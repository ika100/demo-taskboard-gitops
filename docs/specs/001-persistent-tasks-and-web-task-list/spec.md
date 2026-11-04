---
spec_id: 001-persistent-tasks-and-web-task-list
title: Persistent tasks and web task list
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task list

## Problem

The taskboard API keeps tasks in memory, so every restart, redeploy or new image pin in an environment wipes all tasks, and two API instances would each show a different list. The web app is still the starter page: people can't see or create tasks without calling the API by hand. Together this means the system can't be used for real work yet. This spec makes tasks survive restarts by keeping them in the system's Postgres database, and turns the web app into the place where people see and add tasks.

## Stories

As a taskboard user, I want the tasks I created to still be there after the system is redeployed or restarted, so that I can rely on the board as my record of what to do.
**Repos:** demo-taskboard-api, demo-taskboard-gitops

As a taskboard user, I want to open the web app and see every task with its due date, so that I know what is to be done without using the API.
**Repos:** demo-taskboard-web, demo-taskboard-api

As a taskboard user, I want to add a task with a title and an optional due date from the web app, so that I can record work without calling the API by hand.
**Repos:** demo-taskboard-web, demo-taskboard-api

## Acceptance criteria

Persistence (the task API keeps the request and response format and the validation rules of its spec 001-create-and-list-tasks, AC-001.1 to AC-001.10; that spec's AC-001.11 "a restart empties the list" and its non-goals "persistence beyond the process lifetime" and "sharing tasks across replicas" are superseded by this spec and withdrawn in the API's slice):

- **AC-001.1** Given tasks were created through the task API, when the API is restarted or redeployed with a new image, then `GET /tasks` returns the same tasks, each with the same `id`, `title`, `due_date` and `created_at` as when it was created, oldest first.
- **AC-001.2** Given two running instances of the task API in the same environment, when a task is created through one instance, then `GET /tasks` on the other instance contains it.
- **AC-001.3** Given the database cannot be reached, when a client sends `POST /tasks` or `GET /tasks`, then the response is `503` with a JSON error body, no task is created, and the list is never answered with `[]` in place of the stored tasks.
- **AC-001.4** Given the database cannot be reached, when the API's readiness endpoint is called, then it does not answer `200`; once the database is reachable again it answers `200` without the API being restarted.
- **AC-001.5** Given tasks exist in one environment (for example dev), when the task list is read in another environment (for example staging), then it does not contain them: each environment has its own tasks.

Web task list:

- **AC-001.6** Given no tasks exist, when a person opens the web app's root URL, then the page shows the heading "Tasks", the message "No tasks yet." and the form to add a task.
- **AC-001.7** Given tasks "A" (due 2026-12-01) and then "B" (no due date) were created, when a person opens the web app's root URL, then the page lists "A" before "B" and shows "Due 2026-12-01" for "A" and "No due date" for "B" (the due date exactly as stored, `YYYY-MM-DD`, without time zone conversion).
- **AC-001.8** Given a task was created directly through the task API, when a person opens or reloads the web app's root URL, then the task appears in the list.

Web create form:

- **AC-001.9** Given the task list page, when a person enters the title "Buy milk", leaves the due date empty and submits the form, then "Buy milk" appears at the end of the list with "No due date" without leaving the page, the form is cleared, and `GET /tasks` on the API contains the task.
- **AC-001.10** Given the task list page, when a person enters a title, picks the due date 2026-12-01 and submits, then the new task appears at the end of the list with "Due 2026-12-01".
- **AC-001.11** Given the task list page, when a person submits the form with an empty or whitespace-only title, then no task is created, the message "Enter a title." is shown next to the title field and the entered due date is kept.
- **AC-001.12** Given the task list page, when a person types a title, then the title field accepts at most 200 characters, and a task with a title of exactly 200 characters is created.
- **AC-001.13** Given the task API or the database is unavailable, when a person opens the web app's root URL, then the page loads and shows "Tasks could not be loaded. Please try again later." in place of the list, neither an empty list nor a server error page.
- **AC-001.14** Given the task API or the database is unavailable, when a person submits a valid task, then the message "The task could not be saved. Please try again." is shown, the entered title and due date stay in the form, and no task is added to the list.
- **AC-001.15** Given the form was submitted, when the person clicks the submit button again before the first request has finished, then exactly one task is created.

## Non-goals

- Editing, completing or deleting tasks, in the API or in the web app.
- Users, sign-in and per-user lists: the web app is open to everyone who can reach it, in every environment it runs in, and everyone sees and creates every task. Sign-in is a later spec.
- Filtering, sorting options, search and pagination of the list.
- A cap on the number of tasks kept or shown: the web app shows every task. Pagination is a later spec.
- Fields other than title and due date.
- Moving tasks that exist in memory at the time of the switch into the database: they are dropped once, on the first deploy of this feature.
- Copying tasks between environments.
- Database backups and restore: a separate spec, due before the first prod promotion.
- Live updates of the list when someone else adds a task; a reload shows them.
- Changing whether the task API itself is reachable from outside the system.

## Open questions

- ~~Confirm: this spec supersedes AC-001.11 of the API's spec 001-create-and-list-tasks and its non-goals "persistence beyond the process lifetime" and "sharing tasks across replicas"?~~ Answered: yes, the API's slice withdraws them; see the note above AC-001.1, AC-001.1, AC-001.2.
- ~~Confirm: when the database is unreachable the API answers `503` and reports itself not ready?~~ Answered: yes; see AC-001.3, AC-001.4.
- ~~Confirm: each environment has its own database and its own tasks?~~ Answered: yes; see AC-001.5.
- ~~Confirm: the task list replaces the starter page at the web app's root URL?~~ Answered: yes; see AC-001.6 to AC-001.14.
- ~~Confirm the wording users see?~~ Answered: as written; see AC-001.6, AC-001.7, AC-001.11, AC-001.13, AC-001.14.
- ~~Should due dates be shown as stored or in a friendlier form?~~ Answered: as stored, `YYYY-MM-DD`, no time zone conversion; see AC-001.7, AC-001.10.
- ~~Confirm: the web app is open to anyone who can reach it, without sign-in?~~ Answered: yes, in every environment for now; sign-in is a later spec; see Non-goals.
- ~~Confirm: in-memory tasks are dropped, not migrated?~~ Answered: yes; see Non-goals.
- ~~Should the database be backed up as part of this spec?~~ Answered: no, a separate spec before the first prod promotion; see Non-goals.
- ~~Should the list shown in the web app be capped?~~ Answered: no cap; pagination is a later spec; see Non-goals.

## Changelog

- 2026-11-04 created
- 2026-11-04 amended: removed the (smoke) marker from AC-001.6 (a gitops-app spec has no smoke tests; the component slices choose their own); behaviour unchanged.
- 2026-11-04 amended: all 10 open questions answered with the suggested answers and struck through. Clarified the superseded API spec 001 items above AC-001.1 and the stored date format in AC-001.7 (same behaviour); non-goals now state no sign-in for now, no cap on the list, in-memory tasks dropped and backups in a separate spec before the first prod promotion. No criterion added or withdrawn.
