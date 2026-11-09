---
spec_id: 001-persistent-tasks-in-postgres-with-web-ui
title: Persistent tasks in Postgres with web UI
status: done
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with web UI

## Problem

The taskboard7 Task API can create and list tasks, but keeps them in memory: every restart or redeploy empties the board, and two instances of the API would each show a different list. People also have no way to use the board except by calling the API by hand, because the web front end has no task features yet. Keeping tasks in a Postgres database and giving the web app a task page turns the board into something people can actually rely on and use.

## Stories

As a person using the taskboard, I want the tasks I created to still be there after the system restarts or is redeployed, so that I can rely on the board.
**Repos:** taskboard7-api, taskboard7-gitops

As a person using the taskboard, I want to open the web app and see all tasks with their due dates, so that I know what is on the board without calling the API.
**Repos:** taskboard7-web, taskboard7-api

As a person using the taskboard, I want to add a task with a title and an optional due date from the web app, so that I can record work without calling the API.
**Repos:** taskboard7-web, taskboard7-api

## Acceptance criteria

- **AC-001.1** Given tasks A, B and C were created through the Task API, when the Task API is restarted or redeployed and a client then sends `GET /tasks`, then the response is `200` with exactly A, B and C, oldest first, each with the same `id`, `title` and `due_date` as returned on creation.
- **AC-001.2** Given the Task API runs with two instances, when a task is created through one instance, then a `GET /tasks` answered by the other instance contains that task.
- **AC-001.3** Given the database, when clients use `POST /tasks` and `GET /tasks`, then requests, response fields, status codes and validation errors are the same as in the Task API's component spec 001 (create and list tasks): its criteria AC-001.1 to AC-001.7 still hold, except that tasks are no longer lost on restart.
- **AC-001.4** Given the database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response is `503` with a JSON error body and no task is created; the Task API's `/ready` answers not ready while the database is unreachable; and once the database is reachable again, both requests succeed without restarting the Task API.
- **AC-001.5** Given tasks exist, when a person opens the web app's root page, then it shows every task, oldest first, each with its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none; given no task exists, the page shows "No tasks yet".
- **AC-001.6** Given the web app's root page, when a person enters a title and optionally picks a due date and submits the form, then the new task appears at the end of the list without a manual page reload, the form is cleared, and a following `GET /tasks` on the Task API contains the task with that title and due date.
- **AC-001.7** Given the create form, when a person submits a title that is empty, only whitespace or longer than 200 characters, then the form shows "Enter a title" or "Title can be at most 200 characters" respectively next to the title field, keeps the entered values, and no task is created.
- **AC-001.8** Given the Task API cannot be reached or answers with an error, when a person opens the root page, then it shows "Tasks could not be loaded. Try again later." instead of the list; and when a person submits the create form, then it shows "The task could not be saved. Try again." and keeps the entered values.

## Non-goals

- Updating, completing or deleting tasks, in the API or in the web app.
- Filtering, sorting options, paging or search.
- Users, login, ownership or per-person task lists: while the system runs in dev only, everyone who can reach the web app or the Task API may see and create every task; login is a later spec.
- New task fields (description, status, priority, timestamps).
- Carrying over tasks that exist only in the memory of the currently running Task API.
- Database backups and restore, and running the system in staging or prod (the components stay in dev only).

## Assumptions

- **A1** Tasks are stored in one Postgres database declared as a resource of the taskboard7 system and used only by the Task API; the web app never talks to the database directly, only to the Task API (other option: the web app reads the database itself; affects AC-001.1 to AC-001.8).
- **A2** The Task API contract stays exactly as in its component spec 001, with no version prefix and no new fields (other option: a versioned `/v1/tasks`; affects AC-001.3).
- **A3** Tasks that exist only in memory when the change is deployed are not carried over; the persistent board starts empty (in-memory tasks are lost on every restart anyway) (other option: copy them into the database during the switch; affects AC-001.1).
- **A4** When the database is unreachable, the Task API answers `503` with a JSON error and reports not ready, and recovers by itself (other option: `500`, or staying ready; affects AC-001.4).
- **A5** The task list lives on the web app's root page together with the create form, oldest first, the same order as the API (other option: newest first, or a separate "new task" page; affects AC-001.5, AC-001.6).
- **A6** The web form checks the title with the same rules as the Task API (1 to 200 characters, at least one non-whitespace character) before sending, and the due date is chosen with a date picker, so no invalid date can be entered (other option: rely on the API's `422` only; affects AC-001.7).
- **A7** The page wording is English: "No tasks yet", "No due date", "Enter a title", "Title can be at most 200 characters", "Tasks could not be loaded. Try again later.", "The task could not be saved. Try again." (other option: other wording or German; affects AC-001.5, AC-001.7, AC-001.8).
- **A8** The list is loaded when the page opens and after each create from this page; tasks created elsewhere appear after a reload, with no live updates (other option: periodic refresh; affects AC-001.5, AC-001.6).

## Open questions

- ~~Without any login, everyone who can reach the web app (and the Task API, which stays exposed) can see and create every task. Is that acceptable for now?~~ Answered 2026-11-09: yes, open access while the system runs in dev only; login is a later spec, see Non-goals.

## Changelog

- 2026-11-09 created
- 2026-11-09 amended: open access without login confirmed for dev only, recorded as a non-goal; no criteria changed
- 2026-11-09 approved
- 2026-11-09 building
- 2026-11-09 done
