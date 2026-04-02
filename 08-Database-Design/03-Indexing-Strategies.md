# Indexing Strategies

## Title & Summary

**Indexing Strategies** encompasses the techniques and data structures used to accelerate data retrieval in database systems. This pattern covers B-trees, hash indexes, full-text indexes, spatial indexes, and composite indexes, along with strategies for index selection, maintenance, and optimization.

---

## Problem Statement

Without proper indexing, database queries must perform full table scans, examining every row to find matching records. This results in:

- **O(n) time complexity** for lookups on unindexed columns
- **High I/O costs** reading unnecessary data from disk
- **Poor query performance** as dataset size grows
- **Resource contention** blocking other operations
- **Inability to meet SLA requirements** for response times

### Example Scenario

A user table with 10 million records:
- **Without index**: Finding a user by email requires scanning all 10 million rows
- **With B-tree index**: Finding a user by email requires ~24 comparisons (log₂(10,000,000))

---

## Solution

### Index Types and Data Structures

#### 1. B-Tree Index (Balanced Tree)

**Best for**: Range queries, equality queries, ordered data

```
        [100|200|300]
       /    |    |    \
  [50|60] [150] [250] [350|400]
   /  |     |      |      |   \
[45] [55] [140]  [240]  [340] [380] [420]
```

**Characteristics**:
- O(log n) search, insert, delete
- Maintains sorted order
- Supports range queries efficiently
- Self-balancing structure

#### 2. Hash Index

**Best for**: Equality queries only

```
Key: "user123" → Hash: 0x3F7A → Bucket: 5 → [Record Pointer]
Key: "user456" → Hash: 0x8B2C → Bucket: 12 → [Record Pointer]
```

**Characteristics**:
- O(1) average lookup time
- No support for range queries
- No support for prefix matching
- Hash collisions require resolution

#### 3. Full-Text Index (Inverted Index)

**Best for**: Text search, document retrieval

```
Word        → Document IDs
-------------------------
"database"  → [1, 5, 23, 45]
"index"     → [1, 3, 12, 45]
"strategy"  → [1, 8, 23]
```

**Characteristics**:
- Supports word-level search
- Enables phrase matching
- Supports relevance ranking
- Requires tokenization

#### 4. Composite Index

**Best for**: Multi-column queries

```
Index on (last_name, first_name, email)

(last_name, first_name, email)
Smith,     John,      john.s@email.com
Smith,     Jane,      jane.s@email.com
Johnson,   Bob,       bob.j@email.com
```

**Characteristics**:
- Follows leftmost prefix rule
- More selective than single-column indexes
- Supports queries on leading columns

#### 5. Covering Index

**Best for**: Queries that can be satisfied entirely from index

```sql
-- Index includes all columns needed
CREATE INDEX idx_users_email_name ON users(email, first_name, last_name);

-- Query can use index only (no table access needed)
SELECT first_name, last_name FROM users WHERE email = 'user@example.com';
```

### Index Selection Strategy

```
Query Pattern Analysis:
├── Equality on single column → Hash or B-Tree
├── Range queries → B-Tree
├── Text search → Full-Text
├── Multi-column equality → Composite B-Tree
├── Spatial queries → R-Tree / GiST
└── Key-value lookup → Hash
```

---

## When to Use

### Use B-Tree Index When:
- ✅ Performing range queries (`BETWEEN`, `>`, `<`)
- ✅ Needing sorted results (`ORDER BY`)
- ✅ Doing equality queries on frequently queried columns
- ✅ Column has high cardinality (many unique values)

### Use Hash Index When:
- ✅ Only equality queries (`=`) on the column
- ✅ Needing fastest possible point lookups
- ✅ Column has very high cardinality
- ⚠️ Never use for range queries

### Use Full-Text Index When:
- ✅ Searching text content for keywords
- ✅ Needing relevance ranking
- ✅ Performing phrase or proximity searches
- ✅ Working with large text fields

### Use Composite Index When:
- ✅ Queries filter on multiple columns
- ✅ Column order matches query patterns
- ✅ Needing to optimize specific query patterns
- ✅ Following leftmost prefix rule

### Use Covering Index When:
- ✅ Query selects only indexed columns
- ✅ Reducing table lookups is critical
- ✅ Index size is acceptable tradeoff

---

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| **Read Performance** | O(log n) or O(1) lookups | Index maintenance overhead |
| **Write Performance** | - | Slower INSERT/UPDATE/DELETE |
| **Storage** | Faster queries | Additional disk space (10-20% per index) |
| **Memory** | Index pages cached | Reduced buffer pool for data |
| **Complexity** | Optimized queries | Index management overhead |

### Quantitative Tradeoffs

