---
name: ohdsi-cohorts
description: Read this file whenever the user asks to define, extract, or design a clinical cohort (e.g., exposure or outcome), or mentions TCO (Target, Comparator, Outcome) mapping, Circe, ATLAS, or OMOP cohort generation.
---

# Core Directives

- NEVER use the term "cohort" to mean a general population sample. 
- ALWAYS define a cohort strictly as a set of persons who satisfy one or more inclusion criteria for a duration of time.
- When instructed to operationalize a cohort, you must assume the target output relies on the OHDSI Circe JSON standard unless told otherwise.

# Term Definition

We define a cohort as a set of persons who satisfy one or more inclusion criteria for a duration of time. 

**Strict Logical Consequences:**

- One person MAY belong to multiple cohorts.
- One person MAY belong to the same cohort for multiple different time periods.
- One person MAY NOT belong to the same cohort multiple times during the exact same period of time (periods cannot overlap).
- A cohort MAY have zero or more members.

# Cohort Operationalization (Circe Standard)

OHDSI Circe (used in ATLAS) offers the standard way to operationalize a cohort definition. These definitions are expressed as JSON and converted to SQL to instantiate the cohort in a database following the OMOP Common Data Model. 

 Key Elements of a Circe Cohort Definition:
- **Cohort Entry Event:** The initial clinical event (e.g., condition, drug exposure) that qualifies a person for a cohort. This acts as the anchor point.
- **Index Date:** The start date of the cohort entry event. All temporal logic is calculated relative to this date.
- **Inclusion Rules:** Additional criteria that must be met relative to the index date (e.g., requires 365 days of prior continuous observation in the `observation_period` table).
- **Cohort Exit:** The date the person no longer satisfies the cohort criteria, establishing the end of the time-in-cohort duration.
