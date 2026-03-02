# Indexing Strategy Guide

## Core Principles

1. **Every index must justify its existence with a specific query pattern.** If you can't name the query an index serves, remove it.
2. **Indexes are a write-time cost for read-time benefit.** Each index slows INSERT/UPDATE/DELETE and consumes storage.
3. **The query planner is your ally.** Use `EXPLAIN ANALYZE` to verify indexes are actually used.

## Index Types by Database

### PostgreSQL

| Type | Use Case | Example |
|---|---|---|
| B-tree (default) | Equality, range, sorting, prefix LIKE | `CREATE INDEX idx_users_email ON users(email)` |
| Hash | Equality only (rarely better than B-tree) | Rarely recommended |
| GIN | Arrays, JSONB, full-text search (tsvector) | `CREATE INDEX idx_posts_tags ON posts USING gin(tags)` |
| GiST | Geospatial, range types, nearest-neighbor | `CREATE INDEX idx_locations_coords ON locations USING gist(coords)` |
| BRIN | Large tables with natural ordering (timestamps, auto-inc IDs) | `CREATE INDEX idx_events_created ON events USING brin(created_at)` |
| SP-GiST | Trie-based: IP ranges, phone numbers, hierarchical data | `CREATE INDEX idx_network_ip ON networks USING spgist(ip_range)` |

### MySQL / MariaDB

| Type | Use Case |
|---|---|
| B-tree (default) | General purpose |
| Full-text | Text search (InnoDB supports since 5.6) |
| Spatial (R-tree) | Geospatial |
| Hash | Memory engine only, equality only |

### MongoDB

| Type | Use Case |
|---|---|
| Single field | Equality, range on one field |
| Compound | Multi-field queries (order matters) |
| Multikey | Array fields |
| Text | Full-text search |
| 2dsphere | Geospatial |
| Hashed | Shard key distribution |

## Composite Index Design

The order of columns in a composite index matters significantly. Follow the **ESR rule** (Equality → Sort → Range):

1. **Equality** columns first (exact match `WHERE status = 'active'`)
2. **Sort** columns next (`ORDER BY created_at DESC`)
3. **Range** columns last (`WHERE price BETWEEN 10 AND 50`)

**Example:**
Query: `SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2024-01-01' ORDER BY total DESC`

Best index: `(status, total DESC, created_at)` — equality first, then sort, then range.

## Covering Indexes

A covering index includes all columns needed by a query, allowing the database to serve the query entirely from the index without touching the table (index-only scan).

**PostgreSQL** (using INCLUDE):
```sql
CREATE INDEX idx_orders_status_covering
ON orders(status)
INCLUDE (total, customer_id, created_at);
```

Use covering indexes for your most frequent read queries. The trade-off is increased index size.

## Partial Indexes

A partial index only includes rows matching a condition. Extremely useful for:

- Soft-delete patterns: `CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL`
- Status filtering: `CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending'`
- Sparse data: `CREATE INDEX idx_users_phone ON users(phone) WHERE phone IS NOT NULL`

Partial indexes are smaller and faster than full indexes. Use them when queries consistently filter on the same condition.

## Expression Indexes

Index computed values when queries use expressions:

```sql
-- For case-insensitive email lookups
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- For JSONB field access
CREATE INDEX idx_metadata_type ON events((metadata->>'type'));

-- For date truncation queries
CREATE INDEX idx_orders_month ON orders(DATE_TRUNC('month', created_at));
```

The query must use the exact same expression for the index to be used.

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|---|---|---|
| Index every column | Massive write overhead, wasted storage | Index only queried columns |
| No index on foreign keys | Full table scans on JOIN and CASCADE | Add B-tree index on every FK |
| Wrong column order in composite | Index unused for the actual query pattern | Follow ESR rule, check EXPLAIN |
| Duplicate indexes | Same overhead, no benefit | Audit with `pg_stat_user_indexes` |
| Indexing low-cardinality columns alone | B-tree can't help when most rows match | Combine with other columns or use partial index |
| Not using partial indexes for soft deletes | Full index for queries that always filter active | Add `WHERE deleted_at IS NULL` |

## Index Maintenance

- **Monitor unused indexes**: In PostgreSQL, check `pg_stat_user_indexes.idx_scan = 0` for indexes that haven't been used. Drop them after confirming.
- **Monitor bloat**: `REINDEX` or `pg_repack` for heavily updated tables.
- **Statistics**: Run `ANALYZE` after bulk loads to update planner statistics.
- **Index size vs table size**: If total index size exceeds table size, you likely have redundant indexes.

## MongoDB-Specific Indexing

- Use `.explain("executionStats")` to verify index usage
- Compound indexes support queries on any **prefix** of the indexed fields
- `{a: 1, b: 1, c: 1}` supports queries on `{a}`, `{a, b}`, and `{a, b, c}` but NOT `{b}` or `{c}` alone
- Set TTL indexes for auto-expiring documents (sessions, logs)
- Use `background: true` for index creation on production collections (MongoDB 4.2+ builds in background by default)
