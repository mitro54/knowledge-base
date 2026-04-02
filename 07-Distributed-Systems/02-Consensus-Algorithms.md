# Consensus Algorithms

## Title & Summary

Consensus algorithms enable distributed systems to agree on a single value or state despite node failures, network partitions, and message delays. This pattern covers Paxos, Raft, and Zab algorithms that form the foundation of distributed databases, coordination services, and blockchain systems.

## Problem Statement

In distributed systems, multiple nodes must maintain consistent state despite:
- **Node Failures**: Nodes may crash or become unresponsive
- **Network Partitions**: Messages may be delayed, lost, or duplicated
- **Concurrent Updates**: Multiple nodes may propose changes simultaneously
- **Byzantine Failures**: Nodes may behave maliciously or unpredictably

Without consensus, distributed systems cannot guarantee:
- Data consistency across replicas
- Atomic commits in distributed transactions
- Leader election for coordinated operations
- Agreement on event ordering

## Solution

### Core Consensus Properties

**Safety Properties** (must never be violated):
- **Agreement**: All correct nodes decide on the same value
- **Validity**: The decided value must have been proposed
- **Integrity**: Each node decides at most once

**Liveness Property** (must eventually happen):
- **Termination**: All correct nodes eventually decide

### Paxos Algorithm

**Roles**:
- **Proposers**: Propose values for consensus
- **Acceptors**: Accept or reject proposals
- **Learners**: Learn the decided value

**Two-Phase Protocol**:
```
Phase 1 (Prepare):
1. Proposer sends Prepare(N) to majority of Acceptors
2. Acceptors respond with Promise(N) and last accepted value

Phase 2 (Accept):
3. Proposer sends Accept(N, V) to majority of Acceptors
4. Acceptors respond with Accepted(N, V) if not promised higher N
```

**Multi-Paxos**: Optimized for repeated consensus with persistent leader

### Raft Algorithm

**Key Innovation**: Strong leadership with log replication

**Three Sub-problems**:
1. **Leader Election**: Followers request votes; candidate with majority wins
2. **Log Replication**: Leader replicates entries to followers
3. **Safety**: Term-based ordering prevents conflicting decisions

**State Machines**:
- **Leader**: Handles client requests, replicates log
- **Follower**: Passively responds to leader
- **Candidate**: Transitional state for election

### Zab Algorithm (ZooKeeper)

**Design Goals**:
- Total order broadcast
- Crash recovery with checkpointing
- Simple mental model for implementers

**Phases**:
1. **Leader Election**: Based on zxid (epoch, transaction id)
2. **Synchronization**: New leader synchronizes followers
3. **Broadcast**: Leader broadcasts proposals in total order

## When to Use

**Use Consensus Algorithms When**:
- Building distributed databases requiring strong consistency
- Implementing coordination services (like ZooKeeper, etcd)
- Creating replicated state machines
- Building blockchain or distributed ledger systems
- Implementing distributed locks and leader election

**Choose Paxos When**:
- Maximum flexibility and academic rigor required
- Building highly available systems with complex failure models
- Academic or research applications

**Choose Raft When**:
- Ease of understanding and implementation is priority
- Building systems like etcd, Consul, HashiCorp products
- Need clear mental model for debugging

**Choose Zab When**:
- Building ZooKeeper-compatible systems
- Need total order broadcast semantics
- Simple implementation with strong guarantees

## Tradeoffs

| Aspect | Paxos | Raft | Zab |
|--------|-------|------|-----|
| Understandability | Low | High | Medium |
| Flexibility | High | Medium | Low |
| Performance | Medium | High | High |
| Implementation Complexity | High | Medium | Medium |
| Leader Optimization | Multi-Paxos | Built-in | Built-in |

**CAP Theorem Implications**:
- All provide **CP** (Consistency + Partition Tolerance)
- Sacrifice availability during network partitions
- Require majority quorum for progress

**Performance Considerations**:
- **Latency**: O(1) round trips for simple operations
- **Throughput**: Limited by log replication speed
- **Scalability**: Typically 3-5 nodes optimal; more increases latency

**Failure Scenarios**:
- **Leader Failure**: Triggers election (Raft: ~1-2 heartbeat intervals)
- **Network Partition**: Minority partition cannot make progress
- **Split Brain**: Prevented by requiring majority quorum

## Implementation Example

### Raft Leader Election (Python)

