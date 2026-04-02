# NoSQL Database Design

## Title & Summary

**NoSQL Database Design** encompasses non-relational data storage approaches that prioritize flexibility, scalability, and performance over traditional ACID guarantees. NoSQL databases come in four main types: document stores, key-value stores, column-family stores, and graph databases, each optimized for specific use cases and data access patterns.

## Problem Statement

Relational databases face challenges in modern applications:

- **Horizontal Scalability**: Difficult to scale beyond single server capacity
- **Schema Rigidity**: Changing schema requires migrations and downtime
- **JSON/Document Storage**: Storing hierarchical data requires complex joins or text fields
- **High Velocity Data**: Struggles with massive write throughput requirements
- **Flexible Data Models**: Cannot easily store varying attributes per record
- **Geospatial Queries**: Limited native support for location-based queries

**Example of Relational Limitations:**
```json
// Storing flexible product data in relational database
// Problem: Different product types have different attributes

// Electronics need: voltage, wattage, warranty_months
// Clothing needs: size, color, material, fit_type
// Books need: isbn, author, pages, publisher

// Relational approach requires either:
// 1. Many nullable columns (wasteful)
// 2. EAV pattern (complex queries)
// 3. JSON column (loses queryability)
```

## Solution

### 1. Document Stores (MongoDB, CouchDB)

**Store semi-structured documents with flexible schemas.**

```javascript
// MongoDB schema design for e-commerce

// Products collection - flexible schema
const productSchema = {
    _id: ObjectId,
    sku: String,
    name: String,
    description: String,
    price: Number,
    currency: String,
    // Flexible attributes based on product type
    attributes: {
        type: String,  // 'electronics', 'clothing', 'book'
        // Electronics-specific
        voltage: Number,
        wattage: Number,
        warranty_months: Number,
        // Clothing-specific
        sizes: [String],
        colors: [String],
        material: String,
        // Book-specific
        isbn: String,
        author: String,
        pages: Number
    },
    // Embedded documents
    categories: [{
        _id: ObjectId,
        name: String,
        slug: String
    }],
    // Array of embedded reviews
    reviews: [{
        user_id: ObjectId,
        rating: Number,
        comment: String,
        created_at: Date
    }],
    metadata: {
        created_at: Date,
        updated_at: Date,
        version: Number
    }
};

// Query examples
db.products.find({
    'attributes.type': 'electronics',
    price: { $gte: 100, $lte: 500 },
    'attributes.warranty_months': { $gte: 12 }
});

// Aggregation pipeline
db.products.aggregate([
    { $match: { 'attributes.type': 'clothing' } },
    { $group: { 
        _id: '$attributes.material',
        avgPrice: { $avg: '$price' },
        count: { $sum: 1 }
    }},
    { $sort: { count: -1 } }
]);
```

### 2. Key-Value Stores (Redis, DynamoDB)

**Ultra-fast lookups by key, ideal for caching and sessions.**

```python
# Redis data structures for various use cases

import redis

r = redis.Redis(host='localhost', port=6379)

# String - Simple key-value
r.set('user:123:name', 'John Doe')
r.get('user:123:name')  # 'John Doe'

# Hash - Object-like storage
r.hset('user:123', mapping={
    'name': 'John Doe',
    'email': 'john@example.com',
    'age': 30,
    'is_premium': True
})
r.hgetall('user:123')  # All fields
r.hincrby('user:123', 'login_count', 1)  # Atomic increment

# List - Ordered collection
r.lpush('user:123:recent_products', 'prod-456')
r.lrange('user:123:recent_products', 0, 9)  # Last 10 products

# Set - Unique collection
r.sadd('user:123:tags', 'premium', 'verified', 'early_adopter')
r.sismember('user:123:tags', 'premium')  # True

# Sorted Set - Ranked collection
r.zadd('leaderboard', {'player1': 1500, 'player2': 2000})
r.zrevrange('leaderboard', 0, 9, withscores=True)  # Top 10

# Session storage pattern
def create_session(user_id, data):
    session_id = generate_uuid()
    expiry = 3600  # 1 hour
    r.hset(f'session:{session_id}', mapping={
        'user_id': user_id,
        'data': json.dumps(data),
        'created_at': time.time()
    })
    r.expire(f'session:{session_id}', expiry)
    return session_id

# Caching pattern with cache-aside
def get_user_with_cache(user_id):
    cache_key = f'user:{user_id}'
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Cache miss - fetch from database
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    if user:
        r.setex(cache_key, 300, json.dumps(user))  # 5 min cache
    return user
```

