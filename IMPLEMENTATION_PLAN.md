# Implementation plan (v1 web app)

How to build v1 with Claude Code, one milestone at a time. The rules, data model and API live in [BRIEF.md](BRIEF.md). This file says in what order to build, how the code is laid out, and how to know each step is done.

## How to work with Claude Code

- One milestone per session. Start each one in plan mode, check the plan, then let it build.
- Work on a branch per milestone (`m1-skeleton`, `m2-data-model`, ...). Open a PR into `main` when the "Done when" list passes.
- Paste the milestone's prompt from this file. Claude Code reads `CLAUDE.md` on its own, which points it to the brief and this plan.
- Run `/clear` between milestones so old context does not leak in.
- Review the diff yourself before merging. Pay most attention to anything touching reservations, blocks and fees.

## Architecture

```
backend/
  pyproject.toml          deps managed with uv
  alembic/                migrations
  app/
    main.py               FastAPI app, routers, error handler
    core/config.py        pydantic-settings, every number from the brief
    core/clock.py         Clock interface: real clock in prod, fixed clock in tests
    core/security.py      JWT cookies, argon2, role audiences
    db.py                 async engine and session
    models/               SQLAlchemy models, one file per table group
    schemas/              Pydantic request and response models
    api/v1/               routers: auth, me, offers, reservations, partner, admin
    services/             all business rules live here, never in routers
      offers.py           daily offer creation, close offers
      reservations.py     reserve, cancel, lock and counters
      pickups.py          pickup, no-show, undo, fee snapshot
      blocks.py           block and unblock rules
      otp.py              login codes and limits
      statements.py       partner stats, monthly statements, CSV
      rate_limit.py       PostgreSQL counters
    sms/                  SmsProvider, ConsoleSmsProvider, HttpSmsProvider
    jobs/worker.py        APScheduler entry point
    cli.py                create-admin, seed
  tests/
frontend/
  src/
    api/                  typed fetch client, TanStack Query hooks
    i18n/bs.json          all UI text
    routes/user/          customer screens
    routes/partner/       shop screens (lazy loaded)
    routes/admin/         admin screens (lazy loaded)
    components/           shared UI
docker-compose.yml        postgres, backend, worker, frontend
.github/workflows/ci.yml  lint, type check, tests, build
```

Three choices that keep the code testable:

1. **Rules only in `services/`.** Routers parse input, call one service function, return the result. Jobs call the same service functions.
2. **Time comes from `Clock`.** No `datetime.now()` anywhere else. Tests set the clock to "19:05 Sarajevo time" and check what is allowed.
3. **Tests run against real PostgreSQL**, not SQLite. Row locks and time zones behave differently in SQLite. CI starts a Postgres service.

## Milestones

Times assume 5 to 10 hours a week next to client work.

| # | Milestone | Rough time |
| --- | --- | --- |
| 1 | Skeleton and CI | 1 evening |
| 2 | Data model, seed, create-admin | 1 to 2 evenings |
| 3 | Admin API and screens | 2 evenings |
| 4 | Shop panel: weekly plan, daily offers, today screen | 2 evenings |
| 5 | Customer side: offers, login, reserve, cancel | 3 evenings |
| 6 | Pickup flow: pickup, no-show job, blocks, undo | 2 evenings |
| 7 | Numbers and statements | 1 to 2 evenings |
| 8 | Ratings, account delete, PWA, copy pass | 2 evenings |
| 9 | Deploy to the server | 1 evening |

### M1. Skeleton and CI

Tasks:

- Repo layout as above. `uv` project in `backend/`, Vite React TypeScript app in `frontend/`.
- FastAPI app with `GET /api/v1/health` returning `{"status":"ok"}` and the error format from the brief.
- `docker-compose.yml` with postgres, backend (uvicorn with reload), worker (empty scheduler for now), frontend (Vite dev server, `/api` proxied to backend).
- `.env.example` with every setting, `config.py` reading them.
- GitHub Actions: ruff check, ruff format check, pytest with a Postgres service, frontend `tsc --noEmit`, eslint and build.

Done when:

- [ ] `docker compose up` shows the React shell at localhost and `/api/v1/health` answers.
- [ ] CI is green on the PR.

Prompt:

> Read BRIEF.md and IMPLEMENTATION_PLAN.md. Build milestone M1 only. Use uv for the backend. Plan first, then build. Stop when the M1 "Done when" list passes and show me how to run it.

### M2. Data model, seed, create-admin

Tasks:

- SQLAlchemy models and one Alembic migration for every table in brief section 5, with the unique constraints listed there.
- Money as integers in feninga, timestamps as UTC `timestamptz`, weekday 0 = Monday.
- `Clock` in `core/clock.py`.
- `python -m app.cli create-admin --email ...` (asks for the password).
- `python -m app.cli seed`: 1 admin, 6 made-up partners with stores in Centar and Novo Sarajevo, weekly plans, today's offers, 3 demo users. One partner has `free_until` in the past.

Done when:

- [ ] `alembic upgrade head` on an empty database works, and `alembic downgrade base` works too.
- [ ] Seed runs twice without errors or duplicates.
- [ ] Tests check the unique constraints (one offer per store per day, one rating per reservation, unique pickup code per offer).

Prompt:

> Build milestone M2 from IMPLEMENTATION_PLAN.md, following BRIEF.md section 5. Plan first. All shop names in the seed must be made up.

