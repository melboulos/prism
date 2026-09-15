# 🔮 PRISM — Technical Design Specification

**Version:** 1.0 (V1 scope)
**Owner:** Mel Boulos (maintainer), Couchbase
**Deployed as:** Shared Rox Agentflow, per-rep webhook instance
**Status:** Live, validated end-to-end (workflow run `7bf0506b`, 2026-09-14)

## 1. Purpose

PRISM turns a Fresh Catch signal about a company into a ready-to-activate personal
selling conversation with one specific person. It:

1. Receives a per-company POST from Fresh Catch with 2–3 candidate contacts.
2. Resolves each candidate in Rox, checks a global claim registry for
   duplicate-outreach protection, and screens for per-candidate eligibility.
3. Researches company, signal, Couchbase footprint, and each eligible candidate in
   parallel.
4. Selects one primary contact using role fit, signal proximity, personal evidence
   depth, and seniority match.
5. Drafts a Wave 1 discovery email under a strict Personal Selling Test.
6. Authors 5 email templates (Wave 1 real, Waves 2–5 placeholders), creates a
   paused 5-step Rox Sequence, enrolls only the primary, and writes a global claim
   so no other PRISM run duplicates the outreach.
7. Notifies the owning rep with a concise sequence-ready summary; writes a full
   shadow diagnostic to the maintainer's Home.

PRISM never sends email. Every sequence it creates is **PAUSED**.

## 2. Position in the Fresh Catch → PRISM → Pursue flow

| Stage | Owns | Cadence | Output |
|---|---|---|---|
| 🎣 Fresh Catch | Detection: signal + candidate contacts | M/W/F | Lookalike Prospect leads, one webhook POST per company per rep |
| 🔮 PRISM | Interpretation: primary selection + Wave 1 + paused sequence | Per POST | One paused Rox Sequence + one rep notification + one diagnostic |
| 🎯 Pursue | Human conversion: rep activation and reply handling | Rep-driven | Real outreach, conversations, meetings |

The `pursuit_id` (Fresh Catch-generated, per candidate, format
`fc-YYYYMMDD-{initials}-{6-hex}`) is the join key that stitches all three stages.

## 3. Runtime type and multi-tenant deployment

**Runtime type:** Rox agentflow (LLM agent with tool access; no step DAG).
**Trigger:** webhook (unauthenticated in V1 — see §12 for hardening plan).
**Deployment model:** PRISM is a shared workflow. The maintainer (Mel) owns the
definition. Each rep enables their own instance in Rox, which generates a per-rep
webhook URL.

**Rep webhook map:** Fresh Catch resolves the target URL via an org-scoped
`custom_store` map, key `prism_webhooks_by_rep`:

```json
{ "rep.email@couchbase.com": "https://webhooks.backend.rox.com/webhooks/w/<slug>", ... }
```

Keys are lowercased rep emails. Maintained via the Custom Store Editor workflow
(same pattern as `support_by_rep`).

**Onboarding cost per rep:** rep enables PRISM (URL is auto-generated) → maintainer
adds one entry to the map. Reps not in the map get no PRISM handoff; their Fresh
Catch brief still ships normally.

### 3.1 Runtime identity invariants

Each PRISM run executes as the rep who enabled the instance that received the POST.
Consequences:

| Concern | Behavior |
|---|---|
| `{{ metadata.user.* }}` | Resolves to the executing rep, not the maintainer. |
| RQL reads | Governed by the executing rep's data access. |
| `send_notification` | Delivered to the executing rep. |
| `add_html` | Writes to the workflow-owner's Home (maintainer), regardless of executing rep. |

**Invariant PRISM must enforce:** `trigger_data.payload.owning_rep_rox_user_id ==
{{ metadata.user.id }}` AND `owning_rep_email == {{ metadata.user.email }}`
(case-insensitive). If either mismatches, PRISM aborts with `BLOCKED:rep_mismatch`
and records both identities in the diagnostic. This catches Fresh Catch routing bugs.

## 4. Trigger contract (webhook payload schema)

