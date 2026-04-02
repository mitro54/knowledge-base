# Distributed Transactions

## Title & Summary

**Distributed Transactions** are transactions that span multiple databases, services, or systems in a distributed environment. Unlike local transactions that operate on a single resource, distributed transactions must coordinate across multiple independent systems while maintaining consistency, atomicity, and durability guarantees.

## Problem Statement

In distributed systems, business operations often require updates across multiple services or databases:

- **E-commerce Order**: Update inventory, charge payment, create order record, send notification
- **Bank Transfer**: Debit source account, credit destination account, update transaction log
- **User Registration**: Create user record, initialize profile, send welcome email, update analytics

**Challenges:**
- **Partial Failures**: One service succeeds while another fails, leaving inconsistent state
- **Network Partitions**: Services cannot communicate, blocking transaction completion
- **Coordination Overhead**: Synchronizing multiple systems adds latency and complexity
- **CAP Tradeoffs**: Cannot simultaneously guarantee Consistency, Availability, and Partition Tolerance

Without proper distributed transaction patterns, systems face data inconsistency, lost updates, and complex manual reconciliation.

## Solution

### 1. Two-Phase Commit (2PC)

**Coordinator-based protocol ensuring atomic commitment across participants.**

**Phase 1 - Prepare:**
1. Coordinator sends PREPARE to all participants
2. Each participant locks resources and votes COMMIT or ABORT
3. All participants must vote COMMIT for transaction to proceed

**Phase 2 - Commit:**
1. If all voted COMMIT: Coordinator sends COMMIT to all
2. If any voted ABORT: Coordinator sends ABORT to all
3. Participants release locks and confirm completion

```python
class TwoPhaseCommit:
    def __init__(self, coordinator_id):
        self.coordinator_id = coordinator_id
        self.participants = []
    
    def execute(self, operations):
        # Phase 1: Prepare
        prepare_results = []
        for participant_id, operation in operations:
            result = self._prepare(participant_id, operation)
            prepare_results.append(result)
        
        # Phase 2: Commit or Abort
        if all(result['voted_commit'] for result in prepare_results):
            self._commit_all(operations)
            return True
        else:
            self._abort_all(operations)
            return False
    
    def _prepare(self, participant_id, operation):
        # Lock resources, validate, vote
        pass
    
    def _commit_all(self, operations):
        # Send commit to all participants
        pass
    
    def _abort_all(self, operations):
        # Send abort to all participants
        pass
```

### 2. Saga Pattern

**Long-running transaction decomposed into local transactions with compensating actions.**

```python
class SagaOrchestrator:
    def __init__(self):
        self.steps = []
        self.compensations = []
    
    def add_step(self, action, compensation):
        self.steps.append(action)
        self.compensations.append(compensation)
    
    def execute(self, context):
        executed_steps = []
        try:
            for step in self.steps:
                step(context)
                executed_steps.append(step)
            return True
        except Exception as e:
            # Execute compensations in reverse order
            for compensation in reversed(executed_steps):
                try:
                    compensation.compensate(context)
                except:
                    log_error(f"Compensation failed: {compensation}")
            raise e

# Usage: Create Order Saga
create_order_saga = SagaOrchestrator()
create_order_saga.add_step(
    action=lambda ctx: order_service.create_order(ctx),
    compensation=lambda ctx: order_service.cancel_order(ctx['order_id'])
)
create_order_saga.add_step(
    action=lambda ctx: inventory_service.reserve(ctx),
    compensation=lambda ctx: inventory_service.release(ctx)
)
create_order_saga.add_step(
    action=lambda ctx: payment_service.charge(ctx),
    compensation=lambda ctx: payment_service.refund(ctx)
)
create_order_saga.add_step(
    action=lambda ctx: notification_service.send_confirmation(ctx),
    compensation=lambda ctx: notification_service.send_cancellation(ctx)
)
```

### 3. Eventual Consistency with Event Sourcing

**Achieve consistency through event replay and reconciliation.**

