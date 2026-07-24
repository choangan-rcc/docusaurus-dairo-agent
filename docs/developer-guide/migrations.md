---
sidebar_position: 4
title: Database Migrations
---

# Database Migrations

Migrations are Alembic (async) and run against development → staging →
production in sequence. A migration you write today runs on the **real production
database** within days — treat this page as safety-critical.

## One head, always

Each migration points at a parent (`down_revision`), forming a chain whose last
link is the **head**. There must be **exactly one head at all times**. The CI job
`alembic-heads` fails any PR where `alembic heads` returns more than one.

The trap: two developers each add a migration on parallel branches. Git merges
both files with no conflict (different filenames), yet the database now has two
heads and `alembic upgrade head` refuses to run.

## Prefer rebase; merge only shared history

Before opening a PR that adds a migration, rebase onto the latest `dev`, confirm
a single head, **then** generate the migration so it chains onto the newest head:

```bash
git fetch origin && git rebase origin/dev
uv run alembic heads          # expect exactly one
make migration m="add x"      # chains onto current head
```

If you created the migration before rebasing and it now diverges, delete it,
rebase, and regenerate — do not leave two heads.

If two migrations **already coexist on `dev`/`main`** (pushed, possibly already
run in an environment), do **not** rewrite history. Stitch them with a merge
revision:

```bash
uv run alembic heads          # shows 2 heads
make migrate-merge            # alembic merge heads -m "merge heads"
uv run alembic heads          # back to 1
git add migrations/versions/ && git commit -m "chore(db): merge alembic heads"
```

**Golden rule: unshared history → rebase; shared history → merge revision.**

## One migration per PR

Exactly one migration per PR — easier to review, revert, and reason about
ordering.

## Never mutate an applied migration

Once a migration has merged to `dev`/`main` (and run anywhere), its file content
and revision id are frozen. To change schema, add a **new** migration. Editing an
applied migration desyncs code from the database.

## Backward-compatible changes only

During a rollout, the old image and the new migration briefly coexist (and a Helm
rollback does **not** reverse migrations), so migrations must be additive:

- New columns are `NULL`able or carry a `server_default`. Never add a `NOT NULL`
  column without a default to a populated table.
- Renames and drops use **expand–contract** across multiple releases:
  1. **Expand** — add the new column; code writes to both old and new.
  2. **Migrate** — backfill data; switch code to the new column.
  3. **Contract** — once nothing reads the old column, a later migration drops it.
- Keep seed/data changes out of schema migrations — use `make seed`. If a
  backfill must live in a migration, make it **idempotent**.

## Migration PR checklist

- [ ] `uv run alembic heads` → exactly one head
- [ ] `make migrate` runs clean on an empty local DB
- [ ] Change is additive, or follows expand–contract
- [ ] No previously-merged migration was edited
- [ ] A sensible `downgrade()` exists (or a note explains why not)
- [ ] Exactly one migration in this PR
