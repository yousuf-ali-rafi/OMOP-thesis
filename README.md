# Input-to-Output Mapping: \(X \rightarrow y\)

In the context of the thesis, **\(X, y\)** represents a formal **input-to-output mapping function or predictive relationship**. Across the end-to-end data engineering and observational research lifecycle implemented in the project, this relationship operates across **three primary layers**.

---

## 1. Data Standardization ETL Mapping Layer: \(X \rightarrow y\)

During the **Extract-Transform-Load (ETL)** pipeline, raw, heterogeneous clinical data collected from disparate source databases is transformed into standard concepts within the **OMOP Common Data Model (v5.4)** on Microsoft SQL Server.

- **Input (\(X\))**: The raw, unstandardized source text, local clinical code, or native string captured in source systems.
- **Transformation Engine (\(f(X)\))**: Rule-based T-SQL transformation scripts informed by WhiteRabbit profiling, Rabbit In A Hat mapping designs, and OHDSI ATHENA vocabulary crosswalks (`code_to_standard` built using `"Maps to"` relationships).
- **Output (\(y\))**: The resolved standard OHDSI Concept Identifier (`*_concept_id`) loaded into the target OMOP CDM clinical tables.

### Concrete Examples from the Thesis Implementation

#### 1. Demographics Mapping

- **Input (\(X\))**: Raw gender text string (e.g., `"Male"` or `"Female"`) in the Kaggle EHR staging table (`stg_healthcare`).
- **Output (\(y\))**: Standard concept ID `8507` for Male or `8532` for Female populated into `PERSON.gender_concept_id`.

#### 2. Condition / Diagnosis Mapping

- **Input (\(X\))**: Unstandardized diagnostic strings or local ICD codes across the datasets.
- **Output (\(y\))**: Standard **SNOMED CT** concept ID assigned to `CONDITION_OCCURRENCE.condition_concept_id`.

#### 3. Medication Mapping

- **Input (\(X\))**: Free-text prescription names or local brand terms.
- **Output (\(y\))**: Standard **RxNorm** concept ID assigned to `DRUG_EXPOSURE.drug_concept_id`.

#### 4. Procedure & Measurement Mapping

- **Input (\(X\))**: Clinical event text from the Lithuanian kidney-biopsy registry (`omop_3`) or laboratory observation strings from Synthea (`omop_2`).
- **Output (\(y\))**: Standard procedure concept IDs in `PROCEDURE_OCCURRENCE` or **LOINC** measurement concepts in `MEASUREMENT.measurement_concept_id`.

---

## 2. Data Quality & Fallback Exception Handling Layer: \(X \rightarrow y\)

Real-world healthcare records—such as the Lithuanian government nephrology registry (`omop_3`)—contain corrupted dates, missing attributes, or local terms not recognized by standard vocabularies. The framework applies a deterministic fallback rule to preserve data integrity and complete auditability.

- **Input (\(X\))**: Corrupted, ambiguous, or unmapped source text (e.g., mangled age strings, invalid sex entries, or local terms without an ATHENA mapping).
- **Transformation Rule (\(f(X)\))**: Hardened T-SQL error handling with multi-format date parsing, UTF-8 character encoding, and conditional `CASE` statements.
- **Output (\(y\))**:
  - The OMOP standard placeholder **\(y = 0\)** (*"No matching concept"*) written to `*_concept_id`.
  - The verbatim raw text string **\(X\)** preserved in `*_source_value` (e.g., `gender_source_value`, `person_source_value`).

### Concrete Examples from the Thesis Implementation

- **Unresolved Sex/Gender**: When the source text cannot be resolved to Male/Female, \(X = \text{"Unknown"}\) outputs \(y = 0\) in `gender_concept_id`, while preserving \(X\) in `gender_source_value`.
- **Spreadsheet-Corrupted Data**: In `omop_3`, mangled birth dates or unmapped registry codes output \(y = 0\) in standard concept fields. This prevents record loss during ETL and surfaces these data-quality defects directly for evaluation in the **OHDSI Data Quality Dashboard (DQD)**.