The webhook body arrives at `trigger_data.payload`. Fresh Catch produces one POST
per company per rep. See [`schemas/webhook-payload.schema.json`](../schemas/webhook-payload.schema.json)
for the machine-readable contract.

### 4.1 Required fields

| Field | Type | Notes |
|---|---|---|
| `report_id` | string | Fresh Catch's run_id — company-level lineage key |
| `company_name` | string | |
| `company_domain` | string | |
| `owning_rep_rox_user_id` | string | Must equal runtime `{{ metadata.user.id }}` |
| `owning_rep_name` | string | |
| `owning_rep_email` | string | Must equal runtime `{{ metadata.user.email }}` (case-insensitive) |
| `signal_summary` | string | Fresh Catch's summary of why the company surfaced |
| `candidate_contacts` | list[obj] | 1–3 entries; each entry as below |

### 4.2 Optional fields

| Field | Type | Purpose |
|---|---|---|
| `rox_company_id` | string | Skips domain lookup if Fresh Catch already resolved |
| `test` | bool | If truthy: run diagnostic only, no claim/sequence writes |
| `signal_sources` | list[url] | Fresh Catch's dated citations |
| `signal_type` | string | Fresh Catch's classification (e.g. `product_launch`, `funding`) |
| `fresh_catch_couchbase_angle` | string | Prior category-level Couchbase angle. Treat as prior work, not gospel. |
| `fresh_catch_call_opener` | string | Prior suggested opener. Same treatment. |

### 4.3 Candidate contact object

| Field | Type | Notes |
|---|---|---|
| `pursuit_id` | string, required | Fresh Catch's stable per-contact key; PRISM promotes the selected primary's to its claim |
| `name` | string | |
| `email` OR `linkedin_slug` | string | At least one required per candidate |
| `rox_person_id` | string, optional | If Fresh Catch already resolved |
| `title`, `phone` | string, optional | |
| `is_fresh_catch_primary` | bool, optional | Fresh Catch's own primary guess. Advisory only. |

### 4.4 Validation

If `report_id`, `owning_rep_rox_user_id`, `signal_summary`, `company_domain`, or a
non-empty `candidate_contacts` list is missing — or any candidate lacks
`pursuit_id` — PRISM records `BLOCKED:missing_required_payload_field:<name>` and
stops after writing the diagnostic. No claim or sequence side effects.

## 5. Global Lead Protection: the claim registry

### 5.1 Purpose

The claim registry is the source of truth for "PRISM is already working this
person." It prevents duplicate outreach across:

- Multiple PRISM runs firing near-simultaneously for the same person
- Different reps' PRISM instances landing on the same candidate
- The same rep re-triggering on the same person before finishing

### 5.2 Storage

**Backend:** Rox `custom_store`, org-scoped.
**Key pattern:** `prism:claim:<rox_person_id>` — one claim per person. Alternates
are NEVER claimed.
**Value schema:** see [`schemas/claim-registry-value.schema.json`](../schemas/claim-registry-value.schema.json).

```json
{
  "prism_run_id": "prism_<yyyymmddThhmmss>_<8-hex>",
  "claim_owner_rox_user_id": "…",
  "claim_owner_name": "…",
  "claim_status": "CLAIMED | SEQUENCE_CREATED | RELEASED",
  "sequence_campaign_id": "<id>" | null,
  "sequence_name": "<name>" | null,
  "sequence_status": "PAUSED | ACTIVE | RESPONDED | MEETING_BOOKED | COMPLETED | STOPPED" | null,
  "claim_timestamp": "<iso8601>",
  "fresh_catch_report_id": "…",
  "pursuit_id": "<primary_pursuit_id>",
  "error_reason": "<string>"
}
```

`error_reason` is present only after the 11g release path.

### 5.3 Interpretation rules (step 4 of the algorithm)

