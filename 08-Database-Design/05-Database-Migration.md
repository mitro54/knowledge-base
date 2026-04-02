# Database Migration

## Title & Summary

**Database Migration** encompasses the strategies, tools, and patterns for evolving database schemas over time while maintaining data integrity, minimizing downtime, and ensuring backward compatibility. This pattern covers migration strategies, versioning approaches, and techniques for zero-downtime schema changes.

---

## Problem Statement

Database schema evolution presents significant challenges:

- **Downtime requirements** for schema changes affecting availability
- **Data loss risk** during migration operations
- **Backward compatibility** issues with existing applications
- **Rollback complexity** when migrations fail
- **Team coordination** challenges in distributed environments

### Example Scenario

```
Production Database: 500GB, 10 million users, 24/7 availability required

Migration Needed: Add NOT NULL constraint to email column

Naive Approach:
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL;

Result:
- Table locked for 45 minutes
- All writes blocked during migration
- Service unavailable
- Users impacted: 10 million
```

---

## Solution

### Migration Strategies

#### 1. Expand and Contract Pattern

```
Phase 1: EXPAND (Additive Changes)
┌─────────────────────────────────────────┐
│  Add new column/table/index             │
│  - No data loss                        │
│  - Backward compatible                 │
│  - Zero downtime                       │
└─────────────────────────────────────────┘
              ↓
Phase 2: MIGRATE (Data Transformation)
┌─────────────────────────────────────────┐
│  Copy/transform data                   │
│  - Run in background                   │
│  - Batch processing                    │
│  - Can be rolled back                  │
└─────────────────────────────────────────┘
              ↓
Phase 3: CONTRACT (Remove Old)
┌─────────────────────────────────────────┐
│  Remove old column/table/index          │
│  - After verification                  │
│  - Clean up                            │
└─────────────────────────────────────────┘
```

#### 2. Parallel Write Pattern

```
Before Migration:
┌──────────┐
│  Write   │→┌──────────┐
│  to Old  │ │  Old     │
│  Schema  │ │  Column  │
└──────────┘ └──────────┘

During Migration:
┌──────────┐
│  Write   │→┌──────────┐  ┌──────────┐
│  to Both │ │  Old     │  │  New     │
│  Columns │ │  Column  │  │  Column  │
└──────────┘ └──────────┘  └──────────┘

After Migration:
┌──────────┐
│  Write   │→┌──────────┐
│  to New  │ │  New     │
│  Column  │ │  Column  │
└──────────┘ └──────────┘
```

#### 3. Dual Write with Read Migration

```
Step 1: Add new column, dual write
Step 2: Backfill historical data
Step 3: Switch reads to new column
Step 4: Verify data consistency
Step 5: Remove old column
```

### Zero-Downtime Migration Techniques

#### Adding a Column

```sql
-- Step 1: Add nullable column (instant)
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NULL;

-- Step 2: Update application to write to new column
-- (Dual write period)

-- Step 3: Backfill existing data (can be done in batches)
UPDATE users SET email_verified = TRUE WHERE email IS NOT NULL;

-- Step 4: Add constraint after verification
ALTER TABLE users ALTER COLUMN email_verified SET NOT NULL;
ALTER TABLE users ALTER COLUMN email_verified SET DEFAULT FALSE;
```

#### Adding a Not Null Constraint

```sql
-- Step 1: Add column with default value
ALTER TABLE users ADD COLUMN last_login_at TIMESTAMP NULL DEFAULT NULL;

-- Step 2: Application starts writing to column

-- Step 3: Backfill with default
UPDATE users SET last_login_at = '1970-01-01' WHERE last_login_at IS NULL;

-- Step 4: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN last_login_at SET NOT NULL;
```

#### Renaming a Column

