# DomoNest

**A Django + Wagtail household coordination app that connects meal planning, pantry, shopping, recurring chores and practical home knowledge into one daily workflow.**

> **Less to remember. More room to live.**

[**Open the live demo**](https://domonest.onrender.com/) · [Engineering handbook](./docs/00_INDEX.md) · [Deployment runbook](./docs/11_DEPLOYMENT_RUNBOOK.md)

### Demo login

```text
username: demo
password: domonest-demo
```

The demo account is intentionally non-privileged. Its seeded household data can be restored deterministically, so visitors can explore the product without exposing an admin account or real household information.

![DomoNest Today dashboard](./docs/images/domonest-today.png)

> The screenshot comes from the real Django/Wagtail application running against deterministic demo data in Chromium.

## The household problem is not another missing checklist

Home organisation usually gets split across unrelated tools: recipes in one place, a shopping list somewhere else, pantry stock in memory, recurring chores in reminders, and useful household knowledge buried in notes.

The friction appears in the gaps between them:

- choosing dinner does not tell you what is missing;
- noticing low stock does not update the shopping list;
- a recurring chore can be checked off, but the next occurrence still has to be worked out;
- a dashboard can become another place where the same state is copied and eventually drifts;
- private household data and public reference content need very different visibility rules.

DomoNest treats those gaps as the product.

```text
Recipe → Pantry readiness → Missing / low ingredients → Shopping
Recipe → Plan dinner → Today
Pantry low stock → Shopping
Routine → Complete / skip / postpone → Next recurrence
Discover → Public Wagtail knowledge + owner-scoped private household state
```

The result is not eight disconnected CRUD screens. A change in one part of the household can produce a useful next action somewhere else.

## What you can actually do

| Area | Household job | What DomoNest does |
| --- | --- | --- |
| **Today** | Decide what needs attention now | Composes a deterministic action feed from current household state rather than storing a second copy of dashboard data |
| **Plan** | Decide dinner without checking three places | Stores one dinner per day and calculates recipe readiness against Pantry |
| **Shopping** | Capture what must be bought quickly | Quick Add, grouping, buy/reopen, Undo and focused Shopping mode |
| **Pantry** | Keep enough stock without inventory micromanagement | Supports approximate or precise stock and derives low/missing/expiry attention states |
| **Home Rhythm** | Keep recurring work moving | Complete, skip or postpone routines while retaining immutable event history and calculating the next occurrence |
| **Recipes** | Turn editorial recipes into household actions | Structured Wagtail recipes with relational ingredients and links into Pantry/Shopping |
| **Guides** | Keep useful household knowledge actionable | Structured Wagtail content instead of an unbounded note dump |
| **Discover** | Find both public guidance and private household items safely | Searches public Wagtail content and owner-scoped household data without collapsing their privacy boundary |

## A five-minute recruiter walkthrough

The live demo is designed to be inspected rather than merely screenshotted.

1. Open **Today** and see how different domains are composed into one view.
2. Open a recipe and compare its ingredients with current Pantry readiness.
3. Send missing or low ingredients to **Shopping**.
4. Plan that recipe for dinner and return to **Today**.
5. Complete, skip or postpone a **Home Rhythm** routine and inspect how the next occurrence changes.
6. Use **Discover** to see public knowledge and private household results coexist without sharing the same data boundary.

That path exercises the main product idea: **one household action should be able to inform the next without duplicating state across features.**

## Where the engineering work sits

The interesting parts of DomoNest are mostly in the seams between features:

- recipe ingredients and Pantry items use conservative normalized identity rather than fuzzy matching;
- readiness is explicit: `AVAILABLE / LOW / MISSING / UNKNOWN`;
- Recipe/Pantry → Shopping writes are idempotent;
- owner scoping happens at query time for private household data;
- Routine history is immutable while the next occurrence is derived;
- Today and Plan are read models composed from source-of-truth domain state;
- public Wagtail search stays separate from private household queries;
- database constraints protect invariants that should survive UI mistakes or retries.

This keeps the product predictable. The application does not need an AI layer to decide whether an ingredient is missing or when a recurring task is due; those answers come from deterministic domain rules.

## Architecture

DomoNest separates editorial content from private transactional state:

- **Wagtail** owns public pages, publishing, snippets and structured authoring.
- **Django domain models + services** own household state and write invariants.
- **Selectors / read models** compose Today, Plan, recipe readiness and Discover.
- **Views** coordinate authentication, forms, services and responses.
- **Templates** render already-understood state rather than carrying business rules.
- **Database constraints** enforce invariants below the request layer.

The UI stays server-rendered with progressive enhancement because most product interactions are form- and workflow-driven. That keeps one authoritative state path through Django instead of introducing a second client-side state model only to mirror the server.

## Stack

- **Python 3.12–3.14**
- **Django 5.2.17**
- **Wagtail 7.4.3**
- **PostgreSQL** production/integration target
- SQLite for zero-setup local development and supported fallback environments
- **Gunicorn**
- **WhiteNoise** with fingerprinted production static assets
- Server-rendered HTML + CSS
- **Playwright + Chromium**
- **Axe WCAG 2.2 AA** automated browser checks
- **Ruff**, Coverage and `pip-audit`

## Quality evidence

CI checks the repository at several boundaries rather than relying on one end-to-end happy path:

- Python **3.12 / 3.13 / 3.14**;
- Ruff lint + formatting;
- Django/Wagtail system checks;
- migration drift;
- clean database migrations;
- branch coverage threshold;
- full PostgreSQL integration suite;
- production `check --deploy`;
- production `collectstatic`;
- Python dependency audit;
- Chromium golden journey;
- Axe accessibility checks;
- cross-module workflow regression scenarios;
- query-budget regressions for high-value read models.

Browser CI publishes Playwright evidence including desktop/mobile screenshots, traces and failure artifacts.

## Run the demo locally

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

Use the same demo account as the hosted version:

```text
username: demo
password: domonest-demo
```

The seed command forces this account to remain active, non-staff and non-superuser.

## Render deployment

**Live portfolio deployment:** [https://domonest.onrender.com/](https://domonest.onrender.com/)

The repository includes [`render.yaml`](./render.yaml) for a reproducible Render topology with a Frankfurt web service and PostgreSQL 17. The production settings also support SQLite when PostgreSQL environment variables are absent, which is useful for lightweight portfolio/fallback deployments.

The Render configuration covers:

- Python 3.13 pinned through `.python-version`;
- generated Django secret key in Blueprint deployments;
- PostgreSQL credentials wired through Render `fromDatabase` references;
- automatic Render hostname, CSRF origin and Wagtail admin URL discovery;
- `/health/` readiness endpoint;
- production `check --deploy` before process startup;
- migrations at startup on Render Free, where `preDeployCommand` is unavailable;
- deterministic public demo restoration through environment variables;
- optional S3-compatible media storage for Wagtail uploads.

### Reproduce the Blueprint deployment

1. Open Render and choose **New → Blueprint**.
2. Select `MykolaDotsenko/domonest`.
3. Review `render.yaml`.
4. Create the web service and PostgreSQL resources.
5. Wait for the health check to pass.

The public demo credentials are:

```text
username: demo
password: domonest-demo
```

### Verify a deployment

Run the standard-library smoke check:

```bash
python scripts/deployment_smoke.py https://domonest.onrender.com
```

It checks the database-aware health endpoint, root page, no-store health semantics, HTTPS security headers and HSTS.

### Persistent Wagtail media

Render Free has an ephemeral filesystem. The seeded portfolio demo does not depend on uploaded media, so it can run without a media bucket.

For editor uploads, set `AWS_STORAGE_BUCKET_NAME` and the matching S3-compatible credentials from `.env.example`. Production settings then route Wagtail media and image renditions through `storages.s3.S3Storage`; AWS S3, Cloudflare R2 and DigitalOcean Spaces-compatible endpoints are supported.

For a paid Render service, migrations can move to `preDeployCommand`, `DOMONEST_RUN_MIGRATIONS_ON_START` can be disabled, and `DOMONEST_AUTO_SEED_DEMO` should be turned off for real household use.

See the full [deployment runbook](./docs/11_DEPLOYMENT_RUNBOOK.md).

### Re-render the README screenshot

After installing browser dependencies:

```bash
npm install --no-audit --no-fund
npx playwright install chromium
python manage.py migrate
python manage.py seed_demo --reset --username demo --password domonest-demo
python manage.py runserver 127.0.0.1:8000 --noreload
```

In another terminal:

```bash
node scripts/capture-readme-screenshot.mjs
```

The script writes the rendered UI to:

```text
docs/images/domonest-today.png
```

## Production baseline

For non-Render environments, copy `.env.example`, use `mysite.settings.production`, configure PostgreSQL and durable media storage, then run:

```bash
python -m pip install -r requirements.txt
python manage.py check --deploy
python manage.py migrate --noinput
python manage.py collectstatic --noinput
gunicorn mysite.wsgi:application --bind 0.0.0.0:${PORT:-8000}
```

Health/readiness endpoint:

```text
GET /health/
```

The included Docker image honors `PORT`, `WEB_CONCURRENCY`, and `GUNICORN_TIMEOUT` at runtime.

## Engineering handbook

The implementation is documented beyond the README so product rationale, architecture and operational decisions can be inspected independently.

Start with **[docs/00_INDEX.md](./docs/00_INDEX.md)**.

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

## How the product grew

DomoNest was rebuilt as bounded vertical slices:

1. foundation + CI;
2. design system + app shell;
3. Shopping;
4. Pantry;
5. recurring Home Rhythm;
6. Today orchestration;
7. Wagtail content architecture;
8. Recipe domain;
9. Recipe → Pantry → Shopping;
10. dinner planning;
11. privacy-safe Discover/search;
12. production hardening;
13. cross-module browser regression scenarios.

Each slice added a usable household capability while preserving domain invariants, accessibility and testability.
