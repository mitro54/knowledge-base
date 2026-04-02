# Microservices Architecture

An architectural style that structures an application as a collection of loosely coupled, independently deployable services.

## Summary

Microservices architecture is an approach where a single application is composed of many small, independent services that communicate over well-defined APIs. Each service is self-contained, implements a specific business capability, and can be developed, deployed, and scaled independently.

**Key Characteristics:**
- Fine-grained, loosely coupled services
- Each service implements a single business capability
- Independent deployment and scaling
- Decentralized data management
- Communication via lightweight protocols (HTTP/REST, gRPC, messaging)

---

## Problem Statement

### The Challenge

As applications grow in complexity and teams expand, monolithic architectures face significant challenges:

- **Scaling inefficiency**: Must scale entire application when only one feature needs resources
- **Deployment bottlenecks**: Single change requires redeploying everything
- **Technology lock-in**: Cannot use different technologies for different components
- **Team coordination overhead**: Merge conflicts and coordination costs increase with team size
- **Single point of failure**: Application failure affects all functionality

### Context

- **Business Context**: Organizations need faster time-to-market and continuous deployment
- **Technical Context**: Cloud infrastructure enables distributed deployment at scale
- **Team Context**: Large organizations have multiple teams with different expertise

### Consequences of Not Addressing

Without addressing monolithic limitations:
- Deployment frequency decreases as codebase grows
- Time-to-market slows due to coordination overhead
- Innovation is stifled by technology constraints
- System becomes fragile and difficult to modify

---

## Solution

### The Microservices Approach

A microservices architecture decomposes an application into independent services:

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer              │
└───────────────┬──────────────────┬──────────────────┬───────┘
                │                  │                  │
    ┌───────────▼────────┐  ┌─────▼────────┐  ┌─────▼────────┐
    │   User Service    │  │ Product      │  │   Order      │
    │   (Node.js)       │  │  Service     │  │   Service    │
    │   PostgreSQL      │  │  (Java)      │  │  (Go)        │
    └───────────┬────────┘  └─────┬────────┘  └─────┬────────┘
                │                  │                  │
    ┌───────────▼────────┐  ┌─────▼────────┐  ┌─────▼────────┐
    │  User Database    │  │ Product      │  │   Order      │
    │                   │  │  Database    │  │   Database   │
    └───────────────────┘  └──────────────┘  └──────────────┘
```

### Key Components

1. **Services**: Independent, deployable units implementing business capabilities
2. **API Gateway**: Single entry point routing requests to appropriate services
3. **Service Discovery**: Mechanism for services to locate each other
4. **Message Queue**: Asynchronous communication between services
5. **Distributed Data**: Each service manages its own database

### How It Addresses the Problem

- **Independent Scaling**: Scale only services that need resources
- **Technology Diversity**: Each service can use different technology stack
- **Isolated Failures**: Service failure doesn't crash entire system
- **Parallel Development**: Teams work independently on different services
- **Faster Deployment**: Deploy individual services without full redeployment

---

## When to Use

### Appropriate Scenarios

| Scenario | Suitability |
|----------|-------------|
| Large, complex applications | ⭐⭐⭐⭐⭐ Excellent |
| Multiple development teams | ⭐⭐⭐⭐⭐ Excellent |
| Variable load patterns | ⭐⭐⭐⭐⭐ Excellent |
| Need for technology diversity | ⭐⭐⭐⭐⭐ Excellent |
| High availability requirements | ⭐⭐⭐⭐ Very Good |
| Small applications | ⭐ Poor |

### Prerequisites

- Sufficient team size to manage distributed complexity
- Infrastructure for service deployment and monitoring
- Understanding of distributed system challenges
- Clear domain boundaries (Domain-Driven Design helpful)

### Indicators for Suitability

- Different parts of application have different scaling needs
- Multiple teams need to work independently
- You need to use different technologies for different components
- Deployment frequency requirements exceed monolithic capabilities
- Business requires high availability and fault isolation

---

## Tradeoffs

### Advantages

| Advantage | Description |
|-----------|-------------|
| **Independent Scaling** | Scale only services that need resources |
| **Technology Diversity** | Each service can use optimal technology |
| **Fault Isolation** | Service failure doesn't crash entire system |
| **Parallel Development** | Teams work independently with minimal coordination |
| **Faster Deployment** | Deploy individual services without full redeployment |
| **Clear Boundaries** | Service boundaries enforce modular design |
| **Easier Testing** | Smaller codebases are easier to test thoroughly |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| **Distributed Complexity** | Must handle network latency, failures, partial outages |
| **Operational Overhead** | More services to monitor, deploy, and maintain |
| **Data Consistency** | Distributed transactions are complex (see CAP Theorem) |
| **Service Communication** | Network calls are slower and less reliable than function calls |
| **Testing Complexity** | Integration and end-to-end testing are more complex |
| **Debugging Difficulty** | Distributed tracing required for request tracking |
| **Development Overhead** | More infrastructure code per feature |

### Performance Considerations

- **Network Latency**: Service-to-service calls add latency
- **Serialization**: Data must be serialized/deserialized for transmission
- **Caching**: More complex due to distributed nature
- **Database**: Each service has its own connection pool

### Complexity Implications

- **Initial Complexity**: High - requires infrastructure setup
- **Long-term Complexity**: Medium - services remain manageable size
- **Operational Complexity**: High - distributed system management

---

## Implementation Example

### Service Structure

```
microservices-project/
├── api-gateway/
│   ├── src/
│   │   ├── main.go
│   │   ├── routes/
│   │   └── middleware/
│   └── Dockerfile
├── user-service/
│   ├── src/
│   │   ├── main.js
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   └── Dockerfile
├── product-service/
│   ├── src/
│   │   ├── main.java
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   └── Dockerfile
├── order-service/
│   ├── src/
│   │   ├── main.go
│   │   ├── handlers/
│   │   └── models/
│   └── Dockerfile
├── docker-compose.yml
└── kubernetes/
    ├── deployments/
    └── services/
