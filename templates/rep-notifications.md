# Rep notification templates (STEP 12)

Sent via `rox_actions.send_notification` to `owning_rep_rox_user_id`. Body format:
markdown. Cross-reference: [`../docs/design-spec.md`](../docs/design-spec.md) §8.

## Success variant (ENROLLED)

```
🔮 **PRISM — Sequence Ready**

**<primary_name> — <company_name>**

**Why:** <one sentence from thesis — signal + person>
**The angle:** <one sentence — conversation to open>
**Approach:** <one sentence — how the 5-wave arc will evolve>

**PRISM evaluated <N> candidates:** <primary_name> selected.
Alternates: <alternate names, or "none">.

**Status:** 🟢 Claimed by you
**Sequence:** 5-touch conversation (⏸️ PAUSED)

**Wave 1 preview**
Subject: <subject>

<full Wave 1 body>

---
Review the sequence and activate it when ready. Waves 2–5 are placeholders —
a future PRISM workflow will author them adaptively before each send.
Alternate contacts remain available for future outreach.
_pursuit_id: <primary_pursuit_id> · Run id: <prism_run_id>_
```

## ALREADY_WORKED variant

Fired when all candidates unavailable, or when step 11a hit. Names the existing
claim owner and sequence.

```
🔮 **PRISM — Already In Motion**

**<candidate_name(s)> — <company_name>**

This signal's candidate(s) are already being worked:
<claim_owner_name> has an active sequence (<sequence_status>) with
<claimed_contact_name>.

No new outreach was created for this signal.
_Run id: <prism_run_id>_
```

## COOLING_OFF variant

Same shape as ALREADY_WORKED, with wording:

```
🔮 **PRISM — Cooling Off**

**<candidate_name(s)> — <company_name>**

<claimed_contact_name> was recently in a PRISM sequence that ended
(<sequence_status>). Cooling off until <cooling_off_until>. PRISM will
reconsider after that date if a meaningful new signal exists.

No new outreach was created for this signal.
_Run id: <prism_run_id>_
```

## Silent variants (no notification sent)

| Outcome | Rep notification | Diagnostic |
|---|---|---|
| `HOLD:<reason>` | ❌ Not sent | ✅ Written |
| `ERROR:<reason>` (11c–11f failures) | ❌ Not sent | ✅ Written |
| `BLOCKED:<reason>` (rep_mismatch, missing_required_payload_field) | ❌ Not sent | ✅ Written |

Rationale: never tell a rep "sequence is ready" when it isn't; never notify under
a mismatched or malformed identity.

## Wave 2–5 placeholder body (used inside `create_email_template`, step 11c)

This exact text — do not soften or shorten it — so an accidentally-activated
placeholder wave reads as an obvious internal guardrail, not shipped copy:

```
[PRISM V1 placeholder — Wave <N> (<wave_name>) will be reauthored by a future
PRISM wave-authoring workflow with fresh research before send. Do not activate
this wave until then.]
```

Wave names by position: Wave 2 = Curiosity, Wave 3 = Idea, Wave 4 = Connection,
Wave 5 = Close (day offsets 3, 7, 12, 20 respectively — Wave 1 is day 0).
