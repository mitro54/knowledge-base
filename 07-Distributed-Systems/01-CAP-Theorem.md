# CAP Theorem

## Title & Summary

**CAP Theorem** (also known as Brewer's Theorem) states that in a distributed data store, it is impossible to simultaneously provide more than two out of the following three guarantees: **Consistency**, **Availability**, and **Partition Tolerance**. This fundamental theorem shapes the design decisions for distributed systems and database systems.

---

## Problem Statement

Distributed systems face inherent challenges when trying to provide strong guarantees:

1. **Network Unreliability**: Network partitions can occur, separating nodes
2. **Consistency Requirements**: Clients expect to read the latest written value
3. **Availability Expectations**: Clients expect the system to respond to requests
4. **Impossible Trinity**: Cannot have all three properties simultaneously

### The Three Properties

```
┌─────────────────────────────────────────────────────────┐
│                    CAP THEOREM                          │
├───────────────┬───────────────┬─────────────────────────┤
│ Consistency   │ Availability  │ Partition Tolerance     │
│ (C)           │ (A)           │ (P)                     │
├───────────────┼───────────────┼─────────────────────────┤
│ Every read    │ Every request │ System continues        │
│ receives the  │ receives a    │ operating despite       │
│ most recent    │ response      │ network partitions      │
│ write          │ (not error)   │                         │
└───────────────┴───────────────┴─────────────────────────┘
                    
                    YOU CAN ONLY PICK TWO
```

---

## Solution

### Understanding the Tradeoffs

#### CA Systems (Consistency + Availability)
- **Sacrifice**: Partition Tolerance
- **Behavior**: When partition occurs, system becomes unavailable
- **Examples**: Traditional relational databases (single node)
- **Use Case**: Financial transactions requiring strong consistency

```python
# CA System behavior during partition
def handle_request(during_partition=False):
    if during_partition:
        raise UnavailableError("System unavailable due to partition")
    return process_request()  # Consistent response
```

#### CP Systems (Consistency + Partition Tolerance)
- **Sacrifice**: Availability
- **Behavior**: During partition, some operations fail to maintain consistency
- **Examples**: ZooKeeper, HBase, MongoDB (with strong consistency)
- **Use Case**: Systems where data correctness is critical

```python
# CP System behavior during partition
def handle_write(key, value, during_partition=False):
    if during_partition and not can_reach_quorum():
        raise ConsistencyError("Cannot guarantee consistency")
    return replicate_to_quorum(key, value)

def handle_read(key, during_partition=False):
    if during_partition and not can_reach_quorum():
        raise ConsistencyError("Cannot read consistent value")
    return read_from_quorum(key)
```

#### AP Systems (Availability + Partition Tolerance)
- **Sacrifice**: Consistency (provide eventual consistency)
- **Behavior**: During partition, nodes respond with potentially stale data
- **Examples**: Cassandra, DynamoDB, Riak
- **Use Case**: Systems where availability is more important than immediate consistency

```python
# AP System behavior during partition
def handle_write(key, value, during_partition=False):
    # Write to local node, replicate when possible
    write_to_local(key, value)
    if not during_partition:
        replicate_async(key, value)
    return Success()

def handle_read(key, during_partition=False):
    # Read from local node (may be stale)
    return read_from_local(key)
```

### CAP Decision Matrix

```
                    Network Partition?
                       │
                    YES│NO
                      ▼
              ┌───────┴───────┐
              │               │
         Can you         Must you
         tolerate        choose between
         inconsistency?  C and A?
              │               │
           YES│NO          YES│NO
              ▼               ▼
           AP System      CP System
         (Eventual      (Strong
          Consistency)    Consistency)
```

---

## When to Use

### Choose CP When:
- **Financial Systems**: Banking, payment processing
- **Inventory Management**: Prevent overselling
- **Healthcare Systems**: Patient records must be accurate
- **Configuration Systems**: Consistent configuration critical
- **Authorization Systems**: Access decisions must be consistent

```python
# CP choice for banking system
class BankingSystem:
    def transfer(from_account, to_account, amount):
        # Must be consistent - cannot lose money
        if not can_reach_quorum():
            raise TransactionFailed("Try again later")
        deduct(from_account, amount)
        add(to_account, amount)
```

### Choose AP When:
- **Social Media**: Stale likes/comments acceptable
- **E-commerce Catalogs**: Product info can be eventually consistent
- **Content Delivery**: Cache can be stale
- **Collaborative Editing**: Resolve conflicts later
- **IoT Sensor Data**: Missing/delayed data acceptable

```python
# AP choice for social media
class SocialMediaFeed:
    def get_feed(user_id):
        # Stale data acceptable - prioritize availability
        return cache.get(f"feed:{user_id}") or generate_feed(user_id)
    
    def post_status(user_id, content):
        # Fire and forget - replicate eventually
        db.write_local(content)
        replicate_async(content)  # May fail, that's ok
```

### Choose CA When:
- **Single-node databases**: No partition possibility
- **Local transactions**: No distribution involved
- **Simple applications**: Don't need distribution

---

## Tradeoffs

| System Type | Advantages | Disadvantages |
|-------------|------------|---------------|
| **CP** | Strong consistency, data correctness | Lower availability during partitions, higher latency |
| **AP** | High availability, low latency | Stale reads, complex conflict resolution |
| **CA** | Simple mental model | No fault tolerance, single point of failure |

### Practical Considerations

```
Real-world spectrum (not binary):

CP ─────────────────────────────────────── AP
     ↑                                    ↑
   MongoDB                            Cassandra
   (tunable)                          (tunable)

Most systems allow tuning between extremes
```

### Tunable Consistency (MongoDB Example)

```javascript
// Read concern levels
db.collection.find(query, {
    readConcern: "local"    // AP - fastest, may be stale
    readConcern: "majority" // CP - consistent, slower
})

// Write concern levels
db.collection.insert(doc, {
    w: 1,        // AP - acknowledge on primary only
    w: "majority" // CP - wait for majority acknowledgment
})
```

---

## Implementation Example

### CAP-Aware Distributed Database Client

```python
from enum import Enum
from typing import Optional

class ConsistencyLevel(Enum):
    EVENTUAL = "eventual"      # AP
    QUORUM = "quorum"          # CP
    ONE = "one"                # AP
    ALL = "all"                # CP

class AvailabilityLevel(Enum):
    HIGH = "high"              # AP
    STANDARD = "standard"      # Balanced
    LOW = "low"                # CP

class DistributedDatabase:
    def __init__(self, nodes: list, mode: str = "AP"):
        self.nodes = nodes
        self.mode = mode  # "AP", "CP", or "CA"
        
    def write(self, key: str, value: str, 
              consistency: ConsistencyLevel = None) -> bool:
        """Write with specified consistency level"""
        effective_consistency = consistency or self._default_consistency()
        
        if effective_consistency == ConsistencyLevel.ALL:
            # CP - write to all nodes
            return self._write_to_all(key, value)
        elif effective_consistency == ConsistencyLevel.QUORUM:
            # CP - write to quorum
            return self._write_to_quorum(key, value)
        else:
            # AP - write to one, replicate async
            return self._write_to_one(key, value)
    
    def read(self, key: str,
             consistency: ConsistencyLevel = None) -> Optional[str]:
        """Read with specified consistency level"""
        effective_consistency = consistency or self._default_consistency()
        
        if effective_consistency == ConsistencyLevel.ALL:
            # CP - read from all, verify consistency
            return self._read_verify_all(key)
        elif effective_consistency == ConsistencyLevel.QUORUM:
            # CP - read from quorum
            return self._read_quorum(key)
        else:
            # AP - read from any available node
            return self._read_any(key)
    
    def _default_consistency(self) -> ConsistencyLevel:
        if self.mode == "CP":
            return ConsistencyLevel.QUORUM
        else:  # AP mode
            return ConsistencyLevel.ONE

# Usage examples
ap_db = DistributedDatabase(nodes, mode="AP")  # Social media feed
cp_db = DistributedDatabase(nodes, mode="CP")  # Banking system

# AP: Fast writes, eventual consistency
ap_db.write("user:123:feed", feed_data)  # ConsistencyLevel.ONE

# CP: Slower writes, strong consistency
cp_db.write("account:123:balance", balance)  # ConsistencyLevel.QUORUM
```

### Dynamic CAP Adjustment

```python
class AdaptiveDistributedSystem:
    def __init__(self):
        self.mode = "AP"  # Default to availability
        
    def adjust_for_context(self, context: str):
        """Dynamically adjust CAP tradeoff based on context"""
        if context == "peak_traffic":
            self.mode = "AP"  # Availability critical
        elif context == "financial_batch":
            self.mode = "CP"  # Consistency critical
        elif context == "maintenance":
            self.mode = "CA"  # Single node, no partition risk
            
    def handle_request(self, request):
        if request.type == "read_profile":
            return self.read_ap(request.key)  # Stale ok
        elif request.type == "read_balance":
            return self.read_cp(request.key)  # Must be consistent
```

---

## Anti-Pattern

### ❌ Anti-Pattern: "Ignoring CAP Tradeoffs"

Assuming you can have strong consistency AND high availability during partitions:

```python
# ANTI-PATTERN: Expecting impossible guarantees
class NaiveDistributedSystem:
    def write(self, key, value):
        # Assumes always consistent AND available
        for node in all_nodes:
            node.write(key, value)  # Blocks until ALL succeed
        return Success()
    
    def read(self, key):
        # Assumes always returns latest value
        return any_available_node.read(key)  # May be stale!
```

**Problems:**
- **False Expectations**: System won't provide both guarantees during partition
- **Poor User Experience**: Either slow (waiting for all) or inconsistent
- **Hidden Bugs**: Inconsistencies appear only during network issues

### ✅ Correct Approach: Explicit Tradeoffs

```python
# CORRECT: Explicit about tradeoffs
class HonestDistributedSystem:
    def write(self, key, value, consistency_level):
        if consistency_level == "strong":
            # CP: May fail during partition
            return self.write_to_quorum(key, value)
        else:
            # AP: Always succeeds, eventual consistency
            return self.write_to_one(key, value)
    
    def read(self, key, consistency_level):
        if consistency_level == "strong":
            # CP: May fail during partition
            return self.read_from_quorum(key)
        else:
            # AP: May return stale data
            return self.read_from_any(key)
```

---

## Related Patterns

- **[Consensus Algorithms](./02-Consensus-Algorithms.md)** - Achieving agreement in distributed systems
- **[Replication](./04-Replication.md)** - Data copying strategies affecting CAP choices
- **[Distributed Transactions](./05-Distributed-Transactions.md)** - ACID vs BASE approaches
- **[NoSQL Design](../08-Database-Design/02-NoSQL-Design.md)** - Database choices based on CAP
- **[Scalability Patterns](../01-System-Design/04-Scalability-Patterns.md)** - Scaling considerations with CAP
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Handling partitions gracefully