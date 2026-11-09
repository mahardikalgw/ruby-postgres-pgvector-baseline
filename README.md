# PostgreSQL + pgvector baseline

Run a Rails 8 app on PostgreSQL 16 with pgvector in every environment — one schema dump per environment, the Solid Cache/Queue/Cable stores on the same host, CI and Kamal parity, and the local traps that break fresh clones. Use when setting up a database, adding an extension, wiring Solid stores, or debugging "works on my machine" database failures.

One skill, one repository. Part of a twelve-skill pack extracted from a production Rails 8 +
Inertia + React helpdesk monolith — the opinionated layer no public Rails or shadcn skill can know.
Version `2.0.0` tracks the source constitution.

## Install

```jsonc
// skills-lock.json
{
  "version": 1,
  "skills": {
    "postgres-pgvector-baseline": {
      "source": "mahardikalgw/postgres-pgvector-baseline",
      "sourceType": "github",
      "skillPath": "SKILL.md"
    }
  }
}
```

Or clone and symlink:

```bash
git clone git@github.com:mahardikalgw/postgres-pgvector-baseline.git
ln -s "$PWD/postgres-pgvector-baseline" .claude/skills/postgres-pgvector-baseline
```

## When to activate

- Creating a project's database setup.
- Adding an extension (pgvector, pg_trgm, unaccent).
- Wiring Solid Cache / Queue / Cable.
- A CI job cannot reach the database, or a fresh clone cannot boot.
- `db/schema.rb` versus `db/postgres_schema.rb` confusion.

## What's inside

```
postgres-pgvector-baseline/
  SKILL.md
  references/databases.md
```

`SKILL.md` is a router (kept under 250 lines) — the depth lives in `references/`, and the
copy-in code lives in `assets/`.

## Validate

```bash
./bin/check-skill
```

Checks frontmatter, the activation section, link resolution, orphaned references, script
executability, leftover placeholders, and — when the source app is checked out next to this
repository — that vendored scripts still match it byte for byte. Runs in CI on every push.

## Related skills

- [`merge-gate-parity`](https://github.com/mahardikalgw/merge-gate-parity) — one gate list in every file, a script contract, and local/CI triage
- [`rag-knowledge-loop`](https://github.com/mahardikalgw/rag-knowledge-loop) — approved corpus, fused retrieval, reply guardrails, offline test doubles

## Licence

MIT — see [LICENSE](LICENSE).
