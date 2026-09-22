# 🔮 PRISM

**Version:** 1.0
**Owner:** Mel Boulos (maintainer), Couchbase
**Deployed as:** Shared Rox Agentflow, per-rep webhook instance
**Status:** Design spec current as of 2026-09-22. Runnable agent instructions,
notification templates, and schemas were realigned to this spec in the same
push — see [`CHANGELOG.md`](CHANGELOG.md) for validation status against the
new pipeline.

PRISM turns a Fresh Catch signal about a company into a ready-to-activate personal
selling conversation with one specific person — or explicitly holds if there isn't
a genuine conversation worth having. It never sends email — every sequence it
creates is **PAUSED**, waiting on a human rep to review and activate it.

```
🎣 Fresh Catch  →  🔮 PRISM  →  🎯 Pursue
  detection         interpretation      human conversion
```

PRISM is a **Rox agentflow** (an LLM agent with tool access — there is no step DAG).
Its "source code" is therefore the agent's instructions and tool wiring, configured
inside Rox, not a program that runs from this repo. What lives here instead is the
versioned **design contract** around it: the spec, the wire schemas, the instructions
themselves (so they can be diffed and reviewed like code), and the rep-facing
notification copy.

## Repo layout

| Path | Contents |
|---|---|
| [`docs/design-spec.md`](docs/design-spec.md) | Full technical design spec (this is the source of truth) |
| [`schemas/webhook-payload.schema.json`](schemas/webhook-payload.schema.json) | JSON Schema for the Fresh Catch → PRISM trigger payload |
| [`schemas/claim-registry-value.schema.json`](schemas/claim-registry-value.schema.json) | JSON Schema for `custom_store` claim values (`prism:claim:<rox_person_id>`) |
| [`prompts/agent-instructions.md`](prompts/agent-instructions.md) | The 13-step execution algorithm, the enrollment protocol, and the commercial-viability / email-quality evaluators, written as the actual instructions to paste into the Rox agentflow config |
| [`templates/rep-notifications.md`](templates/rep-notifications.md) | Rep- and maintainer-facing notification variants (ENROLLED, ALREADY_WORKED, silent), plus the canonical Wave 2–5 placeholder text (a cross-repo contract with PRISM Bootstrap) |
| [`CHANGELOG.md`](CHANGELOG.md) | Validation run history |

## Key invariants (read before touching anything)

- **Never sends email.** Every sequence is created with `is_fsd: false` and
  `is_automatic_enabled: false` on every step.
- **HOLD is a first-class outcome, not a failure.** If there is nothing genuinely
  worth saying, PRISM says nothing. See design spec §3.1.
- **Token expenditure only increases with confidence.** Cheap deterministic checks
  and cheap triage run before any deep research; deep research runs on the leading
  candidate only. See design spec §3.2, §5.
- **One claim per person, ever.** `prism:claim:<rox_person_id>` — write-claim-first,
  finalize-or-release after. See design spec §6.3, §8.
- **AE-authored, not SE-authored.** Wave 1 reads as an AE writing to another human,
  not an SE diagnosing architecture or pitching product. See design spec §3.3.
- **V1 owns Wave 1 only.** Waves 2–5 are Bootstrap-authored guardrail placeholders
  until genuinely reauthored — see design spec §6.2 and
  [`templates/rep-notifications.md`](templates/rep-notifications.md).
- **Identity check is a hard gate.** `owning_rep_rox_user_id` /
  `owning_rep_email` in the payload must match the executing rep
  (`{{ metadata.user.* }}`) or the run aborts with `BLOCKED:rep_mismatch`.

## Known V1 limitations (tracked for future work, see design spec §11–§12)

- Webhook auth is **off** (`use_auth: false`).
- `custom_store_set` is check-then-set, not compare-and-swap — a narrow race window
  exists between two near-simultaneous runs on the same person.
- No automatic failover to an alternate candidate once a primary is locked (step 6
  exits PASS) — if step 11a detects the primary was claimed since, the run exits
  as `ALREADY_WORKED` rather than retrying against an alternate. Fall-through to a
  different candidate happens only during research (step 6), and only once.
- Bootstrap dependency: PRISM cannot enroll without `prism:wave_template_ids`
  populated, and each maintainer edit to the Wave 2–5 templates requires a
  Bootstrap re-run.
- HOLD is silent to the AE by design — an AE has no signal that Fresh Catch fired
  for a lead PRISM held on. Revisit if this becomes operationally significant.
