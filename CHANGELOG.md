# Changelog

## Validation runs

| Version | Run | Outcome | Notes |
|---|---|---|---|
| Pre-fix (`user_input` body) | `dbb1971a` (2026-09-11 15:27) | `ERROR:create_campaign_steps_created_0` | Orphan campaign; claim released |
| Pre-fix (`user_input` body) | `e9715769` (2026-09-11 15:52) | `ERROR:create_campaign_steps_created_0` | Orphan campaign; claim released |
| Post-fix (template-linked) | `7bf0506b` (2026-09-14 20:23) | ✅ `ENROLLED` | First clean end-to-end run |

## 1.0 — 2026-09-14

- Initial versioned release of the design spec, wire schemas, agent instructions,
  and rep-notification templates.
- Root-cause fix for the `steps_created: 0` bug: `manual_email` steps require a
  linked `template_id` from `create_email_template` — `user_input` is ignored for
  templated step types. See `docs/design-spec.md` §7.1.
