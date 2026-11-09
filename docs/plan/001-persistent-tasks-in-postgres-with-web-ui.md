---
plan_id: 001-persistent-tasks-in-postgres-with-web-ui
spec: 001-persistent-tasks-in-postgres-with-web-ui
feature: Keep tasks in a Postgres database behind the Task API and give the web app a task page to list and add tasks
gitops_app: ika100-demo/taskboard7-gitops
status: draft
repos:
  - id: taskboard7-gitops
    shape: gitops-app
    summary: Declare the postgres resource, give it to the Task API and point the web app at the Task API
    acs: [AC-001.1, AC-001.2, AC-001.5, AC-001.6]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard7-api}
      - {env: {API_URL: 'http://taskboard7-api'}, component: taskboard7-web}
    depends_on: []
    done: false
  - id: taskboard7-api
    shape: service-python
    summary: Store tasks in Postgres so they survive restarts and are shared between instances
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4]
    criteria:
      - "Given tasks A, B and C were created with POST /tasks, when the Task API process is stopped and started again against the same database and a client sends GET /tasks, then the response is 200 with exactly A, B and C, oldest first, each with the same id, title and due_date as returned on creation (system AC-001.1)"
      - "Given two Task API processes connected to the same database, when a task is created with POST /tasks on one of them, then GET /tasks on the other returns that task with the same id, title and due_date (system AC-001.2)"
      - "Given the database, when clients use POST /tasks and GET /tasks, then every criterion of component spec 001 (AC-001.1 to AC-001.7) still holds unchanged: 201 with id, title and due_date (null when absent); 422 in FastAPI's standard validation shape naming title or due_date, with no task stored; GET /tasks returns all tasks oldest first, [] for an empty database; equal titles get different ids (system AC-001.3)"
      - "Given the database is unreachable, when a client sends a valid POST /tasks or a GET /tasks, then the response is 503 with the JSON body {\"detail\": \"Database unavailable\"} within 5 seconds, and once the database is back no task from that POST exists (system AC-001.4)"
      - "Given the database is unreachable, when the platform calls GET /ready, then it answers 503 with {\"status\": \"not ready\"} while GET /health keeps answering 200; given the database is reachable, GET /ready answers 200 with {\"status\": \"ready\"} (system AC-001.4)"
      - "Given the Task API was started or kept running while the database was unreachable, when the database becomes reachable again, then GET /ready, POST /tasks and GET /tasks succeed without restarting the Task API (system AC-001.4)"
    arguments: |
      Connection: read DATABASE_URL (postgresql://...), injected by the gitops `uses: postgres`; the PG* variables are
      also set. Locally and in tests use a throw-away Postgres (testcontainers or the devbox service). The process must
      start even when the database is unreachable (no crash loop): connect lazily, create the schema (or run the
      migration) on first successful connection, and reconnect on the next request.
      Storage: one table `tasks` (id text primary key holding a uuid4 string, title text not null, due_date date null,
      seq bigint generated always as identity) ordered by seq for "oldest first"; tasks in memory are not carried over.
      Replace the in-memory TaskStore behind get_task_store, keep the router and models unchanged.
      Database timeouts: connect 2 s, statement 3 s, /ready check (SELECT 1) 1 s; never retry a POST insert.
    depends_on: [taskboard7-gitops]
    done: false
  - id: taskboard7-web
    shape: web-nextjs
    summary: Task page on the root route that lists tasks and adds a task through the Task API
    acs: [AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given the Task API's GET /tasks returns tasks, when a person opens the root page, then it lists every task in the order the API returns them (oldest first), each with its title and its due_date as YYYY-MM-DD, or \"No due date\" when due_date is null (system AC-001.5)"
      - "Given the Task API's GET /tasks returns [], when a person opens the root page, then it shows \"No tasks yet\" and the create form (system AC-001.5)"
      - "Given the root page, when a person enters a valid title, optionally picks a due date in the date picker and submits, then the web app sends POST /tasks with {\"title\": <as entered, not trimmed>, \"due_date\": \"YYYY-MM-DD\" or null} to the Task API, and on 201 the returned task appears at the end of the list without a page reload and the form is cleared (system AC-001.6)"
      - "Given the create form, when a person submits a title that is empty or only whitespace, then \"Enter a title\" is shown next to the title field; when the title is longer than 200 characters (Unicode code points), then \"Title can be at most 200 characters\" is shown there; in both cases the entered values stay and no POST /tasks is sent (system AC-001.7)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers GET /tasks with anything but 200 and a JSON array, when a person opens the root page, then it shows \"Tasks could not be loaded. Try again later.\" instead of the list (system AC-001.8)"
      - "Given the Task API is unreachable, does not answer within 5 seconds, or answers POST /tasks with anything but 201 (including 422 and 503), when a person submits the create form, then it shows \"The task could not be saved. Try again.\", keeps the entered values and leaves the list unchanged (system AC-001.8)"
    arguments: |
      The browser never calls the Task API: the Next.js server does, at the base URL from the server-only env var
      API_URL (default http://localhost:8080 for local dev; set to http://taskboard7-api in the cluster; never a
      NEXT_PUBLIC_ variable). Render the list on the server (dynamic, no caching) and submit the form through a Server
      Action or an app/api route handler that calls POST /tasks and returns the created task to a client component.
      Replace the starter content of app/page.tsx; document API_URL in docs/env-vars.md. Unit/e2e tests stub the
      Task API (contract below); no live API needed.
    depends_on: []
    done: false
gitops_pin: []
---

The spec spans three repos: the Task API swaps its in-memory store for Postgres, the web app gets its task page, and
this gitops repo wires the database and the web app's link to the API. The Task API contract does not change (spec
assumption A2), so the API and the web app build in parallel against the contract below.

## taskboard7-gitops — Declare the postgres resource, give it to the Task API and point the web app at the Task API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.5, AC-001.6 · **Depends on:** []

Persistence (AC-001.1) and sharing between instances (AC-001.2) need a database that outlives the API's pods:
`{resource: postgres}` declares it in `system.yaml`, `{uses: postgres, component: taskboard7-api}` injects
`DATABASE_URL` and `PG*` into the Task API. The web app's page (AC-001.5, AC-001.6) reaches the Task API through its
cluster Service (`taskboard7-api`, port 80 -> 8080): `API_URL=http://taskboard7-api`. Both components are already
exposed through the Gateway, so no `expose` operation is needed. Only `dev` is affected (staging and prod are
non-goals).

This entry runs first: the Task API built for Postgres is never ready without `DATABASE_URL`, so its rollout would
stall if it reached dev before the wiring. The current in-memory API ignores the new variables, so landing the wiring
early is harmless.

## taskboard7-api — Store tasks in Postgres so they survive restarts and are shared between instances

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4 · **Depends on:** [taskboard7-gitops]

Owns all persistence behaviour. The component spec 001 criteria are its regression suite (AC-001.3): its existing
tests must keep passing against the Postgres store, with the test database emptied between tests. AC-001.2 is checked
in the repo by two app instances (two ASGI apps or two processes) on one database; dev runs one replica, and replicas
are not part of this spec. AC-001.4 needs tests that stop or block the database (testcontainers pause/stop, or a
connection URL to a closed port, then a working one) and check 503 bodies, `/ready` and recovery without a restart.
Smoke tests: keep the read-only `GET /tasks` check; the create-then-list smoke test (`@pytest.mark.writes`) now also
proves persistence across deploys in dev.

## taskboard7-web — Task page on the root route that lists tasks and adds a task through the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.5, AC-001.6, AC-001.7, AC-001.8 · **Depends on:** []

Owns everything a person sees. The starter landing page is replaced by the task page. Client-side title validation
mirrors the API's rules (A6) so AC-001.7 never reaches the API; the date picker (`<input type="date">`) makes an
invalid date impossible. Any failure of the Task API, including an unexpected 422, maps to the two fixed messages of
AC-001.8 (wording A7, exact strings). The list is fetched on page load and the created task is appended after each
successful create; there is no polling (A8). Tests stub the Task API at `API_URL`; the `@smoke` test opens `/` and
checks that either the list, "No tasks yet" or nothing worse than the AC-001.8 message renders.

## Contract

Caller: the taskboard7-web **server** (never the browser). Callee: taskboard7-api at `API_URL` (cluster:
`http://taskboard7-api`). JSON over HTTP, `Content-Type: application/json`, UTF-8. No authentication (non-goal).
Unchanged from the Task API's component spec 001; no version prefix.

**Task** object: `{"id": string, "title": string, "due_date": "YYYY-MM-DD" | null}`. `id` is an opaque, non-empty,
server-generated string (today a uuid4); clients must not parse it. `due_date` is always present, `null` when unset.

`POST /tasks`
- Request: `{"title": string, "due_date": "YYYY-MM-DD" | null}`; `due_date` may be omitted (= null). Unknown fields
  are ignored.
- `title`: a string of 1 to 200 characters (Unicode code points) with at least one non-whitespace character, stored
  exactly as sent (no trimming). `due_date`: a calendar date `YYYY-MM-DD`, no time, past dates allowed.
- `201`: the created Task.
- `422`: invalid input, FastAPI's standard shape `{"detail": [{"loc": ["body", "<field>"], "msg": ..., "type": ...}]}`
  naming `title` or `due_date`. Validation runs before any database access, so invalid input answers 422 even when
  the database is down. Nothing is stored.
- `503`: the database is unreachable or too slow: `{"detail": "Database unavailable"}`. Nothing is stored.

`GET /tasks`
- `200`: a JSON array of all Tasks, oldest first (creation order), no paging; `[]` when there are none.
- `503`: `{"detail": "Database unavailable"}`.

`GET /ready` (platform probe, not called by the web app)
- `200 {"status": "ready"}` when a `SELECT 1` on the database succeeds within 1 s; otherwise
  `503 {"status": "not ready"}`. `GET /health` stays `200 {"status": "ok"}` regardless of the database.

No events.

### Errors

- The Task API's error bodies are JSON with a `detail` field: a list of validation items for `422`, a string for
  `503` (and for any other framework error such as `404`/`405`/`500`).
- The web app treats exactly two responses as success: `200` with a JSON array for `GET /tasks`, `201` with a Task
  object for `POST /tasks`. Every other outcome — any other status (`422`, `503`, `5xx`, `4xx`), a body that is not
  the expected JSON, a connection error or a timeout — is a failure: the page shows "Tasks could not be loaded. Try
  again later." (list) or "The task could not be saved. Try again." (create, values kept). The web app never shows
  the API's `detail` text and never retries a `POST`.
- The web app logs the failure (status or error kind) server-side as structured JSON, without the task title.

### Timeouts

- web -> `GET /tasks`: 5 s total per request, no retry; on timeout the list error message is shown.
- web -> `POST /tasks`: 5 s total per request, no retry (a retry could create a duplicate task); on timeout the save
  error message is shown. A create that timed out on the web side may still have been stored; it shows up on the next
  page load.
- Task API -> Postgres: connect timeout 2 s, statement timeout 3 s, so the API answers `503` within 5 s and before the
  web app gives up; `/ready` checks with a 1 s timeout. The API keeps no request waiting on a dead database: each
  request tries a (pooled) connection and fails fast; it reconnects by itself on the next request once the database is
  back.
- Database down: the Task API stays up (`/health` 200), reports not ready (Kubernetes removes it from the Service),
  and the web app shows the AC-001.8 messages. The web app's own `/api/ready` does not depend on the Task API.

## gitops-app PR

No pins: the components run only in `dev`, which tracks `main` (staging and prod are non-goals), so `gitops_pin` is
empty. The gitops PR carries the three operations above (`cplat resource add postgres`, `cplat component set` for the
Task API's `uses` and the web app's `API_URL`), then `devbox run render` and `devbox run validate`.

After Argo reconciles dev, verify: the postgres resource is healthy; the `taskboard7-api` pod has `DATABASE_URL` and is
ready; the `taskboard7-web` pod has `API_URL=http://taskboard7-api`. After the API and the web app merge and deploy:
`POST /tasks` then `GET /tasks` on `taskboard7-api.taskboard7-dev.localhost`, restart the API pod
(`kubectl rollout restart`) and check the task is still listed; open `taskboard7-web.taskboard7-dev.localhost` and see
the task, add one through the form, and confirm it in `GET /tasks`.
