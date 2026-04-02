# Event-Driven Architecture

## Title & Summary

Event-Driven Architecture (EDA) is an architectural paradigm where system components communicate through the production and consumption of events. Events represent significant state changes or occurrences that trigger asynchronous responses from interested components, enabling loose coupling and high scalability.

## Problem Statement

Traditional request-response architectures face several challenges:

- **Tight Coupling**: Components must know about each other and communicate synchronously
- **Scalability Bottlenecks**: Synchronous calls create blocking operations and resource contention
- **Limited Responsiveness**: Systems cannot react to events in real-time without polling
- **Poor Extensibility**: Adding new functionality requires modifying existing components
- **Single Points of Failure**: Centralized communication creates vulnerability

## Solution

Event-Driven Architecture addresses these challenges through:

### Core Components

1. **Event Producers**: Components that generate events when state changes occur
2. **Event Consumers**: Components that react to events they're interested in
3. **Event Channels/Buses**: Infrastructure that transports events between producers and consumers
4. **Event Stores**: Persistent storage for event history (in event sourcing patterns)

### Communication Patterns

- **Publish-Subscribe**: Producers publish events; consumers subscribe to topics of interest
- **Event Streaming**: Continuous flow of events through a distributed log
- **Event Sourcing**: State changes recorded as immutable event sequence

### Key Characteristics

- **Asynchronous Communication**: Non-blocking, fire-and-forget interactions
- **Loose Coupling**: Producers don't know consumers; consumers don't know producers
- **Event-Driven Triggers**: Actions initiated by events rather than direct calls
- **Decentralized Control**: No central coordinator required

## When to Use

Event-Driven Architecture is appropriate when:

- **Real-time Processing Required**: Stock trading, IoT sensor data, live analytics
- **High Scalability Needed**: Systems with variable, unpredictable loads
- **Multiple Independent Services**: Microservices needing loose coupling
- **Event History Valuable**: Audit trails, replay capabilities, state reconstruction
- **Responsive UIs Needed**: Real-time updates without polling
- **Integration of Heterogeneous Systems**: Connecting systems with different protocols

## Tradeoffs

### Advantages

| Benefit | Description |
|---------|-------------|
| Loose Coupling | Components independent; can be developed, deployed, scaled separately |
| Scalability | Horizontal scaling through partitioned event streams |
| Responsiveness | Real-time reaction to events without polling |
| Extensibility | New consumers added without modifying producers |
| Resilience | Component failures don't cascade; events can be replayed |
| Audit Trail | Event history provides complete system state evolution |

### Disadvantages

| Challenge | Description |
|-----------|-------------|
| Complexity | Event flows harder to trace than synchronous calls |
| Event Ordering | Maintaining order across distributed systems is difficult |
| Message Loss | Events can be lost without proper acknowledgment mechanisms |
| Debugging | Distributed tracing required; harder to reproduce issues |
| Event Schema Evolution | Changing event formats requires careful versioning |
| Overhead | Event infrastructure adds latency and resource consumption |

## Implementation Example

### Basic Pub-Sub with Node.js and Redis

```javascript
// Event Producer
const redis = require('redis');
const pubClient = redis.createClient();

async function publishOrderEvent(order) {
    const event = {
        type: 'ORDER_CREATED',
        timestamp: new Date().toISOString(),
        payload: {
            orderId: order.id,
            customerId: order.customerId,
            totalAmount: order.total,
            items: order.items
        }
    };
    
    await pubClient.publish('order-events', JSON.stringify(event));
    console.log(`Published event: ${event.type}`);
}

// Event Consumer - Inventory Service
const subClient = redis.createClient();
subClient.subscribe('order-events');

subClient.on('message', (channel, message) => {
    const event = JSON.parse(message);
    
    if (event.type === 'ORDER_CREATED') {
        handleOrderCreated(event.payload);
    }
});

async function handleOrderCreated(orderData) {
    console.log(`Processing order: ${orderData.orderId}`);
    // Reserve inventory, update stock levels
}

// Event Consumer - Notification Service
subClient.on('message', (channel, message) => {
    const event = JSON.parse(message);
    
    if (event.type === 'ORDER_CREATED') {
        sendOrderConfirmation(event.payload);
    }
});

async function sendOrderConfirmation(orderData) {
    // Send email, SMS notifications
}
```

