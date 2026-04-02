# Partitioning

## Title & Summary

Partitioning (also known as sharding) is the horizontal division of data across multiple database instances or servers. This pattern enables databases to scale beyond the capacity of a single machine by distributing data storage and query processing across multiple nodes.

## Problem Statement

Single database instances face fundamental limitations:
- **Storage Limits**: Cannot store more data than disk capacity
- **Memory Constraints**: Cannot cache more than available RAM
- **CPU Bottlenecks**: Query processing limited by single machine
- **I/O Saturation**: Disk throughput has hard limits
- **Network Bandwidth**: Single network interface limits

When data exceeds single-node capacity:
- Queries become slower
- Inserts/updates become bottlenecks
- Backup and recovery times increase
- Single point of failure exists

## Solution

### Partitioning Strategies

**Horizontal Partitioning (Sharding)**
- Splits rows across multiple partitions
- Each partition has same schema
- Most common for scaling databases

```
Partition 0: | user_id | name    | email              |
             |----1   | Alice   | alice@example.com  |
             |----3   | Charlie | charlie@example.com|
             |----5   | Eve     | eve@example.com    |

Partition 1: | user_id | name    | email              |
             |----2   | Bob     | bob@example.com    |
             |----4   | David   | david@example.com  |
             |----6   | Frank   | frank@example.com  |
```

**Vertical Partitioning**
- Splits columns across multiple partitions
- Useful for wide tables with sparse access
- Related columns stored together

```
Partition A (Frequently Accessed):    Partition B (Infrequently Accessed):
| user_id | name | email |              | user_id | bio | preferences |
|----1   | Alice| alice@|              |----1   | ... | {...}       |
```

**Directory-Partitioned Tables**
- Physical separation by filesystem directories
- Simple to implement
- Good for archival data

### Partition Keys and Functions

**Hash-Based Partitioning**
```python
def hash_partition(key: str, num_partitions: int) -> int:
    """Distribute data using hash function"""
    return hash(key) % num_partitions

# Example: User data distributed by user_id hash
partition = hash_partition(user_id, 16)  # 16 partitions
```

**Range-Based Partitioning**
```python
def range_partition(value: int, ranges: List[Tuple[int, int]]) -> int:
    """Distribute data based on value ranges"""
    for i, (start, end) in enumerate(ranges):
        if start <= value < end:
            return i
    return len(ranges) - 1

# Example: Orders by date ranges
ranges = [
    (20200101, 20210101),  # Partition 0: 2020
    (20210101, 20220101),  # Partition 1: 2021
    (20220101, 20230101),  # Partition 2: 2022
]
```

**Modulo Partitioning**
```python
def modulo_partition(id: int, num_partitions: int) -> int:
    """Simple modulo-based distribution"""
    return id % num_partitions
```

**Consistent Hashing**
```python
import hashlib

class ConsistentHash:
    def __init__(self, nodes: List[str], replicas: int = 150):
        self.replicas = replicas
        self.circle = {}
        self.sorted_keys = []
        
        for node in nodes:
            for i in range(replicas):
                virtual_node = f"{node}-{i}"
                key = int(hashlib.md5(virtual_node.encode()).hexdigest(), 16)
                self.circle[key] = node
                self.sorted_keys.append(key)
        
        self.sorted_keys.sort()
    
    def get_node(self, key: str) -> str:
        """Find node responsible for key"""
        hash_key = int(hashlib.md5(key.encode()).hexdigest(), 16)
        
        for k in self.sorted_keys:
            if hash_key <= k:
                return self.circle[k]
        
        return self.circle[self.sorted_keys[0]]
```

### Rebalancing Strategies

**Full Rebalancing**
- All data redistributed when partition count changes
- Simple but causes significant disruption

**Incremental Rebalancing**
- Move data gradually in background
- Minimizes impact on availability

**V-Shape Rebalancing (Cassandra)**
- Each node gives to right neighbor, receives from left
- Minimizes data movement

## When to Use

**Use Horizontal Partitioning When**:
- Dataset exceeds single machine capacity
- Need to scale read/write throughput
- Query patterns allow partition key selection
- Can tolerate cross-partition query complexity

**Use Vertical Partitioning When**:
- Tables have many columns with different access patterns
- Hot columns accessed frequently, cold columns rarely
- Want to optimize memory cache utilization

