# Replication

## Title & Summary

Replication maintains multiple copies of data across different nodes or data centers to improve availability, fault tolerance, and read performance. This pattern covers master-slave, multi-master, and master-master replication strategies along with consistency models and conflict resolution techniques.

## Problem Statement

Single data copies face critical limitations:
- **Single Point of Failure**: Node failure makes data unavailable
- **Limited Read Scalability**: Single node handles all reads
- **Geographic Latency**: Users far from data center experience delays
- **Disaster Recovery**: Data center failure loses all data
- **Maintenance Windows**: Updates require downtime

Without replication:
- System unavailable during node failures
- Read performance cannot scale
- High latency for geographically distributed users
- No disaster recovery capability

## Solution

### Replication Topologies

**Master-Slave (Primary-Replica)**
```
        Write
         ↓
    ┌─────────┐
    │ MASTER  │ ← Primary for writes
    └────┬────┘
         │ Replicate
    ┌────┴────┬─────────┬─────────┐
    ↓         ↓         ↓         ↓
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│SLAVE1 │ │SLAVE2 │ │SLAVE3 │ │SLAVE4 │
└───────┘ └───────┘ └───────┘ └───────┘
    ↑         ↑         ↑         ↑
    └──────── Reads (can scale) ────────┘
```

**Characteristics**:
- Single master handles all writes
- Slaves replicate from master (async or sync)
- Reads can be served from any slave
- Simple conflict resolution (master is authoritative)

**Multi-Master**
```
    ┌─────────┐     ┌─────────┐
    │MASTER 1 │────▶│MASTER 2 │
    │(Region A)│◀───│(Region B)│
    └─────────┘     └─────────┐
                              │
                          ┌───┴───┐
                          │MASTER 3│
                          │(Region C)│
                          └───────┘
```

**Characteristics**:
- Multiple masters accept writes
- Changes replicated between masters
- Requires conflict resolution
- Higher availability for writes

**Master-Master (Active-Active)**
```
    ┌─────────────────────────────────┐
    │         Load Balancer           │
    └──────────┬────────────────┬──────┘
               ↓                ↓
        ┌─────────┐        ┌─────────┐
        │ MASTER  │◀──────▶│ MASTER  │
        │   A     │  Sync  │   B     │
        └─────────┘        └─────────┘
```

**Characteristics**:
- Both masters accept reads and writes
- Synchronous or asynchronous replication
- Complex conflict resolution required

### Consistency Models

**Strong Consistency**
```python
def strong_consistency_read(key: str) -> Any:
    """Read returns most recent write"""
    # All replicas must agree before returning
    quorum_read(replicas, key)
    return value

# Guarantees:
# - Read always sees latest write
# - Linearizable operations
# - Higher latency
```

**Eventual Consistency**
```python
def eventual_consistency_read(key: str) -> Any:
    """Read may return stale data temporarily"""
    # Read from any replica
    return random_replica().get(key)

# Guarantees:
# - If no new writes, all reads eventually return last write
# - Lower latency
# - May read stale data
```

**Causal Consistency**
```python
def causal_consistency_read(key: str) -> Any:
    """Respects cause-effect relationships"""
    # Track dependencies between operations
    # Read sees all causally related writes
    return read_with_causal_dependencies(key)
```

**Read-Your-Writes Consistency**
```python
def read_your_writes_read(session_id: str, key: str) -> Any:
    """Client always sees their own writes"""
    # Route reads to replica with client's writes
    # Or use session-based routing
    return session_replica(session_id).get(key)
```

### Conflict Resolution Strategies

**Last-Write-Wins (LWW)**
```python
def lww_resolve(values: List[Tuple[timestamp, value]]) -> Any:
    """Resolve by highest timestamp"""
    return max(values, key=lambda x: x[0])[1]
```

