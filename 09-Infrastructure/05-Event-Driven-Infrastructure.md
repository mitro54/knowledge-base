# Event-Driven Infrastructure

## Title & Summary

Event-Driven Infrastructure is an architectural approach where infrastructure components communicate through asynchronous events, enabling loose coupling, scalability, and resilience. This pattern leverages event buses, message queues, and streaming platforms to orchestrate infrastructure operations, service communication, and system coordination.

## Problem Statement

Traditional request-response infrastructure architectures face several challenges:

- **Tight Coupling**: Services directly depend on each other's availability and response times
- **Synchronous Bottlenecks**: Blocking calls create cascading failures and performance issues
- **Limited Scalability**: Point-to-point connections don't scale well with increasing services
- **Poor Fault Isolation**: Failures propagate quickly through synchronous call chains
- **Difficulty in Coordination**: Complex workflows require intricate orchestration logic
- **Limited Observability**: Hard to trace requests across multiple synchronous boundaries

## Solution

Event-Driven Infrastructure addresses these challenges through:

### Core Components

**Event Bus / Message Broker**
- Central hub for event distribution
- Supports publish-subscribe patterns
- Provides event routing and filtering
- Examples: Apache Kafka, RabbitMQ, AWS SNS/SQS

**Event Producers**
- Services that generate and publish events
- Events represent state changes or significant occurrences
- Events are immutable and timestamped

**Event Consumers**
- Services that subscribe to and process events
- Can process events asynchronously
- Multiple consumers can react to the same event

**Event Store**
- Persistent storage for event history
- Enables event replay and auditing
- Supports event sourcing patterns

### Key Patterns

**Publish-Subscribe (Pub/Sub)**
- Producers publish events without knowing consumers
- Consumers subscribe to event types of interest
- One-to-many communication pattern

**Event Streaming**
- Events stored in ordered, append-only logs
- Consumers can read at their own pace
- Supports replay and reprocessing

**Event Sourcing**
- State changes stored as sequence of events
- Current state derived by replaying events
- Enables temporal queries and audit trails

## When to Use

**Ideal Scenarios:**
- Microservices architectures requiring loose coupling
- Systems needing high scalability and throughput
- Event-heavy domains (e-commerce, IoT, real-time analytics)
- Systems requiring audit trails and event history
- Complex workflows with multiple participants
- Real-time data processing pipelines

**Not Recommended For:**
- Simple, monolithic applications
- Systems requiring strong immediate consistency
- Applications with simple request-response needs
- Teams without experience in distributed systems

## Tradeoffs

| Aspect | Benefits | Costs |
|--------|----------|-------|
| **Coupling** | Loose coupling between services | Increased complexity in event design |
| **Scalability** | Horizontal scaling of consumers | Event ordering challenges at scale |
| **Resilience** | Fault isolation, retry mechanisms | Event loss prevention complexity |
| **Performance** | Asynchronous processing, non-blocking | Latency in event delivery |
| **Complexity** | Simplified service interactions | Complex debugging and tracing |
| **Consistency** | Eventual consistency enables availability | Not suitable for strong consistency needs |

## Implementation Example

### Kafka-Based Event Infrastructure

```yaml
# Infrastructure Configuration
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: event-infrastructure
spec:
  kafka:
    version: 3.5.0
    replicas: 3
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    persistence:
      type: persistent-claim
      size: 100Gi

---
# Topic Definitions
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-events
  labels:
    strimzi.io/cluster: event-infrastructure
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000  # 7 days
    cleanup.policy: delete

---
# Consumer Service Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: processor
        image: order-processor:latest
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "event-infrastructure:9092"
        - name: KAFKA_TOPIC
          value: "order-events"
        - name: KAFKA_GROUP_ID
          value: "order-processor-group"
```

### Event Schema Design

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OrderCreatedEvent",
  "type": "object",
  "properties": {
    "eventId": {
      "type": "string",
      "format": "uuid",
      "description": "Unique event identifier"
    },
    "eventType": {
      "type": "string",
      "const": "OrderCreated"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "version": {
      "type": "integer",
      "minimum": 1
    },
    "data": {
      "type": "object",
      "properties": {
        "orderId": { "type": "string" },
        "customerId": { "type": "string" },
        "totalAmount": { "type": "number" },
        "currency": { "type": "string" },
        "items": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "productId": { "type": "string" },
              "quantity": { "type": "integer" },
              "price": { "type": "number" }
            }
          }
        }
      }
    }
  },
  "required": ["eventId", "eventType", "timestamp", "data"]
}
```

## Anti-Pattern

### ❌ Event Storming Without Boundaries

```python
# ANTI-PATTERN: Publishing events without clear domain boundaries

class OrderService:
    def create_order(self, order_data):
        order = self.create_order_entity(order_data)
        
        # Publishing too many granular events
        self.event_bus.publish("OrderCreated", order)
        self.event_bus.publish("OrderItemAdded", order.items)
        self.event_bus.publish("CustomerOrderCountUpdated", order.customer_id)
        self.event_bus.publish("InventoryReserved", order.items)
        self.event_bus.publish("PaymentInitiated", order.payment_info)
        self.event_bus.publish("NotificationTriggered", order.customer_id)
        
        return order

# ANTI-PATTERN: Synchronous event processing disguised as async
class PaymentProcessor:
    def on_order_created(self, event):
        # Blocking call to external service
        response = external_api.charge_card(event.data.card_number)
        # This defeats the purpose of async processing
        return response
```

### ✅ Correct Pattern

```python
# CORRECT: Clear domain events with proper boundaries

class OrderService:
    def create_order(self, order_data):
        # Domain logic
        order = Order.create(order_data)
        
        # Persist order
        self.order_repository.save(order)
        
        # Publish single domain event
        self.event_bus.publish(
            OrderCreatedEvent(
                order_id=order.id,
                customer_id=order.customer_id,
                total_amount=order.total_amount
            )
        )
        
        return order

# CORRECT: Async event processing with proper error handling
class PaymentProcessor:
    @retry(max_attempts=3, backoff='exponential')
    async def process_payment(self, event: OrderCreatedEvent):
        try:
            # Non-blocking async call
            payment_result = await self.payment_gateway.charge(
                order_id=event.order_id,
                amount=event.total_amount
            )
            
            # Publish result as new event
            await self.event_bus.publish(
                PaymentProcessedEvent(
                    order_id=event.order_id,
                    status=payment_result.status
                )
            )
        except PaymentFailedError as e:
            await self.event_bus.publish(
                PaymentFailedEvent(
                    order_id=event.order_id,
                    error=str(e)
                )
            )
            raise
```

## Related Patterns

- **[Event-Driven Architecture](../01-System-Design/03-Event-Driven-Architecture.md)** - Higher-level architectural pattern
- **[Event-Driven Paradigm](../03-Paradigms/03-Event-Driven-Paradigm.md)** - Programming paradigm perspective
- **[Messaging Patterns](../02-Design-Patterns/03-Behavioral-Patterns.md)** - Related behavioral patterns
- **[Distributed Transactions](../07-Distributed-Systems/05-Distributed-Transactions.md)** - Saga pattern for distributed coordination
- **[Observability](../10-Observability/01-Logging.md)** - Event tracing and monitoring