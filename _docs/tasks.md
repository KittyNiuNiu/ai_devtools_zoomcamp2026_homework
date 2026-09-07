# Backlog — Shared Household Chores Tool

Implementation backlog for [plan.md](plan.md), built on the design in
[architecture.md](architecture.md). Each task is sized for a single session and written to be
picked up cold — where a task depends on earlier work, the description names what it assumes
already exists. The numbering is a suggested order, not a hard sequence.

---

## 1. Bootstrap the project with Docker Compose and a passing test

Goal: An empty Django project that boots in Docker and runs one green test.

Description: Create the Django project under `config/`, a Dockerfile for the app image, and a `docker-compose.yml` with a `db` service (postgres:16, named volume) and a `web` service running `manage.py runserver`. Wire up pytest and pytest-django with a `.env` file supplying `DATABASE_URL` and `SECRET_KEY`. Done when `docker compose run --rm web pytest` passes a single smoke test asserting the app's health/home view returns 200.

---

## 2. Custom user model and email-based authentication

Goal: People can sign up, log in, and log out using an email address.

Description: Add an `accounts` app with a custom `User` model that uses email as the login identifier and carries a `display_name`, wired via `AUTH_USER_MODEL` before the first migration is created. Install and configure django-allauth for signup, login, logout, and password reset, using the local email backend for now. Tests should cover a signup-then-login round trip.

---

## 3. Base template and front-end shell

Goal: A styled layout that every later page can extend.

Description: Add `templates/base.html` pulling in Bootstrap 5 and HTMX from CDN, with a nav bar showing the logged-in user, flash-message rendering, and `title`/`content` blocks. Configure the CSRF token header that HTMX needs for POST requests, set once on the body element. Ship it with a placeholder home page so there is something to look at.

---

## 4. Household and membership models

Goal: Households exist and users belong to them with a role and a stated capacity.

Description: Add a `households` app with `Household` (name, IANA timezone, `daily_cutoff_time`, `min_capacity`, `last_cutoff_run_date`) and `HouseholdMembership` (household, user, role of admin or member, `daily_capacity`, `is_active`, unique per household-user pair). Add a "create household" form that makes the creator an admin and prompts for their capacity. Tests cover creation and the uniqueness constraint.

---

## 5. Household-scoped routing and permission mixins

Goal: Every household page verifies membership, and admin pages verify role.

Description: Add a `HouseholdAccessMixin` that resolves the requesting user's active membership for the `<household_id>` in the URL, returns 404 when there is none, and attaches the membership to the request. Add an `AdminRequiredMixin` layered on top that requires the admin role. Route all household pages under `/h/<household_id>/` and test that a member of household A gets a 404 on household B's pages.

---

## 6. Invite codes and the join-by-code flow

Goal: A new person can join an existing household with a code.

Description: Add an `InviteCode` model (random URL-safe code, optional expiry, optional max uses, use counter, active flag) plus admin-facing create and revoke actions. Build a `/join` form that validates a submitted code, creates a member-role membership, and increments the counter. Tests should cover the happy path alongside expired, revoked, and fully-used codes.

---

## 7. Task catalog model and CRUD

Goal: Each household has a list of chores with point values and recurrence rules.

Description: Add a `chores` app with a `Task` model holding household, name, description, `point_value`, recurrence (none, daily, or weekly), `weekly_weekday`, and `is_active`. Build list, create, and edit views under the household URL prefix, with validation that weekly tasks specify a weekday. Role enforcement on the point value comes in a later task — treat it as an ordinary field here.

---

## 8. Default chore fixture with common-knowledge point values

Goal: A brand-new household starts with a usable chore list instead of a blank page.

Description: Assemble a seed list of roughly fifteen common household chores with default point values that reflect effort and unpleasantness rather than duration, and sensible recurrence settings. Copy these into a household's own `Task` rows when the household is created, so each household can then edit them independently. Add a test asserting a freshly created household has a non-empty task list.

---

## 9. Audit trail with django-simple-history

Goal: Changes to point values and capacity are recorded with who and when.

