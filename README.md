# Dialogue-Datasets

Switchboard Corpus folder contains raw unsupervised text dialogue files and also contains annotated data.
Personachat zip flie contains csv files for unsupervised data and json files for annotated data with dialogue utterances.
MultiWOZ zip foler contains json files for annotated and dialogue utterances.








# Legal Compliance Mining Exercise – EIB Project
This repository contains the solution for the **Exercise on Legal Compliance Mining**  
prepared by Haider Khalid for the **SVV Research Group, SnT, University of Luxembourg**.

**Note:** Per exercise rules, this repository and all associated scripts, outputs, and documentation are **not intended for public release**.

Submission date: **23 January 2026 (AoE)**
---
## Overview

The goal of this exercise is to analyse the evolution of legal requirements in the
**UCITS Directive (Directive 2009/65/EC)** by:
1. Retrieving and normalizing legal texts (Task A),
2. Extracting legal obligations (Task B),
3. Detecting and characterizing changes across versions (Task C).

---

## Expected Runtime and Hardware Assumptions

The full pipeline runs within a few minutes on a standard laptop.

**Approximate runtime by task:**
- **Task A:** Dominated by network downloads from EUR-Lex (typically a few minutes (2-3 minutes), depending on connection).
- **Task B:** Local computation only; completes in a few seconds.
- **Task C:** Local computation only; completes in a few seconds.


### Hardware Assumptions

- Standard laptop or desktop
- ≥ 4 GB RAM
- No GPU required
- Internet connection required only for TASK A - Data Retrieval

## Task A – Objective

The goal of Task A is to:

1. **Programmatically retrieve**:
   - the base version of the UCITS Directive, and
   - two specified consolidated revisions (dated **04/01** and **21/07**) from EUR-Lex.
2. **Convert each version into a structured representation** by:
   - defining a unit of analysis (paragraph-level),
   - splitting each document accordingly,
   - assigning **stable and unique unit identifiers**,
   - preserving the **raw text** of each unit,
   - optionally preserving **article/section identifiers**.

The outputs of Task A are designed to be **reproducible, traceable, and robust**, and to **support Task B (obligation extraction)** and **Task C (change detection)** without structural issues.

---

## Repository Structure (Task A)
```
EIB_Exercise_SVV/
│
├── code/
│ ├── 01_retrieve.py # Download legal texts from EUR-Lex (HTML)
│ ├── 02_extract_text.py # Extract and clean legal text (TXT)
│ ├── 03_make_units.py # Paragraph units + stable hash-based IDs (JSONL)
│ └── 04_sanity_check.py # Validate outputs (uniqueness, preview)
│
├── data/
│ └── texts/
│ ├── raw_html/ # Downloaded HTML files + metadata
│ ├── raw_txt/ # Cleaned legal text (TXT)
│ └── units/ # Structured paragraph units (JSONL)
│
├── sources.txt # EUR-Lex URLs (base + two revisions)
├── requirements.txt # Reproducible Python environment
└── README.md

```
---

## Input Sources

The file `sources.txt` contains the three authoritative EUR-Lex URLs used in Task A:

- Base act (original legal act)
- Consolidated revision dated **04/01**
- Consolidated revision dated **21/07**

These URLs are used as the **single source of truth** for retrieval to ensure reproducibility.

---

## Unit of Analysis

- **Unit type:** Paragraph  
- Each paragraph (including headings such as “Article X”, “CHAPTER”, “SECTION”) is treated as an individual unit.
- This choice provides stability across revisions and supports later obligation extraction and change analysis.

---

## Stable Unit Identifiers (Design Rationale)

Each unit is assigned a **stable, unique identifier** of the form:

- {version}|{article_key}|{text_hash}|{occurrence}

Example:
- v1_base|a05|9f31c2ab42|01