### 3. Column-Family Stores (Cassandra, HBase)

**Optimized for massive scale writes and time-series data.**

```cql
-- Cassandra CQL for time-series sensor data

-- Keyspace with replication
CREATE KEYSPACE sensor_data
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'datacenter1': 3,
    'datacenter2': 3
};

-- Table for sensor readings (wide column store)
CREATE TABLE sensor_readings (
    sensor_id UUID,
    day_date DATE,
    hour INT,
    minute INT,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    PRIMARY KEY ((sensor_id, day_date), hour, minute)
) WITH CLUSTERING ORDER BY (hour ASC, minute ASC)
  AND gc_grace_seconds = 86400
  AND default_ttl = 2592000;  -- 30 days TTL

-- Insert sensor reading
INSERT INTO sensor_readings 
    (sensor_id, day_date, hour, minute, temperature, humidity, pressure)
VALUES 
    (uuid(), dateOf(now()), hourOf(now()), minuteOf(now()), 23.5, 65.2, 1013.25);

-- Query last hour of readings for a sensor
SELECT * FROM sensor_readings
WHERE sensor_id = ? AND day_date = ?
ORDER BY hour DESC, minute DESC
LIMIT 60;

-- Materialized view for different query pattern
CREATE MATERIALIZED VIEW sensor_readings_by_hour AS
SELECT sensor_id, day_date, hour, 
       avg(temperature) as avg_temp,
       min(temperature) as min_temp,
       max(temperature) as max_temp
FROM sensor_readings
PRIMARY KEY (sensor_id, day_date, hour)
```

### 4. Graph Databases (Neo4j, Amazon Neptune)

**Model relationships as first-class citizens.**

```cypher
// Neo4j Cypher queries for social network

// Create nodes and relationships
CREATE (user:User {id: 'user123', name: 'Alice', email: 'alice@example.com'})
CREATE (user2:User {id: 'user456', name: 'Bob'})
CREATE (post:Post {id: 'post789', content: 'Hello World!', timestamp: 1234567890})

// Create relationships
CREATE (user)-[:FOLLOWS]->(user2)
CREATE (user)-[:POSTED]->(post)
CREATE (user2)-[:LIKED]->(post)

// Query: Find friends of friends
MATCH (user:User {id: 'user123'})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE fof <> user
RETURN fof.name, fof.id

// Query: Find mutual friends
MATCH (user1:User {id: 'user123'})-[:FOLLOWS]-(mutual)-[:FOLLOWS]-(user2:User {id: 'user456'})
RETURN mutual.name

// Query: Recommend friends based on common interests
MATCH (user:User {id: 'user123'})-[:INTERESTED_IN]->(interest)
MATCH (recommended:User)-[:INTERESTED_IN]->(interest)
WHERE recommended.id <> 'user123'
  AND NOT EXISTS {(user)-[:FOLLOWS]->(recommended)}
RETURN recommended.name, 
       count(interest) as common_interests
ORDER BY common_interests DESC
LIMIT 10

// Query: Find shortest path between users
MATCH path = shortestPath(
    (user1:User {id: 'user123'})-[:FOLLOWS*]- (user2:User {id: 'user999'})
)
RETURN path, length(path) as degrees_of_separation
```

## When to Use

| Database Type | Best For | Examples |
|--------------|----------|----------|
| **Document** | Content management, catalogs, user profiles | MongoDB, CouchDB |
| **Key-Value** | Caching, sessions, real-time analytics | Redis, DynamoDB |
| **Column-Family** | Time-series, logs, massive scale writes | Cassandra, HBase |
| **Graph** | Social networks, recommendations, fraud detection | Neo4j, Amazon Neptune |

**Use Document Stores when:**
- Data is hierarchical or semi-structured
- Schema needs to evolve frequently
- Read patterns are document-centric
- Need flexible querying with indexing

**Use Key-Value Stores when:**
- Ultra-low latency reads/writes required
- Simple lookup by key is sufficient
- Caching or session storage needed
- Real-time analytics on simple data

**Use Column-Family Stores when:**
- Massive write throughput required
- Time-series or log data
- Query patterns are predictable
- Horizontal scalability is critical

**Use Graph Databases when:**
- Relationships are as important as data
- Need multi-hop queries
- Social networks or recommendations
- Fraud detection or knowledge graphs

## Tradeoffs

| Aspect | Relational | NoSQL |
|--------|-----------|-------|
| **ACID** | Full support | Varies (often eventual) |
| **Schema** | Rigid, enforced | Flexible, dynamic |
| **Joins** | Native support | Application-level or denormalized |
| **Transactions** | Multi-table | Usually single-document/key |
| **Scalability** | Vertical | Horizontal native |
| **Query Language** | SQL (declarative) | Varies, often API-based |
| **Consistency** | Strong | Often eventual |

