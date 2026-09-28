# DomoNest

A Django + Wagtail household application that connects meal planning, pantry stock, shopping and recurring household routines instead of treating them as separate tools.

[**Live demo**](https://domonest.onrender.com/) · [Engineering notes](./docs/00_INDEX.md)

### Demo login

```text
username: demo
password: domonest-demo
```

The demo account is seeded, non-staff and non-superuser.

![DomoNest Today dashboard](./docs/images/domonest-today.png)

> Screenshot captured from the real application with the seeded demo household.

## The idea

Planning dinner often crosses several small workflows:

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
```

DomoNest keeps those workflows connected through the same household state. **Today** is not another task database, and recipe readiness is not another copy of pantry state.

## What works

### Meals, pantry and shopping

- plan a dinner for a date;
- compare recipe ingredients with pantry state;
- derive `AVAILABLE / LOW / MISSING / UNKNOWN`;
- add missing/low ingredients to Shopping without creating duplicates;
- use quick capture, grouped shopping items, buy/reopen and Undo.

### Home Rhythm

Recurring household jobs keep an event history. Completing, skipping or postponing a routine records what happened and derives the next occurrence from that history rather than rewriting old events.

### Wagtail content

Wagtail owns structured recipes, household guides, publishing workflow and reusable snippets. Recipe ingredients can participate in pantry readiness and shopping instead of remaining isolated CMS content.

### Discover

Public Wagtail content and private household data can appear in one search experience, but their queries remain separate so convenience does not weaken owner scoping.

## Rules the UI cannot bypass

The important invariants live below the template layer:

- private household queries are owner-scoped at query time;
- Recipe/Pantry → Shopping writes are idempotent;
- routine events are immutable;
- Today and Plan are derived from source-of-truth domain state;
- conservative normalized identity is used for recipe/pantry matching;
- database constraints backstop rules that must survive retries or duplicate submissions;
- public search never becomes a shortcut around private-data scoping.

These calculations are deterministic application logic. Pantry state and routine schedules are not delegated to an AI model.

## Architecture

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

Views coordinate requests, services own writes, selectors assemble read models and templates receive already-interpreted state.

The UI is server-rendered with progressive enhancement. A separate client state layer would duplicate application state without solving a current requirement.

## Stack

**Python 3.12–3.14 · Django 5.2 · Wagtail 7 · PostgreSQL · Gunicorn · WhiteNoise**

Quality/tooling:

- Ruff and Coverage;
- PostgreSQL integration tests;
- Django/Wagtail system and migration checks;
- Playwright browser journeys;
- Axe WCAG 2.2 AA checks;
- production `check --deploy` and `collectstatic`;
- query budgets around expensive read models;
- Python 3.12, 3.13 and 3.14 in CI.

SQLite is supported for zero-setup local development.

## A useful demo path

1. Open **Today**.
2. Open a recipe and inspect Pantry readiness.
3. Add missing ingredients to **Shopping**.
4. Plan the recipe for dinner.
5. Return to **Today** and see the planned meal in context.
6. Complete or postpone a **Home Rhythm** routine.
7. Try **Discover** to see public and owner-scoped results handled side by side.

## Run locally

```bash
python -m pip install -r requirements-dev.txt
python manage.py migrate
python manage.py seed_demo --reset --username demo --password domonest-demo
python manage.py runserver
```

Open `http://127.0.0.1:8000/` and use the demo credentials above.

## Deployment

The public demo runs on Render:

**https://domonest.onrender.com/**

The repository includes `render.yaml`, production settings and a `/health/` readiness endpoint. Deployment-specific PostgreSQL, media-storage, security and smoke-test details live in the [deployment runbook](./docs/11_DEPLOYMENT_RUNBOOK.md).

## Documentation

Start with [docs/00_INDEX.md](./docs/00_INDEX.md).

The detailed notes cover product intent, UX, architecture, domain modelling, quality/security/accessibility, ADRs, references and deployment. They are kept out of this README so the repository front page stays useful as a technical portfolio overview.
