# DomoNest

DomoNest is a Django + Wagtail app for keeping meal plans, pantry stock, shopping, recurring household jobs and practical home notes in one place.

[**Live demo**](https://domonest.onrender.com/) · [Engineering notes](./docs/00_INDEX.md) · [Deployment runbook](./docs/11_DEPLOYMENT_RUNBOOK.md)

### Demo login

```text
username: demo
password: domonest-demo
```

The demo account is non-staff and non-superuser. It contains seeded household data only.

![DomoNest Today dashboard](./docs/images/domonest-today.png)

> Captured from the real application in Chromium with the seeded demo household.

## Why I built it

I started with a simple problem: planning dinner often means opening a recipe, checking the fridge, remembering what is running low, adding missing items to a shopping list, and then remembering the plan again later.

Those steps are usually handled in separate places. I wanted them to share the same household state.

```text
Recipe
  ↓
Pantry readiness
  ↓
Missing / low ingredients
  ↓
Shopping

Planned dinner ─────────────→ Today
Low pantry stock ───────────→ Shopping
Routine event ──────────────→ Next occurrence
Public guide + private data → Discover
```

The screens are useful on their own, but the main idea is that a change in one place can affect what appears somewhere else.

## Around the house

### Today

Today is not a separate task database. It pulls together useful actions from the current state of dinner plans, routines, shopping and pantry attention.

There is no second copy of the same household state to keep in sync.

### Plan

Choose one dinner for a date and DomoNest checks the recipe against the Pantry.

A planned meal can therefore answer two different questions:

- what are we eating?
- what do we still need?

### Shopping

Shopping supports fast capture as well as the longer Pantry/Recipe flow:

- Quick Add
- grouped items
- buy / reopen
- Undo
- focused Shopping mode
- idempotent additions from Recipe and Pantry actions

### Pantry

Pantry does not force a warehouse-style inventory model onto a kitchen.

An item can be tracked approximately or precisely, while DomoNest still derives whether it is:

`AVAILABLE` · `LOW` · `MISSING` · `UNKNOWN`

Expiry attention is derived from the stored item rather than maintained as another flag.

### Home Rhythm

Recurring household jobs keep an event history.

Completing, skipping or postponing a routine records what happened and derives the next occurrence from that history. Previous events are not rewritten to make the current schedule look tidy.

### Recipes and Guides

Wagtail handles the editorial side of the product:

- structured recipes
- relational ingredients
- practical household guides
- publishing workflow
- reusable snippets

Recipes are not isolated content pages: their ingredients can participate in Pantry readiness and Shopping.

### Discover

Discover has to search two very different kinds of information:

- public Wagtail content;
- private, owner-scoped household data.

Those queries stay separate. A convenient search box is not a reason to blur the privacy boundary.

## A useful path through the demo

This route covers most of the app without needing admin access:

1. Start on **Today**.
2. Open a recipe and inspect its Pantry readiness.
3. Add missing or low ingredients to **Shopping**.
4. Plan the recipe for dinner.
5. Return to **Today** and see the planned meal appear in context.
6. Open **Home Rhythm**, complete or postpone a routine, and see its next occurrence change.
7. Try **Discover** to see public content and household results handled side by side.

## The rules I did not want the UI to be able to break

Most of these rules live below the template layer.

- Recipe ingredients and Pantry items use conservative normalized identity instead of fuzzy matching.
- Recipe/Pantry → Shopping writes are idempotent.
- Private household queries are owner-scoped at query time.
- Routine events are immutable.
- Today and Plan are composed from source-of-truth domain state.
- Database constraints backstop invariants that should survive retries and duplicate submissions.
- Public Wagtail search never becomes a shortcut around private-data scoping.

These calculations are deterministic. Whether milk is missing or when a weekly routine is due comes from application rules, not an AI model.

## Code shape

```text
Wagtail
  public pages · recipes · guides · snippets
        │
        ├──────────────┐
        │              │
Django domain      selectors / read models
  pantry              Today
  shopping            Plan
  routines            readiness
  meal plans           Discover
        │              │
        └──── services ┘
               │
            views
               │
           templates
```

Views coordinate requests; services own writes; selectors assemble read models; templates receive state that has already been interpreted.

The UI is server-rendered with progressive enhancement. Most interactions already map cleanly to Django forms and domain services, so a separate client-side state layer would duplicate state without solving a current problem.

## Stack

**Python 3.12–3.14 · Django 5.2.17 · Wagtail 7.4.3 · PostgreSQL · Gunicorn · WhiteNoise · Playwright · Axe**

Supporting pieces:

- SQLite for zero-setup local development and fallback deployments
- server-rendered HTML + CSS
- Ruff
- Coverage
- `pip-audit`
- Chromium browser tests

## How I check it

The test suite is organised around the failures that matter for this app:

- domain and service tests for household rules;
- PostgreSQL integration coverage;
- migration-drift checks;
- Django/Wagtail system checks;
- production `check --deploy`;
- production `collectstatic`;
- Chromium user journeys;
- Axe WCAG 2.2 AA checks;
- cross-module regressions such as Recipe → Pantry → Shopping;
- query budgets around expensive read models;
- Python 3.12, 3.13 and 3.14 in CI.

Playwright artifacts include desktop/mobile screenshots, traces and failure evidence.

## Run it locally

Create a virtual environment, then:

```bash
python -m pip install -r requirements-dev.txt
python manage.py migrate
python manage.py seed_demo --reset --username demo --password domonest-demo
python manage.py runserver
```

Open:

```text
http://127.0.0.1:8000/
```

Login:

```text
username: demo
password: domonest-demo
```

The seed command always keeps that user active, non-staff and non-superuser.

## Live deployment

**https://domonest.onrender.com/**

The repository contains [`render.yaml`](./render.yaml) for Render and production settings for both PostgreSQL-backed and lightweight fallback deployments.

The Blueprint describes:

- Frankfurt web service
- PostgreSQL 17
- Python 3.13 via `.python-version`
- generated Django secret
- database values through Render `fromDatabase`
- automatic Render hostname / CSRF / Wagtail admin URL handling
- `/health/` readiness endpoint
- `check --deploy` before startup
- migrations on startup for the free Render setup
- deterministic demo restoration
- optional S3-compatible Wagtail media storage

To create the Blueprint stack in Render:

1. Choose **New → Blueprint**.
2. Select `MykolaDotsenko/domonest`.
3. Review `render.yaml`.
4. Create the resources.
5. Wait for the service health check.

Smoke-test a deployment with:

```bash
python scripts/deployment_smoke.py https://domonest.onrender.com
```

The script checks the health endpoint, root page, cache behaviour, HTTPS security headers and HSTS.

### Media on Render

The seeded demo does not depend on uploaded files, so the free hosted demo can run without durable media storage.

For editor uploads, configure `AWS_STORAGE_BUCKET_NAME` plus the S3-compatible settings documented in `.env.example`. Wagtail media and renditions then use `storages.s3.S3Storage`; WhiteNoise continues to serve versioned static assets.

## Re-capture the README image

```bash
npm install --no-audit --no-fund
npx playwright install chromium
python manage.py migrate
python manage.py seed_demo --reset --username demo --password domonest-demo
python manage.py runserver 127.0.0.1:8000 --noreload
```

Then, in another terminal:

```bash
node scripts/capture-readme-screenshot.mjs
```

Output:

```text
docs/images/domonest-today.png
```

## Production outside Render

Copy `.env.example`, use `mysite.settings.production`, configure PostgreSQL and durable media storage, then run:

```bash
python -m pip install -r requirements.txt
python manage.py check --deploy
python manage.py migrate --noinput
python manage.py collectstatic --noinput
gunicorn mysite.wsgi:application --bind 0.0.0.0:${PORT:-8000}
```

Readiness:

```text
GET /health/
```

The Docker image honors `PORT`, `WEB_CONCURRENCY` and `GUNICORN_TIMEOUT`.

## Notes behind the code

More detailed product, architecture and deployment notes are in `docs/`.

- [Product specification](./docs/01_PRODUCT_SPEC.md)
- [UX research and flows](./docs/02_UX_RESEARCH_AND_FLOWS.md)
- [UI design system](./docs/03_UI_DESIGN_SYSTEM.md)
- [Architecture](./docs/04_ARCHITECTURE.md)
- [Domain model](./docs/05_DOMAIN_MODEL.md)
- [Quality, security and accessibility](./docs/06_QUALITY_SECURITY_ACCESSIBILITY.md)
- [Implementation roadmap](./docs/07_IMPLEMENTATION_ROADMAP.md)
- [Official references](./docs/08_REFERENCE.md)
- [Architecture decisions](./docs/09_ADR_LOG.md)
- [Research log](./docs/10_RESEARCH_LOG.md)
- [Deployment runbook](./docs/11_DEPLOYMENT_RUNBOOK.md)

## Build history

I built DomoNest one household workflow at a time:

```text
foundation
→ Shopping
→ Pantry
→ Home Rhythm
→ Today
→ Wagtail content
→ Recipes
→ Recipe / Pantry / Shopping link
→ dinner planning
→ Discover
→ production hardening
→ cross-module browser coverage
```

I used that order because each new workflow depended on behaviour that was already working in the previous one.