```python
import random
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class ServerState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

@dataclass
class RaftNode:
    node_id: int
    current_term: int = 0
    voted_for: Optional[int] = None
    state: ServerState = ServerState.FOLLOWER
    log: List[str] = None
    commit_index: int = 0
    last_applied: int = 0
    
    def __post_init__(self):
        if self.log is None:
            self.log = []
    
    def start_election(self, cluster: List['RaftNode']) -> bool:
        """Start leader election process"""
        self.state = ServerState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        
        votes_received = 1  # Vote for self
        total_nodes = len(cluster)
        
        # Request votes from other nodes
        for node in cluster:
            if node.node_id != self.node_id:
                if self.request_vote(node):
                    votes_received += 1
        
        # Check if won election
        if votes_received > total_nodes // 2:
            self.become_leader()
            return True
        return False
    
    def request_vote(self, follower: 'RaftNode') -> bool:
        """Request vote from a follower"""
        # Vote granted if:
        # 1. Candidate's term >= follower's term
        # 2. Follower hasn't voted or voted for this candidate
        # 3. Candidate's log is at least as up-to-date
        if self.current_term < follower.current_term:
            return False
        
        if follower.voted_for is None or follower.voted_for == self.node_id:
            if self.is_log_up_to_date(follower):
                follower.current_term = self.current_term
                follower.voted_for = self.node_id
                return True
        return False
    
    def is_log_up_to_date(self, other: 'RaftNode') -> bool:
        """Check if this node's log is up-to-date"""
        self_last_index = len(self.log) - 1
        other_last_index = len(other.log) - 1
        
        if self_last_index > other_last_index:
            return True
        elif self_last_index == other_last_index:
            # Compare terms of last entries (simplified)
            return True
        return False
    
    def become_leader(self):
        """Transition to leader state"""
        self.state = ServerState.LEADER
        self.next_index = {}  # Index of next entry to send to each follower
        self.match_index = {}  # Index of highest entry known to be replicated
        
        print(f"Node {self.node_id} became leader for term {self.current_term}")

# Example usage
if __name__ == "__main__":
    cluster = [RaftNode(i) for i in range(5)]
    
    # Simulate election
    candidate = cluster[2]
    won = candidate.start_election(cluster)
    
    print(f"Election won: {won}")
    print(f"Leader: Node {candidate.node_id}, Term: {candidate.current_term}")
```

### Log Replication (Simplified)

```python
def append_entries(self, entries: List[str], follower: 'RaftNode') -> bool:
    """Replicate log entries to follower"""
    if self.state != ServerState.LEADER:
        return False
    
    # Check term validity
    if self.current_term < follower.current_term:
        self.step_down()
        return False
    
    # Update follower's term if needed
    follower.current_term = max(follower.current_term, self.current_term)
    
    # Find conflict point and replicate
    for entry in entries:
        follower.log.append(entry)
    
    follower.commit_index = min(follower.commit_index + len(entries), len(follower.log) - 1)
    return True
```

## Anti-Pattern

### ❌ Anti-Pattern: Voting Without Term Validation

```python
# WRONG: No term validation in vote request
def request_vote_broken(self, follower: 'RaftNode') -> bool:
    # Missing: term comparison
    # Missing: log up-to-date check
    if follower.voted_for is None:
        follower.voted_for = self.node_id
        return True
    return False

# Problems:
# 1. Old leaders can steal elections with stale terms
# 2. Nodes with outdated logs can become leader
# 3. Split brain scenarios possible
# 4. No guarantee of log consistency
```

### ❌ Anti-Pattern: Quorum Without Majority

```python
# WRONG: Using fixed quorum instead of dynamic majority
QUORUM_SIZE = 3  # Fixed, doesn't adapt to cluster size

def check_quorum_broken(votes: int) -> bool:
    return votes >= QUORUM_SIZE

# Problems:
# 1. Doesn't work with different cluster sizes
# 2. May allow split brain in 4-node cluster
# 3. May require impossible quorum in 2-node cluster
```

### ✅ Correct Pattern

```python
def check_majority(self, votes: int, cluster_size: int) -> bool:
    """Correct majority check"""
    return votes > cluster_size // 2

def request_vote_correct(self, follower: 'RaftNode') -> bool:
    """Correct vote request with all validations"""
    # 1. Term must be at least as large
    if self.current_term < follower.current_term:
        return False
    
    # 2. Follower must not have voted, or voted for us
    if follower.voted_for is not None and follower.voted_for != self.node_id:
        return False
    
    # 3. Our log must be at least as up-to-date
    if not self.is_log_up_to_date(follower):
        return False
    
    # Grant vote
    follower.current_term = self.current_term
    follower.voted_for = self.node_id
    return True
```

## Related Patterns

- **[CAP Theorem](./01-CAP-Theorem.md)**: Understanding consistency vs availability tradeoffs
- **[Partitioning](./03-Partitioning.md)**: Data distribution across nodes
- **[Replication](./04-Replication.md)**: Maintaining multiple copies of data
- **[Distributed Transactions](./05-Distributed-Transactions.md)**: Atomic operations across nodes
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)**: Handling node failures gracefully
- **[Reliability Patterns](../01-System-Design/05-Reliability-Patterns.md)**: Building resilient distributed systems

---

*Last Updated: 2024-01-15*