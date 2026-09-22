# 🔮 PRISM — Technical Design Spec

**Version:** 1.0
**Owner:** Mel Boulos
**Runtime:** Rox agentflow

## 1. Overview

PRISM is an LLM agentflow that turns a detected sales signal into a credible Couchbase AE-authored outreach conversation — or explicitly refuses to produce one. It is not an email generator. It is a decision system whose primary output is whether to open a conversation, secondarily what conversation to open, and only lastly the specific email that opens it.

**Inputs:** a webhook POST from an upstream detection agent (Fresh Catch) containing a company, a signal, and 2–3 candidate contacts.

**Outputs:** either (a) a PAUSED 5-step Rox sequence enrolled to one primary contact with an AI-authored Wave 1, ready for human review and send; or (b) a HOLD/BLOCKED/ERROR decision with a structured diagnostic. PRISM never sends email autonomously.

**Core design constraint:** token expenditure must increase only as confidence increases. Cheap deterministic checks precede LLM reasoning. Deep research runs on the leading candidate only. The email is drafted only after commercial viability is validated. HOLD is a first-class outcome, not a failure state.

## 2. System Context

```
┌──────────────┐   webhook    ┌──────────────┐   paused seq    ┌──────────────┐
│ 🎣 Fresh     │─────────────▶│ 🔮 PRISM     │────────────────▶│ 🎯 Pursue    │
│    Catch     │              │  (this doc)  │                 │  (human)     │
└──────────────┘              └──────────────┘                 └──────────────┘
      │                              │
      │                              │ reads/writes
      ▼                              ▼
   Detects signals         ┌─────────────────────┐
   Discovers candidates    │ Rox custom_store    │
   Assigns pursuit_ids     │ - claim registry    │
                           │ - wave templates    │
                           │ - company research  │
                           │ - webhook routing   │
                           └─────────────────────┘
                                     ▲
                                     │ writes wave 2–5 template ids
                                     │
                           ┌─────────────────────┐
                           │ 🛠️ PRISM Bootstrap  │
                           │ (maintainer-run)    │
                           └─────────────────────┘
```

### External contracts

| System | Role | Interface |
|---|---|---|
| Fresh Catch | Signal detection, candidate discovery, compliance/DNC screening | POSTs `PrismWebhookPayload` (§6.1) to per-AE PRISM webhook URLs |
| PRISM Bootstrap | Creates the four prebuilt email templates (Waves 2–5) and writes their IDs atomically | Writes org `custom_store` key `prism:wave_template_ids` (§6.2) |
| Pursue | Human-driven follow-up motion after enrollment | Reads `pursuit_id` from PRISM's claim record; PRISM does not integrate with Pursue directly |
| Rox platform | Workflow runtime, custom_store, RQL, email templates, campaigns, notifications, Home diagnostics | Rox actions catalog (§9) |

## 3. Design Principles

Four principles govern every architectural decision in this system.

### 3.1 If there is nothing genuinely worth saying, say nothing

HOLD is a first-class outcome. A well-reasoned HOLD is more valuable to the AE than a manufactured sequence. The system is optimized for quality of conversation, not number of touches.

### 3.2 Token expenditure must increase only as confidence increases

The pipeline is a strict funnel of increasing token cost:

```
Payload validation      ─── zero LLM
Deterministic eligibility ── zero LLM
Cheap triage           ─── 1 LLM call, no research
Company research (cached) ─ 0 or 1 research call, TTL 30 days
Deep person research   ─── 1 batched research call on #1 only, max 2 per run
Conversation Plan      ─── 1 LLM call (only if research passed)
Commercial evaluator   ─── 1 LLM call (validates the plan)
Wave 1 draft           ─── 1 LLM call (only if plan passed)
Email evaluator        ─── 1 LLM call + max 1 rewrite (only if draft exists)
```

No stage runs until the previous one has justified it. HOLD terminates the pipeline immediately; no "try a different approach" beyond one narrowly-scoped fall-through rule during person research.

### 3.3 AE-authored, not SE-authored