| Claim state | Meaning for a candidate |
|---|---|
| No claim, or `RELEASED` | AVAILABLE |
| `CLAIMED` | CLAIMED_ELSEWHERE — ineligible |
| `SEQUENCE_CREATED` with `sequence_status` ∈ {PAUSED, ACTIVE, RESPONDED, MEETING_BOOKED} | CLAIMED_ELSEWHERE — ineligible |
| `SEQUENCE_CREATED` with `sequence_status` ∈ {COMPLETED, STOPPED} and `cooling_off_until` in future | COOLING_OFF — ineligible |
| `SEQUENCE_CREATED` with `sequence_status` ∈ {COMPLETED, STOPPED} and cooling_off expired or absent | AVAILABLE |

### 5.4 State machine

```
                     ┌────────────────┐
   (no claim) ─────► │    CLAIMED     │  (step 11b)
                     └─────┬──────────┘
                           │
                           ▼
                    ┌───────────────────┐
                    │  SEQUENCE_CREATED │  (step 11f)
                    └─────┬─────────────┘
                          │
                          ▼
                    ┌────────────┐
                    │  RELEASED  │  ◄──── failure between 11c–11f (step 11g)
                    └────────────┘
```

`RELEASED` is a terminal state and is treated as "no claim" for future runs.
Preserved for audit trail; never deleted.

### 5.5 V1 concurrency limitation (documented in every diagnostic)

`custom_store_set` is check-then-set, not compare-and-swap. Two PRISM runs firing
within milliseconds for the same person could both write `CLAIMED`. In practice
this is rare (single-rep-per-instance, Fresh Catch M/W/F cadence, one company per
POST). Documented; V2 will move to a CAS primitive.

### 5.6 Cooling-off — V1 scope

V1 recognizes cooling-off if `cooling_off_until` is present and in the future. V1
never writes it. That's a V2 concern owned by a future sequence-lifecycle workflow
that will react to Rox sequence `COMPLETED`/`STOPPED` events.

## 6. Execution algorithm

See [`prompts/agent-instructions.md`](../prompts/agent-instructions.md) for the
full, runnable version of the 13-step algorithm — this section is the narrative
summary; that file is the literal instructions pasted into the Rox agentflow.

## 7. The enrollment protocol (STEP 11) — canonical order

The most subtle part of PRISM. Order matters because it prevents both race
conditions and orphaned state. Full detail, including the exact tool call shapes,
lives in [`prompts/agent-instructions.md`](../prompts/agent-instructions.md) §11.

### 7.1 Sequence step configuration — non-negotiable

All three templated `step_type` values in Rox (`email`, `manual_email`,
`automated_email`) copy their body from a linked email template. The step's
`user_input` is IGNORED for these types. Consequence:

| Field on step | Value | Rationale |
|---|---|---|
| `step_type` | `"manual_email"` | Templated, PAUSED, rep manually reviews and sends |
| `templates` | `[<template_id>]` (exactly one) | Body source. A step without a template silently drops. |
| `is_automatic_enabled` | `false` | Belt-and-suspenders alongside campaign-level pause |
| `user_input` | UNSET | Ignored for templated step types; setting it just confuses diagnostics |
| `generation_type` | UNSET | Default is correct. Setting `"agent"` would let runtime AI rewrite the body at send time. |
| `step_subject_line` | UNSET | Template supplies the subject |
| `is_new_thread` | `true` on Wave 1; UNSET on Waves 2–5 | Wave 1 starts a new thread; Waves 2–5 reply within it |
| `day` | 0, 3, 7, 12, 20 | Discovery / Curiosity / Idea / Connection / Close cadence |

**Root cause of the pre-fix bug** (workflow run `e9715769`, 2026-09-11): earlier
revisions of the instructions treated `manual_email` as "user_input goes in
verbatim." Rox correctly reported `steps_created: 0` because there was no template
linked — the step body source was empty. Fix landed 2026-09-14:
`create_email_template` first, then link the returned `template_id` on the step.
Validated by run `7bf0506b` returning `steps_created: 5` and `read_sequence`
showing 5 `manual_email` steps at correct day offsets, `auto_send: false`.

### 7.2 Why placeholders exist for Waves 2–5

