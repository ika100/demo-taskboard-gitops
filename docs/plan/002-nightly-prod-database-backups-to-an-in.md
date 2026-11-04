---
plan_id: 002-nightly-prod-database-backups-to-an-in
spec: 002-nightly-prod-database-backups-to-an-in
feature: Nightly prod database backups with continuous archiving to an in-cluster bucket, landed with the first prod promotion and followed by a restore drill
gitops_app: ika100/demo-taskboard-gitops
status: draft
repos:
  - id: demo-taskboard-gitops
    shape: gitops-app
    summary: Add an in-cluster backup bucket, back the prod database up to it nightly, promote the API and the web app to prod in the same PR, then run the restore drill
    acs: [AC-002.1, AC-002.2, AC-002.3, AC-002.4, AC-002.5, AC-002.6, AC-002.7, AC-002.8, AC-002.9, AC-002.11, AC-002.13, AC-002.15, AC-002.16, AC-002.17]
    arguments: |
      Build this entry by hand in one PR, not through /system:ship's entry automation: the steps are
      prose in the body, not `gitops:` operations (plan.py only knows `postgres` as a resource, and the
      operations cannot express `--type bucket`, the `backup:` block or the prod promotion). Bucket resource name: `backups` (SeaweedFS bucket demo-taskboard-backups),
      default size. Backup block (hand-edited in system.yaml, no cplat flag exists):
      resources.postgres.instances.prod.backup: {to: backups, schedule: "0 3 * * *", retention: 30d}.
      Prod promotion: /instance:promote demo-taskboard-api demo-taskboard-web --to prod
      (release tags 0.1.0 and 0.1.1), committed on the same branch so it is one PR (AC-002.15).
    gitops:                       # PLACEHOLDER: already in system.yaml, changes nothing; plan.py rejects an empty list and knows only postgres.
      - {uses: postgres, component: demo-taskboard-api}   # the real steps (bucket backups, uses backups, prod backup block, prod promotion) are prose in the body, one hand-built PR
    depends_on: []
    done: false
gitops_pin: []                    # the prod pins of both components are part of the gitops entry itself (AC-002.15); plan-check only accepts pins of plan repos
---

## demo-taskboard-gitops — Add an in-cluster backup bucket, back the prod database up to it nightly, promote the API and the web app to prod in the same PR, then run the restore drill

**Shape:** `gitops-app` · **Criteria:** AC-002.1 to AC-002.9, AC-002.11, AC-002.13, AC-002.15, AC-002.16, AC-002.17 · **Depends on:** []

