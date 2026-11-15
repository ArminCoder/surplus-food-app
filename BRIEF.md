# Build brief: surplus food app (v1)

Working name: TBD. Use `APP_NAME` from config everywhere, never hard-code a name.

## 1. What we are building

A web app for Sarajevo where bakeries, restaurants and shops sell unsold food at the end of the day as cheap "bags". Users reserve a bag in the app, go to the shop in a set time window, show a pickup code and pay the shop directly.

Same model as Too Good To Go. Three kinds of people use it:

- User: finds offers nearby, reserves, picks up.
- Partner staff: sets a weekly plan of bags, confirms pickups.
- Admin (the owner): creates partners, watches numbers, exports monthly statements.

Key business facts that shape the build:

- The user pays the shop at pickup. No payments in the app in v1.
- The partner later pays a fee per bag picked up (default 1 KM), after a free period. No fee for no-shows or cancelled bags. v1 only tracks this. It does not send invoices.
- The first launch is one area of Sarajevo with 15 to 20 partners. Keep everything small and simple.
- Partner staff use it during the evening rush. The partner screens must work with one or two taps.

## 2. Scope

### In v1

- User: phone login, list and map of offers, offer detail, reserve, cancel, my reservations, pickup code, rate after pickup.
- Partner: login, weekly plan, today's offer (change quantity, cancel today), today's reservations, confirm pickup, mark no-show, simple monthly numbers.
- Admin: create and edit partners and stores, create partner logins, see all offers and reservations, block and unblock users, monthly statement per partner with CSV export.
- Background jobs: create daily offers from weekly plans, close offers, mark no-shows, apply blocks.
- Bosnian UI. All text in one translation file so English can be added later.

### Not in v1 (do not build)

- Card or any online payment.
- Native iOS or Android apps. Build a web app that installs as a PWA.
- Delivery.
- Invoices, tax logic, fiscal receipts.
- Push notifications, chat, loyalty points, referral codes.
- Partner self-signup. Admin creates partners by hand.

## 3. Tech stack

Backend (Python is a fixed requirement):

- Python 3.12, FastAPI, Pydantic v2
- SQLAlchemy 2 (async) with Alembic migrations
- PostgreSQL 16
- pytest for tests, ruff for lint and format
- Background jobs: APScheduler inside a separate worker process (same codebase, own entry point)
- No Redis in v1. Rate limits are counted in PostgreSQL (see rule 18).

Frontend:

- React with TypeScript, Vite
- React Router, TanStack Query
- Tailwind CSS
- Leaflet with OpenStreetMap tiles for the map. Show the OpenStreetMap attribution on the map.
- PWA manifest and service worker (install to home screen, cache the app shell only)

Project layout:

```
/backend      FastAPI app, models, migrations, jobs, tests
/frontend     React app (user, partner and admin areas in one app, split by route)
/docker-compose.yml   postgres, backend, worker, frontend for local dev
/README.md    how to run, test and seed
```

One frontend app with three route groups: `/` (user), `/partner`, `/admin`. Load partner and admin code lazily.

## 4. Conventions

- Money: store as integer feninga (1 KM = 100). Never use floats. Show as "5,00 KM".
- Time: store UTC in the database. All business rules use the `Europe/Sarajevo` time zone. Put the zone in config.
- Weekdays: 0 = Monday, 6 = Sunday (same as Python `date.weekday()`).
- IDs: UUIDs.
- Language: code, comments and commit messages in English. UI text in Bosnian, from `frontend/src/i18n/bs.json`.
- Config: all settings through environment variables. Provide `.env.example`. Never commit secrets.
- Errors: API returns `{ "code": "machine_code", "message": "human text" }`. The frontend maps `code` to Bosnian text.

## 5. Data model

Use these tables. Add indexes where queries need them.

**users**

- id, phone (unique, E.164 format, nullable after account deletion), name (optional), created_at, deleted_at (nullable)
- blocked_until (nullable timestamp), block_reason (nullable), block_source (`auto` or `admin`, nullable)

**otp_codes**

- id, phone, ip, code_hash, expires_at, attempts, created_at

**partners** (the business)

- id, name, legal_name (optional), contact_name, contact_phone, contact_email
- fee_per_bag (integer feninga, default 100)
- free_until (date, fee is 0 for pickups up to and including this date; default when created = today + 90 days)
- active (bool), created_at

**partner_staff** (logins for the partner panel)

- id, partner_id, email (unique), password_hash, name, active

**stores** (a physical location, a partner can have many)

- id, partner_id, name, address, municipality, lat, lng
- category (bakery, pastry, restaurant, grocery, hotel, other)
- photo_url (optional), description (optional), active

**weekly_plans** (one row per store per weekday)

- id, store_id, weekday (0 to 6)
- quantity, price (feninga), original_value (feninga)
- pickup_start (local time), pickup_end (local time)
- bag_description (short text, for example "Mix peciva")
- active
- unique (store_id, weekday)

**offers** (one per store per day, created from the weekly plan or by hand)

- id, store_id, date
- quantity_total, quantity_reserved
- price, original_value, pickup_start, pickup_end (timestamps), bag_description
- status: `active`, `cancelled`, `ended`
- unique (store_id, date)