The sequence needs valid, enrollable steps end-to-end for the PAUSED sequence to
be a coherent artifact. But V1 owns Wave 1 only — Waves 2–5 will be authored by a
future PRISM wave-authoring workflow with fresh research before each wave sends.
See [`templates/rep-notifications.md`](../templates/rep-notifications.md) for the
exact placeholder text.

The placeholder text is deliberately obviously-not-customer-copy. If a rep
accidentally activates a placeholder wave, the prospect sees the guardrail text,
not shipped-looking generic prose. This is a designed-in safety property, not a
limitation.

## 8. Rep notification (STEP 12)

See [`templates/rep-notifications.md`](../templates/rep-notifications.md) for all
four variants (ENROLLED, ALREADY_WORKED, COOLING_OFF, silent).

## 9. Shadow diagnostic (STEP 13)

Written via `rox_actions.add_html` regardless of outcome. Because `add_html`
writes to the workflow-owner's Home, every diagnostic lands on the maintainer's
Home — never on the executing rep's. This is the maintainer's telemetry surface
for tuning PRISM's angle-finding over time.

### 9.1 Title convention

```
🔮 PRISM diagnostic — <rep_name> — <company_name> — <decision> — <prism_run_id>
```

Decisions: `ENROLLED | ALREADY_WORKED | ALL_CANDIDATES_UNAVAILABLE | COOLING_OFF
| HOLD:<reason> | BLOCKED:<reason> | ERROR:<message>`

### 9.2 Required contents

- Lineage keys at the top: `prism_run_id`, `report_id`, primary `pursuit_id` (if
  selection completed).
- Executing rep identity (`metadata.user.name`, `metadata.user.email`) — so the
  maintainer knows whose instance ran.
- Full trigger payload snapshot (nothing redacted).
- Full candidate evaluation table: for every candidate PRISM received — name,
  role, `pursuit_id`, `is_fresh_catch_primary`, eligibility status (AVAILABLE /
  CLAIMED_ELSEWHERE / COOLING_OFF / INELIGIBLE:reason), research findings summary,
  PRISM's score reasoning, final designation (PRIMARY / ALTERNATE / SKIPPED /
  BLOCKED).
- Explicit call-out of whether PRISM agreed with Fresh Catch's
  `is_fresh_catch_primary`, and if not, why.
- Claim state before/after for the primary (all status transitions).
- Full internal Conversation Thesis (all 7 fields).
- Wave 1 draft + Personal Selling Test result per check.
- Template ids created (or the subset created before failure) with their names.
- Sequence campaign id and `steps_created`, plus any per-step errors
  `create_campaign` surfaced.
- Any tool errors.

Rendered as structured HTML (tables, headings, colored decision banner).

## 10. Tool inventory

### 10.1 Attached actions (workflow tools array)

| Package | Action | Purpose |
|---|---|---|
| `rox_actions` | `lookup_account_by_domain` | Step 2 account resolution |
| `rox_actions` | `find_contact` | Step 3 per-candidate resolution |
| `rox_actions` | `create_contact` | Step 3 fallback for unresolved candidates with email |
| `rox_actions` | `custom_store_get` | Step 4 claim read; step 11a re-check |
| `rox_actions` | `custom_store_set` | Steps 11b, 11f, 11g claim writes |
| `rox_actions` | `create_campaign` | Step 11d PAUSED sequence creation |
| `rox_actions` | `add_contact_to_campaign` | Step 11e primary enrollment |
| `rox_actions` | `send_notification` | Step 12 rep notification |
| `rox_actions` | `add_html` | Step 13 shadow diagnostic to maintainer's Home |
| `agent_outputs` | `generate_agent_response` | Step 6 research (batched via built-in `batch_generate_agent_response`) |
| `email` | `list_emails` | Step 5 engagement check |

### 10.2 Built-in runtime tools (never listed in `tools`)

