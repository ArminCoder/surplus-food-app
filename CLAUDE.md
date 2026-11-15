# CLAUDE.md

Project: a web app where Sarajevo shops sell unsold end-of-day food as low-price bags. Customers reserve in the app and pay the shop at pickup.

## Read first

- `BRIEF.md`: what to build. Rules, data model, API, screens. It is the source of truth.
- `IMPLEMENTATION_PLAN.md`: build order, code layout, and the "Done when" checks per milestone.
- `docs/index.html`: clickable mockup. Customer and shop screens should look and behave like it.

## Rules

- Build only the milestone you are asked for. Plan first, then build.
- Business rules go in `backend/app/services/`. Routers and jobs only call services.
- Never call `datetime.now()` directly. Use the `Clock` from `app/core/clock.py`.
- Money is integer feninga. Never floats.
- UI text only in `frontend/src/i18n/bs.json`.
- Tests run against PostgreSQL, never SQLite.
- Ask before adding a dependency not listed in BRIEF.md section 3, or anything from the "Not in v1" list.
- If two rules conflict or one is unclear, stop and ask.
- Small commits, one topic each, messages in English.

## Commands

Filled in during milestone M1 (run, test, lint, migrate, seed).