```python
class EventSourcedTransaction:
    def __init__(self, event_store):
        self.event_store = event_store
        self.pending_events = []
    
    def execute(self, transaction_id, events):
        # Publish events to all interested parties
        for event in events:
            event['transaction_id'] = transaction_id
            event['timestamp'] = datetime.utcnow()
            self.event_store.append(event)
            self.pending_events.append(event)
        
        # Subscribe to acknowledgments
        return self._wait_for_acknowledgments(transaction_id)
    
    def _wait_for_acknowledgments(self, transaction_id):
        # Wait for all services to acknowledge events
        pass
```

### 4. Outbox Pattern

**Ensure reliable event publishing alongside data changes.**

```python
class OutboxPattern:
    def __init__(self, database, event_bus):
        self.database = database
        self.event_bus = event_bus
    
    def execute_transaction(self, data_changes, events):
        with self.database.transaction() as tx:
            # Execute data changes
            for change in data_changes:
                tx.execute(change)
            
            # Store events in outbox table
            for event in events:
                tx.execute(
                    "INSERT INTO outbox_events (event_data, published) VALUES (?, FALSE)",
                    (json.dumps(event),)
                )
            
            tx.commit()
        
        # Separate process publishes events from outbox
        self._publish_pending_events()
    
    def _publish_pending_events(self):
        events = self.database.query(
            "SELECT event_data FROM outbox_events WHERE published = FALSE"
        )
        for event_row in events:
            self.event_bus.publish(json.loads(event_row['event_data']))
            self.database.execute(
                "UPDATE outbox_events SET published = TRUE WHERE id = ?",
                (event_row['id'],)
            )
```

## When to Use

| Pattern | Use Case | Consistency Model |
|---------|----------|------------------|
| **2PC** | Financial systems, strict consistency required | Strong consistency |
| **Saga** | Microservices, long-running processes | Eventual consistency |
| **Event Sourcing** | Audit trails, state reconstruction | Eventual consistency |
| **Outbox Pattern** | Reliable event publishing | Eventual consistency |

**Use 2PC when:**
- Strong consistency is required (financial transactions)
- Latency requirements allow blocking coordination
- Number of participants is small and known

**Use Saga when:**
- Operating in microservices architecture
- Long-running business processes
- Can tolerate eventual consistency
- Need to avoid distributed locks

**Use Event Sourcing when:**
- Complete audit trail is valuable
- State can be reconstructed from events
- Multiple read models needed

**Use Outbox Pattern when:**
- Reliable event publishing is critical
- Cannot afford lost events
- Want to separate transaction from publishing

## Tradeoffs

| Aspect | 2PC | Saga | Event Sourcing | Outbox |
|--------|-----|------|----------------|--------|
| **Consistency** | Strong | Eventual | Eventual | Eventual |
| **Availability** | Low (blocking) | High | High | High |
| **Latency** | High | Medium | Medium | Low |
| **Complexity** | Medium | High | High | Medium |
| **Scalability** | Limited | Good | Good | Good |
| **Recovery** | Coordinator needed | Compensation logic | Replay events | Checkpoint tracking |

## Anti-Pattern

### ❌ Distributed Transaction Anti-Patterns

**1. Chatty Distributed Transactions**
```python
# ANTI-PATTERN: Multiple round trips per operation
def process_order(order):
    # Round trip 1
    inventory = inventory_service.check(order.items)
    # Round trip 2
    payment = payment_service.validate(order.payment)
    # Round trip 3
    order_record = order_service.create(order)
    # Round trip 4
    inventory_service.reserve(order.items)
    # Round trip 5
    payment_service.charge(order.payment)
    # Total: 5 network calls, high latency, multiple failure points
```

**2. Ignoring Partial Failures**
```python
# ANTI-PATTERN: No compensation for partial failures
def transfer_funds(from_account, to_account, amount):
    debit_service.debit(from_account, amount)  # Succeeds
    credit_service.credit(to_account, amount)   # Fails!
    # Money disappeared - no compensation!
```

**3. Using 2PC for Everything**
```python
# ANTI-PATTERN: Overusing 2PC causes blocking and poor scalability
# 2PC blocks resources during coordination
# Not suitable for high-throughput, low-latency systems
```

