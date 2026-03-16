# Session Storage: JSONL to PostgreSQL Migration Plan

## Current State

OpenClaw stores conversation history in JSONL files at:

- `~/.openclaw/agents/<agentId>/sessions/sessions.json` — index mapping session keys to session IDs
- `~/.openclaw/agents/<agentId>/sessions/<session-id>.jsonl` — per-session transcript

### JSONL Schema (per line)

```json
{
  "type": "session" | "message",
  "timestamp": "ISO8601",
  "message": {
    "role": "user" | "assistant" | "toolResult",
    "content": [{ "type": "text" | "thinking" | "toolCall", ... }],
    "usage": { "cost": { "total": number } }
  }
}
```

### Limitations of JSONL

1. **No indexing** — grep/jq required for all searches; O(n) across all files
2. **No concurrent writes** — file locking issues with concurrent sessions
3. **No relational queries** — cannot efficiently join sessions, messages, costs
4. **Fragmented metadata** — sessions.json index separated from transcript data
5. **No built-in pagination** — must read entire files for range queries
6. **Backup complexity** — entire directory must be copied; no incremental
7. **Schema evolution** — adding fields requires migration scripts

---

## PostgreSQL Solution

### Database Schema

```sql
-- Core tables
CREATE TABLE agents (
  id TEXT PRIMARY KEY,          -- agent ID from system prompt
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  metadata JSONB
);

CREATE TABLE sessions (
  id UUID PRIMARY KEY,
  agent_id TEXT NOT NULL REFERENCES agents(id),
  session_key TEXT NOT NULL,   -- provider:account:thread (e.g., "telegram:123:456")
  title TEXT,
  started_at TIMESTAMPTZ NOT NULL,
  ended_at TIMESTAMPTZ,
  deleted_at TIMESTAMPTZ,      -- soft delete timestamp
  metadata JSONB,
  UNIQUE(agent_id, session_key)
);

CREATE TABLE messages (
  id UUID PRIMARY KEY,
  session_id UUID NOT NULL REFERENCES sessions(id),
  type TEXT NOT NULL,           -- 'session' (metadata) or 'message'
  role TEXT NOT NULL,          -- 'user', 'assistant', 'toolResult'
  content JSONB NOT NULL,      -- [{ type, text, thinking, toolCall, ... }]
  usage JSONB,                 -- { cost: { total }, tokens: {...} }
  created_at TIMESTAMPTZ DEFAULT NOW(),
  line_number INTEGER          -- position in original JSONL (for reconstruction)
);

-- Indexes for common queries
CREATE INDEX idx_messages_session ON messages(session_id);
CREATE INDEX idx_messages_role ON messages(role);
CREATE INDEX idx_messages_created ON messages(created_at);
CREATE INDEX idx_messages_content_fts ON messages USING gin(to_tsvector('english', content::text));
CREATE INDEX idx_sessions_agent ON sessions(agent_id);
CREATE INDEX idx_sessions_started ON sessions(started_at);
CREATE INDEX idx_sessions_deleted ON sessions(deleted_at) WHERE deleted_at IS NULL;
```

### Why This Schema Beats JSONL

| Capability         | JSONL                                               | PostgreSQL                                       |
| ------------------ | --------------------------------------------------- | ------------------------------------------------ |
| Search by date     | `jq` + file iteration                               | `WHERE started_at BETWEEN ...`                   |
| Search by role     | `jq 'select(.message.role=="user")'`                | `WHERE role = 'user'`                            |
| Full-text search   | `rg` across files                                   | `to_tsvector` + `ts_rank`                        |
| Cost aggregation   | `jq -s '[.[] \| .message.usage.cost.total] \| add'` | `SELECT SUM((usage->'cost'->>'total')::numeric)` |
| Pagination         | `tail`/`head`                                       | `LIMIT ... OFFSET` or keyset                     |
| Concurrent writes  | File locking risks                                  | Row-level locking, ACID                          |
| Soft delete        | Rename with `.deleted.<ts>`                         | `UPDATE SET deleted_at = NOW()`                  |
| Schema migration   | Custom scripts                                      | `ALTER TABLE ADD COLUMN`                         |
| Incremental backup | Full copy                                           | `pg_dump --incremental` / WAL                    |

