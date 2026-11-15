# Surplus Food App (working name)

A web app for Sarajevo where bakeries, restaurants and shops sell unsold food at the end of the day as low-price "bags". Customers reserve a bag in the app, pick it up in a set time window, show a code and pay the shop directly.

**Live clickable mockup:** https://armincoder.github.io/surplus-food-app/ (best on a phone)

## How it works

**Customer**

1. Sees offers nearby, on a list or a map.
2. Reserves a bag and gets a short pickup code.
3. Goes to the shop in the pickup window, shows the code and pays the shop.

**Shop**

1. Sets a weekly plan once: how many bags, price, pickup window.
2. Each day the offer goes live by itself. Staff can change the number of bags or cancel the day with one tap.
3. Staff confirm each pickup by tapping "Preuzeto" or typing the customer's code.
4. Once a month the shop sees bags sold, money earned and the fee.

**Rules that protect shops:** phone-number login, at most 2 open reservations per customer, a 30-day block after 2 no-shows, and cancelling only up to 1 hour before pickup.

## Status

| Part | Status |
| --- | --- |
| Product spec for v1 ([BRIEF.md](BRIEF.md)) | Done |
| Implementation plan ([IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md)) | Done |
| Clickable mockup ([docs/index.html](docs/index.html)) | Done |
| Backend and frontend v1 | In progress |

## Planned stack

- Backend: Python 3.12, FastAPI, SQLAlchemy 2, PostgreSQL 16, Alembic, APScheduler worker
- Frontend: React, TypeScript, Vite, TanStack Query, Tailwind CSS, Leaflet with OpenStreetMap
- Delivery: installable web app (PWA), Docker Compose for local development

## Build plan

1. Skeleton: repo layout, Docker Compose, health endpoint, app shell, CI
2. Data model, migrations and seed data
3. Admin: partners, stores, staff logins
4. Shop panel: weekly plan, daily offer job, today screen
5. Customer side: offers list and map, phone login, reserve, cancel, pickup code
6. Pickup flow: confirm pickup, no-show job, blocks, undo
7. Numbers: shop stats, admin dashboard, monthly statements with CSV
8. Ratings, PWA install and Bosnian copy pass
9. Deploy

Full rules, data model and API are in [BRIEF.md](BRIEF.md). Code layout and milestone checks are in [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md).

## Author

Armin Balija, full-stack developer, Sarajevo.

© 2026 Armin Balija. All rights reserved.
