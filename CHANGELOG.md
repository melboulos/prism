# 🔮 PRISM — Changelog

## Validation runs

| Version | Run | Outcome | Notes |
|---|---|---|---|
| Pre-fix (`user_input` body) | `dbb1971a` (2026-09-11 15:27) | `ERROR:create_campaign_steps_created_0` | Orphan campaign; claim released |
| Pre-fix (`user_input` body) | `e9715769` (2026-09-11 15:52) | `ERROR:create_campaign_steps_created_0` | Orphan campaign; claim released |
| Post-fix (template-linked) | `7bf0506b` (2026-09-14 20:23) | ✅ `ENROLLED` | First clean end-to-end run, validated against the pre-1.0 3-stage pipeline |

⚠️ No end-to-end run has yet validated the redesigned 13-step pipeline below.
Run `7bf0506b` validated an earlier architecture (single research batch across
all eligible candidates, 7-field Conversation Thesis, single 6-check Personal
Selling Test gate) that this version's design spec, agent instructions, and
notification templates have since replaced. Treat this repo as design-complete
but not yet re-validated until a new run is logged here.

## 1.0 (redesign) — 2026-09-22

- Full pipeline redesign, documented in `docs/design-spec.md`: 13 sequential
  steps replacing the prior architecture, with cheap deterministic checks and
  cheap candidate triage ahead of any LLM research spend; cached (30-day TTL)
  company research; deep person research on the leading candidate only, with
  a single bounded fall-through to the next candidate on person-specific
  (not structural) research failures; a 16-field Conversation Plan replacing
  the prior 7-field Conversation Thesis; and a two-evaluator gate split —
  an 11-gate commercial viability evaluator (no rewrite budget) followed by a
  20-gate email quality evaluator (max 1 rewrite) — replacing the prior
  single 6-check Personal Selling Test.
- `prompts/agent-instructions.md`, `templates/rep-notifications.md`, and both
  files in `schemas/` were rewritten to match. The `COOLING_OFF` notification
  variant and the `cooling_off_until` claim field were removed — that concept
  does not exist in this version. A maintainer-copy notification (in addition
  to the rep notification) was added for `ENROLLED` and `ALREADY_WORKED`
  outcomes, per design-spec §10.2.
- Waves 2–5 template ownership moved to PRISM Bootstrap: PRISM now reads their
  IDs from `prism:wave_template_ids` at Step 1 preflight rather than creating
  per-run placeholder templates. The exact guardrail placeholder text was
  confirmed absent from any Bootstrap-side documentation and is preserved in
  `templates/rep-notifications.md` as the canonical, cross-repo contract PRISM
  Bootstrap's templates must satisfy.
- **Not yet re-validated end-to-end** — see the warning above the validation
  run table.

## 1.0 (initial) — 2026-09-14

- Initial versioned release of the design spec, wire schemas, agent instructions,
  and rep-notification templates.
- Root-cause fix for the `steps_created: 0` bug: `manual_email` steps require a
  linked `template_id` from `create_email_template` — `user_input` is ignored for
  templated step types. This fix is preserved and unaffected by the 2026-09-22
  redesign — see `docs/design-spec.md` §5, note under the pipeline table.