**4. Assuming Strong Consistency**
```python
# ANTI-PATTERN: Reading stale data after write
def create_user_and_profile(user_data):
    user_service.create(user_data)  # Write to user service
    # Immediately read - may get stale data!
    profile = profile_service.get(user_data.id)  # May not exist yet
```

## Examples

### Complete Saga Implementation

```python
from enum import Enum
from typing import Callable, Dict, Any

class SagaStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

class SagaStep:
    def __init__(self, name: str, action: Callable, compensation: Callable):
        self.name = name
        self.action = action
        self.compensation = compensation

class DistributedSaga:
    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.steps: list[SagaStep] = []
        self.status = SagaStatus.PENDING
        self.executed_steps: list[str] = []
        self.context: Dict[str, Any] = {}
    
    def add_step(self, name: str, action: Callable, compensation: Callable):
        self.steps.append(SagaStep(name, action, compensation))
        return self
    
    def execute(self, initial_context: Dict[str, Any] = None):
        self.context = initial_context or {}
        self.status = SagaStatus.IN_PROGRESS
        
        try:
            for step in self.steps:
                print(f"Executing step: {step.name}")
                self.context = step.action(self.context)
                self.executed_steps.append(step.name)
            
            self.status = SagaStatus.COMPLETED
            return self.context
            
        except Exception as e:
            print(f"Step failed: {e}")
            self.status = SagaStatus.COMPENSATING
            return self._compensate()
    
    def _compensate(self):
        # Execute compensations in reverse order
        for step_name in reversed(self.executed_steps):
            step = next(s for s in self.steps if s.name == step_name)
            try:
                print(f"Compensating step: {step.name}")
                self.context = step.compensation(self.context)
            except Exception as e:
                print(f"Compensation failed for {step.name}: {e}")
                # Log and continue - don't let compensation fail the recovery
        self.status = SagaStatus.FAILED
        return self.context

# Define saga steps
def create_order_step(context):
    order_id = order_service.create_order(context['order_data'])
    context['order_id'] = order_id
    return context

def reserve_inventory_step(context):
    inventory_service.reserve(context['order_id'], context['items'])
    return context

def charge_payment_step(context):
    payment_id = payment_service.charge(context['order_id'], context['amount'])
    context['payment_id'] = payment_id
    return context

def send_confirmation_step(context):
    notification_service.send_order_confirmation(context['order_id'])
    return context

# Compensation steps
def cancel_order_compensation(context):
    order_service.cancel_order(context['order_id'])
    return context

def release_inventory_compensation(context):
    inventory_service.release(context['order_id'])
    return context

def refund_payment_compensation(context):
    if 'payment_id' in context:
        payment_service.refund(context['payment_id'])
    return context

def send_cancellation_compensation(context):
    notification_service.send_order_cancellation(context['order_id'])
    return context

# Execute saga
saga = DistributedSaga("order-123")
saga.add_step("create_order", create_order_step, cancel_order_compensation)
saga.add_step("reserve_inventory", reserve_inventory_step, release_inventory_compensation)
saga.add_step("charge_payment", charge_payment_step, refund_payment_compensation)
saga.add_step("send_confirmation", send_confirmation_step, send_cancellation_compensation)

result = saga.execute({
    'order_data': {'customer_id': 'cust-456', 'items': [...]},
    'amount': 99.99
})
```

## Related Patterns

- **[CAP Theorem](./01-CAP-Theorem.md)** - Understanding consistency tradeoffs
- **[Event-Driven Architecture](../01-System-Design/03-Event-Driven-Architecture.md)** - Event-based coordination
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Handling failures gracefully
- **[Circuit Breaker](../02-Design-Patterns/04-Concurrency-Patterns.md)** - Preventing cascade failures

## References

- **Paper**: "Sagas" - Glenn Ammons et al., 1987
- **Book**: "Microservices Patterns" - Chris Richardson
- **Article**: "Transactions in Distributed Systems" - Jim Gray, 1984
- **Pattern**: "Outbox Pattern" - Microservices Patterns catalog