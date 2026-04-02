# Query Optimization

## Title & Summary

**Query Optimization** encompasses the techniques and strategies for improving database query performance through execution plan analysis, query rewriting, index utilization, and resource management. This pattern covers understanding query execution, identifying bottlenecks, and applying optimization techniques systematically.

---

## Problem Statement

Poorly optimized queries can severely impact application performance:

- **Slow response times** causing poor user experience
- **High resource consumption** (CPU, memory, I/O)
- **Database contention** blocking other operations
- **Increased infrastructure costs** from over-provisioning
- **Cascading failures** when queries timeout

### Example Scenario

```sql
-- Unoptimized query on 10 million row table
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 'pending' AND o.created_at > '2024-01-01'
ORDER BY o.created_at DESC;

-- Execution time: 15.2 seconds
-- Rows examined: 10,000,000
-- Temporary tables: Yes
-- Filesort: Yes
```

---

## Solution

### Query Execution Plan Analysis

#### Understanding Execution Plans

```
Query Execution Flow:
┌─────────────────────────────────────────────────────┐
│                  Parser                              │
│           (Syntax validation)                        │
└─────────────────┬───────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────┐
│                Binder                                │
│      (Resolve tables, columns, types)                │
└─────────────────┬───────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────┐
│              Optimizer                               │
│  (Choose indexes, join order, access methods)        │
└─────────────────┬───────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────┐
│              Executor                                │
│        (Run query, return results)                   │
└─────────────────────────────────────────────────────┘
```

#### Reading PostgreSQL EXPLAIN Output

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

Output:
┌─────────────────────────────────────────────────────┐
│  Bitmap Heap Scan on orders                          │
│    Cost: 4.21..179.42                               │
│    Actual Time: 0.045..12.345 rows=1500 width=256   │
│    Heap Blocks: exact=156                           │
│  └─>  Bitmap Index Scan on idx_orders_customer       │
│        Cost: 0.00..4.19                             │
│        Actual Time: 0.012..0.012 rows=1500          │
│        Index Cond: (customer_id = 123)              │
│└─────────────────────────────────────────────────────┘
│  Planning Time: 0.234 ms                            │
│  Execution Time: 15.678 ms                          │
└─────────────────────────────────────────────────────┘
```

#### Reading MySQL EXPLAIN Output

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

+----+-------------+-------+------------+------+---------------+
| id | select_type | table | partitions | type | possible_keys |
+----+-------------+-------+------------+------+---------------+
|  1 | SIMPLE      | orders| NULL       | ref  | idx_customer  |
+----+-------------+-------+------------+------+---------------+
|  1 | SIMPLE      | orders| NULL       | ALL  | NULL          |  ← Full table scan!
+----+-------------+-------+------------+------+---------------+
```

### Query Optimization Techniques

#### 1. SELECT Optimization

```sql
-- ❌ Bad: SELECT * retrieves unnecessary data
SELECT * FROM users WHERE id = 123;

-- ✅ Good: Select only needed columns
SELECT id, name, email FROM users WHERE id = 123;

-- ✅ Better: Use covering index
SELECT id, name, email FROM users WHERE email = 'user@example.com';
-- Index: CREATE INDEX idx_users_email_name ON users(email, name);
```

#### 2. JOIN Optimization

```sql
-- ❌ Bad: Unnecessary joins
SELECT u.*, c.*, o.*, oi.*, p.*
FROM users u
JOIN customers c ON u.id = c.user_id
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.id = 123;

-- ✅ Good: Only join what's needed
SELECT u.name, o.total, oi.quantity
FROM users u
JOIN orders o ON u.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
WHERE u.id = 123;
```

#### 3. WHERE Clause Optimization

```sql
-- ❌ Bad: Function on indexed column prevents index use
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ Good: Range query allows index use
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
  AND created_at < '2025-01-01';

-- ❌ Bad: Leading wildcard prevents index use
SELECT * FROM users WHERE email LIKE '%@example.com';

-- ✅ Good: Leading prefix allows index use
SELECT * FROM users WHERE email LIKE 'john%@example.com';
```

#### 4. Subquery vs JOIN

```sql
-- ❌ Bad: Correlated subquery (executes per row)
SELECT u.name, 
       (SELECT COUNT(*) FROM orders WHERE customer_id = u.id) as order_count
FROM users u;

-- ✅ Good: JOIN with aggregation
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.customer_id
GROUP BY u.id, u.name;
```

#### 5. LIMIT for Large Result Sets

```sql
-- ❌ Bad: Returning all 10 million rows
SELECT * FROM logs ORDER BY created_at DESC;

-- ✅ Good: Paginate results
SELECT * FROM logs ORDER BY created_at DESC LIMIT 100 OFFSET 0;
SELECT * FROM logs ORDER BY created_at DESC LIMIT 100 OFFSET 100;

-- ✅ Better: Keyset pagination (faster for deep pages)
SELECT * FROM logs 
WHERE created_at < '2024-01-01 12:00:00'
ORDER BY created_at DESC LIMIT 100;
```

### Advanced Optimization Techniques

#### Query Hints

```sql
-- MySQL: Force index usage
SELECT * FROM orders 
FORCE INDEX (idx_orders_customer) 
WHERE customer_id = 123;

-- PostgreSQL: Enable/disable parallel query
SET max_parallel_workers_per_gather = 4;
SELECT COUNT(*) FROM large_table;

-- MySQL: Join optimization hints
SELECT /*+ BKA(o) BNL(c) */ *
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

#### Materialized Views

```sql
-- Create materialized view for complex query
CREATE MATERIALIZED VIEW mv_customer_order_summary AS
SELECT 
    c.id,
    c.name,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Refresh periodically