The executing rep is a Couchbase Account Executive. Wave 1 reads as an AE writing to another human: technically credible, but not diagnosing the customer's architecture and not pitching product. SE involvement is a later-conversation possibility that Wave 1 sets up, never an identity Wave 1 assumes. SE-only enablements are explicitly out of scope.

### 3.4 The successful commercial chain

Every ENROLLED run establishes:

```
person-specific evidence
  → current problem
    → Couchbase-relevant problem
      → legitimate AE reason for contact
        → diagnostic question
          → expected response
            → next conversation
              → SE handoff when appropriate
```

If that chain cannot be established honestly, the run holds.

## 4. Deployment Architecture

### 4.1 Multi-tenant shared workflow

PRISM is one workflow definition shared across an org. Each AE enables their own instance, which:

- Runs as that AE (`{{ metadata.user.* }}` resolves to them)
- Governs all RQL reads by their data access
- Generates a per-AE webhook URL
- Delivers rep notifications to them

The workflow owner (the maintainer) receives shadow diagnostics on their Home regardless of which AE's instance ran — because Rox's `add_html` writes to the workflow owner's Home, not the executing user's.

### 4.2 Routing map

Fresh Catch resolves the invoking AE to a PRISM webhook via the org-scoped `custom_store` key `prism_webhooks_by_rep`:

```json
{
  "ae.email@couchbase.com": "https://webhooks.backend.rox.com/webhooks/w/<slug>",
  ...
}
```

Keys are lowercased AE emails. Values are per-AE PRISM webhook URLs. PRISM does not read this map — it is a webhook receiver; whoever POSTed to its URL has already resolved routing.

**Onboarding:** AE enables PRISM → Rox generates their URL → maintainer adds one map entry → Fresh Catch begins routing signals for that AE.

### 4.3 Identity invariant

At runtime, PRISM enforces:

```
payload.owning_rep_rox_user_id == metadata.user.id
payload.owning_rep_email       == metadata.user.email (case-insensitive)
```

If either mismatches, Fresh Catch has POSTed to the wrong AE's instance. Abort with `BLOCKED:rep_mismatch` and record both identities in the diagnostic. This guards against misconfigured routing map entries.

## 5. The Pipeline

13 sequential steps. Each has a terminal state; the only intra-run backtracking is the deterministic fall-through in Step 6.

| # | Stage | LLM calls | External calls | Terminal outcomes |
|---|---|---|---|---|
| 1 | Payload validation + preflight | 0 | 1 `custom_store_get` | `BLOCKED:rep_mismatch`, `BLOCKED:missing_required_payload_field:*`, `BLOCKED:missing_wave_template_config:*` |
| 2 | Deterministic eligibility | 0 | N × `custom_store_get`, `lookup_account_by_domain`, `find_contact`/`create_contact`, RQL, `email.list_emails` | `HOLD:account_unresolved`, `HOLD:all_candidates_unresolved`, `HOLD:all_candidates_ineligible`, `ALREADY_WORKED` |
| 3 | Cheap candidate triage | 1 | 0 | Produces `ranked_candidates` and `triage_selection` |
| 4 | Company research (cached, 30-day TTL) | 0 or 1 | 1 `custom_store_get`, 0 or 1 `custom_store_set` | Cache HIT or MISS |
| 5 | Deep person research on `ranked_candidates[0]` | 1 batched call (person research + Couchbase RQL) | RQL | Produces Person research object + `couchbase_relationship` |
| 6 | Interpret `research_result` classifier | 0 or 1 (fall-through) | RQL (reused) | PASS → step 7; `HOLD:insufficient_signal`, `HOLD:conflicting_evidence`, `HOLD:research_tool_failure`, `HOLD:insufficient_research_all_candidates_tried` |
| 7 | Compact research summary + Conversation Plan | 1 | 0 | `HOLD:plan_construction_failed:*` |
| 8 | Commercial viability evaluator (11 gates) | 1 | 0 | `HOLD:commercial_viability_failed:<gate>` |
| 9 | Draft Wave 1 | 1 | 0 | Produces Wave 1 draft |
| 10 | Email quality evaluator (20 gates) + max 1 rewrite | 1 or 2 (plus 1 draft on rewrite) | 0 | `HOLD:email_quality_failed:<gate>` |
| 11 | Atomic claim + template + enroll | 0 | `custom_store_get`/`set`, `create_email_template`, `create_campaign`, `add_contact_to_campaign` | `ENROLLED`, `ALREADY_WORKED` (11a race), `ERROR:*` |
| 12 | Notify (AE + maintainer copy) | 0 | 1–2 `send_notification` | Notifications sent or silent per outcome |
| 13 | Shadow diagnostic to maintainer Home | 0 | 1 `add_html` | Always fires |

