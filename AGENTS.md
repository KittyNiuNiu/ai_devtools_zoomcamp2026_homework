# AGENTS.md

Django app for shared household chores. Scope in `_docs/plan.md`, design in
`_docs/architecture.md`, backlog in `_docs/tasks.md` (task N = GitHub issue #N).

## Documents

- `_docs/process.md` - how work is organized

## Commands

- `docker compose up -d db` - Postgres; must be running before tests
- `uv sync` - install dependencies
- `uv run pytest` - the whole suite
- `uv run pytest apps/chores/tests/test_assignment.py` - one test file
- `docker compose up` - full stack; app on :8000, mail inbox on :8025

## Rules

- Dependencies are added in `pyproject.toml` via `uv add`. Do not add one without
  asking.
- Never edit a committed migration. Add a new one.
- Use model `.save()`, not queryset `.update()` — `.update()` skips signals and silently
  loses the django-simple-history audit trail that the whole governance model depends on.
- Datetimes are stored in UTC. Derive "today" from the household's timezone, never the
  server's.