REFRESH MATERIALIZED VIEW mv_customer_order_summary;

-- Query the materialized view (fast!)
SELECT * FROM mv_customer_order_summary WHERE id = 123;
```

#### Partitioning for Large Tables

```sql
-- Range partitioning by date
CREATE TABLE logs (
    id BIGINT,
    message TEXT,
    created_at TIMESTAMP
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Query only relevant partition
SELECT * FROM logs WHERE created_at > '2024-06-01';
-- Only scans p2024 partition!
```

---

## When to Use

### Use EXPLAIN/EXPLAIN ANALYZE When:
- ✅ Query takes longer than expected
- ✅ Debugging performance issues
- ✅ Before deploying new queries to production
- ✅ After schema or index changes

### Use Covering Indexes When:
- ✅ Query selects specific, known columns
- ✅ High-frequency queries
- ✅ Index size is acceptable tradeoff

### Use Materialized Views When:
- ✅ Complex aggregations run frequently
- ✅ Data can be stale (not real-time)
- ✅ Read-heavy workloads

### Use Query Rewriting When:
- ✅ Subqueries cause performance issues
- ✅ JOINs can be simplified
- ✅ Unnecessary data is being retrieved

---

## Tradeoffs

| Technique | Benefit | Cost |
|-----------|---------|------|
| **Covering Index** | No table lookup needed | Larger index, slower writes |
| **Materialized View** | Fast complex queries | Storage, refresh overhead |
| **Query Caching** | Instant repeated queries | Memory, stale data risk |
| **Denormalization** | Fewer JOINs needed | Data redundancy, consistency |
| **Partitioning** | Faster filtered queries | Complexity, maintenance |

### Quantitative Example

```
Original Query: 15.2 seconds, 10M rows examined

After Optimization:
1. Add covering index: 2.1 seconds
2. Rewrite subquery to JOIN: 0.8 seconds  
3. Add LIMIT for pagination: 0.05 seconds (per page)

Total improvement: 304x faster
```

---

## Implementation Example

### Comprehensive Query Optimization Process

```sql
-- Step 1: Identify slow queries
-- PostgreSQL: Query log analysis
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- Step 2: Analyze execution plan
EXPLAIN ANALYZE 
SELECT u.name, o.total, p.name as product_name
FROM users u
JOIN orders o ON u.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.status = 'active'
  AND o.created_at > '2024-01-01';

-- Step 3: Create supporting indexes
CREATE INDEX idx_orders_customer_created 
ON orders(customer_id, created_at);

CREATE INDEX idx_order_items_order_product
ON order_items(order_id, product_id);

-- Step 4: Rewrite query if needed
SELECT u.name, o.total, p.name as product_name
FROM users u
INNER JOIN orders o ON u.id = o.customer_id 
    AND o.created_at > '2024-01-01'  -- Filter early
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
WHERE u.status = 'active';

-- Step 5: Verify improvement
EXPLAIN ANALYZE <optimized_query>;
```

### Query Optimization Checklist

```sql
-- ✅ Does query select only needed columns?
-- ✅ Are appropriate indexes in place?
-- ✅ Can JOINs be simplified or eliminated?
-- ✅ Are WHERE conditions sargable (index-friendly)?
-- ✅ Can subqueries be converted to JOINs?
-- ✅ Is pagination used for large result sets?
-- ✅ Can query be cached or materialized?
-- ✅ Are statistics up to date? (ANALYZE)
```

---

## Anti-Pattern

### ❌ Anti-Pattern: N+1 Query Problem

```python
# Bad: N+1 queries
def get_orders_with_customers(order_ids):
    orders = Order.query.filter(Order.id.in_(order_ids)).all()  # 1 query
    result = []
    for order in orders:
        order.customer = Customer.query.get(order.customer_id)  # N queries!
        result.append(order)
    return result

# For 100 orders: 101 queries executed!
```

### ❌ Anti-Pattern: SELECT * in Production

```sql
-- Bad: Retrieving all columns including large text blobs
SELECT * FROM orders;  -- May include 10KB JSON column!

-- Good: Only needed columns
SELECT id, customer_id, total, status, created_at FROM orders;
```

### ❌ Anti-Pattern: OR Conditions Preventing Index Use

```sql
-- Bad: OR prevents index use
SELECT * FROM products 
WHERE category_id = 1 OR brand_id = 5;

-- Good: UNION combines indexed queries
SELECT * FROM products WHERE category_id = 1
UNION
SELECT * FROM products WHERE brand_id = 5;
```

### ✅ Correct Approach

```python
# Good: Eager loading with JOIN
def get_orders_with_customers(order_ids):
    return Order.query\
        .filter(Order.id.in_(order_ids))\
        .join(Customer)\
        .all()
    # Single query with JOIN!
```

```sql
-- Good: Use IN for multiple values
SELECT * FROM products WHERE category_id IN (1, 2, 3);

-- Good: Use EXISTS for membership checks
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM order_items oi 
    WHERE oi.order_id = o.id AND oi.quantity > 10
);
```

---

## Related Patterns

- **[Indexing Strategies](03-Indexing-Strategies.md)** - Indexes are fundamental to query optimization
- **[Relational Design](01-Relational.md)** - Schema design impacts query performance
- **[NoSQL Design](02-NoSQL.md)** - Different query optimization in NoSQL
- **[Caching Patterns](../02-Design-Patterns/02-Structural-Patterns.md)** - Query result caching
- **[Performance Patterns](../01-System-Design/07-Performance-Patterns.md)** - Database performance optimization