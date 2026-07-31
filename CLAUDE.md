# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Teaching repo for Quantic's MS SDE "Software Testing" course. The app ("Hangry Hippo", a fictional
fast-food ordering system) exists as a vehicle for demonstrating unit / functional / acceptance /
security testing and CI automation. Many artifacts here are course scaffolding rather than
production code — see "Course scaffolding" below before "fixing" things that look broken.

Two independent apps in one repo:

- `backend/hangry_api/` — Django 4.0 + Django REST Framework, SQLite (`db.sqlite3` is committed on purpose)
- `frontend/` — Create React App (react-scripts 5), React 18, React Router 6, SCSS modules

They are deployed separately to AWS Elastic Beanstalk and communicate only over HTTP.

## Commands

### Backend (run from `backend/`)

```bash
# one-time: venv lives at backend/hangry_api/env
python3 -m venv hangry_api/env && . hangry_api/env/bin/activate
pip install -r hangry_api/requirements.txt

# run the server (from backend/hangry_api/)
./manage.py runserver            # http://127.0.0.1:8000/api/...

# tests + coverage — MUST be run from backend/, not backend/hangry_api/
pip install pytest django_mock_queries six coverage
coverage run -m --source=./hangry_api pytest
coverage report

# single file / single test
pytest hangry_api/tests/test_DeliveryCost.py
pytest hangry_api/tests/test_DeliveryCost.py::test_LotsOfItems
```

Two things that will bite you:

- There is no `pytest.ini`/`conftest.py`. Tests do `from api.controllers import ...`, which only
  resolves because `hangry_api/tests/__init__.py` exists, making pytest insert `backend/hangry_api/`
  onto `sys.path`. Running pytest from a different directory, or deleting that `__init__.py`,
  breaks all imports.
- Test dependencies (`pytest`, `django_mock_queries`, `six`, `coverage`) are deliberately **not** in
  `requirements.txt` — they're installed separately in the README and in every CI workflow. Keep it
  that way unless the course material changes.

### Frontend (run from `frontend/`)

```bash
npm install
npm run start                                   # http://localhost:3000
npm test                                        # jest watch mode
npm test -- --coverage --watchAll=false         # what CI runs
npm test -- --watchAll=false Home.test.js       # single file
npm test -- --watchAll=false -t "Test Render"   # single test by name

node_modules/.bin/cypress run                   # acceptance tests (headless)
node_modules/.bin/cypress open
```

To point the frontend at a local backend, edit `API_URL` in `frontend/src/utils/constants.js`
(defaults to the shared hosted API at `https://hangryhippo-api.quantic.host/`).

## Architecture

### Backend: logic lives in `controllers.py`, not views

`backend/hangry_api/api/` is a single Django app:

- `models.py` — `Category` → `Food` (FK) → `Order` (FK to `Food`, plus a free-text `name` that acts
  as the cart identifier; there is no user model and no auth)
- `controllers.py` — **all pricing logic**: `Delivery`, `Subtotal`, `Tax`, `Total`. These are plain
  classes whose `calculate` methods take no `self` — called as `Delivery.calculate(order, distance)`.
  They accept anything iterable of objects with `.quantity` / `.item.price`, which is exactly why
  the unit tests can substitute `django_mock_queries` `MockSet`/`MockModel` with no DB.
- `views.py` — thin DRF `APIView`s. Every response is `{"status": "success"|"error", "data": ...}`.
  The price views (`delivery`, `subtotal`, `tax`, `total`) each re-query `Order` by name and re-run
  the controllers; they hold no logic of their own.
- `urls.py` — mounted under `/api/` by `hangry_api/hangry_api/urls.py`.

The layering is the point of the course: **new business rules go in `controllers.py` so they stay
unit-testable without a database**, and tests go in `backend/hangry_api/tests/` (note: the
Django-generated `api/tests.py` is unused and empty).

Quirk you'll see in `Tax.calculate` / `Total.calculate`: `(0, round(x,2))[round(x,2) > 0]` — an
index-by-boolean trick that returns 0 for non-positive amounts.

### Frontend: cart state in one context

`OrderContext` (`src/context/OrderContext.js`) wraps the whole app and owns the cart. It generates a
random slug at mount (`random-word-slugs`) as the order name and uses that as the server-side cart
key — every `GET /api/order/<name>` and `POST /api/order/` goes through this context, so components
never call the order API directly. Pages (`src/pages/Home`, `src/pages/Order`) fetch menu data with
`axios` directly; presentational pieces live in `src/components/*/index.js` with a sibling
`*.module.scss`.

Elements that tests assert on are marked with `data-testid` (e.g. `category-item`) — those
attributes are load-bearing for both the jest tests and the Cypress specs; don't remove them.

### Test strategy (the actual subject matter)

| Layer | Tool | Location |
|---|---|---|
| Unit | pytest + django_mock_queries | `backend/hangry_api/tests/` |
| Functional | jest + React Testing Library, `axios` mocked via `jest.spyOn` | `frontend/src/**/*.test.js` |
| Acceptance | Cypress | `frontend/cypress/e2e/` |
| Security (DAST) | OWASP ZAP baseline action | `.github/workflows/lesson-6.yaml` |

Backend unit tests follow an explicit `#Arrange / #Act / #Assert` comment structure — match it when
adding tests. Cypress specs and one jest test ("Test Integration Render") hit the **live shared
hosted environment**, not localhost, so they can fail for reasons unrelated to local changes.

## Course scaffolding

`.github/workflows/` holds two kinds of files:

- `backend.yaml` / `frontend.yaml` — the real pipelines. Trigger on push/PR to `main` filtered by
  path, run tests, then deploy to Elastic Beanstalk. The backend deploy job flips `DEBUG = True` to
  `False` in `settings.py` with `sed` at build time, which is why `DEBUG = True` is committed.
- `lesson-1.yaml` … `lesson-6.yaml` — per-lesson exercises that trigger only on pushes to matching
  `lesson-*` branches. They end in placeholder deploy steps (`echo Placeholder for Deployment`), and
  some contain deliberate defects for students to find (e.g. `lesson-2.yaml` uses `runs_on:` and a
  job that lists itself in `needs:`). Do not "correct" these unless explicitly asked.

Likewise, `settings.py` ships a committed `SECRET_KEY`, `ALLOWED_HOSTS = ["*"]`, and
`CORS_ORIGIN_ALLOW_ALL = True` — these are intentional targets for the security-testing lessons.