## Anti-Pattern

### ❌ NoSQL Anti-Patterns

**1. Using NoSQL for Everything**
```javascript
// ANTI-PATTERN: Using document store for financial transactions
// Problem: Lack of ACID guarantees, no foreign keys

db.orders.insertOne({
    order_id: 'ord-123',
    customer_id: 'cust-456',
    total: 99.99,
    items: [{ product_id: 'prod-789', quantity: 1, price: 99.99 }]
});

// No guarantee that customer exists
// No transaction rollback if insert fails
// No referential integrity

// PATTERN: Use relational for financial data, NoSQL for other data
```

**2. Unbounded Arrays**
```javascript
// ANTI-PATTERN: Storing unlimited data in arrays
db.users.findOneAndUpdate(
    { _id: userId },
    { $push: { activity_log: newActivity } }
);
// Document grows indefinitely, hits size limits

// PATTERN: Use separate collection with references
db.activity_logs.insertOne({
    user_id: userId,
    activity: newActivity,
    timestamp: new Date()
});
```

**3. Missing Indexes**
```javascript
// ANTI-PATTERN: Querying without indexes
db.products.find({ category: 'electronics', price: { $lt: 100 } });
// Full collection scan on millions of documents

// PATTERN: Create compound indexes
db.products.createIndex({ category: 1, price: 1 });
```

**4. Embedding Everything**
```javascript
// ANTI-PATTERN: Embedding frequently updated data
{
    _id: 'order-123',
    customer: { /* Full customer object */ },
    items: [{
        product: { /* Full product object */ },
        quantity: 1
    }]
}
// Updating product name requires updating all orders

// PATTERN: Reference frequently updated data
{
    _id: 'order-123',
    customer_id: 'cust-456',
    items: [{
        product_id: 'prod-789',
        product_name_snapshot: 'Widget',  // Snapshot at order time
        quantity: 1
    }]
}
```

## Examples

### Multi-Database Architecture

```python
# Using different databases for different purposes

class MultiDatabaseApplication:
    def __init__(self):
        # PostgreSQL for transactions and user data
        self.relational_db = PostgreSQLConnection()
        
        # Redis for caching and sessions
        self.cache = RedisConnection()
        
        # MongoDB for product catalog
        self.document_db = MongoDBConnection()
        
        # Cassandra for analytics/events
        self.timeseries_db = CassandraConnection()
    
    def create_order(self, user_id, items):
        # 1. Cache user data (Redis)
        user = self.cache.get(f'user:{user_id}')
        if not user:
            user = self.relational_db.query(
                "SELECT * FROM users WHERE id = ?", user_id
            )
            self.cache.setex(f'user:{user_id}', 300, user)
        
        # 2. Create order transaction (PostgreSQL)
        with self.relational_db.transaction() as tx:
            order_id = tx.execute(
                "INSERT INTO orders (user_id, total) VALUES (?, ?) RETURNING id",
                (user_id, calculate_total(items))
            )
            
            for item in items:
                tx.execute(
                    "INSERT INTO order_items (order_id, product_id, quantity) VALUES (?, ?, ?)",
                    (order_id, item['product_id'], item['quantity'])
                )
        
        # 3. Store order details in document DB (MongoDB)
        self.document_db.orders.insert_one({
            'order_id': order_id,
            'user_id': user_id,
            'items': items,
            'created_at': datetime.utcnow()
        })
        
        # 4. Record analytics event (Cassandra)
        self.timeseries_db.execute(
            "INSERT INTO order_events (event_type, user_id, order_id, timestamp) VALUES (?, ?, ?, ?)",
            ('order_created', user_id, order_id, datetime.utcnow())
        )
        
        return order_id
```

## Related Patterns

- **[Relational Design](./01-Relational.md)** - Traditional database approach
- **[Indexing](./03-Indexing.md)** - Query performance in all database types
- **[Database Migration](./05-Migration.md)** - Schema evolution strategies
- **[Event Sourcing](../07-Distributed-Systems/05-Distributed-Transactions.md)** - Alternative data model

## References

- **Book**: "MongoDB: The Definitive Guide" - Shannon Bradshaw
- **Book**: "Redis in Action" - Josiah L. Carlson
- **Book**: "Cassandra: The Definitive Guide" - Eben Hewitt
- **Paper**: "Dynamo: Amazon's Highly Available Key-value Store" - Dean et al., 2007