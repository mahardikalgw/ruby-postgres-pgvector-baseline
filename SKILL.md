---
name: postgres-pgvector-baseline
description: Run a Rails 8 app on PostgreSQL 16 with pgvector in every environment — one schema dump per environment, the Solid Cache/Queue/Cable stores on the same host, CI and Kamal parity, and the local traps that break fresh clones. Use when setting up a database, adding an extension, wiring Solid stores, or debugging "works on my machine" database failures.
version: 2.0.0
---

# PostgreSQL + pgvector baseline

One database engine in **development, test, CI and production**. Not SQLite locally and Postgres in
production: every difference between them is a bug that only appears after deploy.

## When to activate

- Creating a project's database setup.
- Adding an extension (pgvector, pg_trgm, unaccent).
- Wiring Solid Cache / Queue / Cable.
- A CI job cannot reach the database, or a fresh clone cannot boot.
- `db/schema.rb` versus `db/postgres_schema.rb` confusion.

## 1. One engine, four databases in production

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= ENV.fetch("PGHOST", "localhost") %>
  port: <%= ENV.fetch("PGPORT", 5432) %>
  username: <%= ENV.fetch("PGUSER", ENV.fetch("USER", "postgres")) %>
  password: <%= ENV["PGPASSWORD"] %>

development:
  <<: *default
  database: <%= ENV.fetch("POSTGRES_DB", "app_development") %>
  schema_dump: postgres_schema.rb

test:
  <<: *default
  database: <%= ENV.fetch("POSTGRES_TEST_DB", "app_test") %>
  schema_dump: postgres_schema.rb

production:
  primary:
    <<: *default
    database: <%= ENV.fetch("POSTGRES_DB", "app_production") %>
    schema_dump: postgres_schema.rb
  cache:
    <<: *default
    database: <%= ENV.fetch("POSTGRES_CACHE_DB", "app_production_cache") %>
    migrations_paths: db/cache_migrate
  queue:
    <<: *default
    database: <%= ENV.fetch("POSTGRES_QUEUE_DB", "app_production_queue") %>
    migrations_paths: db/queue_migrate
  cable:
    <<: *default
    database: <%= ENV.fetch("POSTGRES_CABLE_DB", "app_production_cable") %>
    migrations_paths: db/cable_migrate
```

Four databases on one server: the primary plus three Solid stores. They run as a Kamal accessory, so
there is no managed-database dependency and the whole product is reproducible from a compose file.

## 2. Schema files

| File | Owner |
|---|---|
| `db/postgres_schema.rb` | the application (one dump, every environment) |
| `db/cache_schema.rb` | Solid Cache |
| `db/queue_schema.rb` | Solid Queue |
| `db/cable_schema.rb` | Solid Cable |

## 3. pgvector

```ruby
class EnableVector < ActiveRecord::Migration[8.0]
  def change
    enable_extension "vector"
  end
end
```

```ruby
create_table :knowledge_entries do |t|
  t.vector :embedding, limit: 1536        # match your provider's dimensions
  # ...
end

add_index :knowledge_entries, :embedding, using: :hnsw, opclass: :vector_cosine_ops
```

- Install the extension for the server version: `brew install pgvector` on macOS, or use the
  `pgvector/pgvector:pg16` image everywhere else.
- The dimensions in the column must match the embedding model. Changing models is a migration, not a
  config change.
- `nearest_neighbors(:embedding, vector, distance: "cosine", threshold: 0.5)` is the query interface.
- An install without embeddings must still work: gate on "does this workspace have embedded rows" and fall
  back to a lexical path.

## 4. The traps that break fresh clones

| Trap | Symptom | Fix |
|---|---|---|
| `npm install` skipped | `bin/rails test` fails with a Vite manifest error | install Node deps before tests, always |
| pgvector missing on the server | `extension "vector" is not available` during `db:prepare` | install for the right major version, or use the pgvector image |
| `PGUSER` defaulted to a nonexistent role | `role "…" does not exist` | export `PGHOST`/`PGUSER`/`PGPASSWORD` in CI |
| OAuth port drift | redirect URI mismatch on sign-in | pin the dev port in one place, register the same URI in the provider |
| Seeds replanted in test | fixture FK validation fails on the next run | `db:test:prepare` at the end of the seed step |
| System test without Chrome | `Selenium::WebDriver::Error` | install Chrome (or chromium) in CI |
| `libvips` missing | Active Storage variant errors | install `libvips` in the Dockerfile and CI |

## 5. Local development, one command

```bash
brew services start postgresql@16
bin/setup                # gems, create + migrate, node deps
bin/rails db:seed
bin/dev                  # Rails + Vite together
```

`bin/setup` must be idempotent: run it twice and nothing breaks. A new contributor's first hour is the
only time they will tolerate a broken setup, and every hour after that is spent working around it.

### The OAuth port pin

If sign-in redirects to `localhost:3100/session/.../callback`, the port is part of the credential. Pin it
in one place the whole toolchain obeys — a `bin/dev` that exports `PORT` **and** a `Procfile.dev` that
passes it explicitly, because the process manager injects its own:

```
# Procfile.dev
vite: bin/vite dev
web: bin/rails s -p 3100
```

Two places is one too many, but a process manager and a shell export are the two that actually exist. A
comment on both lines stating why the number is repeated is the cheapest fix.

## 6. CI parity

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
steps:
  - run: sudo apt-get update && sudo apt-get install --no-install-recommends -y libvips
  - run: bin/rails db:test:prepare test
    env:
      RAILS_ENV: test
      PGHOST: localhost
      PGUSER: postgres
      PGPASSWORD: postgres
```

The CI database image is the same major version as production. A green build on a different Postgres is a
green build for a database you do not deploy.

## References

- [references/databases.md](references/databases.md) — extensions, Solid store wiring, migrations, deployment shape