**Terminal state discipline.** Each stage either promotes to the next or terminates. The only exception is Step 6's controlled fall-through to `ranked_candidates[1]` on person-specific classifier failures. Nothing else "tries a different approach" — no adaptive retry, no re-reasoning, no alternate generation strategy.

**Hard ceilings:**

- Max 2 deep-research operations per run, ever. `ranked_candidates[2]` is never researched.
- Max 1 rewrite on Wave 1. Second failure → HOLD.
- Max 1 retry on any deterministic tool error. Second failure → move to diagnostic.

**Root-cause note carried forward from v1.0 pre-release validation** (workflow runs `dbb1971a`, `e9715769`, fixed and confirmed clean in `7bf0506b`): Rox's templated step types (`manual_email`, `email`, `automated_email`) copy their body from a linked email template — `user_input` is ignored for these step types and a step created without a linked `template_id` silently produces `steps_created: 0`. Step 11 (§9, §11.7 below) enforces `create_email_template` before `create_campaign`, with the returned `template_id` linked on every step, specifically to prevent regression of this bug.

## 6. Data Contracts

### 6.1 Webhook payload (`PrismWebhookPayload`)

```json
{
  "report_id": "string, required — Fresh Catch's run_id (company-level lineage)",
  "test": "bool, optional — if true, skip claim/sequence, diagnostic only",

  "rox_company_id": "uuid, optional — if Fresh Catch resolved",
  "company_name": "string, required",
  "company_domain": "string, required",

  "owning_rep_rox_user_id": "uuid, required (identity invariant)",
  "owning_rep_name": "string, required",
  "owning_rep_email": "string, required (identity invariant, case-insensitive)",

  "signal_summary": "string, required",
  "signal_sources": ["url", "..."],
  "signal_type": "string, optional",
  "fresh_catch_couchbase_angle": "string, optional — prior work, not gospel",
  "fresh_catch_call_opener": "string, optional — prior work, not gospel",

  "candidate_contacts": [
    {
      "pursuit_id": "string, required per candidate — format fc-YYYYMMDD-{initials}-{6-hex}",
      "rox_person_id": "uuid, optional",
      "name": "string",
      "email": "string",
      "linkedin_slug": "string",
      "title": "string",
      "phone": "string",
      "is_fresh_catch_primary": "bool, optional — advisory only"
    }
  ]
}
```

**Validation rules:**

- `report_id`, `owning_rep_rox_user_id`, `signal_summary`, `company_domain`, non-empty `candidate_contacts` are required
- Each candidate needs `pursuit_id` and (`email` OR `linkedin_slug`)
- Any missing required field → `BLOCKED:missing_required_payload_field:<name>`

### 6.2 Custom store keys

| Key | Scope | Writer | Reader | Purpose |
|---|---|---|---|---|
| `prism_webhooks_by_rep` | org | maintainer (manual) | Fresh Catch | Route AE → PRISM webhook URL |
| `prism:wave_template_ids` | org | PRISM Bootstrap | PRISM (Step 1 preflight) | Waves 2–5 template IDs |
| `prism:claim:<rox_person_id>` | org | PRISM (Steps 11b, 11f, 11g) | PRISM (Steps 2, 11a), other PRISM instances | Global lead protection registry |
| `prism:company_research:<domain>` | org | PRISM (Step 4 MISS) | PRISM (Step 4) | Company background cache, 30-day TTL |

### 6.3 Claim registry schema