```

### API Gateway (Go)

```go
package main

import (
    "github.com/gorilla/mux"
    "net/http"
)

func main() {
    r := mux.NewRouter()
    
    // Route to User Service
    r.PathPrefix("/api/users").Handler(http.StripPrefix("/api/users", 
        http.ProxyURL("http://user-service:3001"))).Methods("GET", "POST", "PUT", "DELETE")
    
    // Route to Product Service
    r.PathPrefix("/api/products").Handler(http.StripPrefix("/api/products",
        http.ProxyURL("http://product-service:8080"))).Methods("GET", "POST", "PUT", "DELETE")
    
    // Route to Order Service
    r.PathPrefix("/api/orders").Handler(http.StripPrefix("/api/orders",
        http.ProxyURL("http://order-service:8081"))).Methods("GET", "POST", "PUT", "DELETE")
    
    log.Println("API Gateway running on :8080")
    http.ListenAndServe(":8080", r)
}
```

### User Service (Node.js)

```javascript
// src/main.js
const express = require('express');
const mongoose = require('mongoose');
const userRoutes = require('./routes/users');

const app = express();

// Connect to User Service database
mongoose.connect(process.env.USER_DB_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true
});

app.use(express.json());
app.use('/api/users', userRoutes);

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
    console.log(`User Service running on port ${PORT}`);
});

// src/routes/users.js
const { getUser, createUser } = require('../services/user.service');

router.get('/:id', async (req, res) => {
    try {
        const user = await getUser(req.params.id);
        res.json(user);
    } catch (error) {
        res.status(404).json({ error: 'User not found' });
    }
});

router.post('/', async (req, res) => {
    try {
        const user = await createUser(req.body);
        res.status(201).json(user);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - user-service
      - product-service
      - order-service

  user-service:
    build: ./user-service
    environment:
      - USER_DB_URI=mongodb://user-db:27017/users
    depends_on:
      - user-db

  product-service:
    build: ./product-service
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://product-db:3306/products
    depends_on:
      - product-db

  order-service:
    build: ./order-service
    environment:
      - ORDER_DB_URI=postgres://order-db:5432/orders
    depends_on:
      - order-db

  user-db:
    image: mongo:4.4

  product-db:
    image: mysql:8.0

  order-db:
    image: postgres:13
```

---

## Anti-Pattern

### Common Mistakes

#### 1. Distributed Monolith
```
❌ ANTI-PATTERN: Services tightly coupled like monolith

┌─────────────────────────────────────────────────────────┐
│              Shared Database (All Services)             │
└─────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    ┌────┴────┐    ┌───┴───┐    ┌────┴────┐
    │ Service │    │Service│    │ Service │
    │   A     │◄──►│   B   │◄──►│    C    │
    └─────────┘    └───────┘    └─────────┘
         │              │              │
         └──────────────┴──────────────┘
                    Synchronous
                   Tight Coupling
```

```javascript
// ❌ ANTI-PATTERN: Direct database access across services
async function processOrder(order) {
    // Order service directly accessing User service database
    const user = await db.collection('users').findOne({ id: order.userId });
    
    // Direct call to another service's internal API
    const inventory = await fetch('http://product-service/internal/inventory');
    
    // Synchronous cascade of calls
    await paymentService.charge(order.total);
    await notificationService.sendEmail(user.email);
    await analyticsService.track(order);
}
```

#### 2. Fine-Grained Services
```
❌ ANTI-PATTERN: Services too small, excessive overhead

├── greeting-service/
├── farewell-service/
├── comma-service/
├── period-service/
└── newline-service/
```

#### 3. Shared Libraries Between Services
```
❌ ANTI-PATTERN: Shared code creates coupling

shared-libraries/
├── common-utils/
├── domain-models/
└── validation-rules/

# Each service depends on shared libraries
# Changes require coordination and redeployment
```

### Warning Signs

- Services share databases
- Services call each other synchronously in chains
- Data consistency requires distributed transactions
- Services are too small (less than one business capability)
- Team coordination required for simple changes

### What NOT to Do

1. **Don't** share databases between services
2. **Don't** create synchronous call chains
3. **Don't** make services too fine-grained
4. **Don't** share libraries between services
5. **Don't** use microservices for simple applications

---

## Related Patterns

### Complementary Patterns

- [API Gateway Pattern](02-Design-Patterns/02-Structural-Patterns.md) - Entry point for microservices
- [Circuit Breaker](05-Safety-Engineering/02-Fault-Tolerance.md) - Handle service failures
- [Event-Driven Architecture](01-System-Design/03-Event-Driven-Architecture.md) - Asynchronous communication

### Alternative Approaches

- [Monolithic Architecture](01-System-Design/01-Monolithic-Architecture.md) - Single unit approach
- [Event-Driven Architecture](01-System-Design/03-Event-Driven-Architecture.md) - Message-based approach

### Evolution Path

- Start with [Monolithic Architecture](01-System-Design/01-Monolithic-Architecture.md)
- Apply [Modular Monolith](02-Design-Patterns/05-Architectural-Patterns.md) principles
- Extract services when boundaries are clear
- Adopt [Microservices Architecture](01-System-Design/02-Microservices-Architecture.md)

### See Also

- [CAP Theorem](07-Distributed-Systems/01-CAP-Theorem.md) - Distributed system tradeoffs
- [Service Mesh](09-Infrastructure/04-Service-Mesh.md) - Service communication infrastructure
- [Containerization](09-Infrastructure/01-Containerization.md) - Deployment packaging