| Tool | Used in step | Purpose |
|---|---|---|
| `create_email_template` | 11c | Author each wave's template; returns `template_id` |
| `batch_generate_agent_response` | 6 | Parallel research prompts in ONE call |
| `plan_and_execute_rql_query`, `search_rql_catalog` | 5, 6 | Data reads (deal ownership, footprint, sequence state) |
| `set_run_name` | 1, adjusted post-selection | Human-readable run label |
| `add_todo`, `mark_todo_done`, `update_task_list` | throughout | Agent planning scratch |
| `write_file`, `read_file`, `run`, `search` | 6 | Scratchpad for parsing research payloads |

Built-in tools are supplied by the runtime and never appear in `<<tool:…>>` action
tokens.

### 10.3 Explicitly NOT attached

- `sql.*` actions. RQL is served via the built-in `plan_and_execute_rql_query` +
  `search_rql_catalog` — the agent discovers objects, writes the SQL, handles
  errors itself. Never attach `sql.*`.
- Web-search primitives. Attached via `agent_outputs.generate_agent_response`;
  the research agent handles internet access internally.

## 11. Data-model interactions (RQL surface)

PRISM reads (never writes) these objects via RQL. Reads are governed by the
executing rep's data access.

| Object | Used for |
|---|---|
| `deal` | Step 5 — open opportunity conflict on the account (`stage_name`, `updated_at`, `rox_owner_id`, `rox_company_id`) |
| `company` | Step 6 — Couchbase footprint (`using_couchbase`, `estimated_couchbase_apps`, `couchbase_app_estimate_v2`, `couchbase_mobile_requirement`) |
| `person` | Step 5 — engagement history (`last_email`, `last_meeting`, `opt_out_email`); step 6 — candidate context |
| `sequence` | Step 4 & step 5 sanity — existing sequence state per person (`status`, `rox_person_id`, `name`) |

PRISM writes via actions only, never via RQL: contacts (`create_contact`),
campaign/sequence (`create_campaign`, `add_contact_to_campaign`), claim registry
(`custom_store_set`), rep notifications (`send_notification`), diagnostic artifact
(`add_html`), email templates (built-in `create_email_template`).

## 12. Security posture (V1) and V2 hardening

### 12.1 V1 posture

- **Webhook auth:** OFF (`use_auth: false`). Any caller with the URL can trigger a
  run.
- **Rate limiting:** none at the workflow layer — Rox platform-level protections
  only.
- **Payload trust:** the run enforces rep-identity match against
  `{{ metadata.user.* }}` (§3.1), which mitigates the primary abuse — a caller
  cannot get PRISM to enroll under a different rep. But a caller could still
  trigger PRISM runs on the correct rep's PRISM instance with crafted payloads,
  potentially spamming the claim registry.

### 12.2 V2 hardening plan

- Enable webhook signing (`use_auth: true`); Fresh Catch signs with the shared
  secret.
- Move `custom_store_set` writes to a CAS primitive to close the millisecond race
  window (§5.5).
- Introduce `cooling_off_until` writes owned by a separate sequence-lifecycle
  workflow that subscribes to Rox sequence `COMPLETED`/`STOPPED` events.
- Automatic failover to the next-best alternate when 11a detects the primary was
  claimed since step 4.
- Author Waves 2–5 adaptively via a wave-authoring workflow triggered before each
  wave's day boundary, replacing the placeholder templates.

## 13. Failure taxonomy