**Vector Clocks**
```python
class VectorClock:
    def __init__(self, node_id: int, num_nodes: int):
        self.node_id = node_id
        self.clock = [0] * num_nodes
    
    def tick(self):
        self.clock[self.node_id] += 1
    
    def update(self, other: 'VectorClock'):
        for i in range(len(self.clock)):
            self.clock[i] = max(self.clock[i], other.clock[i])
    
    def happens_before(self, other: 'VectorClock') -> bool:
        """True if this clock happens before other"""
        all_less_or_equal = all(
            self.clock[i] <= other.clock[i] 
            for i in range(len(self.clock))
        )
        at_least_one_less = any(
            self.clock[i] < other.clock[i] 
            for i in range(len(self.clock))
        )
        return all_less_or_equal and at_least_one_less
    
    def concurrent(self, other: 'VectorClock') -> bool:
        """True if events are concurrent (conflict)"""
        return not self.happens_before(other) and not other.happens_before(self)
```

**CRDTs (Conflict-Free Replicated Data Types)**
```python
class G-Counter:
    """Grow-only counter CRDT"""
    def __init__(self, node_id: int, nodes: List[int]):
        self.node_id = node_id
        self.values = {node: 0 for node in nodes}
    
    def increment(self):
        self.values[self.node_id] += 1
    
    def value(self) -> int:
        return sum(self.values.values())
    
    def merge(self, other: 'G-Counter'):
        """Merge two counters (element-wise max)"""
        for node in self.values:
            self.values[node] = max(
                self.values[node], 
                other.values.get(node, 0)
            )
```

## When to Use

**Use Master-Slave When**:
- Write throughput is lower than read throughput
- Simple conflict resolution needed
- Single source of truth acceptable
- Building read-heavy applications

**Use Multi-Master When**:
- Need write availability across regions
- Can handle conflict resolution complexity
- Building globally distributed applications
- Write latency critical for user experience

**Use Strong Consistency When**:
- Financial transactions
- Inventory management
- User account balances
- Cannot tolerate stale reads

**Use Eventual Consistency When**:
- Social media feeds
- Comment counts
- Recommendation systems
- Can tolerate temporary staleness

**Use Read-Your-Writes When**:
- User editing their own content
- Shopping cart operations
- Form submissions
- User-specific data

## Tradeoffs

| Model | Consistency | Availability | Latency | Complexity |
|-------|------------|--------------|---------|------------|
| Strong | High | Medium | High | Medium |
| Eventual | Low | High | Low | Low |
| Causal | Medium | High | Medium | High |
| Read-Your-Writes | Medium | High | Low | Medium |

**Replication Lag Considerations**:
```
Sync Replication:    Async Replication:
Write → [M→S] → ACK   Write → M → ACK
       (blocking)        (background)
       
Latency: High           Latency: Low
Durability: High        Durability: Medium (risk of loss)
```

**CAP Theorem Tradeoffs**:
- **CP Systems**: Sacrifice availability during partitions (e.g., ZooKeeper, etcd)
- **AP Systems**: Sacrifice consistency during partitions (e.g., Cassandra, DynamoDB)

## Implementation Example

### Python Replication Manager