```sql
-- Step 1: Add new column with new name
ALTER TABLE users ADD COLUMN new_email VARCHAR(255);

-- Step 2: Copy data
UPDATE users SET new_email = email;

-- Step 3: Add index if needed
CREATE INDEX idx_users_new_email ON users(new_email);

-- Step 4: Update application to use new column
-- (Dual read/write period)

-- Step 5: Remove old column (after verification)
ALTER TABLE users DROP COLUMN email;

-- Step 6: Rename new column
ALTER TABLE users RENAME COLUMN new_email TO email;
```

#### Changing Column Type

```sql
-- Step 1: Add new column with new type
ALTER TABLE users ADD COLUMN age_new INTEGER;

-- Step 2: Copy and transform data
UPDATE users SET age_new = CAST(age AS INTEGER);

-- Step 3: Add constraints
ALTER TABLE users ALTER COLUMN age_new SET NOT NULL;
ALTER TABLE users ADD CONSTRAINT age_check CHECK (age_new >= 0 AND age_new <= 150);

-- Step 4: Update application

-- Step 5: Remove old column and rename
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users RENAME COLUMN age_new TO age;
```

### Migration Tools and Frameworks

#### Flyway Migration Example

```sql
-- V1__initial_schema.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- V2__add_phone_column.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- V3__add_email_index.sql
CREATE INDEX idx_users_email ON users(email);

-- V4__rename_phone_to_mobile.sql
ALTER TABLE users ADD COLUMN mobile VARCHAR(20);
UPDATE users SET mobile = phone;
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users RENAME COLUMN mobile TO phone;
```

#### Liquibase Migration Example

```xml
<!-- changelog.xml -->
<databaseChangeLog>
    <changeSet id="1" author="dev">
        <createTable tableName="users">
            <column name="id" type="BIGINT">
                <constraints primaryKey="true"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
    
    <changeSet id="2" author="dev">
        <addColumn tableName="users">
            <column name="phone" type="VARCHAR(20)"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
```

---

## When to Use

### Use Expand and Contract When:
- ✅ Schema changes require data transformation
- ✅ Zero downtime is required
- ✅ Changes can be broken into phases
- ✅ Application can support dual state

### Use Parallel Write When:
- ✅ Adding new columns or tables
- ✅ Changing data formats
- ✅ Needing gradual migration
- ✅ Can tolerate temporary duplication

### Use Direct Migration When:
- ✅ Adding nullable columns
- ✅ Adding indexes
- ✅ Adding new tables
- ✅ Low-traffic maintenance windows acceptable

---

## Tradeoffs

| Strategy | Downtime | Complexity | Risk | Rollback |
|----------|----------|------------|------|----------|
| **Direct ALTER** | High | Low | Medium | Easy |
| **Expand/Contract** | None | High | Low | Medium |
| **Parallel Write** | None | High | Low | Medium |
| **Table Swap** | Low | Medium | Medium | Hard |

### Quantitative Comparison

```
Migration: Add NOT NULL constraint to 10M row table

Direct ALTER:
- Downtime: 45 minutes
- Complexity: Low
- Risk: Medium (lock contention)

Expand/Contract:
- Downtime: 0 minutes
- Complexity: High (multiple steps)
- Risk: Low (gradual)
- Additional storage: 80MB (new column)
```

---

## Implementation Example

### Complete Zero-Downtime Migration