| Class | Reasons | Rep notification | Claim result |
|---|---|---|---|
| `BLOCKED:rep_mismatch` | Payload identity ≠ runtime identity | ❌ | none written |
| `BLOCKED:missing_required_payload_field:<name>` | Schema failure at step 1 | ❌ | none written |
| `HOLD:account_unresolved` | Step 2 — Fresh Catch shouldn't produce this | ❌ | none written |
| `HOLD:all_candidates_unresolved` | Step 3 — every candidate failed to resolve or create | ❌ | none written |
| `HOLD:<per-candidate reason>` | Step 5 — engagement / opp / opt-out killed the pool | ❌ | none written |
| `HOLD:insufficient_research` | Step 7 or 8 — no cited personal evidence AND thin category angle | ❌ | none written |
| `HOLD:personal_selling_test_failed:<check>` | Step 10 — Wave 1 failed on second attempt | ❌ | none written |
| `ALL_CANDIDATES_UNAVAILABLE` | Step 4 — every candidate CLAIMED_ELSEWHERE or COOLING_OFF | ✅ ALREADY_WORKED | none written |
| `ALREADY_WORKED` | Step 11a — lost race between step 4 and 11a | ✅ ALREADY_WORKED | none written |
| `ERROR:create_email_template_failed:wave<N>` | 11c failure | ❌ | RELEASED |
| `ERROR:create_campaign_steps_created_<N>_expected_5` | 11d-verify failure | ❌ | RELEASED |
| `ERROR:add_contact_to_campaign_failed` | 11e failure | ❌ | RELEASED |
| `ERROR:finalize_claim_failed` | 11f failure | ❌ | RELEASED |
| `ENROLLED` | Success | ✅ Sequence Ready | SEQUENCE_CREATED |

Every case writes a shadow diagnostic (§9).

**Invariant:** a `CLAIMED` claim without a completed sequence enrollment is never
left in the registry. Either 11f promotes it to `SEQUENCE_CREATED`, or 11g
transitions it to `RELEASED`.

## 14. Observability

- Rox execution trace (per run, via `get_workflow_run_traces`): every tool call,
  input, output, error, in order.
- Shadow diagnostic on maintainer's Home: the primary human-readable audit trail
  per run.
- Claim registry itself: stateful record of every primary PRISM has touched,
  current status, current owner.
- Rox sequence library: every PRISM-authored sequence carries `prism_run_id` in
  its name for grep-ability.
- Rox email template library: every PRISM-authored template carries
  `prism_run_id` in its name.
- Run name convention: `🔮 PRISM — <primary_name> @ <company_name>` (set
  post-selection via `set_run_name`).

## 15. Voice standard (Wave 1)

Plain, intelligent, conversational, specific, curious, confident, low pressure.
Avoid: marketing jargon, buzzwords, generic AI language, exaggerated enthusiasm,
fake familiarity, "just following up," forced product pitches. The recipient
should think: "Huh. That's an interesting observation."

A synthetic calibration example is preserved in
[`prompts/agent-instructions.md`](../prompts/agent-instructions.md) §15 as a
worked reference the agent should be graded against. Run `7bf0506b` validated
that PRISM's actual output clears this bar, but its real Wave 1 content is
customer-identifying and is intentionally not reproduced in either document.

## 16. What PRISM V1 is NOT

- Not a mass-email engine
- Not a sender — every sequence is PAUSED, `is_automatic_enabled: false` on every
  step, `is_fsd: false` at the campaign level
- Not autonomous across waves — V1 owns Wave 1; Waves 2–5 are placeholder-only
- Not an automatic failover engine — if the chosen primary becomes unavailable at
  11a, V1 stops; V2 will retry against alternates
- Not a cooling-off manager — V1 recognizes cooling-off, never writes it
- Not a Couchbase product-pitch bot — Couchbase enters at Wave 4, and V1 doesn't
  author Wave 4
- Not a replacement for the rep — it hands them a sequence worth activating, or it
  stays silent

## 17. Fresh Catch handoff contract

**PRISM inherits from Fresh Catch** (does not redo):

- Company discovery, ICP fit, account net-new-ness
- Compliance screen (DNC, opt-out)
- Signal identification, source citation, why-now
- Baseline contact discovery (2–3 candidates per company)
- Per-contact `pursuit_id`

**PRISM decides** (Fresh Catch does not):

- Which candidate is primary
- Deep person research per candidate
- Whether the angle is worth a personal-selling conversation
- Whether Fresh Catch's category angle and call opener are actually good
- The full Conversation Thesis
- The Wave 1 draft
- Whether to enroll at all (`HOLD` is a valid outcome)

PRISM never overrides Fresh Catch on: compliance status, company/account identity,
the signal's factual content, per-contact `pursuit_id` values.

## 18. Validation status

See [`CHANGELOG.md`](../CHANGELOG.md).
