# 🔮 PRISM — Rep notification templates (STEP 12)

Sent via `rox_actions.send_notification`. Body format: markdown. Cross-reference:
[`../docs/design-spec.md`](../docs/design-spec.md) §6.2, §10.2, §10.3, Appendix A.

Two audiences per notifying outcome:

- **Owning rep** (`owning_rep_rox_user_id`) — gets the AE-facing variant below.
- **Maintainer copy** — for `ENROLLED` and `ALREADY_WORKED` only, a second
  notification fires to the workflow maintainer, unless the maintainer IS the
  executing AE (in which case it would be a duplicate — skip it). The maintainer
  copy is NOT a plain forward of the rep notification: it adds per-alternate
  comparative reasoning and a Diagnostic Notes section surfacing run mechanics,
  so the maintainer can monitor every AE's instance without opening Home per
  event (design-spec.md §10.2).

## Success variant (ENROLLED) — rep copy

```
🔮 **PRISM — Sequence Ready**

**<primary_name> — <company_name>**

**Why:** <one line — current_signal + why_now, from the Conversation Plan>
**The angle:** <one line — couchbase_relevant_problem + question>
**Approach:** <one line — how the 5-wave arc will evolve>

**PRISM evaluated <N> candidates:** <primary_name> selected.
Alternates: <alternate names, or "none">.

**Status:** 🟢 Claimed by you
**Sequence:** 5-touch conversation (⏸️ PAUSED)

**Wave 1 preview**
Subject: <subject>

<full Wave 1 body>

---
Review the sequence and activate it when ready. Waves 2–5 use the standard
prebuilt templates — a future PRISM workflow will author them adaptively before
each send. Alternate contacts remain available for future outreach.
_pursuit_id: <primary_pursuit_id> · Run id: <prism_run_id>_
```

## Success variant (ENROLLED) — maintainer copy

```
🔮 **PRISM — Sequence Ready** (<owning_rep_name>'s instance)

**<primary_name> — <company_name>**

<same Why / Angle / Approach / Wave 1 preview as the rep copy>

**Diagnostic Notes**
- Cache: company research <HIT | MISS>
- Candidates researched: <1 | 2> of <N> eligible (<fall-through reason, if any>)
- Commercial evaluator: pass on first attempt
- Email evaluator: <pass on first attempt | passed after 1 rewrite — reason: <gate>>
- Per-alternate reasoning: <why each non-primary ranked lower — one line each>

_pursuit_id: <primary_pursuit_id> · Run id: <prism_run_id> · Executing rep:
<owning_rep_name> (<owning_rep_email>)_
```

## ALREADY_WORKED variant — rep copy

Fired when every candidate is `claimed_elsewhere` (Step 2), or when Step 11a
detects a lost race. Names the existing claim owner and sequence where known.

```
🔮 **PRISM — Already In Motion**

**<candidate_name(s)> — <company_name>**

This signal's candidate(s) are already being worked:
<claim_owner_name> has an active claim (<claim_status> / <sequence_status>)
with <claimed_contact_name>.

No new outreach was created for this signal.
_Run id: <prism_run_id>_
```

## ALREADY_WORKED variant — maintainer copy

```
🔮 **PRISM — Already In Motion** (<owning_rep_name>'s instance)

**<candidate_name(s)> — <company_name>**

<claim_owner_name> holds <claim_status> / <sequence_status> on
<claimed_contact_name>. No new outreach created.

**Diagnostic Notes**
- Stopping point: <Step 2 all_candidates_ineligible | Step 11a race>
- Candidates evaluated: <N>, all classified <claimed_elsewhere | lost race>

_Run id: <prism_run_id> · Executing rep: <owning_rep_name> (<owning_rep_email>)_
```

## Silent variants (no notification sent — diagnostic only)

HOLD, BLOCKED, and ERROR outcomes send no rep or maintainer notification. This
is a design choice (design-spec.md §10.3): an AE hearing about every HOLD would
drown out the successful outcomes, and PRISM never wants to announce a sequence
that isn't actually ready.

| Outcome family | Examples (Appendix A) | Rep notification | Maintainer notification | Diagnostic |
|---|---|---|---|---|
| `HOLD:<reason>` | `account_unresolved`, `all_candidates_unresolved`, `all_candidates_ineligible`, `insufficient_research_all_candidates_tried`, `insufficient_signal`, `conflicting_evidence`, `research_tool_failure`, `plan_construction_failed:*`, `commercial_viability_failed:*`, `email_quality_failed:*` | ❌ Not sent | ❌ Not sent | ✅ Written |
| `BLOCKED:<reason>` | `rep_mismatch`, `missing_required_payload_field:*`, `missing_wave_template_config:*` | ❌ Not sent | ❌ Not sent | ✅ Written |
| `ERROR:<reason>` (11c–11f failures) | `create_email_template_failed:wave1`, `create_campaign_steps_created_<N>_expected_5`, `add_contact_to_campaign_failed`, `finalize_claim_failed` | ❌ Not sent | ❌ Not sent | ✅ Written |

Rationale: never tell a rep "sequence is ready" when it isn't; never notify
under a mismatched or malformed identity; never surface an internal HOLD
reasoning chain to a rep who didn't ask for it (see design-spec.md §12,
"HOLD-to-AE notification" — deferred by design, may become necessary at scale).

## Wave 2–5 placeholder body — canonical text (cross-repo contract)

Waves 2–5 are org-wide prebuilt templates authored once by **PRISM Bootstrap**
(a separate, maintainer-run process) and referenced by ID via the org
`custom_store` key `prism:wave_template_ids` (design-spec.md §6.2). PRISM itself
does not create these templates per run — it only reads their IDs at Step 1
preflight and links them onto sequence steps 2–5 at Step 11c.

Because Bootstrap is out of this repo's direct control, this file preserves the
exact required guardrail text as PRISM's contract with Bootstrap: whatever
Bootstrap writes into any Wave 2–5 template body, until that wave is genuinely
reauthored with real research, MUST be this text, verbatim — not softened,
shortened, or paraphrased:

```
[PRISM V1 placeholder — Wave <N> (<wave_name>) will be reauthored by a future
PRISM wave-authoring workflow with fresh research before send. Do not activate
this wave until then.]
```

Rationale: if a rep accidentally activates a not-yet-reauthored wave, the
prospect must see obvious internal guardrail language, not copy that reads as
shipped, customer-ready prose. This is a designed-in safety property, not a
placeholder-quality issue — hence it is specified here exactly, rather than
described in prose, so it survives edits to either repo without silent drift.

If Bootstrap's own documentation is later confirmed to pin this text down
authoritatively and keep it in sync with this file, this section can defer to
that source instead of duplicating it. As of this writing, no such
authoritative source was found, so this file is the canonical copy.

Wave names by position: Wave 2 = Curiosity, Wave 3 = Idea, Wave 4 = Connection,
Wave 5 = Close (day offsets 3, 7, 12, 20 respectively — Wave 1 is day 0).