```python
from typing import Dict, List, Any, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum
from collections import defaultdict
import time
import hashlib
import threading

class ConsistencyLevel(Enum):
    STRONG = "strong"
    EVENTUAL = "eventual"
    CAUSAL = "causal"
    READ_YOUR_WRITES = "read_your_writes"

class ReplicationType(Enum):
    MASTER_SLAVE = "master_slave"
    MULTI_MASTER = "multi_master"
    MASTER_MASTER = "master_master"

@dataclass
class ReplicatedValue:
    value: Any
    timestamp: float
    vector_clock: Dict[int, int] = field(default_factory=dict)
    node_id: int = 0

class Replica:
    """Single replica node"""
    def __init__(self, node_id: int):
        self.node_id = node_id
        self.storage: Dict[str, ReplicatedValue] = {}
        self.vector_clock: Dict[int, int] = {node_id: 0}
    
    def write(self, key: str, value: Any) -> ReplicatedValue:
        """Write value with vector clock"""
        self.vector_clock[self.node_id] = self.vector_clock.get(self.node_id, 0) + 1
        
        replicated_value = ReplicatedValue(
            value=value,
            timestamp=time.time(),
            vector_clock=dict(self.vector_clock),
            node_id=self.node_id
        )
        self.storage[key] = replicated_value
        return replicated_value
    
    def read(self, key: str) -> Optional[ReplicatedValue]:
        """Read value from storage"""
        return self.storage.get(key)
    
    def replicate_from(self, value: ReplicatedValue):
        """Replicate value from another node"""
        # Merge vector clocks
        for node_id, clock in value.vector_clock.items():
            self.vector_clock[node_id] = max(
                self.vector_clock.get(node_id, 0), 
                clock
            )
        
        # Check if should update (newer or concurrent)
        existing = self.storage.get(key)
        if existing is None or self._should_replace(existing, value):
            self.storage[key] = value
    
    def _should_replace(self, existing: ReplicatedValue, new: ReplicatedValue) -> bool:
        """Determine if new value should replace existing"""
        # Check vector clock ordering
        if self._happens_before(existing.vector_clock, new.vector_clock):
            return True
        if self._happens_before(new.vector_clock, existing.vector_clock):
            return False
        # Concurrent - use timestamp as tiebreaker
        return new.timestamp > existing.timestamp
    
    def _happens_before(self, vc1: Dict[int, int], vc2: Dict[int, int]) -> bool:
        """Check if vc1 happens before vc2"""
        all_less_or_equal = all(
            vc1.get(k, 0) <= vc2.get(k, 0) 
            for k in set(vc1.keys()) | set(vc2.keys())
        )
        at_least_one_less = any(
            vc1.get(k, 0) < vc2.get(k, 0) 
            for k in set(vc1.keys()) | set(vc2.keys())
        )
        return all_less_or_equal and at_least_one_less

class ReplicationManager:
    """Manages replication across multiple replicas"""
    
    def __init__(self, num_replicas: int, 
                 replication_type: ReplicationType,
                 consistency_level: ConsistencyLevel):
        self.replicas = [Replica(i) for i in range(num_replicas)]
        self.replication_type = replication_type
        self.consistency_level = consistency_level
        self.master_id = 0  # For master-slave mode
    
    def write(self, key: str, value: Any, node_id: Optional[int] = None) -> bool:
        """Write with replication"""
        if self.replication_type == ReplicationType.MASTER_SLAVE:
            return self._master_slave_write(key, value)
        elif self.replication_type == ReplicationType.MULTI_MASTER:
            return self._multi_master_write(key, value, node_id)
        else:
            raise NotImplementedError
    
    def _master_slave_write(self, key: str, value: Any) -> bool:
        """Write to master, replicate to slaves"""
        # Write to master
        self.replicas[self.master_id].write(key, value)
        
        # Async replicate to slaves
        for i, replica in enumerate(self.replicas):
            if i != self.master_id:
                value = self.replicas[self.master_id].read(key)
                if value:
                    replica.replicate_from(value)
        
        return True
    
    def _multi_master_write(self, key: str, value: Any, node_id: int) -> bool:
        """Write to any master"""
        self.replicas[node_id].write(key, value)
        # Replication happens asynchronously
        return True
    
    def read(self, key: str, node_id: Optional[int] = None) -> Any:
        """Read with consistency guarantees"""
        if self.consistency_level == ConsistencyLevel.STRONG:
            return self._strong_read(key)
        elif self.consistency_level == ConsistencyLevel.EVENTUAL:
            return self._eventual_read(key, node_id)
        elif self.consistency_level == ConsistencyLevel.READ_YOUR_WRITES:
            return self._read_your_writes_read(key, node_id)
        else:
            raise NotImplementedError
    
    def _strong_read(self, key: str) -> Any:
        """Read with strong consistency (quorum)"""
        values = []
        for replica in self.replicas:
            value = replica.read(key)
            if value:
                values.append(value)
        
        if not values:
            return None
        
        # Return value with highest vector clock
        return max(values, key=lambda v: sum(v.vector_clock.values())).value
    
    def _eventual_read(self, key: str, node_id: Optional[int] = None) -> Any:
        """Read from any replica (eventual consistency)"""
        if node_id is not None:
            return self.replicas[node_id].read(key)
        # Random replica
        import random
        replica = random.choice(self.replicas)
        value = replica.read(key)
        return value.value if value else None
    
    def _read_your_writes_read(self, key: str, client_node_id: int) -> Any:
        """Read from client's node for read-your-writes consistency"""
        value = self.replicas[client_node_id].read(key)
        return value.value if value else None

# Example Usage
if __name__ == "__main__":
    # Create replication manager with 3 replicas
    manager = ReplicationManager(
        num_replicas=3,
        replication_type=ReplicationType.MASTER_SLAVE,
        consistency_level=ConsistencyLevel.EVENTUAL
    )
    
    # Write data
    manager.write("user:1", {"name": "Alice", "email": "alice@example.com"})
    
    # Read data
    value = manager.read("user:1")
    print(f"Read value: {value}")
    
    # Check replication
    for i, replica in enumerate(manager.replicas):
        val = replica.read("user:1")
        print(f"Replica {i}: {val.value if val else None}")
```

