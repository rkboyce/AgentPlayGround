---
name: phenotype-make-computable
description: Create a review-gated OHDSI cohort definition from a well-specified narrative cohort statement by calling StudyAgent's phenotype_make_computable ACP endpoint. Use when a user wants a validated function-form Capr R definition and Circe JSON, or needs clarification or concept-set review before creating one.
---

# Phenotype make computable

Call `POST ${STUDY_AGENT_ACP_URL:-http://127.0.0.1:8765}/flows/phenotype_make_computable`. Retain the narrative, confirmed scope, and reviewed concept sets locally and resubmit them in each emission request. Large review sessions are immutable but short-lived; download their review package for durable continuation.

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
3. After the user explicitly confirms every scope decision, call with `confirmed_scope: true` and `concept_review_mode: "required"`. Count the returned candidates and state retrieval provenance. If the count is 10 or fewer, display every candidate in one review table with concept ID, name, domain, vocabulary, concept class, and standard/non-standard status. Do not choose or rank a concept. End the turn.
4. If the count exceeds 10, do not dump the candidate table into the terminal. A normal `review_delivery: "auto"` response supplies `review_delivery: "session"`, a `review_id`, and `review_urls`. Ask the user for permission to download the immutable `candidates_csv` review artifact and stop. If the user declines, offer a paginated display through `review_urls.candidates` and state exactly `Returned N candidates; showing M of N`, identifying omitted IDs. Never describe a partial table as “all candidates.” Do not proceed to selection.
5. After permission, download both `review_urls.candidates_csv` and `review_urls.manifest` to the user-specified review directory, or `./phenotype-review/` when no path is specified. Use sibling names `<safe-narrative-slug>_concept_review.csv` and `<safe-narrative-slug>_concept_review_manifest.json`. The CSV contains one frozen candidate per row. `proposed_*` fields record only a provisional policy actually suggested by the LLM; blank means no proposed policy. State both written paths, the human-readable `review_expires_at`, and that the package is unreviewed. Then give the user this concise review handoff:
   - Edit only `review_*` columns; mark chosen cells with `x` and leave all other review cells blank.
   - For a row, mark either `review_include_concept` or `review_exclude_concepts`, never both. Descendants/mapped marks require the corresponding include/exclude root mark.
   - Blank review fields mean omit the candidate. `not_assessed_retrieval_context` is retained evidence, not an automatic choice; include it only deliberately.
   - Preserve the manifest beside the CSV. Save the CSV, then tell the agent the CSV path and the intended concept-set name. The agent will validate and display the exact submission object for explicit approval.
   - Before expiry, the agent can read the session URLs. After expiry or ACP restart, the saved CSV plus manifest still supports validation and submission; only an undownloaded session requires a new review request.
   Do not fabricate a CSV from a truncated terminal response.
6. Treat a user-edited CSV as review input, not submission approval. When the user points to it, run `uv run python ../../scripts/phenotype_review_csv_to_concept_sets.py --csv <path> --manifest <adjacent-manifest-path> --concept-set-name <user-approved-name>` from `sandbox/AgentPlayGround` (adjust only if the active workspace differs). The reader accepts `x` case-insensitively and blank as false, rejects contradictory/incomplete markings, and emits both policy-bearing `concept_sets` and `approval_preview`. Render every `approval_preview` row in a human-readable table with concept ID, concept name, domain, policy, assessment status, precision eligibility, and relationship evidence before showing the exact emitted `concept_sets` object. Highlight manually included `not_assessed_retrieval_context` rows; those rows are never automatic choices. Request explicit approval before calling `provided_only`, even when the review session has expired.
7. If the user explicitly asks for a provisional LLM proposal, call once with `concept_review_mode: "propose"`. Use `concept_build_mode: "grounded"` only when the user explicitly requests grounded concept building; otherwise omit it (the service defaults to `search_only`). For 10 or fewer candidates, display the complete candidate table, `concept_build` terms and step counts when present, the exact `concept_provenance` (including PHOEBE relationship expansion), every `candidate_assessment` with its precision-eligibility rationale, the complete `proposed_plan`, a policy table for every proposed concept-set item, assumptions, warnings, and proposal diagnostics. For more than 10 candidates, first display status, proposal-validation status/errors, candidate count, assessment scope/count, and provenance counts, then ask permission to download the CSV artifact in step 5 and stop. Only after that separate permission may it fetch `review_urls.proposal` or the CSV; these GETs read the immutable review session and are not proposal retries. Label every proposal unreviewed. Search terms and relationship evidence are retrieval hints, not approved concepts or clinical decisions. Never turn that proposal into an artifact.
8. Only on a later user message that supplies or explicitly approves the exact proposed `concept_sets` object, call with `concept_review_mode: "provided_only"`. An approval to run curl or another tool is insufficient.
9. For `ok`, report the returned Capr source, Circe JSON, validation evidence, provenance, and assumptions. Describe this only as technical validation: R sourced the generated function, Capr wrote Circe JSON, and CirceR generated SQL. Do not claim clinical validity or database-level cohort validation.

## Request shapes

Use `Content-Type: application/json`. For an LLM proposal request, use `curl --max-time 120`; a successful grounded proposal can take more than 30 seconds. This changes only the client wait allowance, never the one-request/no-retry rule.

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

For a one-call LLM proposal, use the complete confirmed scope, `concept_review_mode: "propose"`, `"review_delivery": "auto"`, and an empty `concept_sets` list. Add `"concept_build_mode": "grounded"` only when the user explicitly requests the grounded vocabulary pipeline. In that mode, show `concept_build` and `concept_provenance` exactly as returned; do not treat generated terms, candidates, or the LLM plan as approval. Its response is review material only; do not resubmit it automatically. A session response is intentionally compact; preserve its `review_id` and URLs for later user-authorized read/download actions.

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
- `unavailable` or failed validation: display `proposal_validation_status` and every `proposal_validation_errors` entry before any summary; then surface diagnostics. Do not search logs or make another ACP request after a JSON response, and do not fabricate a partial artifact.