Where:
- `version` identifies the document version (base / revision),
- `article_key` encodes the article context (e.g., Article 5 → `a05`),
- `text_hash` is a SHA-1 hash (prefix) of a canonicalized version of the paragraph text,
- `occurrence` disambiguates repeated identical paragraphs.

**Why this matters:**
- Guarantees uniqueness (no duplicate unit IDs),
- Preserves stability across versions when text is unchanged,
- Supports precise traceability for Task B,
- Enables robust diffing for Task C.

---

## Environment Setup (Reproducibility)

### Requirements
- Python **3.9+**
- OS: Windows, macOS, or Linux

### Create a virtual environment

From the repository root:

```bash
python -m venv .venv

```

### Activate a virtual environment
 - Windows (PowerShell)
```bash
 - .venv\Scripts\Activate.ps1
```
- macOS / Linux
```bash
- source .venv/bin/activate
```
### Install dependencies
```bash
- pip install -r requirements.txt
```
### How to Reproduce Task A (End-to-End)

Run the following commands from the repository root in order:

1. Retrieve legal texts from EUR-Lex

Downloads the three document versions as HTML and stores metadata.
```bash
- python code/01_retrieve.py
```
Outputs: 
data/texts/raw_html/
  ├── v1_base.html
  ├── v2_20110104.html
  ├── v3_20110721.html
  └── _meta.json

2. Extract and normalize legal text

Extracts the main legal text from HTML and saves it as clean TXT files.
```bash
- python code/02_extract_text.py
```
Outputs:
data/texts/raw_txt/
  ├── v1_base.txt
  ├── v2_20110104.txt
  └── v3_20110721.txt

3. Create structured paragraph units

Splits each document into paragraph units and assigns stable identifiers.
```bash
- python code/03_make_units.py
```
Outputs:
data/texts/units/
  ├── v1_base.jsonl
  ├── v2_20110104.jsonl
  ├── v3_20110721.jsonl
  └── _meta_units.json

Each JSONL file contains one paragraph unit per line with:

version and CELEX reference,
article context,
unique stable unit ID,
raw text and canonicalized text.

4. Sanity checks (recommended)

Verifies that:
there are no duplicate unit IDs,
units are generated correctly,
sample previews are printed for inspection.
```bash
- python code/04_sanity_check.py
```
Execution stops with an error if any duplicate identifiers are detected.



## Task B — Extract Obligations

**Prerequisites:**  
Requires Task A outputs in `data/texts/units/`.

### Objective
Task B focuses on extracting **legal obligations** from each version of the UCITS Directive using the structured text produced in Task A.  
An obligation is defined as a **normative requirement that imposes a duty on an actor**, typically expressed using a deontic modality (e.g., *shall*, *must*, *may not*).

The goal is to:
1. Identify obligation spans in the legal text,
2. Extract structured obligation fields,
3. Ensure full traceability to the source text,
4. Manually evaluate a random sample of extracted obligations and report precision.

---

### Obligation Definition (Applied)
An extracted obligation must include at least:
- **Actor / Subject** (who is obliged),
- **Deontic modality** (e.g., *shall*, *must*, *may not*),
- **Action** (what must be done or avoided).

Optional elements (captured where present):
- Conditions or exceptions (e.g., *unless*, *where*, *in the case of*),
- Object or scope of the action.

Statements that are **definitions**, **scope/applicability rules**, or **interpretative clauses** are not considered obligations, even if they contain deontic-looking language.

---

### Extraction Approach
A **rule-based linguistic approach** is used, designed for transparency and reproducibility.

**Key characteristics:**
- Paragraph-level units from Task A are processed.
- Obligation candidates are detected using explicit **deontic modality triggers**, including:
  - *shall*, *shall not*, *must*, *must not*, *is required to*, *may not*.
- Paragraphs are heuristically split into sentence-level spans to isolate individual obligation statements.
- Each detected obligation span is extracted as the **full sentence containing the modality**.

