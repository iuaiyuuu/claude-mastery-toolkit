# Migration Safety Playbook

## Golden Rules

1. **Every migration must be reversible.** Always write both the forward and rollback steps. If a migration is genuinely irreversible (dropping data), make this explicit and require confirmation.
2. **Never mix schema changes and data migrations in one step.** Schema change first (deploy), data migration second (deploy), cleanup third (deploy).
3. **Test against production-sized data.** A migration that takes 50ms on 1,000 rows may take 4 hours on 50 million rows and lock the table the entire time.
4. **Every migration has a blast radius.** Know what breaks if it fails halfway.

## Zero-Downtime Migration Patterns

### Adding a Column

**Safe:**
```sql
-- Step 1: Add column with default (Postgres 11+ doesn't rewrite table for constant defaults)
ALTER TABLE orders ADD COLUMN priority varchar(20) DEFAULT 'normal';

-- Step 2: Add constraint after backfill (if needed)
ALTER TABLE orders ADD CONSTRAINT chk_orders_priority
  CHECK (priority IN ('low', 'normal', 'high', 'urgent'));
```

**Unsafe:**
```sql
-- LOCKS TABLE while rewriting every row (pre-Postgres 11 without default optimization)
ALTER TABLE orders ADD COLUMN priority varchar(20) NOT NULL;
```

### Removing a Column

**Safe sequence:**
1. Stop application code from reading/writing the column (deploy code change)
2. Wait for all old application instances to drain
3. Drop the column: `ALTER TABLE orders DROP COLUMN legacy_field;`

**Unsafe:** Dropping a column while application code still references it → instant errors.

### Renaming a Column

Never rename directly. Instead:

```
Step 1: Add new column
  ALTER TABLE users ADD COLUMN full_name varchar(255);

Step 2: Backfill (batched)
  UPDATE users SET full_name = name WHERE full_name IS NULL LIMIT 10000;
  -- Repeat until done

Step 3: Deploy dual-write code (writes to both old and new columns)

Step 4: Verify data consistency

Step 5: Switch reads to new column (deploy code change)

Step 6: Stop writing to old column (deploy code change)

Step 7: Drop old column
  ALTER TABLE users DROP COLUMN name;
```

### Adding an Index

**PostgreSQL — always use CONCURRENTLY:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
```

Without `CONCURRENTLY`, `CREATE INDEX` acquires a table-level lock that blocks all writes for the duration of the index build. On a 50M row table, this could be minutes to hours.

**Caveats of CONCURRENTLY:**
- Cannot run inside a transaction block
- If it fails, it leaves an invalid index — check `pg_index.indisvalid` and drop it if invalid
- Takes roughly 2-3x longer than a regular index build

**MySQL:**
```sql
ALTER TABLE orders ADD INDEX idx_orders_customer (customer_id), ALGORITHM=INPLACE, LOCK=NONE;
```

### Changing a Column Type

Type changes that are safe without table rewrite (PostgreSQL):
- `varchar(N)` → `varchar(M)` where M > N (widening)
- `varchar(N)` → `text`
- Adding or removing a default

Type changes that require a rewrite (treat as rename):
- `integer` → `bigint`
- `varchar` → `integer`
- Any narrowing change

For rewrite-requiring changes, follow the rename pattern: add new column → backfill → dual-write → switch → drop old.

### Adding a NOT NULL Constraint

**Safe sequence:**
```sql
-- Step 1: Add the constraint as NOT VALID (doesn't scan existing rows)
ALTER TABLE orders ADD CONSTRAINT orders_customer_id_not_null
  CHECK (customer_id IS NOT NULL) NOT VALID;

-- Step 2: Backfill any NULL values
UPDATE orders SET customer_id = 0 WHERE customer_id IS NULL;
-- (Use appropriate default or handle upstream)

-- Step 3: Validate the constraint (scans existing rows but doesn't lock writes)
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_id_not_null;

-- Step 4: Optionally convert to true NOT NULL
ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT orders_customer_id_not_null;
```

### Adding a Foreign Key

**Safe sequence (PostgreSQL):**
```sql
-- Step 1: Add constraint as NOT VALID (instant, no scan)
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;

-- Step 2: Validate (scans but doesn't hold ACCESS EXCLUSIVE lock for long)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;
```

### Dropping a Table

**Safe sequence:**
1. Verify no application code references the table
2. Verify no foreign keys reference the table
3. Optionally: rename the table first and wait (e.g., `ALTER TABLE old_data RENAME TO _old_data_to_delete`)
4. Wait a deployment cycle to confirm nothing breaks
5. Drop: `DROP TABLE _old_data_to_delete;`

## Batched Data Migrations

For large data migrations, never run a single `UPDATE` on millions of rows:

```sql
-- BAD: Locks the entire table, generates massive WAL, may OOM
UPDATE orders SET new_column = compute(old_column);

-- GOOD: Batch in chunks
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE orders
    SET new_column = compute(old_column)
    WHERE id IN (
      SELECT id FROM orders
      WHERE new_column IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    );
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;
    COMMIT;
    PERFORM pg_sleep(0.1); -- Brief pause to reduce load
  END LOOP;
END $$;
```

For very large tables (100M+ rows), consider:
- Running the migration during off-peak hours
- Using `pg_repack` for table-rewriting operations
- Creating a new table, copying data, and swapping via rename

## Migration Tooling Recommendations

- **PostgreSQL**: Use a migration framework (Flyway, Liquibase, golang-migrate, Alembic, Prisma Migrate) — never run ad-hoc SQL in production
- **MongoDB**: Use a migration runner (migrate-mongo, mongock) — collection restructuring needs the same discipline as SQL migrations
- **All databases**: Migrations must be version-controlled, idempotent (safe to re-run), and applied in order

## Pre-Migration Checklist

Before executing any migration in production:

- [ ] Migration tested against a production-sized dataset
- [ ] Rollback script written and tested
- [ ] Estimated execution time documented
- [ ] Table lock implications analyzed (ACCESS EXCLUSIVE?)
- [ ] Application code compatible with both old and new schema during transition
- [ ] Monitoring in place (replication lag, lock wait times, query performance)
- [ ] Backup taken (or point-in-time recovery confirmed available)
- [ ] Runbook documented for the team