```json
{
  "prism_run_id": "prism_<yyyymmddThhmmss>_<8-hex>",
  "claim_owner_rox_user_id": "uuid",
  "claim_owner_name": "string",
  "claim_status": "CLAIMED | SEQUENCE_CREATED | RELEASED",
  "sequence_campaign_id": "uuid | null",
  "sequence_name": "string | null",
  "sequence_status": "PAUSED | ACTIVE | RESPONDED | MEETING_BOOKED | COMPLETED | STOPPED | null",
  "claim_timestamp": "iso8601",
  "fresh_catch_report_id": "string",
  "pursuit_id": "string — primary's pursuit_id",
  "error_reason": "string | absent"
}
```

**Invariants:**

- One claim per `rox_person_id`. Alternates are never claimed.
- Only the primary is enrolled. Alternates remain available for other motions.
- State transitions: `AVAILABLE → CLAIMED → SEQUENCE_CREATED`, or `CLAIMED → RELEASED` on any failure after Step 11b.
- PRISM only writes `PAUSED` for `sequence_status`. Other values are readable but written by future companion workflows.
- **Concurrency limitation:** `custom_store_set` is check-then-set, not compare-and-swap. Two PRISM runs firing within milliseconds could both write. Mitigated by (a) recheck at Step 11a, (b) writing claim BEFORE creating template or sequence, so a lost race releases cleanly.

### 6.4 Company research cache

```json
{
  "researched_at": "iso8601 — sole source of truth for TTL",
  "domain": "string",
  "prism_run_id_source": "string",
  "current_initiatives": [{"claim": "...", "source": "...", "date": "..."}],
  "architecture_signals": ["..."],
  "hiring_signals": ["..."],
  "strategic_priorities": ["..."],
  "known_couchbase_footprint": "string or 'none found'",
  "background_summary": "2–4 line prose"
}
```

**Cache-corroboration rule:** cached facts may inform prompts but may NEVER be cited as current evidence in Wave 1 or the Conversation Plan without corroboration from Fresh Catch's `signal_summary`/`signal_sources` OR current person research. On conflict, current signal wins.

### 6.5 Person research object

```json
{
  "pursuit_id": "carried",
  "rox_person_id": "carried",
  "person_role_relevance": "one line",
  "current_initiative": "one line",
  "recent_signal": "one line — cited public evidence",
  "technical_tension": "one line",
  "business_impact": "one line",
  "couchbase_relevance": "one line or 'none credible yet'",
  "evidence": [{"fact": "...", "source": "...", "date": "..."}],
  "confidence": "HIGH | MEDIUM | LOW",
  "research_result": "PASS | INSUFFICIENT_PERSON_EVIDENCE | INSUFFICIENT_CURRENT_SIGNAL | INSUFFICIENT_ROLE_RELEVANCE | INSUFFICIENT_COUCHBASE_RELEVANCE | CONFLICTING_EVIDENCE"
}
```

### 6.6 Research result classifier semantics (Step 6)

| Classifier | Category | Fall-through action |
|---|---|---|
| `PASS` | Success | Lock candidate as primary → Step 7 |
| `INSUFFICIENT_PERSON_EVIDENCE` | Person-specific failure | Fall through to `ranked_candidates[1]` if exists |
| `INSUFFICIENT_ROLE_RELEVANCE` | Person-specific failure | Fall through to `ranked_candidates[1]` if exists |
| `INSUFFICIENT_COUCHBASE_RELEVANCE` | Person-specific failure | Fall through to `ranked_candidates[1]` if exists |
| `INSUFFICIENT_CURRENT_SIGNAL` | Structural/premise failure | HOLD — different person won't fix a broken premise |
| `CONFLICTING_EVIDENCE` | Structural/premise failure | HOLD — do not paper over conflicting facts |
| (tool error × 2) | Hard-deterministic failure | `HOLD:research_tool_failure` — no fall-through |

The person-specific/structural distinction is the deterministic contract that keeps fall-through bounded to a single candidate and prevents wasted research when the underlying signal itself is bad.

### 6.7 Conversation Plan

The commercial-reasoning artifact. 16 fields, produced in Step 7 for the locked primary only.