**Field extraction heuristics:**
- **Actor:** text preceding the modality,
- **Modality:** matched deontic trigger,
- **Action:** text following the modality,
- **Condition (optional):** extracted using markers such as *unless*, *where*, *in the case of*.

This approach prioritizes **precision and traceability** over recall, which is appropriate for regulatory compliance analysis.

---

### Traceability
Each extracted obligation is fully traceable to the legal source, satisfying the traceability requirement:

For every obligation, the output includes:
- `version` (base / revision),
- `celex` identifier,
- `source_url`,
- `article` and `article_key`,
- `unit_id` (stable identifier from Task A),
- `span_text` (exact obligation text),
- `span_start` and `span_end` (character offsets within the paragraph).

This allows any obligation to be traced back to:
- the **exact document version**, and
- the **exact text span** from which it was extracted.

---

### Output Format
Obligations are stored in a **machine-readable JSONL format**, one file per version:

data/obligations/
├── v1_base_obligations.jsonl
├── v2_20110104_obligations.jsonl
├── v3_20110721_obligations.jsonl
└── _meta_obligations.json


Each JSON object represents **one extracted obligation**.

---

### Manual Evaluation (Precision)
To assess extraction quality, a manual precision evaluation was conducted on a random
sample of ten extracted obligation candidates…

`precision = (number of true obligations) / 10`

**Observed precision on the 10-sample:** `0.70 (7/10)`.


**Procedure:**
1. Ten obligations are randomly sampled **across all versions** (fixed random seed for reproducibility).
2. Each sampled item is manually assessed against the obligation definition.
3. Each item is labeled as:
   - `true` (valid obligation), or
   - `false` (non-obligation, e.g., definition or applicability rule).

**Evaluation artifacts:**

data/evaluation/
├── manual_sample_10.jsonl
└── precision_report.json


### The evaluation intentionally applies a **strict interpretation** of obligations, which leads to some false positives being correctly rejected during manual review.

---

### Reproducibility — How to Run Task B

**From the repository root:**

```bash
python code/05_extract_obligations.py
python code/06_sample_for_manual_eval.py
```
# After manually labelling the sample file:
### Guide for manual labelling
- In the Explorer (left sidebar), open:

data/
evaluation/
manual_sample_10.jsonl

## You will see 10 lines, and each line is one JSON object.
- Each JSON line already contains these two fields for you to fill:

- "manual_is_obligation": null
- "manual_notes": ""
- You will change them to:
- "manual_is_obligation": true OR "manual_is_obligation": false
- "manual_notes": "your short reason"

NOTE:  Use true/false (lowercase, no quotes) because it is JSON.
       Don’t write "True" or "False" (with quotes) → that becomes a string, not boolean
       Don’t write TRUE/FALSE (uppercase) → invalid JSON.
       Validate you didn’t break the JSONL format

### After successfully lablelled run the below prompt to calculate precision
```bash
python code/07_compute_precision.py

```
  - When the script succeeds, it prints precision and writes:
    - data/evaluation/precision_report.json

## Task C — Detect and Characterize Changes Across Versions

**Prerequisites:**  
Requires Task A outputs in `data/texts/units/`.  
Optional obligation mapping requires Task B outputs in `data/obligations/`.

### Objective
Task C focuses on identifying and characterizing **changes across multiple versions** of the UCITS Directive.  
Changes are detected at the **textual level**, as required, and categorized into **addition**, **removal**, or **replacement**.  
An optional mapping between textual changes and obligation changes is also implemented.

The comparisons performed are:
- **Base version → Revision 04/01**
- **Revision 04/01 → Revision 21/07**

---

### Definition of Change (Applied)
In accordance with the exercise specification, changes are defined as:
- **Addition**: new text inserted,
- **Removal**: existing text deleted,
- **Replacement**: text substituted (a deletion and an insertion aligned at the same location).

---