## Anti-Pattern

### ❌ Anti-Pattern: Ignoring Replication Lag

```python
# WRONG: Write then immediately read expecting consistency
def update_and_verify_broken(user_id: int, new_email: str):
    # Write to master
    db.write(f"user:{user_id}", {"email": new_email})
    
    # Immediately read from slave - may be stale!
    result = slave_db.read(f"user:{user_id}")
    
    if result["email"] != new_email:
        raise Error("Write not visible!")  # May fail due to lag

# Problems:
# 1. Non-deterministic behavior
# 2. User sees stale data
# 3. Verification logic fails
```

### ❌ Anti-Pattern: No Conflict Resolution

```python
# WRONG: Multi-master without conflict resolution
def multi_master_write_broken(node_id: int, key: str, value: Any):
    replicas[node_id].storage[key] = value
    # No conflict detection or resolution!

# Problems:
# 1. Lost updates
# 2. Split brain scenarios
# 3. Data inconsistency
```

### ✅ Correct Pattern

```python
# CORRECT: Handle replication lag appropriately
def update_and_verify_correct(user_id: int, new_email: str):
    # Write to master
    db.write(f"user:{user_id}", {"email": new_email})
    
    # Read from master for immediate consistency
    result = master_db.read(f"user:{user_id}")
    
    # Or wait for replication if reading from slave
    time.sleep(REPLICATION_LAG_MS / 1000)
    result = slave_db.read(f"user:{user_id}")
    
    return result["email"] == new_email

# CORRECT: Use vector clocks for conflict resolution
def resolve_conflict(values: List[ReplicatedValue]) -> ReplicatedValue:
    # Find causally latest value
    latest = max(values, key=lambda v: sum(v.vector_clock.values()))
    
    # Check for concurrent updates
    concurrent = [v for v in values if v != latest and 
                  not happens_before(v.vector_clock, latest.vector_clock)]
    
    if concurrent:
        # Merge or use application-specific resolution
        return merge_values([latest] + concurrent)
    
    return latest
```

## Related Patterns

- **[CAP Theorem](./01-CAP-Theorem.md)**: Understanding consistency tradeoffs
- **[Partitioning](./03-Partitioning.md)**: Combining replication with partitioning
- **[Consensus Algorithms](./02-Consensus-Algorithms.md)**: Achieving strong consistency
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)**: Using replication for availability
- **[Database Design](../08-Database-Design/01-Relational.md)**: Database-level replication
- **[Availability Patterns](../01-System-Design/06-Availability-Patterns.md)**: Building highly available systems

---

*Last Updated: 2024-01-15*