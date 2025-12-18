# TumorBoard Variant Narrator

A research tool for generating explainable narratives for somatic variants using multi-agent LLMs.

**TL;DR:**
Molecular tumor boards spend significant time "telling the story" of each variant—synthesizing evidence from multiple databases, weighing conflicting signals, and contextualizing findings for patient care. TumorBoard Variant Narrator automates this storytelling layer by:

1. **Aggregating evidence** from CIViC, COSMIC, OncoKB, ClinVar, gnomAD, SpliceAI, AlphaMissense, and FDA drug labels
2. **Generating rich variant narratives** that explain what is known about a variant, highlight the most clinically relevant evidence, and provide structured take-home points for tumor board review
3. **Detecting and explaining conflicts** when data sources disagree (e.g., high somatic recurrence vs high population frequency)
4. **Using multi-agent reasoning** (Advocate, Skeptic, Arbiter) to debate interpretations and produce balanced, transparent conclusions
5. **Grounding outputs with RAG** via Honeybee embeddings—retrieving similar historical cases, guideline excerpts, and tool documentation to anchor narratives in established patterns
6. **Real-time literature integration** from PubMed and bioRxiv via RSS feeds and API queries, filtered for relevance to the gene/variant of interest

**Note:** This is a research prototype, not a clinical tool.

***

## Why this exists

Somatic variant interpretation is rarely a clean yes/no decision. Real-world cases often involve:

- High recurrence in COSMIC and strong CIViC oncogenic evidence vs high population frequency in gnomAD.  
- Strong splice-impact predictions from SpliceAI vs seemingly synonymous or low-impact annotation from SnpEff/VEP.  
- Divergent computational scores (e.g., AlphaMissense vs other in silico predictors) vs curated clinical assertions (ClinVar, CIViC, OncoKB).  

ACMG/AMP and AMP/ASCO/CAP guidelines acknowledge that **conflicting evidence is common**, especially in cancer, but they do not fully specify how to weigh heterogeneous sources for every edge case.  In practice, expert molecular tumor boards spend significant effort **telling the story of a variant**: what is known, what conflicts, and how to contextualize that for patient care.

TumorBoard Variant Narrator is designed to automate that *storytelling layer* for research and decision-support experimentation.

***

## What the app does

### 1. Evidence aggregation and normalization

The engine collects and normalizes variant-level evidence from multiple sources:

- Somatic clinical/oncology databases: CIViC, COSMIC, OncoKB, CGI biomarkers, FDA drug labels.  
- Population frequency: gnomAD and related population datasets.  
- Functional and consequence annotation: SnpEff (and/or VEP/ANNOVAR-style annotations).  
- Computational predictors: SpliceAI, AlphaMissense, and other in silico tools as available.  

Evidence is standardized onto consistent models (gene, transcript, HGVS, consequence categories, effect types, AF, scores, etc.) to reduce discrepancies from nomenclature and tool syntax.

### 2. Mapping to guideline-like evidence codes

The app then maps raw evidence onto **guideline-style evidence categories** inspired by ACMG/AMP and AMP/ASCO/CAP (adapted to somatic oncology):

- Population-based evidence (e.g., BA1/BS1 analogs) from high AF in gnomAD.
- Computational evidence (PP3/BP4 analogs) from SpliceAI, AlphaMissense, and other predictors.
- Functional/experimental evidence (PS3/BS3-like) from curated functional data where available.
- Somatic clinical actionability evidence (AMP/ASCO/CAP Tier I/II drivers, guideline-backed biomarkers, FDA labels).

This structured representation provides a **deterministic backbone** for interpretation and sets up the narrative and multi-agent reasoning.

### 3. Conflict detection across datasets

Given the mapped evidence, the engine detects **predefined conflict archetypes**, such as:

- COSMIC/CIViC/OncoKB suggest a strong oncogenic driver vs gnomAD AF implies likely benign or common polymorphism.  
- SpliceAI predicts strong splice disruption vs consequence annotation suggests synonymous/low-impact.  
- Multiple annotation tools disagree on transcript or variant consequence.  
- Computational tools strongly disagree with each other or with curated clinical assertions.

These conflicts are not the *end product*, but they shape the narrative and trigger multi-agent reasoning.

***

## The Variant Narrative

The **primary output** of the app is a **variant narrative**: a structured, human-readable explanation suitable for tumor boards, reports, and curation discussions.

Each narrative typically includes:

- A concise description of the variant  
  - Gene, protein change, genomic coordinates, consequence, and key annotations.

- A synthesized evidence summary  
  - Top-line points from CIViC, COSMIC, ClinVar, gnomAD, SnpEff, SpliceAI, AlphaMissense, FDA labels, etc.  
  - Highlighted evidence that is most relevant to actionability under AMP/ASCO/CAP.

- Agreement vs conflict across data sources  
  - Explicit enumeration of where sources align (e.g., multiple databases flag as oncogenic) and where they diverge (e.g., population vs somatic evidence, computational vs clinical).

- Contextual interpretation  
  - How the evidence fits into guideline-style reasoning (ACMG/AMP, AMP/ASCO/CAP), with clear statements like “computational evidence supports but does not override high-quality clinical data” or “population data argues against a high-tier classification despite recurrent somatic reports.”