```json
{
  "person": {"name": "", "pursuit_id": "", "rox_person_id": ""},
  "current_signal": "restated in one line",
  "why_now": "why this signal makes the question relevant now",
  "person_specific_evidence": [{"fact": "", "source": "", "date": ""}],
  "role_relevance": "why THIS person's responsibilities intersect the signal",

  "problem_to_confirm": "the SINGLE problem the recipient can confirm/deny/clarify",
  "business_technical_tension": "why it matters commercially or technically",

  "couchbase_relevant_problem": "problem restated in terms Couchbase would recognize",
  "couchbase_entry_point": "architecturally specific reason Couchbase is relevant (NOT 'distributed systems', 'data', 'scalability', 'AI', 'modern architecture')",
  "why_couchbase": "connection from problem to Couchbase capability",

  "ae_reason_for_contact": "why a Couchbase AE specifically — not equally a consultant/SE/analyst/vendor",
  "ae_positioning": "soft context ('I work with teams dealing with this class of problem')",

  "question": "the exact diagnostic question — YES/NO/SOMETHING DIFFERENT answerable",
  "expected_response": {"likely_answer": "", "why_it_matters": ""},

  "next_step_if_yes": "real conversation move, not 'ask for a meeting'",
  "next_step_if_no": "what the AE asks or learns next — do not abandon",
  "next_step_if_different": "follow the newly revealed problem without forcing the original",

  "se_handoff_trigger": "condition under which an SE could add value later",
  "research_to_question_trace": "evidence → problem → Couchbase → question chain"
}
```

Every field must be supported by the compact research summary or explicitly marked as an inference. The Plan is the philosophical center of the redesign: we validate the thinking here before spending tokens on the prose.

## 7. Gates and Evaluators

Two evaluators, run in sequence. The commercial evaluator validates the reasoning; the email evaluator validates the prose. Splitting them is deliberate: plan-level failures cost less to detect at plan time than after drafting.

### 7.1 Commercial viability gates (Step 8, 11 gates)