Description: Install django-simple-history and register it on the `Task` model (tracking `point_value`) and the `HouseholdMembership` model (tracking `daily_capacity` and `role`). Add the history middleware so `history_user` is populated from the request rather than left null. Tests assert that editing each tracked field creates a history row attributed to the correct actor.

---

## 10. Admin-only point value editing

Goal: Only household admins can change what a chore is worth.

Description: Split the task edit form so `point_value` renders read-only for members and editable for admins, and guard the underlying view with the admin role check so the rule holds even against a hand-crafted POST. On the admin form, show the field's recent change history inline for context. Tests cover a member being rejected and an admin succeeding.

---

## 11. TaskInstance model and status transitions

Goal: Concrete per-day chore occurrences exist with a well-defined lifecycle.

Description: Add a `TaskInstance` model (household, optional task link, name and point-value snapshots, `due_date`, `original_due_date`, status of open/claimed/assigned/done/missed, `responsible_user`, `assignment_source`, claim/assign/complete timestamps, `is_debt`, `days_overdue`) with a unique constraint on task plus due date. Implement the permitted status transitions as model methods that raise on invalid moves. Tests should walk each valid transition plus a few rejected ones.

---

## 12. Background worker and job scheduler

Goal: Scheduled and async jobs run in a separate process.

Description: Install Django-Q2 configured to use the Django ORM as its broker (no Redis), and add a `worker` service to Docker Compose running `manage.py qcluster` against the same image and database. Add a `setup_schedules` management command that registers the project's recurring jobs idempotently, so re-running it never duplicates entries. Include one trivial scheduled job to prove the loop works end to end.

---

## 13. Recurring task instance generation

Goal: Daily and weekly chores automatically produce instances each day.

Description: Write a `generate_recurring_instances` job that, for every active household, resolves the household's local date and creates one `TaskInstance` per active daily task and per weekly task whose weekday matches, snapshotting the name and point value onto the instance. The job must be fully idempotent so it can safely run hourly. Tests use a frozen clock to assert both correct generation and that a second run creates nothing.

---

## 14. Task pool page with first-come-first-served claiming

Goal: Members can see the open pool and claim tasks without race conditions.

Description: Build `/h/<id>/pool/` listing today's open instances with their point values and a Claim button per row. The claim view runs inside a transaction using `select_for_update()` on the instance and returns a conflict response if it is no longer open, which the front end renders as "already taken". Include a concurrency test where two threads claim the same instance and exactly one succeeds.

---

## 15. Marking tasks as done

Goal: Members can complete the tasks they are responsible for.

Description: Add a mark-done action moving an instance from claimed, assigned, or missed into done, stamping `completed_by` and `completed_at` and clearing the debt flag. Only the responsible user or a household admin may complete an instance. Tests cover the permission boundary and the debt flag being cleared on completion.

---

## 16. Fairness ratio calculation

Goal: Every member has a comparable "how loaded am I" number.

Description: Implement a `household_ratios(household)` service returning each active member's cumulative assigned points divided by their stated capacity, with the divisor floored at the household's `min_capacity` to prevent division by zero. The numerator sums every instance the member is responsible for regardless of completion status, and never resets. Cover the arithmetic with table-driven unit tests including the zero-capacity guard.

---

## 17. Automatic assignment engine

Goal: Unclaimed tasks can be distributed to whoever has the most slack.

Description: Implement `assign_open_instances(household, now)`, which locks the household's open instances and greedily assigns each one — highest point value first — to the member whose resulting fairness ratio would be lowest, updating running totals as it goes. Tie-breaking must be deterministic so the same inputs always produce the same allocation. This is pure logic with no HTTP or job machinery, so test it table-driven.

---

## 18. Daily cutoff job

Goal: At each household's cutoff time, unclaimed tasks get assigned automatically.

Description: Implement `run_daily_cutoff(household_id)` which resolves the household's local date, bails out if the cutoff already ran today, auto-assigns the remaining open instances, and stamps `last_cutoff_run_date`. Add a `run_cutoffs_due` job scheduled every fifteen minutes that calls it for each household whose local cutoff time has passed. Tests must confirm running the cutoff twice for one date is a no-op.

---

## 19. Rollover and debt tracking

Goal: Unfinished tasks stay owed rather than quietly disappearing.