### Event Sourcing Pattern

```javascript
class OrderAggregate {
    constructor(orderId) {
        this.orderId = orderId;
        this.events = [];
        this.state = this.reconstructState();
    }
    
    addEvent(event) {
        this.events.push({
            ...event,
            timestamp: new Date().toISOString(),
            version: this.events.length + 1
        });
        this.applyEvent(event);
    }
    
    applyEvent(event) {
        switch(event.type) {
            case 'ORDER_CREATED':
                this.state = { status: 'PENDING', items: event.items };
                break;
            case 'ORDER_CONFIRMED':
                this.state.status = 'CONFIRMED';
                break;
            case 'ORDER_SHIPPED':
                this.state.status = 'SHIPPED';
                this.state.trackingNumber = event.trackingNumber;
                break;
            case 'ORDER_DELIVERED':
                this.state.status = 'DELIVERED';
                break;
        }
    }
    
    reconstructState() {
        let state = { status: 'UNKNOWN', items: [] };
        this.events.forEach(event => this.applyEvent(event));
        return state;
    }
    
    // Domain methods
    create(items) {
        this.addEvent({ type: 'ORDER_CREATED', items });
    }
    
    confirm() {
        if (this.state.status === 'PENDING') {
            this.addEvent({ type: 'ORDER_CONFIRMED' });
        }
    }
    
    ship(trackingNumber) {
        if (this.state.status === 'CONFIRMED') {
            this.addEvent({ type: 'ORDER_SHIPPED', trackingNumber });
        }
    }
}
```

## Anti-Pattern

### Common Mistakes to Avoid

**1. Event Storming Without Boundaries**
```
❌ BAD: Creating events for every minor state change
- User moved mouse
- Character typed in text field
- Scroll position changed
```

```
✅ GOOD: Events represent meaningful business state changes
- ORDER_CREATED
- PAYMENT_PROCESSED
- USER_REGISTERED
```

**2. Synchronous Expectations on Asynchronous System**
```
❌ BAD: Waiting for event processing to complete
const result = await publishEvent(event);
return result; // Events are fire-and-forget!
```

```
✅ GOOD: Use callbacks or separate flows for responses
publishEvent(event);
// Handle response through separate mechanism if needed
```

**3. Large Event Payloads**
```
❌ BAD: Including entire objects in events
{
    type: 'USER_UPDATED',
    payload: { /* Entire user object with 100+ fields */ }
}
```

```
✅ GOOD: Minimal, focused event data
{
    type: 'USER_UPDATED',
    payload: {
        userId: '123',
        changedFields: ['email', 'preferences'],
        timestamp: '2024-01-01T00:00:00Z'
    }
}
```

**4. Missing Event Versioning**
```
❌ BAD: No versioning - breaking changes break consumers
{ type: 'ORDER_CREATED', orderId: '123' }
```

```
✅ GOOD: Versioned events for safe evolution
{ 
    type: 'ORDER_CREATED', 
    version: '1.0',
    orderId: '123' 
}
```

## Related Patterns

- **[Microservices Architecture](./02-Microservices-Architecture.md)** - EDA often complements microservices
- **[Scalability Patterns](./04-Scalability-Patterns.md)** - Event-driven systems scale horizontally
- **[Reliability Patterns](./05-Reliability-Patterns.md)** - Event replay for fault tolerance
- **[Event-Driven Paradigm](../03-Paradigms/03-Event-Driven-Paradigm.md)** - Programming paradigm perspective
- **[Event-Driven Infrastructure](../09-Infrastructure/05-Event-Driven-Infrastructure.md)** - Infrastructure implementation