**Use Hash Partitioning When**:
- Need even data distribution
- Point queries by partition key are common
- Range queries not required

**Use Range Partitioning When**:
- Range queries are common
- Data has natural ordering (time series, IDs)
- Need to isolate hot partitions

**Use Consistent Hashing When**:
- Nodes frequently added/removed
- Want to minimize data movement on rebalancing
- Building distributed cache or CDN

## Tradeoffs

| Aspect | Hash | Range | Consistent Hash |
|--------|------|-------|-----------------|
| Data Distribution | Even | May skew | Even |
| Range Queries | Expensive | Efficient | Expensive |
| Rebalancing | Full move | Partial | Minimal |
| Hot Spots | Rare | Possible | Rare |
| Complexity | Low | Medium | High |

**Advantages of Partitioning**:
- **Scalability**: Linear scale with partition count
- **Performance**: Parallel query execution
- **Availability**: Failure isolated to partition
- **Maintenance**: Can maintain partitions independently

**Disadvantages of Partitioning**:
- **Complexity**: Cross-partition queries expensive
- **Joins**: Cannot join across partitions efficiently
- **Rebalancing**: Data movement required on resize
- **Skew**: Uneven distribution possible
- **Transactions**: Distributed transactions complex

**Cross-Partition Operations**:
```
Single Partition Query:    Cross-Partition Query:
    O(1) partition            O(N) partitions
    Fast                      Slow
    Cheap                     Expensive
```

## Implementation Example

### Python Partitioning Library

```python
from typing import Dict, List, Any, Optional, Callable
from dataclasses import dataclass
from enum import Enum
import hashlib
import json

class PartitionStrategy(Enum):
    HASH = "hash"
    RANGE = "range"
    MODULO = "modulo"
    CONSISTENT_HASH = "consistent_hash"

@dataclass
class PartitionConfig:
    strategy: PartitionStrategy
    num_partitions: int
    partition_key: str
    ranges: Optional[List[tuple]] = None  # For range partitioning

class DataPartitioner:
    """Generic data partitioning implementation"""
    
    def __init__(self, config: PartitionConfig):
        self.config = config
        self.partitions: Dict[int, List[Dict[str, Any]]] = {
            i: [] for i in range(config.num_partitions)
        }
    
    def get_partition(self, record: Dict[str, Any]) -> int:
        """Determine which partition a record belongs to"""
        key_value = record[self.config.partition_key]
        
        if self.config.strategy == PartitionStrategy.HASH:
            return self._hash_partition(key_value)
        elif self.config.strategy == PartitionStrategy.RANGE:
            return self._range_partition(key_value)
        elif self.config.strategy == PartitionStrategy.MODULO:
            return self._modulo_partition(key_value)
        else:
            raise ValueError(f"Unknown strategy: {self.config.strategy}")
    
    def _hash_partition(self, key: Any) -> int:
        """Hash-based partition selection"""
        key_str = str(key)
        hash_value = int(hashlib.md5(key_str.encode()).hexdigest(), 16)
        return hash_value % self.config.num_partitions
    
    def _range_partition(self, value: Any) -> int:
        """Range-based partition selection"""
        if not self.config.ranges:
            raise ValueError("Ranges not configured for range partitioning")
        
        for i, (start, end) in enumerate(self.config.ranges):
            if start <= value < end:
                return i
        return len(self.config.ranges) - 1
    
    def _modulo_partition(self, value: int) -> int:
        """Modulo-based partition selection"""
        return abs(value) % self.config.num_partitions
    
    def insert(self, record: Dict[str, Any]) -> None:
        """Insert record into appropriate partition"""
        partition_id = self.get_partition(record)
        self.partitions[partition_id].append(record)
    
    def query(self, partition_key_value: Any) -> List[Dict[str, Any]]:
        """Query records from specific partition"""
        partition_id = self.get_partition({self.config.partition_key: partition_key_value})
        return [
            r for r in self.partitions[partition_id]
            if r[self.config.partition_key] == partition_key_value
        ]
    
    def scan_all(self) -> List[Dict[str, Any]]:
        """Scan all partitions (expensive!)"""
        all_records = []
        for partition in self.partitions.values():
            all_records.extend(partition)
        return all_records
    
    def get_partition_stats(self) -> Dict[int, int]:
        """Get record count per partition"""
        return {pid: len(records) for pid, records in self.partitions.items()}

# Example Usage
if __name__ == "__main__":
    # Configure hash-based partitioning
    config = PartitionConfig(
        strategy=PartitionStrategy.HASH,
        num_partitions=4,
        partition_key="user_id"
    )
    
    partitioner = DataPartitioner(config)
    
    # Insert sample data
    users = [
        {"user_id": "user001", "name": "Alice", "email": "alice@example.com"},
        {"user_id": "user002", "name": "Bob", "email": "bob@example.com"},
        {"user_id": "user003", "name": "Charlie", "email": "charlie@example.com"},
        {"user_id": "user004", "name": "Diana", "email": "diana@example.com"},
        {"user_id": "user005", "name": "Eve", "email": "eve@example.com"},
    ]
    
    for user in users:
        partitioner.insert(user)
    
    # Query specific user
    results = partitioner.query("user001")
    print(f"Query results: {results}")
    
    # Check partition distribution
    stats = partitioner.get_partition_stats()
    print(f"Partition stats: {stats}")
```