Description: Extend the cutoff job so instances due before today that are still not done become `missed`, gain `is_debt = True`, and increment `days_overdue` while keeping their responsible user. Overdue instances that were never claimed get an owner assigned at the same moment, so every debt item has a holder. Tests cover the transition, the repeated-day increment, and the flag clearing when the task is finally completed.

---

## 20. My Tasks page with a debt section

Goal: A member can see what they owe today and what is overdue.

Description: Build `/h/<id>/me/` showing today's claimed and assigned instances, plus a visually distinct debt section listing overdue items with their days-overdue count. Every row carries a mark-done button that updates in place. Write the empty states carefully — "nothing assigned" and "no debt" are the common cases and should read as good news.

---

## 21. Household dashboard

Goal: The whole household can see everyone's load at a glance.

Description: Build `/h/<id>/` as a table of every active member showing cumulative points, stated capacity, current fairness ratio, today's task count, and outstanding debt count, sorted by ratio so imbalance is immediately visible. It is a read-only view sitting on top of the ratio service. Include a short legend explaining what the ratio means, since it is the core mechanic.

---

## 22. Members page with transparent, editable capacity

Goal: Everyone can see everyone's capacity, and edit their own.

Description: Build `/h/<id>/members/` listing every active member alongside their stated daily capacity, visible to the whole household by design. Each person can edit only their own value through an inline form that saves without a full page reload, and every save writes an audit history row. Admins additionally get controls to change another member's role.

---

## 23. Activity log page

Goal: Point-value and capacity changes are visible to the household.

Description: Build `/h/<id>/activity/` rendering a merged, reverse-chronological feed drawn from the task and membership history tables, expressed as readable sentences such as "Alex changed Clean bathroom from 3 to 5 points". Paginate it, since history accumulates indefinitely. This page is what makes the governance model work, so favour clarity over density.

---

## 24. Household settings page

Goal: Admins can configure how their household runs.

Description: Build `/h/<id>/settings/` where an admin can edit the household name, timezone, daily cutoff time, and minimum capacity, and manage the household's invite codes. Non-admins must not be able to reach or submit the page. Changes to timezone or cutoff time should take effect on the next scheduled run without needing a worker restart.

---

## 25. Email delivery and notification logging

Goal: The app can send email locally and you can actually see it.

Description: Add a `mailpit` service to Docker Compose and point Django's SMTP settings at it, so outbound mail lands in a browsable local inbox. Add a `NotificationLog` model recording recipient, kind, related task instance, and sent timestamp, plus a small helper that sends an email and writes the log entry in one call. Include a management command that fires a test email so the pipeline is verifiable on its own.

---

## 26. Unclaimed pool reminder job

Goal: People get nudged about open tasks before the cutoff hits.

Description: Add a daily scheduled job that emails active members of any household whose pool still holds open instances for today, listing those tasks and linking to the pool page. Skip anyone who already received this reminder today according to the notification log. Tests should assert the deduplication holds and that households with an empty pool trigger nothing.

---

## 27. Overdue debt reminder job

Goal: People get reminded of what they still owe.

Description: Add a daily scheduled job that emails each member holding a non-empty debt list, itemising each overdue task with its days-overdue count. Skip members already sent an overdue reminder that day, using the same notification log pattern as the pool reminder. Tests cover the dedup behaviour and confirm members with no debt receive nothing.

---

## 28. New-member fairness seeding

Goal: Someone joining an established household is not buried in auto-assignments.

Description: Add a `baseline_points` field to membership, set when a person joins to approximately the household's median fairness ratio multiplied by their stated capacity, so they start near parity rather than at a ratio of zero. Include this baseline in the ratio numerator. Tests should compare a fresh joiner's first auto-assignment load with and without the seeding to show the effect.

---

## 29. Demo data command and end-to-end walkthrough

Goal: One command produces a realistic household you can click through.

Description: Write a management command that creates a demo household with several members, varied capacities, a week of completed and missed task history, and some outstanding debt, so every page has meaningful content. Use it to walk the full flow by hand — join, claim, cutoff, rollover, complete — and fix whatever reads badly or breaks. This doubles as the manual QA checklist for the MVP.