### Migration Strategy

**Phase 1: Dual-Write (Backwards-Compatible)**

- Keep JSONL as primary; write to PostgreSQL on each message
- Read from PostgreSQL if available, fallback to JSONL
- `sessions.json` remains the source of truth for session keys

**Phase 2: Read Replication**

- Migrate read paths to PostgreSQL
- Keep JSONL for write path until validated

**Phase 3: JSONL Deprecation**

- Make PostgreSQL primary
- Export remaining JSONL sessions on-demand
- Remove JSONL paths from `sessions.json` resolution

### Connection Management

```typescript
// pgclient.ts — singleton with connection pooling
import pg from "pg";
const { Pool } = pg;

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
});

export const query = (text: string, params?: unknown[]) => pool.query(text, params);
```

### Config Integration

```typescript
// Config keys
interface OpenClawConfig {
  "db.session.store": "jsonl" | "postgres";
  "db.session.pg-url": string; // postgresql://user:pass@host:5432/openclaw
  "db.session.pg-pool-size": number; // default 20
}
```

### Query Examples (SQL vs JSONL)

```bash
# JSONL: List sessions by date and size
for f in ~/.openclaw/agents/dev-expert/sessions/*.jsonl; do
  date=$(head -1 "$f" | jq -r '.timestamp' | cut -dT -f1)
  size=$(ls -lh "$f" | awk '{print $5}')
  echo "$date $size $(basename $f)"
done | sort -r

# PostgreSQL
SELECT id, started_at, (metadata->>'size') as size
FROM sessions WHERE agent_id = 'dev-expert'
ORDER BY started_at DESC;
```

```bash
# JSONL: Extract user messages
jq -r 'select(.message.role == "user") | .message.content[]? | select(.type == "text") | .text' session.jsonl

# PostgreSQL
SELECT content->0->>'text' FROM messages
WHERE session_id = 'uuid' AND role = 'user' AND content->0->>'type' = 'text';
```

```bash
# JSONL: Search across all sessions
rg -l "phrase" ~/.openclaw/agents/*/sessions/*.jsonl

# PostgreSQL
SELECT DISTINCT s.id FROM sessions s
JOIN messages m ON m.session_id = s.id
WHERE to_tsvector('english', m.content::text) @@ to_tsquery('english', 'phrase');
```

---

## Implementation Notes

### Handling Large Content

- Use `JSONB` for content column; allows partial updates and indexing
- For extremely long messages, consider TOAST compression (`ALTER TABLE ... SET (toast_tuple_target = 2048)`)

### Schema Versioning

- Add `schema_version` table to track migrations
- Use migrations folder: `migrations/001_initial.sql`, `002_add_indexes.sql`

### Backward Compatibility

- Provide `SessionStore` interface that both JSONL and Postgres implement
- Default to JSONL if `DATABASE_URL` not set

### Docker Compose

```yaml
# For local dev
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: openclaw
      POSTGRES_USER: openclaw
      POSTGRES_PASSWORD: devpass
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```

---

## Open Questions

1. **Multi-instance** — If running multiple gateway instances, do they share one DB or have separate?
2. **Retention** — Should there be automatic pruning? (currently manual via `.deleted.<ts>`)
3. **Embedding** — Currently memory system uses session hash for deduplication; how to migrate that?
4. **Cost tracking** — Is the current `usage.cost.total` per-message aggregation sufficient, or need per-token breakdown?

---

## Summary

PostgreSQL replaces JSONL with:

- Indexed queries (date, role, full-text)
- ACID compliance for concurrent sessions
- Relational cost aggregation
- Soft deletes and schema migrations
- Standard backup tools

Migration is staged dual-write → read replication → deprecation to ensure stability.
