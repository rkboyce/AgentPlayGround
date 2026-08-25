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
- Do not use `concept_review_mode: "provided_only"` unless a later user message explicitly supplies the exact reviewed `concept_sets` object or explicitly approves that exact object, including every item's descendant, mapped, and exclusion policies.
- Tool-execution approval authorizes only the local command. It is never clinical, scope, or concept-set approval.
- Do not interpret “continue”, silence, a candidate's rank, a parent-concept relationship, or an LLM proposal as approval. Never select a candidate or choose `include_descendants` on the user's behalf.
- After any ACP request returns a JSON body, do not retry it automatically. Treat the JSON as the response, render it faithfully, and stop when its status requires review. If it cannot be parsed, show the raw body and ask the user whether to retry.
- Do not emit, save, or display Capr/Circe artifacts before explicit concept-set approval.

## Workflow

1. Send the narrative-only clarification request below whenever scope is incomplete. Present the returned checklist verbatim enough for the user to decide.
2. Keep `index_event` as the user's plain clinical term (for example, `"Cirrhosis of liver"`). Put event selection only in `entry_limit` (`"First"` or `"All"`); do not rewrite the event as “first qualifying … condition occurrence”. Use that same plain term as the matching key in `criterion_domains`.
3. After the user explicitly confirms every scope decision, call with `confirmed_scope: true` and `concept_review_mode: "required"`. Count the returned candidates and display **every returned candidate** in one review table with concept ID, name, domain, vocabulary, concept class, and standard/non-standard status. State the count and retrieval provenance. Do not choose or rank a concept. End the turn.
4. If a client display limit makes a complete table impossible, state exactly `Returned N candidates; showing M of N`, identify which IDs are omitted, and offer to show the remainder. Never describe a partial table as “all candidates.” Do not proceed to selection.
5. If the user explicitly asks for a provisional LLM proposal, call once with `concept_review_mode: "propose"`. For a successful response, display the complete candidate table, the complete `proposed_plan`, a policy table for every proposed concept-set item, assumptions, warnings, and proposal diagnostics. Label every proposal unreviewed and end the turn. Never turn that proposal into an artifact.
6. Only on a later user message that supplies or explicitly approves the exact proposed `concept_sets` object, call with `concept_review_mode: "provided_only"`. An approval to run curl or another tool is insufficient.
7. For `ok`, report the returned Capr source, Circe JSON, validation evidence, provenance, and assumptions. Describe this only as technical validation: R sourced the generated function, Capr wrote Circe JSON, and CirceR generated SQL. Do not claim clinical validity or database-level cohort validation.

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

For a one-call LLM proposal, use the complete confirmed scope, `concept_review_mode: "propose"`, and an empty `concept_sets` list. Its response is review material only; do not resubmit it automatically.

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
      "domain": "Condition",
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
