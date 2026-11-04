---
spec_id: 002-nightly-prod-database-backups-to-an-in
title: Nightly prod database backups to an in-cluster bucket
status: draft
priority: P1
shape: gitops-app
---

# 002 — Nightly prod database backups to an in-cluster bucket

## Problem

Since spec 001 the taskboard keeps every task in the environment's Postgres database, so the database is now the only record of users' work. In prod nothing copies that data anywhere: a bad migration, an accidental delete or a broken database volume would lose all tasks for good, and there is no way to go back to yesterday's state. Spec 001 therefore withheld the first prod promotion until prod has backups. This spec gives the prod database a nightly backup into a bucket inside the cluster, with continuous capture of changes in between, and a tested way to restore from those backups, so that the API and the web app can be promoted to prod.

## Stories

As the operator of the taskboard, I want the prod database backed up every night without anyone doing it by hand, so that I can always go back to a recent state of the tasks.
**Repos:** demo-taskboard-gitops

As the operator, I want to restore the prod database to a chosen point in time, check the result and only then make the system use it, so that a restore never makes things worse than they already are.
**Repos:** demo-taskboard-gitops

As the operator, I want to see when the last backup succeeded and to notice when one fails, so that I do not discover missing backups on the day I need one.
**Repos:** demo-taskboard-gitops

As a taskboard user, I want my tasks to come back after a data loss in prod, so that the board stays my reliable record of what to do.
**Repos:** demo-taskboard-gitops (the API and the web app are unchanged: the API keeps reading its database connection settings as today)

## Acceptance criteria

Backups:

- **AC-002.1** Given the prod database is running, when 03:00 cluster time passes, then a full backup of the prod database is completed and stored in a bucket inside the cluster, and the list of prod backups shows it with its completion time.
- **AC-002.2** Given the prod database is running, when tasks are created between two nightly backups, then those changes are also kept in the bucket continuously, so that every change committed at least 5 minutes ago can be restored (see AC-002.6), not only the state of the last nightly backup.
- **AC-002.3** Given the operator asks for an on-demand backup of the prod database, when it completes, then it appears in the list of prod backups next to the nightly ones, and the running system keeps serving tasks during the backup.
- **AC-002.4** Given backups exist, when a backup is older than 30 days, then it is removed from the bucket, and every point in time within the last 30 days stays restorable.
- **AC-002.5** Given the backup bucket, when anyone tries to reach it through the system's public entry point (the Gateway) in any environment, then it is not reachable: the bucket is only reachable from inside the cluster.

Restore:

- **AC-002.6** Given prod backups exist and tasks "A" and then "B" were created, when the operator restores the prod database to a time after "A" was created and before "B" was created (at least 5 minutes in the past), then the restored database contains "A" with the same `id`, `title`, `due_date` and `created_at`, and does not contain "B".
- **AC-002.7** Given a restore of the prod database was started, when it has finished and the operator has not yet switched to it, then the running system still uses the current prod database: `GET /tasks` in prod returns the same tasks as before the restore, and new tasks can still be created.
- **AC-002.8** Given a finished restore, when the operator switches prod to it, then `GET /tasks` in prod returns the tasks of the restored point in time, without a new image or a change in the API or the web app, and after the switch nightly backups continue for the database now in use.
- **AC-002.9** Given a finished restore, when the operator discards it, then the restored copy is removed and prod keeps using its current database with its tasks unchanged.
- **AC-002.10** Given a restore or a switch in prod is requested, when the operator has not explicitly confirmed that it targets prod, then nothing is restored or switched and the operator is told that prod needs explicit confirmation.
- **AC-002.11** Given a restore to a point in time older than the oldest kept backup (more than 30 days ago) or in the future is requested, when it is started, then it is refused with a message naming the earliest and latest restorable times, and prod is unchanged.

Visibility and failures:

- **AC-002.12** Given the operator looks at the status overview of the system, when prod has backups configured, then it shows for the prod database the completion time of the last successful backup, and marks it as failing when the most recent scheduled backup did not succeed or no backup succeeded in the last 26 hours.
- **AC-002.13** Given the backup bucket cannot be reached, when a backup is due, then the backup is reported as failed (AC-002.12), and the prod system keeps answering `GET /tasks` and `POST /tasks` as before.

Before the first prod promotion:

- ~~**AC-002.14**~~ withdrawn 2026-11-04: prod's database only exists from the API's first prod promotion, so the drill cannot happen before it; replaced by AC-002.15.
- **AC-002.15** Given demo-taskboard-api is promoted to prod for the first time, when that promotion is merged, then the same change also turns on the nightly prod backups, and right after it, before prod is used (no task has been created in prod except by the drill), a completed on-demand prod backup and a restore of it into a separate copy that was then discarded are on record with their times, and prod's database is unchanged by the drill.

## Non-goals

- Protection against losing the whole cluster: the bucket lives in the same cluster as the database, so a lost cluster loses the backups too. Copies outside the cluster are a later spec.
- Backups of the dev and staging databases.
- Backups of anything other than the prod database (the bucket's own contents, cluster configuration, components without data).
- Changes to the task API or the web app, including any maintenance or "restore in progress" page.
- Restoring single tasks or a single table: a restore always covers the whole database.
- Copying prod data into dev or staging.
- Automatic restores or automatic switching: every restore and switch is started by the operator.
- Encryption of backups beyond what the in-cluster bucket provides, and access rules for the bucket beyond AC-002.5.
- Regular restore drills after the first one (AC-002.15); a recurring drill can be a later spec.
- Keeping the backup bucket's credentials away from the task API: the platform requires a prod component to use the bucket, so demo-taskboard-api receives them; a platform change to back up without that is a later spec.

## Open questions

- ~~At what time should the nightly backup run?~~ Answered: 03:00 cluster time; see AC-002.1, AC-002.12.
- ~~How long should backups be kept, and so how far back can prod be restored?~~ Answered: 30 days; see AC-002.4, AC-002.11.
- ~~Should dev and staging be backed up too?~~ Answered: no, prod only; see Non-goals.
- ~~Is a restore drill in prod (one on-demand backup, a restore into a separate copy, then discard) required before the first prod promotion?~~ Answered: yes; see AC-002.15 (was AC-002.14).

## Changelog

- 2026-11-04 created
- 2026-11-04 amended: 4 open questions answered (03:00, 30 days, prod only, drill before first prod promotion); no criterion added or withdrawn.
- 2026-11-04 amended: AC-002.14 drill moves to right after the first prod promotion (prod's database only exists from then); the API receives the backup bucket's credentials (platform constraint), noted as accepted risk. AC-002.14 withdrawn and replaced by AC-002.15 (behaviour changed, so a new id per the amending rules).