### Change Detection Approach
A **paragraph-level diffing approach** is used, building directly on the normalized paragraph units produced in Task A.
Changes are recorded at the paragraph-unit level (units defined in Task A).

**Key steps:**
1. Paragraph units from consecutive versions are aligned using a **version-agnostic matching key** derived from:
   - article identifier,
   - canonicalized text hash,
   - occurrence index.
2. The ordered sequences of units are compared using Python’s `SequenceMatcher`.
3. Diff operations are interpreted as:
   - `insert` → **addition**,
   - `delete` → **removal**,
   - `replace` → **replacement**.
4. Replacement blocks are further processed to pair old and new units where possible; unmatched units are classified as additions or removals.

This approach ensures deterministic, reproducible, and traceable change identification.

---

### Optional Mapping to Obligation Changes
As an optional (value-added) step, detected textual changes are mapped to **obligation-level changes** using the outputs of Task B.

For each changed paragraph unit:
- obligations associated with the old and/or new unit are retrieved,
- obligations are compared using a lightweight semantic key derived from actor, modality, action, and condition,
- obligation changes are summarized as:
  - introduced,
  - removed,
  - unchanged.

This mapping allows identification of which **legal obligations were added, removed, or modified** due to a textual change.

---

### Output Files
All outputs are stored in machine-readable JSON formats:

data/changes/
├── base_to_20110104_changes.jsonl
├── 20110104_to_20110721_changes.jsonl
├── base_to_20110104_summary.json
└── 20110104_to_20110721_summary.json


Each change record includes:
- change type (addition/removal/replacement),
- source and target versions,
- article context,
- stable unit identifiers,
- raw and normalized text,
- optional obligation change summaries.

---

### Reproducibility — How to Run Task C
From the repository root, execute:

```bash
python code/08_detect_changes.py
python code/09_summarize_changes.py
```

##  Checklist Against Task C Requirements (PDF)

### Task C.1 — Compute Textual Diffs

The following version comparisons are performed:

- **Base → Revision 04/01**
- **Revision 04/01 → Revision 21/07**

 **Met by `code/08_detect_changes.py`:**

Produces:
- `data/changes/base_to_20110104_changes.jsonl`
- `data/changes/20110104_to_20110721_changes.jsonl`

These outputs correspond exactly to the required version comparisons.


---

### Task C.2 — Categorize Each Change

Required change categories:
- **addition**
- **removal**
- **replacement**

 **Met:**

Change categorization is implemented using `SequenceMatcher` opcodes:
- `insert` → **addition**
- `delete` → **removal**
- `replace` → **replacement**

Any leftover units within a replacement block are explicitly emitted as additions or removals, ensuring correctness and transparency.

This aligns precisely with the PDF definitions:
- **Addition** = new text inserted  
- **Removal** = text deleted  
- **Replacement** = delete + insert aligned at the same location  



---

### Task C.3 — Optional Mapping Between Textual and Obligation Changes

 **Implemented (optional / extra):**

If Task B outputs are available, each change record includes an `obligation_change` summary with:
- obligation keys added / removed / unchanged,
- short obligation text snippets for readability.

If Task B outputs are not present, the script still executes correctly and produces textual change outputs only.

This satisfies the optional component of Task C.



---

### Deliverable C — Script + Output Files

 **Met:**

**Scripts:**
- `08_detect_changes.py`
- `09_summarize_changes.py`

**Outputs:**
- JSONL change files for both version comparisons
- Summary JSON files containing change counts per comparison

This fulfills the requirement for a **change identification script plus output files**.



---

##  Rules & Constraints Compatibility

### Reproducibility 
- Deterministic outputs given the same Task A inputs
- No external services or APIs required

### Traceability 
Each change record includes:
- document versions,
- article information,
- stable unit identifiers,
- full text fields (`text_raw`, `text_norm`).

This ensures that every change can be traced back to the exact legal text and corresponding Task A units, fully satisfying the traceability requirement across the exercise.