```
For a table with 1 million rows:

Without Index:
- Query time: ~100ms (full scan)
- Write time: ~1ms
- Storage: 100MB

With B-Tree Index:
- Query time: ~2ms (index scan)
- Write time: ~3ms (data + index update)
- Storage: 120MB (data + index)
```

### Index Overhead Calculation

```
Index Overhead = (Index Size / Table Size) × 100%

Example:
- Table: 100MB
- Index 1 (on id): 15MB → 15% overhead
- Index 2 (on email): 25MB → 25% overhead
- Index 3 (composite): 35MB → 35% overhead
- Total overhead: 75%
```

---

## Implementation Example

### PostgreSQL Index Implementation

```sql
-- Basic B-Tree Index (default)
CREATE INDEX idx_users_email ON users(email);

-- Hash Index
CREATE INDEX idx_users_hash ON users USING HASH (user_id);

-- Full-Text Search Index
CREATE INDEX idx_posts_content ON posts USING GIN (to_tsvector('english', content));

-- Composite Index
CREATE INDEX idx_orders_customer_status 
ON orders(customer_id, status, created_at);

-- Partial Index (for subset of data)
CREATE INDEX idx_active_users 
ON users(email) WHERE status = 'active';

-- Expression Index
CREATE INDEX idx_users_lower_email 
ON users(LOWER(email));

-- Covering Index (INCLUDE clause in PostgreSQL 11+)
CREATE INDEX idx_orders_covering 
ON orders(customer_id) INCLUDE (total_amount, status);
```

### MySQL Index Implementation

```sql
-- Basic Index
CREATE INDEX idx_products_category ON products(category_id);

-- Full-Text Index
CREATE FULLTEXT INDEX idx_articles_content ON articles(content, title);

-- Spatial Index
CREATE SPATIAL INDEX idx_locations_geom ON locations(geom);

-- Generated Column Index
ALTER TABLE users ADD COLUMN email_hash 
GENERATED ALWAYS AS (SHA2(email, 256)) VIRTUAL;
CREATE INDEX idx_users_email_hash ON users(email_hash);
```

### Index Analysis and Optimization

```sql
-- PostgreSQL: Analyze index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,      -- Times index was used
    idx_tup_read,  -- Rows read via index
    idx_tup_fetch  -- Rows fetched via index
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT indexname, schemaname, tablename
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Analyze index size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexname::regclass) DESC;
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Index Overload

```sql
-- Creating indexes without analysis
CREATE INDEX idx_users_first_name ON users(first_name);
CREATE INDEX idx_users_last_name ON users(last_name);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_city ON users(city);
CREATE INDEX idx_users_country ON users(country);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_updated ON users(updated_at);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_age ON users(age);
-- ... and more

-- Result:
-- - 20+ indexes on one table
-- - Writes are 10x slower
-- - 200% storage overhead
-- - Query optimizer confused
-- - Maintenance nightmare
```

### ❌ Anti-Pattern: Low Cardinality Indexes

```sql
-- Bad: Index on boolean or low-cardinality column
CREATE INDEX idx_users_is_active ON users(is_active);  -- Only 2 values!
CREATE INDEX idx_orders_status ON orders(status);       -- Only 5 values!

-- These indexes provide minimal benefit but add overhead
-- The optimizer often ignores them anyway
```

### ❌ Anti-Pattern: Indexing Unqueried Columns

```sql
-- Creating indexes on columns never used in WHERE/JOIN/ORDER BY
CREATE INDEX idx_users_middle_name ON users(middle_name);
CREATE INDEX idx_logs_debug_info ON logs(debug_info);

-- Wasted storage and write overhead
```

### ✅ Correct Approach

```sql
-- 1. Analyze query patterns first
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- 2. Create only necessary indexes
CREATE INDEX idx_users_email ON users(email);  -- High cardinality, frequently queried

-- 3. Use composite indexes wisely
CREATE INDEX idx_orders_customer_date 
ON orders(customer_id, created_at DESC);  -- Matches common query pattern

-- 4. Use partial indexes for filtered queries
CREATE INDEX idx_recent_logs 
ON logs(created_at) WHERE created_at > NOW() - INTERVAL '7 days';

-- 5. Monitor and remove unused indexes
-- Run monthly analysis and drop indexes with zero usage
```

---

## Related Patterns

- **[Relational Design](01-Relational.md)** - Indexing supports relational query optimization
- **[Query Optimization](04-Query-Optimization.md)** - Indexes are fundamental to query plans
- **[NoSQL Design](02-NoSQL.md)** - Different indexing strategies in NoSQL databases
- **[Scalability Patterns](../01-System-Design/04-Scalability-Patterns.md)** - Indexing impacts database scalability
- **[Performance Patterns](../01-System-Design/07-Performance-Patterns.md)** - Index optimization for performance