Evaluated against the Conversation Plan. Failure → HOLD immediately (no rewrite budget; re-planning the same facts won't produce a different plan).

| Gate | Checks |
|---|---|
| `V_COUCHBASE_CONVERSATION_PATH` | Confirmation creates a real commercial conversation, not just an interesting chat |
| `V_COUCHBASE_ENTRY_POINT` | Architecturally specific, not generic |
| `V_WHY_COUCHBASE` | Real problem → Couchbase capability connection, not generic vendor claim |
| `V_AE_REASON_FOR_CONTACT` | Legitimate for a Couchbase AE specifically |
| `V_QUESTION_DIAGNOSES_PROBLEM` | Question diagnoses the single problem, not abstract architecture opinion |
| `V_NO_DEAD_END` | Yes/no/different each create real, distinct conversation moves |
| `V_SE_HANDOFF_TRIGGER` | Credible technical follow-up condition exists |
| `V_RESEARCH_TO_QUESTION_TRACE` | Evidence → problem → Couchbase → question logically holds |
| `V_WHY_THIS_PERSON` | Question doesn't equally fit another executive at the same company |
| `V_WHY_NOW` | Current signal creates a real present-tense reason |
| `V_CONVERSATION_VALUE` | Recipient's answer materially changes next action |

Evaluator output:

```json
{
  "pass": true,
  "failed_gates": [{"gate": "V_*", "reason": "one line"}],
  "notes": "string | null"
}
```

Passing gates get no prose. Failed gates get one line of "why". Never enumerate PASS.

### 7.2 Email quality gates (Step 10, 20 gates)

Evaluated against the drafted Wave 1 with the approved Plan as reference. Failure → 1 rewrite (must still express the same Plan) → HOLD.

Original 18 human-quality gates (`PERSONAL_RESEARCH_GATE`, `RESPONSE_WORTHINESS_GATE`, `EASY_REPLY_GATE`, `RECIPIENT_PERSPECTIVE_GATE`, `I_TOOK_THE_TIME_TEST`, `HUMAN_GATE`, `KINDNESS_GATE`, `SINCERITY_GATE`, `SPECIFICITY_GATE`, `INSIGHT_GATE`, `NO_FLUFF_GATE`, `NON_SALESY_GATE`, `ONE_IDEA_GATE`, `TRUTH_GATE`, `ROLE_RELEVANCE_GATE`, `FRESHNESS_GATE`, `THREAD_AWARENESS_GATE`, `PERSONAL_SELLING_TEST`) plus two commercial-craft gates:

| Gate | Checks |
|---|---|
| `AE_SENDER_POSITIONING_GATE` | Reads as an AE, not an SE / consultant / analyst / generic vendor |
| `PLAN_FIDELITY_GATE` | Draft honors the approved Plan (same problem, same question, same entry point) |

Plus 4 Ultimate PRISM test questions checked in the same reasoning pass:

1. Would the recipient know they were personally researched?
2. Does the recipient have a genuine reason to respond?
3. Would the recipient believe the sender took the time?
4. Would a thoughtful Couchbase AE actually send this?

Evaluator output:

```json
{
  "pass": true,
  "failed_gates": [{"gate": "*", "reason": "one line"}],
  "expected_response": "string | null"
}
```

**Rewrite discipline:** ONE rewrite max, ONE re-evaluation. Never a second rewrite. Never re-evaluate a passing draft "to be sure". Never try "a different generation strategy". HOLD beats bad outreach.

## 8. Failure Modes and Retry Rules

| Failure kind | Retry policy | Terminal state |
|---|---|---|
| Deterministic tool error (`custom_store`, `create_email_template`, `create_campaign`, `add_contact_to_campaign`, `lookup_account_by_domain`, `find_contact`, `create_contact`, `add_html`, `send_notification`) | 1 retry max | `ERROR:*` on 2nd failure; never spend LLM reasoning to "figure out what happened" |
| Research call failure (`generate_agent_response`, `batch_generate_agent_response`) | 1 retry max | Step 5: `HOLD:research_tool_failure`; Step 4: treat as cache MISS, proceed with `company_background = null` |
| Commercial evaluator returns `pass=false` | No rewrite | `HOLD:commercial_viability_failed:<gate>` |
| Email evaluator returns `pass=false` | 1 rewrite max | `HOLD:email_quality_failed:<gate>` on 2nd failure |
| Anything else | None | `ERROR:<message>` |

### Claim safety

If claim written in Step 11b and any subsequent 11c/11d/11d-verify/11e/11f fails: transition to `claim_status: RELEASED`, record `error_reason`. Never leave a `CLAIMED` claim without a completed enrollment. Orphaned Wave 1 templates remain in the library as unreferenced drafts (do NOT attempt to delete — no rollback path).

### Enrollment protocol ordering

Order matters. This is the anti-race-condition protocol:

```
11a. Re-check claim registry           → race detected? → ALREADY_WORKED
11b. Write claim (CLAIMED)             → claim owned before any downstream work
11c. Create Wave 1 template            → fail → 11g (RELEASE)
11d. Create PAUSED sequence (is_fsd=false)
11d-verify. steps_created == 5         → fail → 11g (RELEASE)
11e. Enroll primary only               → fail → 11g (RELEASE)
11f. Update claim to SEQUENCE_CREATED  → success
11g. Release claim on any failure between 11c–11f
```

## 9. Rox Actions Used

| Action | Purpose | Steps |
|---|---|---|
| `rox_actions.custom_store_get` | Read claim registry, wave template config, company research cache | 1, 2, 4, 11a |
| `rox_actions.custom_store_set` | Write claim, write company research | 4, 11b, 11f, 11g |
| `rox_actions.lookup_account_by_domain` | Resolve account when `rox_company_id` not supplied | 2 |
| `rox_actions.find_contact` | Resolve candidate contacts | 2 |
| `rox_actions.create_contact` | Create missing contacts under `rox_company_id` | 2 |
| `email.list_emails` | Detect per-contact recent engagement | 2 |
| `agent_outputs.generate_agent_response` | Cheap triage (Step 3), company research (Step 4), Plan (Step 7), commercial evaluator (Step 8), Wave 1 draft (Step 9), email evaluator (Step 10) | 3, 4, 7, 8, 9, 10 |
| `rox_actions.create_campaign` | Create the PAUSED 5-step sequence | 11d |
| `rox_actions.add_contact_to_campaign` | Enroll primary only | 11e |
| `rox_actions.send_notification` | AE and maintainer notifications | 12 |
| `rox_actions.add_html` | Shadow diagnostic to maintainer Home | 13 |

Built-in runtime tools (not in the tools list): `batch_generate_agent_response` (Step 5), `create_email_template` (Step 11c), RQL tools (`search_rql_catalog`, `plan_and_execute_rql_query`, `discover_join_keys` — used ad-hoc for account/opportunity checks and Couchbase relationship lookup), `set_run_name` (Step 1, Step 6).

### Sequence step configuration (non-negotiable)

Every one of the 5 steps in the created sequence must have:

- `step_type`: `"manual_email"`
- `is_automatic_enabled`: `false`
- `templates`: `["<template_id>"]` — exactly one entry
- `user_input`: UNSET (ignored for templated step types)
- `generation_type`: UNSET (default; `"agent"` would let runtime AI rewrite at send — never)
- `step_subject_line`: UNSET (template supplies subject)
- Wave 1: `is_new_thread: true`; Waves 2–5: unset
- Days: `[0, 3, 7, 12, 20]`

Wave 1 uses the per-run template created in 11c. Waves 2–5 use the org-wide prebuilt templates from `wave_template_config`.

`create_campaign.is_fsd: false` sets the campaign to PAUSED at the campaign level. No email can send until an AE manually reviews and activates.

## 10. Observability

### 10.1 Progressive diagnostics

The shadow diagnostic (`add_html` to maintainer Home, Step 13) is populated progressively based on how far the run got. This is deliberate: seeing where PRISM stops in the funnel is how we tune it.

| Stopping point | Diagnostic contents |
|---|---|
| Step 1 BLOCKED | Payload snapshot, decision |
| Step 2 HOLD | + per-candidate eligibility outcome |
| Step 6 HOLD | + triage record, ranked candidates, cache HIT/MISS, Person research object(s), Couchbase relationship, fall-through decision, step 6 outcome category |
| Step 7 HOLD | + compact research summary (verbatim) |
| Step 8 HOLD | + full Conversation Plan (verbatim) + commercial evaluator output (which specific gate failed and why) |
| Step 10 HOLD | + Wave 1 draft + email evaluator output + rewrite attempt if any |
| ENROLLED | + Wave 1 template ID, wave_template_config, sequence campaign ID, steps_created, claim state transitions |

Every diagnostic tails with any tool errors (with retry attempt) and maintainer-copy notification status.

### 10.2 Maintainer notification

For `ENROLLED` and `ALREADY_WORKED` outcomes, a separate maintainer-copy notification fires (unless the maintainer is the executing AE). The maintainer copy adds per-alternate comparative reasoning and Diagnostic Notes bullets that surface run mechanics. This lets the maintainer monitor runs across every AE's instance without opening Home for each event.

### 10.3 Silent outcomes

HOLD/BLOCKED/ERROR outcomes send no rep or maintainer notifications — diagnostic only. This is a design choice: an AE hearing about every HOLD would drown out the successful outcomes. The tradeoff is that an AE has no signal that Fresh Catch fired for a lead PRISM held on. Revisit if HOLD-to-AE feedback becomes needed.

### 10.4 Telemetry to watch as volume grows

The diagnostic structure makes these queryable across runs:

- Where the funnel loses candidates (which gate fails most)
- Cheap-triage accuracy (does `triage_selection.selected` predict the eventual `research_result` classifier?)
- Company research cache hit rate
- Deep-research fall-through rate (how often does #1 fail and #2 succeed?)
- Wave 1 rewrite rate
- Commercial evaluator failure distribution across the 11 gates
- Race conditions at Step 11a

These are the signals that will drive future tuning.

## 11. Known Limitations

- **Not adaptive post-enrollment.** Rox does not currently expose pre-send hooks, reply-received triggers, or sequence-lifecycle events. Waves 2–5 are prebuilt templates written by Bootstrap; PRISM does not adapt them per prospect and does not react to replies.
- **No post-lock failover.** Once a primary is locked (Step 6 exits PASS), if Step 11a detects a race and the claim is now taken, the run exits as `ALREADY_WORKED`. It does not automatically retry against an alternate. Fall-through happens ONLY during research (Step 6).
- **Concurrency limitation on claim registry.** `custom_store_set` is check-then-set, not compare-and-swap. Two PRISM runs firing within milliseconds could both write. Mitigated by Step 11a recheck and by writing claim before creating template/sequence, but not eliminated.
- **RQL account-level dedup is bounded by the executing AE's data access.** A "0 deals on this account" reading in Step 2 reflects the AE's governed view, not organization-wide reality.
- **HOLD is silent to the AE.** By design (see §10.3). Revisit when HOLD volume becomes operationally significant.
- **Bootstrap dependency.** PRISM cannot enroll without `prism:wave_template_ids` populated. Bootstrap must run first, and each maintainer copy edit requires a Bootstrap re-run.
- **Company research cache is background only.** Cached facts inform prompts but cannot be cited as current evidence in Wave 1 or the Plan without corroboration from Fresh Catch's signal or current person research. This is an intentional constraint against stale evidence.
- **AE-only sender model.** SE-only enablements are out of scope. `AE_SENDER_POSITIONING_GATE` would fail confusingly for an SE-authored run.

## 12. Future Work

- **Pre-plan sanity check.** If Step 8 becomes the dominant HOLD point in production diagnostics, a cheap pre-plan check between Steps 6 and 7 could catch the most obvious "no viable Couchbase conversation possible" cases before spending tokens on the full 16-field Plan. Not built now — waiting for evidence.
- **HOLD-to-AE notification.** Concise notification to the AE when PRISM holds, so they know the signal was evaluated and stopped rather than silently dropped. Deferred by design; may become necessary at scale.
- **Sequence lifecycle companion workflow.** A separate workflow that reacts to `sequence_status` changes (`RESPONDED`, `MEETING_BOOKED`, `COMPLETED`, `STOPPED`) and updates the claim registry accordingly. PRISM's claim schema already accommodates these states; nothing writes them yet.
- **Reply-received adaptation.** Contingent on Rox exposing a reply-received trigger. Would allow Waves 2–5 to become prospect-specific.
- **Cross-AE triage.** Currently, `INELIGIBLE:claimed_elsewhere` treats another AE's claim as terminal for that person. A future extension could surface these as opportunities for AE-to-AE handoff or collaboration.
- **Web research budget instrumentation.** The person research call currently self-manages its stop conditions. Explicit budget tracking (source count, search-query count, time budget) would let us tune the research cost/quality tradeoff more precisely.

## Appendix A — Terminal decision reference

**Per-run HOLD reasons:** `account_unresolved`, `all_candidates_unresolved`, `all_candidates_ineligible`, `insufficient_research_all_candidates_tried`, `insufficient_signal`, `conflicting_evidence`, `research_tool_failure`, `plan_construction_failed:<reason>`, `commercial_viability_failed:<gate>`, `email_quality_failed:<gate>`

**Per-run BLOCKED reasons:** `rep_mismatch`, `missing_required_payload_field:<name>`, `missing_wave_template_config:<reason>`

**Per-candidate INELIGIBLE reasons** (narrow the pool, don't kill the run): `active_opp_conflict` (account-level; kills all remaining), `recent_two_way_engagement`, `contact_opted_out`, `claimed_elsewhere`

## Appendix B — Global Lead Protection rules

- One claim per person. Primary only. Alternates never claimed.
- First valid claim wins. Never overwrite `CLAIMED` or `SEQUENCE_CREATED`.
- Claim before template, template before sequence. Anti-race protocol.
- `RELEASED` is available. Treated exactly like no claim.
- Account-level dedup is context only for cheap-triage reasoning, not a hard block on unrelated people at the same company.

---

*Last updated: 2026-09-22. Runtime version: 1.*