```sql
-- Migration: Add required 'status' column to orders table
-- Table size: 50 million rows, 24/7 availability required

-- PHASE 1: EXPAND (Add new column)
ALTER TABLE orders ADD COLUMN status_new VARCHAR(20) NULL;

-- PHASE 2: DUAL WRITE (Application updated)
-- Application now writes to both 'state' and 'status_new'

-- PHASE 3: BACKFILL (Run in batches to avoid locking)
-- Batch 1
UPDATE orders 
SET status_new = CASE 
    WHEN state = 1 THEN 'pending'
    WHEN state = 2 THEN 'processing'
    WHEN state = 3 THEN 'completed'
    WHEN state = 4 THEN 'cancelled'
END
WHERE id BETWEEN 1 AND 1000000 AND status_new IS NULL;

-- Batch 2, 3, ... until complete
-- Monitor progress: SELECT COUNT(*) FROM orders WHERE status_new IS NULL;

-- PHASE 4: VERIFY
SELECT 
    state,
    status_new,
    COUNT(*) as count
FROM orders
GROUP BY state, status_new;

-- PHASE 5: SWITCH READS (Application updated)
-- Application now reads from 'status_new' only

-- PHASE 6: ADD CONSTRAINTS
ALTER TABLE orders ALTER COLUMN status_new SET NOT NULL;
ALTER TABLE orders ADD CONSTRAINT status_check 
CHECK (status_new IN ('pending', 'processing', 'completed', 'cancelled'));

-- PHASE 7: CONTRACT (Remove old column)
ALTER TABLE orders DROP COLUMN state;
ALTER TABLE orders RENAME COLUMN status_new TO status;

-- PHASE 8: CLEANUP
-- Remove any temporary indexes, triggers, etc.
```

### Migration Verification Script

```sql
-- Verify migration completeness
SELECT 
    'orders' as table_name,
    COUNT(*) as total_rows,
    COUNT(status_new) as migrated_rows,
    COUNT(*) - COUNT(status_new) as remaining_rows,
    ROUND(100.0 * COUNT(status_new) / COUNT(*), 2) as percent_complete
FROM orders;

-- Verify data consistency
SELECT 
    state as old_value,
    status_new as new_value,
    COUNT(*) as count
FROM orders
WHERE state IS NOT NULL AND status_new IS NOT NULL
GROUP BY state, status_new
HAVING COUNT(*) > 0;

-- Find any inconsistencies
SELECT * FROM orders
WHERE state = 1 AND status_new != 'pending'
   OR state = 2 AND status_new != 'processing'
   OR state = 3 AND status_new != 'completed'
   OR state = 4 AND status_new != 'cancelled';
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Direct Schema Changes in Production

```sql
-- Bad: Making breaking changes directly
ALTER TABLE users DROP COLUMN legacy_field;  -- Data loss!
ALTER TABLE orders ALTER COLUMN total SET DATA TYPE DECIMAL(10,2);  -- May fail!
TRUNCATE TABLE logs;  -- Data loss!

-- These operations:
-- - Lock tables for extended periods
-- - Cannot be easily rolled back
-- - May cause data loss
-- - Block all application traffic
```

### ❌ Anti-Pattern: Manual Migrations

```bash
# Bad: Running migrations manually via SQL client
mysql -u root -p production_db < migration.sql

# Problems:
# - No tracking of applied migrations
# - Easy to skip or duplicate migrations
# - No rollback mechanism
# - No audit trail
# - Team coordination issues
```

### ❌ Anti-Pattern: Migration Without Testing

```sql
-- Bad: Running migration directly on production
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT TRUE;

-- What if:
-- - Table has 100M rows? (Long-running operation)
-- - Default value is wrong for existing users?
-- - Column name conflicts with application code?
```

### ✅ Correct Approach

```sql
-- Good: Use migration framework with versioning
-- Flyway/Liquibase/Django migrations

-- 1. Write migration in version control
-- 2. Test on staging with production-like data
-- 3. Run on production during low-traffic window
-- 4. Monitor for issues
-- 5. Have rollback plan ready

-- Example rollback migration:
-- V4__rollback_add_status_column.sql
ALTER TABLE orders DROP COLUMN IF EXISTS status;
```

---

## Related Patterns

- **[Relational Design](01-Relational.md)** - Schema evolution of relational databases
- **[Scalability Patterns](../01-System-Design/04-Scalability-Patterns.md)** - Migration at scale
- **[Deployment Strategies](../11-DevOps/02-Deployment-Strategies.md)** - Coordinating schema and app deployments
- **[Feature Flags](../11-DevOps/04-Feature-Flags.md)** - Gradual feature rollout with migrations
- **[Database Design](02-NoSQL.md)** - Migration strategies for NoSQL databases