### M3. Admin API and screens

Tasks:

- Admin login with argon2 and httpOnly cookies, separate token audience.
- CRUD for partners, stores and partner staff. Validation from brief rule 1 where it applies.
- Admin screens: plain tables and forms, no styling effort.
- Login rate limit from rule 18, counted in PostgreSQL.

Done when:

- [ ] Admin can log in, create a partner, a store with coordinates, and a staff login.
- [ ] A partner token cannot call admin endpoints (test).
- [ ] 11th failed login in an hour is refused (test).

Prompt:

> Build milestone M3. Keep admin screens plain. Put rate limiting in services/rate_limit.py using PostgreSQL.

### M4. Shop panel: weekly plan, daily offers, today screen

Tasks:

- Partner login. A staff member only sees stores of their own partner.
- Weekly plan GET and PUT, with "copy to all days" in the UI.
- `services/offers.py`: create today's offers from active plans (rule 2), idempotent. Worker runs it at 05:00 Sarajevo time and once at start.
- Today screen: offered, reserved, picked up; plus and minus for quantity (rule 11); cancel today with confirm (rule 12, SMS through the provider). Reservation list stays empty until M5.
- Phone-first layout for the shop screens, big buttons.

Done when:

- [ ] Running the job twice creates each offer once (test).
- [ ] A plan change today does not change today's offer (rule 3, test).
- [ ] Staff of partner A gets 404 on partner B's store (test).

Prompt:

> Build milestone M4. All rules go in services/, the worker calls the same functions. Use the Clock for every time check.

### M5. Customer side: offers, login, reserve, cancel

Tasks:

- `GET /offers` sorted by distance (plain haversine in SQL is fine for one city), filters by category.
- Phone login: OTP request and verify, rules 18 and 19, ConsoleSmsProvider in dev.
- Reserve with `SELECT ... FOR UPDATE` on the offer (rules 4 to 9). Pickup code from the safe alphabet, retry on the rare clash.
- Cancel (rule 10). Show "Ova rezervacija se ne može otkazati" before confirming a late reservation.
- Screens: offers list, map with Leaflet and OpenStreetMap attribution, offer detail, login sheet, reservation with big code, my reservations.
- Today screen in the shop panel now lists reservations.

Done when:

- [ ] Concurrency test: 10 users reserve the last bag at the same time, exactly 1 succeeds.
- [ ] Tests for rules 4 to 10 and 18 to 19, each with a fixed clock.
- [ ] On a phone-size browser, a demo user can find an offer, log in with the console code and see the pickup code.

Prompt:

> Build milestone M5. Start with the service and its tests, including the concurrency test with real PostgreSQL sessions, then the API, then the screens. Match the look of docs/index.html (the mockup) for the customer screens.

### M6. Pickup flow

Tasks:

- Pickup by tap and by code (rule 13) with `fee_snapshot`.
- No-show job every 10 minutes and offer closing (rule 15).
- Automatic blocks (rule 16) and admin blocks (rule 17).
- Undo until midnight (rule 14): clears the fee, re-checks the automatic block.

Done when:

- [ ] Fee is 0 inside the free period and `fee_per_bag × quantity` after it (test, both sides of `free_until`).
- [ ] 2 no-shows in 30 days block the user; undoing one lifts an automatic block but never an admin block (tests).
- [ ] Undo after midnight is refused (test).

Prompt:

> Build milestone M6. Write the tests for rules 13 to 17 first, then make them pass.

### M7. Numbers and statements

Tasks:

- Partner stats per month.
- Admin dashboard for today.
- Monthly statements per partner and CSV export (UTF-8 with BOM so Excel shows č, ć, š, ž, đ).

Done when:

- [ ] Statement totals match a hand-made fixture month (test).
- [ ] CSV opens in Excel with correct Bosnian letters.

Prompt:

> Build milestone M7. Statements count only picked_up reservations and sum fee_snapshot.

### M8. Ratings, account delete, PWA, copy pass

Tasks:

- Ratings (rule 20), account delete (rule 21).
- PWA manifest, icons as neutral placeholders, service worker caching the app shell only.
- Empty states and error texts for every `code` the API returns, all in `bs.json`.
- Placeholder pages for terms and privacy.

Done when:

- [ ] The "Done means" check at the end of BRIEF.md section 11 passes on a real phone.
- [ ] Lighthouse PWA check passes on the built frontend.

Prompt:

> Build milestone M8. Then walk through the full demo flow from BRIEF.md section 11 and list anything that does not work.

### M9. Deploy

Tasks:

- Production compose file: postgres with a volume, backend with gunicorn and uvicorn workers, worker, frontend built and served as static files by nginx.
- nginx site with HTTPS (Let's Encrypt) on the existing server, next to the other apps.
- Daily `pg_dump` backup kept for 14 days.
- A real SMS provider plugged into `HttpSmsProvider` only after one is chosen.

Done when:

- [ ] The app runs on its domain over HTTPS.
- [ ] A restore from last night's backup works on a test database.

Prompt:

> Build milestone M9: production compose, nginx config and a backup script. Do not run anything on the server. Give me the exact commands to run there myself.

## Open decisions (not for Claude Code)

- App name and domain. Until then everything uses `APP_NAME`.
- SMS provider and price per message.
- Who reviews the terms of use and the partner agreement.