**reservations**

- id, offer_id, user_id, quantity (1 or 2)
- pickup_code (5 characters, no look-alike characters such as 0/O and 1/I). Unique (offer_id, pickup_code). An offer is one store on one day, so this makes the code unique per store per day.
- status: `reserved`, `picked_up`, `cancelled_by_user`, `cancelled_by_partner`, `no_show`
- price_snapshot (feninga, set at reservation), fee_snapshot (feninga, set at pickup using the partner's fee and free period, null otherwise)
- created_at, picked_up_at, cancelled_at

**ratings**

- id, reservation_id (unique), stars (1 to 5), comment (optional), created_at

**admins**

- id, email, password_hash

**rate_limit_hits** (if you need it for rule 18)

- id, key (for example `login:ip:1.2.3.4`), created_at

## 6. Business rules

Put every number below in config so it can change without code edits.

Weekly plan and offers:

1. Validation for a weekly plan row and an offer: price is lower than original_value, quantity is at least 1, and pickup_end is after pickup_start on the same day (no window across midnight).
2. A job runs every day at 05:00 local time and creates today's offers from active weekly plans, for active stores of active partners. If an offer for that store and date already exists, skip it. The same job also runs once when the worker starts, so a worker that was down at 05:00 still creates the day's offers.
3. A change to the weekly plan applies from the next day. Today's offer is changed only from the Today screen.

Reserving:

4. A user can reserve only from an `active` offer with bags left, and only before `pickup_end` minus 15 minutes.
5. One reservation per user per offer, for 1 or 2 bags.
6. A user can have at most 2 open (`reserved`) reservations at a time.
7. A blocked user (`blocked_until` in the future) cannot reserve. Show the date the block ends.
8. Reserving must be safe under load. Lock the offer row (`SELECT ... FOR UPDATE`) and check `quantity_reserved + quantity <= quantity_total` in the same transaction. Two users must never get the last bag.
9. `quantity_reserved` counts bags in `reserved`, `picked_up` and `no_show`. Any cancel lowers it by the reservation's quantity in the same transaction.

Cancelling:

10. A user can cancel until 60 minutes before `pickup_start`. The bags go back on offer. If the user reserves later than that, show "Ova rezervacija se ne može otkazati" on the confirm step before they reserve.
11. A partner can lower today's quantity, but not below `quantity_reserved`.
12. A partner can cancel today's offer. All open reservations become `cancelled_by_partner` and each user gets an SMS. This does not count against the user.

Pickup:

13. Staff confirm pickup from the list of today's reservations, or by typing the code. Status becomes `picked_up`, and `fee_snapshot` is saved (0 if the pickup date is on or before `free_until`, else `fee_per_bag` times quantity).
14. Staff can undo a pickup or a no-show until midnight of the same day. The status goes back to `reserved`. Undoing a pickup clears `picked_up_at` and `fee_snapshot`. Undoing a no-show re-runs the block check in rule 16: if the user now has fewer than 2 no-shows in 30 days and the block came from the automatic rule, lift it.

No-shows and blocks:

15. A job runs every 10 minutes. Reservations still `reserved` 15 minutes after `pickup_end` become `no_show`. Offers past `pickup_end` become `ended`.
16. A user with 2 no-shows in the last 30 days (by offer date) is blocked for 30 days, with `block_source = auto`. Run this check every time a reservation becomes `no_show`.
17. Admin can block or unblock any user by hand (`block_source = admin`). An admin block is never lifted by rule 14.

Login and limits:

18. A login code has 6 digits, lasts 5 minutes, and allows 5 tries. At most 3 codes per phone per hour and 10 per IP per hour, counted from `otp_codes`. Store only a hash of the code. Partner and admin logins: at most 10 failed tries per email per hour. Reserve endpoint: at most 20 calls per user per hour. Count these in PostgreSQL, not in memory.
19. In v1 only phone numbers that start with a prefix from config are accepted (default `+387`).

Ratings and accounts:

20. A user can rate only a `picked_up` reservation, once, within 7 days of pickup.
21. Deleting an account (`DELETE /me`) is refused while the user has an open reservation or an active block. On delete: clear phone and name, set `deleted_at`, end all sessions. Keep the reservation rows, so partner statements stay correct.

## 7. Auth

- Users: phone number plus SMS code. On success set an access token (15 minutes) and a refresh token (30 days) as httpOnly, Secure, SameSite=Lax cookies.
- Partner staff and admins: email plus password (argon2 hash). Same cookie approach, separate token audience per role.
- The first admin is created with a CLI command (`python -m app.cli create-admin --email ...`), which asks for the password. No admin signup page.
- SMS: build an `SmsProvider` interface with two versions: `ConsoleSmsProvider` (prints the message, used in dev and tests) and a placeholder `HttpSmsProvider` driven by config. The real provider is not chosen yet. Do not pick or sign up for one.
- SMS is sent in only two cases: login codes, and when a partner cancels an offer.

## 8. API

Prefix everything with `/api/v1`. Generate OpenAPI docs.

User:

- `POST /auth/otp/request` (phone)
- `POST /auth/otp/verify` (phone, code)
- `POST /auth/refresh`, `POST /auth/logout`
- `GET /me`, `PATCH /me` (name), `DELETE /me` (see rule 21)
- `GET /offers?lat=&lng=&radius_km=&category=` (today's active offers, nearest first, with bags left)
- `GET /offers/{id}`
- `POST /offers/{id}/reservations` (quantity)
- `GET /reservations` (mine, newest first)
- `POST /reservations/{id}/cancel`
- `POST /reservations/{id}/rating` (stars, comment)

Partner:

- `POST /partner/auth/login`, `POST /partner/auth/logout`
- `GET /partner/stores`
- `GET /partner/stores/{id}/weekly-plan`, `PUT /partner/stores/{id}/weekly-plan`
- `GET /partner/stores/{id}/today` (offer plus reservations)
- `PATCH /partner/offers/{id}` (quantity)
- `POST /partner/offers/{id}/cancel`
- `POST /partner/reservations/{id}/pickup`
- `POST /partner/reservations/pickup-by-code` (store_id, code)
- `POST /partner/reservations/{id}/no-show`
- `POST /partner/reservations/{id}/undo`
- `GET /partner/stats?month=YYYY-MM` (bags offered, sold, picked up, no-shows, money earned by the shop, fee)

Admin:

- `POST /admin/auth/login`
- CRUD for partners, stores and partner staff
- `GET /admin/offers`, `GET /admin/reservations` (filters: date, store, status)
- `GET /admin/users`, `POST /admin/users/{id}/block`, `POST /admin/users/{id}/unblock`
- `GET /admin/statements?month=YYYY-MM` (per partner: bags picked up, fee total)
- `GET /admin/statements.csv?month=YYYY-MM`
- `GET /admin/dashboard` (today: active offers, bags, reserved, picked up, no-show rate)

## 9. Screens

Design for a phone first. Large tap targets. Plain, friendly look.

User:

1. Offers: a list of cards (store name, category, distance, price, crossed-out original value, pickup window, bags left) with a toggle to the map view. Ask for location; if refused, default to the centre of Sarajevo.
2. Offer detail: photo, description, address with a link that opens the phone's map app, price, pickup window, "Rezerviši" button.
3. Login: phone number, then code. Shown only when the user tries to reserve.
4. Reservation: a large pickup code, address, time window, a clear line "Plaćate u radnji", and a cancel button while cancelling is allowed.
5. My reservations: open ones first, then history with a rate button.
6. Profile: name, logout, delete account, links to terms and privacy.

Partner:

1. Today (home screen): bags offered, reserved, picked up. Plus and minus buttons for quantity. "Otkaži za danas" with a confirm step. Below it, the list of reservations with code and name, each with one large "Preuzeto" button. A code input at the top.
2. Weekly plan: seven rows, one per day: on or off, quantity, price, original value, pickup from and to. A "copy to all days" button.
3. Numbers: this month and last month.

Admin: simple tables and forms. Looks do not matter here, speed of building does.

## 10. Quality bar

- Tests for every rule in section 6, with special care for rule 8 (concurrent reservations), rules 14 and 16 (blocks and undo) and the fee snapshot in rule 13.
- API tests for the happy path of each role.
- A seed script that creates: 1 admin, 6 demo partners with stores at real-looking Sarajevo coordinates (Centar and Novo Sarajevo), weekly plans, today's offers, and 3 demo users. Use made-up shop names, not real businesses. Give one demo partner a `free_until` in the past, so its pickups show the 1 KM fee.
- `docker compose up` must start everything. The README explains run, test, seed, the create-admin command and the list of environment variables.
- Rate limiting as in rule 18.
- Structured logs. Never log phone numbers in full, login codes or tokens.

## 11. Build order

Work in this order. Finish and show each step before starting the next.

1. Skeleton: repo layout, Docker Compose, FastAPI health endpoint, React app shell, CI running lint and tests.
2. Data model: models, migrations, seed script, create-admin command.
3. Admin basics: admin login, CRUD for partners, stores and staff.
4. Partner panel: login, weekly plan, daily offer job, today screen without reservations.
5. User side: offers list and map, offer detail, phone login with the console SMS provider, reserve, cancel, pickup code.
6. Pickup flow: partner confirms pickup, no-show job, blocks, undo.
7. Numbers: partner stats, admin dashboard, monthly statements with CSV.
8. Ratings and polish: ratings, account delete, PWA install, empty states, error texts, Bosnian copy pass.

Done means: on a phone, a demo user can find an offer from the partner whose free period has ended, reserve it, see the code, a demo partner can confirm the pickup, and the admin statement for the month shows that bag with a 1 KM fee.

## 12. Working rules for you

- Ask before adding anything from the "Not in v1" list or any paid service.
- Ask before adding a new major dependency that is not in section 3.
- If a rule here is unclear or two rules conflict, stop and ask. Do not guess.
- Keep commits small, one topic each.
- Do not invent a brand name, logo or colours beyond a neutral placeholder theme. Those will come later.
- Do not write legal texts. Add placeholder pages for terms and privacy.
