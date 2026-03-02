# Technology Selection Matrix

Use this reference when deciding which database technology to recommend. The goal is to be decisive — pick the best fit, don't enumerate every option.

## Decision Tree

```
START
├── Is the data highly relational (many entities, complex joins)?
│   ├── Yes → PostgreSQL (default) or MySQL
│   └── No ↓
├── Is schema flexibility critical (rapidly evolving, polymorphic documents)?
│   ├── Yes → Document store (MongoDB, DynamoDB)
│   └── No ↓
├── Is the primary access pattern graph traversal (friends-of-friends, shortest path)?
│   ├── Yes → Graph database (Neo4j, Neptune)
│   └── No ↓
├── Is this time-series data with time-based aggregations?
│   ├── Yes → TimescaleDB (if already using Postgres) or InfluxDB
│   └── No ↓
├── Is this a caching/session/ephemeral data layer?
│   ├── Yes → Redis / Valkey
│   └── No ↓
└── Default → PostgreSQL
```

## Detailed Comparison

### PostgreSQL
**Best for**: General-purpose OLTP, complex queries, ACID compliance, JSON hybrid workloads, geospatial (PostGIS), full-text search.
**Scales to**: Single node handles millions of rows comfortably. Read replicas for read scaling. Citus for horizontal write scaling.
**Avoid when**: Write throughput exceeds what a single beefy node can handle and you can't use Citus; pure document workloads with no relational needs.
**Key strength**: Extensibility — JSONB, PostGIS, pg_trgm, ltree, hstore all reduce the need for auxiliary databases.

### MySQL / MariaDB
**Best for**: High-throughput simple queries, web application backends, when the team already knows it.
**Scales to**: Read replicas, group replication, Vitess for sharding.
**Avoid when**: Complex queries with CTEs, window functions, or advanced JSON are needed (MySQL is catching up but Postgres is ahead). Also avoid when you need fine-grained constraint enforcement.
**Key difference from Postgres**: More permissive defaults (silent truncation, implicit type coercion) — tighten `sql_mode` to `STRICT_TRANS_TABLES`.

### SQLite
**Best for**: Embedded applications, mobile local storage, single-user apps, development/testing, edge computing.
**Scales to**: Single writer only. Use for read-heavy workloads or where single-writer is acceptable.
**Avoid when**: Multiple concurrent writers, network-accessible database needed, or multi-tenant SaaS.

### MongoDB
**Best for**: Rapid prototyping with evolving schemas, content management, catalog data with varying attributes, document-oriented access patterns.
**Scales to**: Horizontal sharding built-in. Good for distributed workloads.
**Avoid when**: Data is inherently relational, you need multi-document ACID transactions frequently, or you need strong schema enforcement from day one.
**Warning**: "Schemaless" is a myth in production — you always have an implicit schema. Use MongoDB's schema validation to make it explicit.

### DynamoDB
**Best for**: Serverless architectures on AWS, key-value or simple document lookups, extreme read/write throughput at predictable latency.
**Scales to**: Effectively unlimited (AWS manages sharding).
**Avoid when**: Complex queries, ad-hoc analytics, or when you're not on AWS. The single-table design pattern adds significant complexity.
**Key constraint**: Design around partition key + sort key. If you can't express your access patterns in terms of these, DynamoDB is the wrong choice.

### Redis / Valkey
**Best for**: Caching (session, query results), rate limiting, leaderboards, pub/sub, real-time counters, queue-like patterns.
**Scales to**: Cluster mode for horizontal scaling.
**Avoid when**: Data durability is critical (even with AOF, it's not a primary store). Never use as the sole store for data you can't afford to lose.

### TimescaleDB
**Best for**: Time-series data when you're already using PostgreSQL. IoT sensor data, application metrics, financial tick data.
**Key advantage**: Full SQL + PostgreSQL ecosystem. Hypertables provide automatic time-based partitioning.

### InfluxDB
**Best for**: Pure time-series workloads, especially metrics and monitoring. Purpose-built query language (Flux/InfluxQL).
**Avoid when**: You need to join time-series data with relational data frequently — use TimescaleDB instead.

### Neo4j
**Best for**: Social networks, recommendation engines, fraud detection, knowledge graphs — anything where relationship traversal depth > 2 is a primary query pattern.
**Avoid when**: Data is tabular and relationships are simple 1:N or M:N lookable via JOINs. Graph databases add operational complexity.

### Elasticsearch / OpenSearch
**Best for**: Full-text search, log analytics, faceted search.
**Key rule**: Always use alongside a primary datastore. Elasticsearch is a search index, not a system of record. Data flows from the primary store → Elasticsearch.

## Hybrid Architecture Patterns

When recommending multiple databases, always define:

1. **Source of truth**: Which database owns each piece of data
2. **Sync mechanism**: How data flows between databases (CDC, application-level dual-write, ETL)
3. **Consistency model**: What the consistency guarantee is at each boundary
4. **Failure modes**: What happens when the sync breaks

### Common valid hybrid patterns:
- PostgreSQL (primary) + Redis (cache) — most common, low complexity
- PostgreSQL (primary) + Elasticsearch (search) — for full-text search requirements
- PostgreSQL (OLTP) + ClickHouse/BigQuery (OLAP) — for analytics workloads
- PostgreSQL (primary) + TimescaleDB (time-series) — can share the same Postgres instance

### Anti-pattern hybrids:
- MongoDB + PostgreSQL for the same domain (pick one relational strategy)
- Redis as primary store + PostgreSQL as backup (invert this)
- Multiple relational databases for different microservices when a single Postgres with schemas would suffice