### Range Partitioning for Time Series

```python
class TimeSeriesPartitioner(DataPartitioner):
    """Specialized partitioner for time series data"""
    
    def __init__(self, num_days: int, partition_key: str = "timestamp"):
        from datetime import datetime, timedelta
        
        today = datetime.now().date()
        ranges = []
        
        for i in range(num_days):
            end_date = today - timedelta(days=i)
            start_date = end_date - timedelta(days=1)
            ranges.append((
                int(start_date.strftime("%Y%m%d")),
                int(end_date.strftime("%Y%m%d"))
            ))
        
        config = PartitionConfig(
            strategy=PartitionStrategy.RANGE,
            num_partitions=num_days,
            partition_key=partition_key,
            ranges=ranges
        )
        super().__init__(config)
    
    def insert_with_timestamp(self, record: Dict[str, Any], timestamp: int) -> None:
        """Insert record with date-based partition key"""
        record["date_partition"] = timestamp
        self.insert(record)
```

## Anti-Pattern

### ❌ Anti-Pattern: Partitioning by Random Value

```python
# WRONG: Using random as partition key
import random

def insert_random_partition(record: dict):
    partition_id = random.randint(0, NUM_PARTITIONS - 1)
    partitions[partition_id].append(record)

# Problems:
# 1. Cannot query by record - don't know which partition
# 2. Must scan all partitions for any lookup
# 3. Defeats purpose of partitioning
```

### ❌ Anti-Pattern: Partitioning by Timestamp (High Cardinality)

```python
# WRONG: Using millisecond timestamp as partition key
def partition_by_millisecond(timestamp_ms: int) -> int:
    return timestamp_ms % NUM_PARTITIONS

# Problems:
# 1. Creates too many empty partitions
# 2. Hot partition problem - all writes to current time
# 3. Poor query performance for time ranges
```

### ❌ Anti-Pattern: Ignoring Data Skew

```python
# WRONG: Modulo partitioning without considering skew
def naive_modulo(user_id: int) -> int:
    return user_id % NUM_PARTITIONS

# If user_ids are mostly even numbers, 
# even partitions get 2x the traffic!
```

### ✅ Correct Pattern

```python
# CORRECT: Use hash for even distribution
def hash_partition(user_id: str) -> int:
    import hashlib
    hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    return hash_val % NUM_PARTITIONS

# CORRECT: Use date-based partitioning for time series
def date_partition(timestamp: datetime) -> int:
    # Partition by day, not millisecond
    days_since_epoch = (timestamp.date() - EPOCH_DATE).days
    return days_since_epoch % NUM_PARTITIONS
```

## Related Patterns

- **[CAP Theorem](./01-CAP-Theorem.md)**: Tradeoffs in distributed data storage
- **[Replication](./04-Replication.md)**: Combining partitioning with replication
- **[Consensus Algorithms](./02-Consensus-Algorithms.md)**: Coordinating across partitions
- **[Database Migration](../08-Database-Design/05-Migration.md)**: Rebalancing partitioned data
- **[Scalability Patterns](../01-System-Design/04-Scalability-Patterns.md)**: Horizontal scaling strategies
- **[Indexing Strategies](../08-Database-Design/03-Indexing-Strategies.md)**: Indexing in partitioned databases

---

*Last Updated: 2024-01-15*