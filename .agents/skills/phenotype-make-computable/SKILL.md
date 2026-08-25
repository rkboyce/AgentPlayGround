---
name: phenotype-make-computable
description: Create a review-gated OHDSI cohort definition from a well-specified narrative cohort statement by calling StudyAgent's phenotype_make_computable ACP endpoint. Use when a user wants a validated function-form Capr R definition and Circe JSON, or needs clarification or concept-set review before creating one.
---

# Phenotype make computable

Call `POST ${STUDY_AGENT_ACP_URL:-http://127.0.0.1:8765}/flows/phenotype_make_computable`. Treat the endpoint as stateless: retain the narrative, confirmed scope, and reviewed concept sets locally and resubmit them in each request.

Never send PHI or row-level data. Never invent scope decisions, concept IDs, domain policies, or Capr/Circe code.

## Non-negotiable review gates

- Do not infer omitted scope. If the user did not explicitly provide every scope decision, submit only a clarification request with `confirmed_scope: false`.
- Treat `needs_clarification` and `needs_concept_review` as terminal responses for the current turn. Show the result, ask the user for the missing decision or review, and stop. Do not issue another ACP request in that turn.
- Do not use `concept_review_mode: "provided_only"` unless a later user message explicitly approves or supplies every concept-set object, including each item's descendant, mapped, and exclusion policies.
- Do not interpret “continue”, silence, a candidate's rank, or a parent-concept relationship as approval. Never select a candidate or choose `include_descendants` on the user's behalf.
- Do not emit, save, or display Capr/Circe artifacts before explicit concept-set approval.

## Workflow

1. Send the narrative-only clarification request below whenever scope is incomplete. Present the returned checklist verbatim enough for the user to decide.
2. After the user explicitly confirms every scope decision, call with `confirmed_scope: true` and `concept_review_mode: "required"`. Present the candidates with ID, name, domain, vocabulary, and provenance. End the turn.
3. On a later user message that explicitly provides or approves the reviewed concept-set object, call with `concept_review_mode: "provided_only"`.
4. If the user explicitly asks for a provisional proposal, use `concept_review_mode: "propose"`; label it unreviewed and end the turn. Never turn that proposal into an artifact.
5. For `ok`, report the returned Capr source, Circe JSON, validation evidence, provenance, and assumptions. Describe this only as technical validation: R sourced the generated function, Capr wrote Circe JSON, and CirceR generated SQL. Do not claim clinical validity or database-level cohort validation.

## Request shapes

Use `Content-Type: application/json`.

Start an incomplete narrative with this request; do not add guessed scope fields:

```json
{
  "narrative_statement": "<user narrative>",
  "confirmed_scope": false,
  "concept_review_mode": "required",
  "concept_sets": []
}
```

After explicit scope confirmation, include the complete user-supplied `scope` and still leave `concept_sets` empty:

```json
{
  "narrative_statement": "<same narrative>",
  "confirmed_scope": true,
  "scope": {
    "index_event": "<user-confirmed event>",
    "criterion_domains": {"<event>": "<user-confirmed OMOP domain>"},
    "entry_limit": "<First or All>",
    "prior_observation": "<integer days>",
    "index_day_boundary": "<included or excluded>",
    "windows": "<user-confirmed temporal semantics>",
    "exit_strategy": "<observation, end_of_observation, or fixed object>",
    "visit_overlap": false
  },
  "concept_review_mode": "required",
  "concept_sets": []
}
```

Only after an explicit later approval, submit a **wrapped concept set**, not an item directly in `concept_sets`:

```json
{
  "narrative_statement": "<same narrative>",
  "confirmed_scope": true,
  "scope": {"<the unchanged confirmed scope>": "..."},
  "concept_review_mode": "provided_only",
  "concept_sets": [
    {
      "name": "<user-approved name>",
      "items": [
        {
          "concept_id": 123,
          "domain": "Condition",
          "include_descendants": false,
          "include_mapped": false,
          "is_excluded": false
        }
      ]
    }
  ]
}
```

Use the policies exactly as approved. Preserve exclusions as exclusions. A mixed-domain reviewed set also needs an explicit `multi_domain_entry_policy`; ask whether each domain defines entry or supports evidence and stop for the answer.

## Response handling

- `needs_clarification`: show the unresolved design choice and stop.
- `needs_concept_review`: show candidates or a provisional plan and stop.
- `ok`: present the returned artifacts and technical-validation evidence.
- `unavailable` or failed validation: surface structured diagnostics; do not fabricate a partial artifact.