---

## 3. Downstream Patient-Level Prediction & Machine Learning Layer: \(X \rightarrow y\)

Once raw records are converted into standardized OMOP CDM tables across the three databases (`omop_1`, `omop_2`, `omop_3`), they serve as structured inputs for observational health analytics and predictive modeling, such as **OHDSI HADES Patient-Level Prediction**.

### Input Feature Matrix (\(X\))

A multi-dimensional patient feature vector compiled from historical observation windows (`OBSERVATION_PERIOD`) can include:

- **Demographics**: Age derived from `YEAR(GETDATE()) - year_of_birth` and sex (`8507`/`8532`).
- **Encounter Mix**: Counts of visit types (Inpatient `9201`, Outpatient `9202`, Emergency `9203`).
- **Clinical History**: Prior condition occurrences (`CONDITION_OCCURRENCE`), drug exposures (`DRUG_EXPOSURE`), and procedure events (`PROCEDURE_OCCURRENCE`).
- **Biomarkers**: Quantitative laboratory values stored in `MEASUREMENT.value_as_number`.

### Target Outcome Variable (\(y\))

A downstream clinical target event or binary indicator:

\[
y \in \{0,1\}
\]

defined over a specified risk window.

#### Mortality Prediction

\[
y =
\begin{cases}
1, & \text{if a record exists in DEATH within the specified follow-up window} \\
0, & \text{otherwise}
\end{cases}
\]

#### Complication / Readmission Risk

\[
y =
\begin{cases}
1, & \text{if the patient experiences a secondary condition or emergency visit following a procedure} \\
0, & \text{otherwise}
\end{cases}
\]

For example, the outcome may represent a complication or readmission following a kidney biopsy in `omop_3`.

---

## Overall Relationship

The complete thesis workflow can therefore be represented as a sequence of transformations:

```text
Raw Clinical Data
       │
       ▼
┌──────────────────────────────┐
│ ETL Standardization          │
│ X → f(X) → y                 │
│ OMOP CDM Concept Mapping     │
└──────────────────────────────┘
       │
       ▼
Standardized OMOP CDM
       │
       ▼
┌──────────────────────────────┐
│ Data Quality & Exception     │
│ Handling                     │
│ Unmapped X → concept_id = 0  │
│ Preserve X in source_value   │
└──────────────────────────────┘
       │
       ▼
Validated OMOP Data
       │
       ▼
┌──────────────────────────────┐
│ Patient-Level Prediction     │
│                              │
│ Feature Matrix X             │
│          │                   │
│          ▼                   │
│ Predictive Model f(X)        │
│          │                   │
│          ▼                   │
│ Target Outcome y ∈ {0,1}     │
└──────────────────────────────┘
```

Thus, **\(X \rightarrow y\)** is used at multiple stages of the thesis: first as a deterministic ETL mapping from source data to standardized OMOP concepts, then as a controlled fallback mechanism for unresolved data, and finally as a predictive relationship between patient-level features and a defined clinical outcome.

---

## Summary

| Layer | Input \(X\) | Transformation \(f(X)\) | Output \(y\) |
|---|---|---|---|
| **ETL Standardization** | Raw text, local codes, source values | T-SQL rules + vocabulary mappings | Standard OMOP `concept_id` |
| **Data Quality / Fallback** | Missing, corrupted, or unmapped values | Validation + `CASE` logic + error handling | `concept_id = 0` + preserved source value |
| **Patient-Level Prediction** | Patient features from OMOP CDM | Statistical / machine-learning model | Clinical outcome \(y \in \{0,1\}\) |

This establishes a clear conceptual link between **data engineering**, **OMOP standardization**, **data quality management**, and **downstream observational/predictive analytics** within the thesis.