- A structured “take-home” section  
  - A short, bullet-point summary of the most important considerations for a human reviewer or tumor board.  

Narratives are generated by a constrained LLM that is **grounded with Honeybee embeddings**, retrieving:

- Similar historical conflict cases and their outcomes.  
- Relevant guideline excerpts (ACMG/AMP, AMP/ASCO/CAP, somatic specifications).  
- Tool documentation and model-card explanations (e.g., SpliceAI limitations, AlphaMissense behavior).  

***

## The Multi-Agent Solution

Instead of asking a single model to make a monolithic judgment, TumorBoard Variant Narrator uses a **multi-agent LLM architecture** that mimics the dynamics of a molecular tumor board.

### The Advocate Agent

- Focus: **Argues for higher actionability / pathogenicity**.  
- Perspective:
  - Emphasizes clinical and somatic evidence from CIViC, COSMIC, OncoKB, FDA labels, and guidelines that support treating the variant as a driver with therapeutic or prognostic implications.
  - Highlights strong functional data, recurrent hotspot status, and concordant expert assertions that favor a higher AMP/ASCO/CAP tier.

### The Skeptic Agent

- Focus: **Challenges the Advocate, especially using population and computational evidence**.
- Perspective:
  - Brings in gnomAD AF, gene constraint metrics, and any benign or conflicting clinical submissions to argue for a more conservative classification.
  - Leverages SpliceAI, AlphaMissense, and other in silico tools (while aware of their limitations) to question over-interpretation of weak somatic evidence.
  - Points out technical caveats: transcript choice, annotation discrepancies, low coverage, or biases in databases.

### The Arbiter Agent

- Focus: **Weighs arguments and synthesizes the final narrative and tier recommendation**.  
- Responsibilities:
  - Reads the Advocate and Skeptic arguments (plus retrieved context via embeddings).
  - Applies an explicit ruleset aligned with AMP/ASCO/CAP and ACMG-style criteria to weigh different evidence types.
  - Produces the **final variant narrative**, including:  
    - A balanced explanation of why each side is compelling or limited.  
    - A structured recommendation for how a human curator or deterministic tiering engine should treat the variant (e.g., “Tier II with caution,” “Tier III – VUS due to unresolved conflict,” etc.).  

The Arbiter does **not** operate as an unconstrained oracle: its outputs are restricted to a JSON schema and narrative template, and final clinical decisions remain with deterministic logic and/or human review.

***

## Embeddings and RAG: Honeybee as the memory layer

To keep reasoning grounded and reproducible, the system uses **Honeybee embeddings** as a memory and retrieval layer:

- Indexing  
  - Historical variant cases, including conflicts and how they were resolved.  
  - Guideline excerpts and expert commentary.  
  - Tool documentation and caveats for COSMIC, CIViC, gnomAD, SpliceAI, AlphaMissense, SnpEff, etc.  

- Retrieval  
  - For a new variant, the agents retrieve similar conflicts and relevant guidance, anchoring their arguments in known patterns and recommendations.  

- Benefits  
  - More consistent narratives for similar variants.  
  - Enhanced transparency (you can track which prior cases or guideline snippets influenced a given narrative).  

***

## Deterministic tiering and auditability

Although the variant narrative is the centerpiece, TumorBoard Variant Narrator can integrate with or provide a **deterministic tiering layer**:

- Rule-based tiering  
  - Uses explicit, configurable rules to map evidence codes and Arbiter recommendations into AMP/ASCO/CAP-style tiers (Tier I–IV).  

- Logging and audit
  - Logs all evidence, agent arguments, retrieved context, and final narrative in a structured format.
  - Supports post-hoc analysis of:
    - Where agents disagreed.
    - How often the Arbiter sided with Advocate vs Skeptic.
    - The impact of multi-agent reasoning on reclassification and VUS rates.

This makes the system suitable for **research on LLM-assisted variant interpretation**, rather than opaque automation.

***

## Intended use

- **Research and methods development**
  - Studying multi-agent LLM reasoning for variant interpretation and conflict resolution in somatic oncology.

- **Tumor board support (exploratory)**
  - Providing variant narratives and structured pros/cons to inform, not replace, multidisciplinary deliberations.

- **Education and training**
  - Teaching trainees how different evidence types interact and how guideline-style logic applies when data sources disagree.

**Important:** This is a **research prototype**, not a regulated medical device, and is **not intended for unsupervised clinical decision-making**.

***

## High-level usage

1. **Setup**
   - Create a Python 3.11+ environment and install the package in development mode.
   - Configure API keys for LLMs, Honeybee embeddings, and external annotation services.

2. **Run a variant through the Narrator**  
   - Provide a variant (or batch) via CLI or API (e.g., gene, HGVS, tumor type).  
   - The engine aggregates evidence, maps codes, runs the multi-agent dialogue, and produces:  
     - A structured variant narrative.  
     - A machine-readable conflict analysis and recommendation object.  

3. **Evaluate impact**
   - Use the built-in validation harness and your gold-standard labels to compare:
     - Baseline rule-only tiering vs multi-agent–informed tiering.
     - Narrative quality and usefulness in tumor board-style workflows.

Further installation and configuration details can be documented in `/docs` and CLI help as the project evolves.