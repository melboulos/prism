# 🔮 PRISM

**Version:** 1.0 (V1 scope)
**Owner:** Mel Boulos (maintainer), Couchbase
**Deployed as:** Shared Rox Agentflow, per-rep webhook instance
**Status:** Live, validated end-to-end (workflow run `7bf0506b`, 2026-09-14)

PRISM turns a Fresh Catch signal about a company into a ready-to-activate personal
selling conversation with one specific person. It never sends email — every sequence
it creates is **PAUSED**, waiting on a human rep to review and activate it.

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
| [`prompts/agent-instructions.md`](prompts/agent-instructions.md) | The 13-step execution algorithm, the enrollment protocol, and the Personal Selling Test, written as the actual instructions to paste into the Rox agentflow config |
| [`templates/rep-notifications.md`](templates/rep-notifications.md) | The four rep-notification variants (ENROLLED, ALREADY_WORKED, COOLING_OFF, silent) |
| [`CHANGELOG.md`](CHANGELOG.md) | Validation run history |

## Key invariants (read before touching anything)

- **Never sends email.** Every sequence is created with `is_fsd: false` and
  `is_automatic_enabled: false` on every step.
- **One claim per person, ever.** `prism:claim:<rox_person_id>` — write-claim-first,
  finalize-or-release after. See design spec §5–§7.
- **V1 owns Wave 1 only.** Waves 2–5 are guardrail placeholders, not shipped copy.
- **Identity check is a hard gate.** `owning_rep_rox_user_id` /
  `owning_rep_email` in the payload must match the executing rep
  (`{{ metadata.user.* }}`) or the run aborts with `BLOCKED:rep_mismatch`.

## Known V1 limitations (tracked for V2, see design spec §12.2)

- Webhook auth is **off** (`use_auth: false`).
- `custom_store_set` is check-then-set, not compare-and-swap — a narrow race window
  exists between two near-simultaneous runs on the same person.
- No automatic failover to an alternate candidate if the primary gets claimed between
  candidate screening (step 4) and the enrollment re-check (step 11a).
- Cooling-off is *read*, never *written*, in V1.