**Single repo.** Everything in this spec is configuration of the system plus operator actions, and the platform already provides the database backup machinery (spec 117: CloudNativePG with the Barman Cloud plugin, `ObjectStore` + `ScheduledBackup` rendered from `backup:`; `cplat resource backup` / `cplat resource restore` with `restore-state.yaml`; the Backups section of `/system:status`). No component repo changes: `demo-taskboard-api` receives `S3_*` / `AWS_*` env vars from `uses: [backups]` and ignores them (accepted risk in the spec's non-goals); its `DATABASE_URL` keeps pointing at the `<system>-postgres-app` Secret, which a switch repoints (AC-002.8 without a new image). `demo-taskboard-web` only gets its first prod pin.

**What the PR contains (one PR, AC-002.15):**

1. `cplat resource add backups --type bucket` — an in-cluster SeaweedFS bucket. Buckets get no `HTTPRoute`; only components with `expose` do, so the bucket is reachable only inside the cluster (AC-002.5).
2. `cplat component set demo-taskboard-api --uses backups` — the platform requires a component that runs in prod to use the backup target (render.py `check_backup`). Side effect: `uses` is component-wide, so the bucket also gets instances in `dev` and `staging` (the API runs there) and the API gets the bucket credentials there too. No backup is configured in dev or staging (non-goal respected); the extra bucket instances are unused.
3. Hand edit in `system.yaml`: `resources.postgres.instances.prod.backup: {to: backups, schedule: "0 3 * * *", retention: 30d}`. This renders the `ObjectStore`, the `ScheduledBackup` (03:00, AC-002.1) and `isWALArchiver: true` on the prod `Cluster` (continuous WAL archiving, AC-002.2) with Barman retention 30d (AC-002.4).
4. `/instance:promote demo-taskboard-api demo-taskboard-web --to prod` — writes `instances.prod` for both and pins `0.1.0` / `0.1.1`. The prod database (3 replicas, 20Gi, already declared) only exists from this point, so backups are on from its creation.
5. `devbox run render`, then `devbox run ci` (prod has `policy: enforce`: the new SeaweedFS and CNPG workloads in prod must pass the Kyverno checks of `validate`; a violation is fixed in `system.yaml`, never in generated files).

**How each criterion is owned:**

- AC-002.1, AC-002.4: `ScheduledBackup` at `0 3 * * *` and retention `30d`. The schedule is evaluated in UTC (render.py: "five-field cron in UTC"); "cluster time" in the spec is UTC on this cluster.
- AC-002.2: WAL archiving through the plugin. The 5-minute bound comes from CloudNativePG's default `archive_timeout` of `5min`; render.py sets no Postgres parameters, so the default applies. Verified after reconcile (see gitops-app PR).
- AC-002.3: `/resource:backup postgres --env prod` (online base backup from the cluster; the API keeps serving).
- AC-002.5: no route for the bucket in any environment (verified in the rendered overlays).
- AC-002.6, AC-002.7, AC-002.9, AC-002.11: `/resource:restore postgres --env prod --to <time>` creates a separate generation `demo-taskboard-postgres-g<n>`; the active cluster and the app Secret are untouched until a switch; a time outside `[earliest, latest]` is refused with both bounds named (`dbrestore.py`); `--discard --confirm prod` removes the copy and its volumes.
- AC-002.8: `--switch --confirm prod` repoints the app Secret and restarts the API pods (no new image) and recreates the `ScheduledBackup` for the new active cluster.
- AC-002.16: `--switch` and `--discard` in prod refuse without `--confirm prod` and change nothing (`dbrestore.py`).
- AC-002.17, AC-002.13: the Backups section of `/system:status` shows the last successful backup and marks the database `failing` when continuous archiving fails or the newest finished backup failed; the app keeps serving because archiving and backups run beside the database (WAL-volume risk below).
- AC-002.15: steps 1 to 4 in one PR, then the drill right after merge (see gitops-app PR).

**Build by hand, notes and risks:**

1. **Built by hand in one PR:** the entry's only `gitops:` operation, `{uses: postgres, component: demo-taskboard-api}`, is a placeholder that is already in `system.yaml` and changes nothing; it is there because `plan.py` rejects an empty list and knows only `postgres`, so the real steps are the prose above. `plan-check` only knows the resource `postgres` (`KNOWN_RESOURCES` in `scripts/plan.py`), and `/system:ship` maps `{resource: X}` to `cplat resource add X` without `--type`, has no operation for `backup:` and lands the gitops entry as its own PR before any promotion. Steps 1 to 5 above are run by hand on one branch, as one PR (AC-002.15).
2. **AC-002.2 relies on an operator default:** render.py sets no Postgres parameters, so the 5-minute window is CloudNativePG's default `archive_timeout` of `5min`; checked after reconcile (gitops-app PR, step 2).
3. **AC-002.13, risk:** while the bucket is unreachable, WAL segments that cannot be archived accumulate on the prod database volume (20Gi). A long bucket outage can fill it and then stop writes; the status overview shows `failing` long before that. Not a criterion breach as long as the failure is noticed.

## Contract

No contract between repos: this plan has one repo. The interfaces the drill and the operator use are platform commands, run from this repo:

- `/resource:backup postgres --env prod` — creates an on-demand `Backup`; listed with nightly ones by `/system:status`.
- `/resource:restore postgres --env prod --to <RFC 3339 time with offset | latest>` — adds a restored generation (commit `chore(restore): restore postgres/prod to <time>`), waits up to 30 minutes until it is ready.
- `/resource:restore postgres --env prod --switch --confirm prod` — makes the restored generation active.
- `/resource:restore postgres --env prod --discard --confirm prod` — removes the restored (or previous) generation and its volumes.
- `/system:status` — Backups section: per database and environment the last success, archiving state, state (`ok`, `pending`, `failing`, `unknown`) and the earliest and latest restore points.

The task API's HTTP contract (spec 001) is unchanged.

### Errors

Every refusal of the commands above changes nothing and prints why: a time outside the restore points names `earliest` and `latest`; a prod switch or discard without confirmation says `add --confirm prod`; a second restore while a restored or previous generation exists says to discard it first; a cluster that cannot be read gives state `unknown` with the kubectl reason. Any other non-zero exit is treated as "nothing done": re-read `/system:status` before retrying.

### Timeouts

- Each cluster read of the commands: 60 s (`backupstate.TIMEOUT_S`); on timeout the command stops with `unknown`, nothing is changed.
- A restore waits up to 30 minutes for the restored generation to be ready; on timeout it stays in place for inspection and is removed with `--discard --confirm prod`.
- The API's own timeouts to Postgres (connect 2 s, statement 3 s, spec 001 plan) are unchanged; backups and WAL archiving run beside the database and never block a query, so a slow or unreachable bucket does not change `GET /tasks` or `POST /tasks` (AC-002.13).

## gitops-app PR

One PR (branch `promote/prod-with-backups` or similar) with steps 1 to 5 above; title e.g. `feat(prod): first prod promotion with nightly database backups`. The prod pins (`demo-taskboard-api` `0.1.0`, `demo-taskboard-web` `0.1.1`) are written by `/instance:promote` inside this PR, not by a later pin PR, so `gitops_pin` is empty. Review the rendered `addons/prod/` (Cluster with the Barman plugin, ObjectStore, ScheduledBackup; SeaweedFS and Bucket objects) and the two prod pins.

After merge and Argo reconciling `demo-taskboard-prod`, before anyone uses prod:

1. `/system:status`: prod Postgres archiving `ok`; the ScheduledBackup's immediate first backup completed (or `pending` until it does); no HTTPRoute for the bucket in any environment (AC-002.5).
2. `SHOW archive_timeout` on the prod primary returns `5min` (AC-002.2; read by a human with cluster access).
3. Drill (AC-002.15): `/resource:backup postgres --env prod` until completed; `/resource:restore postgres --env prod --to latest`; check the restored generation is ready and prod's `GET /tasks` is unchanged (AC-002.7); `/resource:restore postgres --env prod --discard --confirm prod`. The backup in the backup list and the two `chore(restore)` commits in this repo are the record with their times.
4. Optional, at the drill: create task "A", wait at least 5 minutes, create "B", restore to a time between them, check the copy holds "A" with identical fields and not "B" (AC-002.6), discard. If done, delete "A" and "B" from prod afterwards so prod starts empty.
5. Next morning: the 03:00 UTC backup is listed with its completion time (AC-002.1).
