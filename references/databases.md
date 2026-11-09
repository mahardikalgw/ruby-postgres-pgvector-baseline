# Databases: extensions, Solid stores, deployment

## Extension checklist

```ruby
enable_extension "pgcrypto"     # gen_random_uuid for primary keys that must not leak volume
enable_extension "pg_trgm"      # ILIKE '%term%' at scale; add a gin_trgm_ops index
enable_extension "vector"       # pgvector
```

- `pg_trgm` with `add_index :tickets, :title, using: :gin, opclass: :gin_trgm_ops` turns a full scan into
  an index scan. Without it, a search box is fine until the table has a million rows.
- `vector` requires the extension to be installable on the server, not just requested by a migration. The
  failure surfaces at `db:prepare`, which is the worst moment.
- Every extension is a change to the schema dump. Commit the dump in the same commit as the migration.

## Indexes that pay for themselves

| Index | Query it serves |
|---|---|
| `[workspace_id, status, last_activity_at]` | a filtered queue list, newest first |
| `[workspace_id, requester_id]` | "my tickets" |
| `[ticket_id, created_at]` | the timeline on a detail page |
| `[user_id, read_at]` | unread count for the bell |
| `gin_trgm_ops` on the searchable column | the search box |
| `hnsw vector_cosine_ops` on the embedding | retrieval |

A composite index is only useful in the order its columns are filtered. `[workspace_id, status]` does not
serve a query filtered by status alone.

## Solid Cache / Queue / Cable

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }
```

```yaml
# config/cable.yml
production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

- Each store owns its schema file and its migrations path; `db/cache_migrate`, `db/queue_migrate`,
  `db/cable_migrate`.
- In development use `:memory_store` / `:async` adapter, so a fresh clone does not need three extra
  databases to boot. In test use `:test` delivery and the inline or test adapter.
- Copy the store schemas from the gem's install task once, then commit them. A store upgrade is a
  migration, not a reinstall.

## Migrations on pgvector

```ruby
class CreateKnowledgeEntries < ActiveRecord::Migration[8.0]
  def change
    create_table :knowledge_entries do |t|
      t.references :workspace, null: false, foreign_key: true
      t.string :source_ref, null: false
      t.text :question, null: false
      t.text :answer, null: false
      t.vector :embedding, limit: 1536
      t.datetime :approved_at
      t.timestamps
    end

    add_index :knowledge_entries, [ :workspace_id, :source_ref ], unique: true
    add_index :knowledge_entries, :embedding, using: :hnsw, opclass: :vector_cosine_ops
  end
end
```

`t.vector` is provided by the `neighbor` gem (or a hand-written column type). Either way the column is a
float array with a fixed dimension; a dimension mismatch fails at insert, which is the right place.

## Deployment shape (Kamal)

```yaml
# config/deploy.yml
accessories:
  db:
    image: pgvector/pgvector:pg16
    host: <%= ENV["DB_HOST"] %>
    port: 5432
    env:
      clear:
        POSTGRES_USER: app
        POSTGRES_DB: app_production
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data
```

The database is an accessory: the same image as CI, the volume on the host, the password in the secret
store. `bin/docker-entrypoint` runs `db:prepare` on boot, which is what creates the three Solid databases
on the first deploy.

## Verification queries

```sql
-- which extensions are actually installed
SELECT extname, extversion FROM pg_extension;

-- is the vector index used, or is it a sequential scan?
EXPLAIN ANALYZE
SELECT * FROM knowledge_entries
ORDER BY embedding <=> '[0.1,0.2,...]' LIMIT 6;

-- indexes on a hot table
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'tickets';
```

Run these before blaming the application. A missing index and a wrong query plan look identical from the
Rails log.
