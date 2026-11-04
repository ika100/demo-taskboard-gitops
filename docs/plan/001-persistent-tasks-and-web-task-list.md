---
plan_id: 001-persistent-tasks-and-web-task-list
spec: 001-persistent-tasks-and-web-task-list
feature: Tasks are kept in each environment's Postgres database and the web app lists and creates them
gitops_app: ika100/demo-taskboard-gitops
status: draft
repos:
  - id: demo-taskboard-gitops
    shape: gitops-app
    summary: Declare the system's Postgres database and wire it into the task API
    acs: [AC-001.1, AC-001.2, AC-001.5]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: demo-taskboard-api}
    depends_on: []
    done: false
  - id: demo-taskboard-api
    shape: service-python
    summary: Keep tasks in Postgres and report not-ready and 503 when the database is unreachable
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5]
    arguments: |
      Withdraw AC-001.11 of this repo's spec 001-create-and-list-tasks and its non-goals
      "persistence beyond the process lifetime" and "sharing tasks across replicas"
      (superseded by system spec 001-persistent-tasks-and-web-task-list).
      Configuration comes only from the injected DATABASE_URL (PG* are also set); no
      other DB settings. Replace the in-memory TaskStore with a Postgres-backed store;
      the request/response format and validation of AC-001.1 to AC-001.10 of spec
      001-create-and-list-tasks stay unchanged. The schema (one `tasks` table) is created
      by the service itself, idempotently, under a Postgres advisory lock so two replicas
      starting together do not race; the service must start and serve /health even when
      the database is down at startup, and create the schema on the first successful
      connection (no restart needed, AC-001.4). Order by created_at then a monotonic
      sequence column so tasks created in the same instant keep creation order.
      Tests run against a real Postgres (testcontainers or a devbox service), plus a
      test that stops it to cover 503 and not-ready. Smoke: keep AC-001.8's smoke out
      (the list is no longer empty after a deploy); AC-001.1 of the old spec stays the smoke.
    depends_on: [demo-taskboard-gitops]
    done: false
  - id: demo-taskboard-web
    shape: web-nextjs
    summary: Replace the starter page with the task list and the create-task form
    acs: [AC-001.6, AC-001.7, AC-001.8, AC-001.9, AC-001.10, AC-001.11, AC-001.12, AC-001.13, AC-001.14, AC-001.15]
    arguments: |
      The API is reached server-side only, at the env var API_URL (already set to
      http://demo-taskboard-api in system.yaml); the browser never calls the API directly
      (it is a cluster-internal name). Use a Server Component for the list and a Server
      Action (or a route handler under app/api/) for create. Smoke test: AC-001.6's page
      shape (heading "Tasks" and the form are present), written so it also passes when
      tasks exist; replace the starter smoke in e2e/smoke.spec.ts. Unit/e2e tests mock
      the API with the contract below. /api/ready stays independent of the API (the page
      must load when the API is down, AC-001.13).
    depends_on: []
    done: false
gitops_pin:
  - service: demo-taskboard-api
    overlay: staging
    apply_after: merge_of_all
    note: First staging instance of the API; promote together with the web app so staging never runs the new web app against the in-memory API. /instance:promote adds the staging instance, which gets its own Postgres cluster (AC-001.5).
  - service: demo-taskboard-web
    overlay: staging
    apply_after: merge_of_all
    note: First staging instance of the web app, promoted with the API it calls.
---

## demo-taskboard-gitops — Declare the system's Postgres database and wire it into the task API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.5 · **Depends on:** []

The database is configuration of the system, so it is declared here: `cplat resource add postgres` adds a CloudNativePG cluster per environment the API runs in (each in that environment's namespace, so dev and staging have separate databases: AC-001.5), and `uses: [postgres]` on `demo-taskboard-api` injects `DATABASE_URL` and `PG*` from the CNPG-generated Secret. A database outside the pod is what makes tasks survive restarts and redeploys (AC-001.1) and be shared by every API replica (AC-001.2). No `expose` or `env` changes: the web app already has `API_URL: http://demo-taskboard-api`, and whether the API is exposed stays as is (spec non-goal). `devbox run validate` must pass after render; no backup block (separate spec, before the first prod promotion).

This entry runs first: the new API image reports not-ready without a database, so the wiring must be in dev before the API's PR merges and pin-dev rolls it out.

## demo-taskboard-api — Keep tasks in Postgres and report not-ready and 503 when the database is unreachable

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5 · **Depends on:** [demo-taskboard-gitops]

The API owns the storage behind `POST /tasks` and `GET /tasks`: every read and write goes to the database named by `DATABASE_URL`, with no in-process cache, so a restart (AC-001.1) or another replica (AC-001.2) sees the same list, and an environment sees only its own database (AC-001.5, by configuration only). It owns the failure behaviour: a database error or timeout on either endpoint answers `503` with the error body below and never an empty list (AC-001.3); `/ready` checks the database on every call and recovers by itself (AC-001.4). `/health` stays a pure liveness check so Kubernetes does not restart pods during a database outage. Tasks in memory at the switch are dropped (spec non-goal); no migration code.

Test concerns: restart is simulated by building a new app instance against the same database; two replicas by two app instances; outage by stopping or pausing the database container, checking both 503 and that nothing was written once it is back.

## demo-taskboard-web — Replace the starter page with the task list and the create-task form

**Shape:** `web-nextjs` · **Criteria:** AC-001.6 to AC-001.15 · **Depends on:** []

The web app owns everything a person sees. It builds against the contract below, in parallel with the API (the contract is unchanged from the API's current behaviour except for the new `503`). The list is fetched server-side on each request (no caching: `dynamic = "force-dynamic"` or `cache: "no-store"`), so a task created directly through the API shows on reload (AC-001.8). It renders the API's order as-is (no client sort) and shows `due_date` as the raw string, never through `Date` (AC-001.7, AC-001.10). The form validates the title before sending (trimmed empty: "Enter a title.", AC-001.11; `maxLength=200`, AC-001.12), disables submit while a request is pending (AC-001.15), appends the returned task and clears the form on `201` (AC-001.9), and keeps the inputs and shows the save error on any failure (AC-001.14). A failed list fetch renders the page with the load error in place of the list (AC-001.13), never `error.tsx`.

## Contract

The task API is reached inside each environment's namespace at `http://demo-taskboard-api` (port 80, the Service; env var `API_URL` in the web app). Only the web app's server side calls it. JSON over HTTP, `Content-Type: application/json`, UTF-8. No authentication.

### `GET /tasks`

- `200`: a JSON array of every task, oldest first (creation order). Empty array `[]` only when the database holds no tasks.
- `503`: the database cannot be reached (body in Errors). Never `200 []` in that case.

### `POST /tasks`

Request body:

| field | type | rules |
|---|---|---|
| `title` | string, required | surrounding whitespace trimmed by the API; after trimming 1 to 200 characters |
| `due_date` | string `YYYY-MM-DD` or `null`, optional | a real calendar date, no time part; past dates accepted |

The web app sends `due_date: null` (or omits it) when the date field is empty, and the `<input type="date">` value (`YYYY-MM-DD`) otherwise.

- `201`: the created task (object below).
- `422`: validation failed, no task created (body in Errors).
- `503`: the database cannot be reached, no task created.

The API has no idempotency key: two requests create two tasks. Exactly-once from a double click (AC-001.15) is the web app's job (one in-flight request per form).

### Task object

```json
{"id": "3f0c…", "title": "Buy milk", "due_date": "2026-12-01", "created_at": "2026-11-04T10:15:30.123456Z"}
```

| field | type | notes |
|---|---|---|
| `id` | non-empty string | opaque; clients must not parse it |
| `title` | string | trimmed, 1 to 200 characters |
| `due_date` | string `YYYY-MM-DD` or `null` | exactly as stored, no time zone |
| `created_at` | string, ISO 8601 UTC with `Z` | set by the API; unchanged on every later read |

### `GET /ready` (probe, not called by the web app)

- `200` `{"status": "ready"}` when a trivial query (`SELECT 1`) against the database succeeds.
- `503` `{"status": "not ready"}` otherwise. Checked on every call, so it returns to `200` once the database is back, without a restart.
- `GET /health` stays `200` `{"status": "ok"}` regardless of the database.

### Errors

- `422` uses FastAPI's standard validation format: `{"detail": [{"loc": ["body", "<field>"], "msg": "<text>", "type": "<code>"}, ...]}`; the field name is the last element of `loc` (`title` or `due_date`).
- `503` body: `{"detail": "Task storage is unavailable."}`. The API logs the underlying database error; it never puts it in the response.
- Unexpected server failures answer `500` with `{"detail": "Internal Server Error"}`.
- **How the web app treats responses:**
  - `GET /tasks`: anything other than `200` with a JSON array (any 4xx/5xx, a body that is not an array, a network error, a timeout) shows "Tasks could not be loaded. Please try again later." in place of the list; the page itself answers `200`.
  - `POST /tasks`: `201` appends the returned task. `422` naming `title` shows "Enter a title." (it should not happen, the form checks first). Any other status, including `422` on `due_date`, `503`, `500`, an unlisted status, a network error or a timeout, shows "The task could not be saved. Please try again." and keeps the inputs; nothing is added to the list.
- Web app internal: when the server action / route handler fails it returns a typed result to the client (`{ok: true, task}` or `{ok: false, reason: "invalid-title" | "unavailable"}`), never throws to `error.tsx`.

### Timeouts

- **Web app to API, `GET /tasks`:** 5 s total (connect plus response). On timeout or connection refused: the load error message, no retry within the request; the next page load tries again.
- **Web app to API, `POST /tasks`:** 5 s total. On timeout: the save error message, inputs kept, **no automatic retry** (a retry could create a duplicate). The submit button is enabled again after the response or the timeout.
- **API to Postgres:** connect timeout 2 s, statement timeout 3 s per query, so the API answers `503` within about 4 s, inside the web app's 5 s. Connections come from a pool that drops broken connections and reconnects on the next request; no in-API retries.
- **`/ready` to Postgres:** at most 500 ms for the check (the kubelet probe timeout is 1 s); on timeout `/ready` answers `503`.
- When the API pod is down or not ready, the Service has no endpoints and the web app sees a connection error: same handling as a timeout.

## gitops-app PR

1. The `demo-taskboard-gitops` entry lands first: `cplat resource add postgres` and `cplat component set demo-taskboard-api --uses postgres`, then `devbox run render` and `devbox run validate`. After Argo reconciles `demo-taskboard-dev`: the CNPG `Cluster` is healthy and the API Deployment has `DATABASE_URL` from the `-app` Secret. The old in-memory API keeps running unaffected (it ignores the variable).
2. The API and web PRs merge in either order; pin-dev pins each in dev. Verify in dev: `/ready` of the API is `200`, a task created at `http://demo-taskboard-web.demo-taskboard-dev.localhost:8088/` is still listed after `kubectl rollout restart` of the API (done by a human), and the web smoke (AC-001.6) is green.
3. After both merged (`merge_of_all`): `/instance:promote demo-taskboard-api demo-taskboard-web --to staging` adds the staging instances (staging gets its own Postgres cluster through the resource) and pins `sha-<40>`. Verify that a task created in dev is not listed in staging (AC-001.5). No prod pin in this plan: database backups are a separate spec due before the first prod promotion.
