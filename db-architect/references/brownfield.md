# Brownfield Database Audit Checklist

When the user provides an existing schema (SQL dump, ORM models, migration files, or verbal description), perform a systematic audit before proposing changes. The goal is to understand what exists, identify risks, and propose a prioritized improvement plan.

## Audit Process

### Step 1: Schema Inventory

Catalog what exists:
- List all tables/collections with row count estimates (ask the user if not provided)
- Map all relationships (FKs, implicit references, embedded documents)
- Identify primary key strategy (auto-inc, UUID, composite, natural key)
- Note all indexes
- Note all constraints (CHECK, UNIQUE, FK, NOT NULL)
- Identify any stored procedures, triggers, or views

### Step 2: Anti-Pattern Scan

Check for these issues systematically. For each finding, rate severity as **Critical**, **High**, or **Medium**:

**Critical (fix immediately):**
- Missing primary keys
- No foreign key constraints on obvious relationships
- Money stored as float/double
- No unique constraints on natural business keys (email, SKU, etc.)
- Unbounded text fields accepting user input (SQL injection surface if not parameterized)

**High (fix in next sprint):**
- Missing indexes on foreign key columns
- No timestamps (created_at / updated_at) on mutable tables
- VARCHAR(255) used universally without meaningful size constraints
- Comma-separated values stored in a single column
- Polymorphic associations without discriminator
- Repeated column groups (phone1, phone2, phone3)
- Status columns as unconstrained VARCHAR (no CHECK/ENUM)

**Medium (plan for next quarter):**
- Inconsistent naming conventions
- Over-indexing (more total index size than table size)
- Unused indexes (check pg_stat_user_indexes)
- Missing partial indexes for soft-delete patterns
- N+1 query patterns visible from schema structure
- Denormalization without clear justification
- Missing comments on non-obvious columns or tables

### Step 3: Relationship Integrity Analysis

For every relationship:
- Is there a foreign key constraint? If not, are there orphaned rows?
- What's the ON DELETE behavior? Is it appropriate?
  - CASCADE on a critical table is a ticking time bomb
  - SET NULL without application handling creates data holes
  - RESTRICT/NO ACTION is safest default
- Are there implicit relationships (columns named `*_id` without FK constraints)?
- Are there circular references? Document them.

### Step 4: Performance Assessment

Based on the schema (not production metrics — you don't have those):
- Are the obvious hot-path queries indexed?
- Are there composite indexes that could serve multiple query patterns?
- Are there large tables that should be partitioned?
- Are there missing covering indexes for high-frequency read patterns?
- Are there tables that mix hot and cold data (consider archiving)?

### Step 5: Migration Risk Assessment

For every proposed change, assess:
- **Table size impact**: Will this require a table rewrite on a large table?
- **Lock behavior**: Will this acquire ACCESS EXCLUSIVE and block writes?
- **Data dependency**: What application code reads/writes this table?
- **Rollback complexity**: How hard is it to undo this change?
- **Ordering constraints**: What must be done before this change?

## Audit Output Format

Present findings as a prioritized improvement plan:

```markdown
## Database Audit: {Project Name}

### Current State Summary
- X tables, Y total estimated rows
- Key findings: [brief summary of the most important issues]

### Critical Issues (fix immediately)
1. Issue description
   - Impact: what breaks or corrupts
   - Fix: specific migration steps
   - Risk: migration complexity

### High Priority Issues (next sprint)
1. Issue description
   - Impact: ...
   - Fix: ...
   - Risk: ...

### Medium Priority Issues (next quarter)
1. Issue description
   - Impact: ...
   - Fix: ...

### Recommended Migration Sequence
Ordered list of changes with dependencies noted.
Each step includes the migration SQL and rollback SQL.
```

## Brownfield Design Principles

When improving an existing schema:

1. **Don't boil the ocean.** Propose incremental improvements, not a full rewrite. The user's application is running on this schema — respect that.
2. **Preserve backward compatibility during transition.** Every intermediate state must be a working state.
3. **Document assumptions.** If you infer a relationship or constraint from naming conventions, say so — the user may correct you.
4. **Ask before assuming.** If the schema seems intentionally denormalized or uses an unusual pattern, ask why before flagging it. There may be a valid reason (performance, legacy integration, regulatory requirement).
5. **Prioritize data integrity fixes over performance fixes.** Corrupt data is permanent; slow queries are temporary.
