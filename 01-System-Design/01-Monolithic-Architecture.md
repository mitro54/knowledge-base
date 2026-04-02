# Monolithic Architecture

A traditional application architecture where all components are contained within a single codebase and deployed as a single unit.

## Summary

Monolithic architecture is the traditional approach to building software applications, where the entire application is built as a single, unified unit. All components—user interface, business logic, data access, and infrastructure—are tightly coupled and deployed together as one executable or service.

**Key Characteristics:**
- Single codebase containing all functionality
- Single deployment unit
- Shared memory and resources across components
- Synchronous communication between components
- Single database connection (typically)

---

## Problem Statement

### The Challenge

When building software applications, developers face the fundamental challenge of organizing code into a coherent, maintainable structure. The monolithic approach emerged as the natural solution before distributed systems became practical.

### Context

- **Historical Context**: Monolithic architecture has been the default approach since the early days of computing
- **Team Context**: Small teams working on simple applications find monolithic structure intuitive
- **Technical Context**: Single-server deployments make monolithic architecture the obvious choice

### Consequences of Not Addressing

Without a clear architectural approach:
- Code becomes disorganized and difficult to navigate
- Components become tightly coupled without intention
- Scaling becomes impossible without complete redeployment
- Team collaboration suffers from merge conflicts

---

## Solution

### The Monolithic Approach

A monolithic architecture organizes an application into a single, cohesive unit with the following structure:

```
┌─────────────────────────────────────┐
│           Application               │
│  ┌─────────────────────────────────┐│
│  │         Presentation Layer      ││
│  │    (UI, Controllers, Routes)    ││
│  └─────────────────────────────────┘│
│  ┌─────────────────────────────────┐│
│  │       Business Logic Layer      ││
│  │   (Services, Managers, Utils)   ││
│  └─────────────────────────────────┘│
│  ┌─────────────────────────────────┐│
│  │        Data Access Layer        ││
│  │   (Repositories, Models, ORM)   ││
│  └─────────────────────────────────┘│
│  ┌─────────────────────────────────┐│
│  │         Database                ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

### Key Components

1. **Presentation Layer**: Handles user interface and HTTP requests
2. **Business Logic Layer**: Contains core application functionality
3. **Data Access Layer**: Manages database interactions
4. **Shared Resources**: Memory, threads, database connections

### How It Addresses the Problem

- **Simplicity**: Single codebase is easy to understand and navigate
- **Development Speed**: No distributed system complexity to manage
- **Testing**: Easier to test with in-memory databases and mocks
- **Deployment**: Single artifact to deploy and manage

---

## When to Use

### Appropriate Scenarios

| Scenario | Suitability |
|----------|-------------|
| Small applications | ⭐⭐⭐⭐⭐ Excellent |
| Prototypes and MVPs | ⭐⭐⭐⭐⭐ Excellent |
| Small team (< 5 developers) | ⭐⭐⭐⭐⭐ Excellent |
| Simple domain logic | ⭐⭐⭐⭐⭐ Excellent |
| Single-server deployment | ⭐⭐⭐⭐⭐ Excellent |
| Low traffic applications | ⭐⭐⭐⭐ Very Good |

### Prerequisites

- Application scope is well-defined and limited
- Team size is small
- Expected traffic is moderate
- Single skill set covers all layers

### Indicators for Suitability

- You can describe the entire application in one paragraph
- All components have similar scaling requirements
- Team can deploy multiple times per day without coordination
- No need for independent component updates

---

## Tradeoffs

### Advantages

| Advantage | Description |
|-----------|-------------|
| **Simplicity** | Single codebase is easy to understand and navigate |
| **Development Speed** | No distributed system complexity; faster initial development |
| **Testing** | Easier to test with in-memory databases and mocks |
| **Deployment** | Single artifact to build and deploy |
| **Debugging** | Stack traces are straightforward; no distributed tracing needed |
| **Performance** | No network latency between components; in-memory communication |
| **Transaction Management** | ACID transactions are straightforward with single database |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| **Scalability** | Must scale entire application even if only one component needs resources |
| **Deployment Risk** | Any change requires redeploying entire application |
| **Technology Lock-in** | All components must use same technology stack |
| **Codebase Bloat** | Application grows continuously without natural boundaries |
| **Team Coordination** | Merge conflicts increase with team size |
| **Single Point of Failure** | Application failure affects all functionality |

### Performance Considerations

- **Memory**: Entire application loaded into memory
- **CPU**: All components compete for CPU resources
- **Network**: No inter-component network latency
- **Database**: Single connection pool for all operations

### Complexity Implications

- **Initial Complexity**: Low - straightforward structure
- **Long-term Complexity**: High - grows with application size
- **Operational Complexity**: Low - single deployment unit

---

## Implementation Example

### Basic Monolithic Structure

```
my-application/
├── src/
│   ├── main.js              # Application entry point
│   ├── config/
│   │   └── database.js      # Database configuration
│   ├── controllers/
│   │   ├── user.controller.js
│   │   └── product.controller.js
│   ├── services/
│   │   ├── user.service.js
│   │   └── product.service.js
│   ├── models/
│   │   ├── user.model.js
│   │   └── product.model.js
│   ├── routes/
│   │   └── index.js
│   └── utils/
│       └── helpers.js
├── tests/
│   ├── unit/
│   └── integration/
├── package.json
└── README.md
```

### Code Example (Node.js/Express)

```javascript
// src/main.js
const express = require('express');
const db = require('./config/database');
const userRoutes = require('./routes/user.routes');
const productRoutes = require('./routes/product.routes');

const app = express();

// Middleware
app.use(express.json());

// Routes
app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Monolithic application running on port ${PORT}`);
});

// src/controllers/user.controller.js
const userService = require('../services/user.service');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await userService.findAll();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.getUserById = async (req, res) => {
  try {
    const user = await userService.findById(req.params.id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Database Schema

```sql
-- Single database for all entities
CREATE DATABASE monolithic_app;

USE monolithic_app;

CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

## Anti-Pattern

### Common Mistakes

#### 1. God Object / God Class
```javascript
// ❌ ANTI-PATTERN: God Object
class Application {
    constructor() {
        this.db = new Database();
        this.auth = new Auth();
        this.email = new EmailService();
        this.payment = new PaymentGateway();
        this.analytics = new Analytics();
        // ... 20 more dependencies
    }
    
    // Handles everything
    handleRequest(request) {
        // 500+ lines of mixed concerns
    }
}
```

#### 2. Tight Coupling Without Structure
```javascript
// ❌ ANTI-PATTERN: Direct database access in controllers
const mysql = require('mysql');

exports.createUser = async (req, res) => {
    const connection = mysql.createConnection({ /* config */ });
    
    // Direct SQL in controller
    await connection.query(
        'INSERT INTO users SET ?', 
        req.body
    );
    
    // Then query another table directly
    await connection.query(
        'INSERT INTO audit_log SET ?', 
        { action: 'user_created', user_id: lastId }
    );
    
    connection.end();
};
```

#### 3. No Layer Separation
```javascript
// ❌ ANTI-PATTERN: Mixed concerns in single file
const express = require('express');
const mysql = require('mysql');

// Database config, business logic, and routes all mixed
app.post('/users', async (req, res) => {
    // Validation
    if (!req.body.email) {
        return res.status(400).json({ error: 'Email required' });
    }
    
    // Business logic
    const hashedPassword = bcrypt.hash(req.body.password, 10);
    
    // Database access
    const conn = mysql.createConnection({ /* config */ });
    await conn.query('INSERT INTO users...');
    
    // Email sending
    await sendWelcomeEmail(req.body.email);
    
    // Response
    res.json({ success: true });
});
```

### Warning Signs

- Single files exceeding 1000 lines
- Controllers directly accessing database
- Business logic scattered across layers
- Import cycles between modules
- "It works but I don't know why" syndrome

### What NOT to Do

1. **Don't** put all code in a single file
2. **Don't** access database directly from controllers
3. **Don't** mix validation, business logic, and data access
4. **Don't** ignore the need for structure just because it's monolithic
5. **Don't** let the monolith grow without refactoring

---

## Related Patterns

### Complementary Patterns

- [Clean Code](04-Best-Practices/01-Clean-Code.md) - Essential for maintaining monolithic codebases
- [SOLID Principles](04-Best-Practices/02-SOLID-Principles.md) - Helps organize monolithic code
- [Code Organization](04-Best-Practices/05-Code-Organization.md) - Structuring monolithic applications

### Alternative Approaches

- [Microservices Architecture](01-System-Design/02-Microservices-Architecture.md) - Distributed alternative
- [Event-Driven Architecture](01-System-Design/03-Event-Driven-Architecture.md) - Asynchronous alternative

### Evolution Path

- Start with [Monolithic Architecture](01-System-Design/01-Monolithic-Architecture.md)
- Apply [Clean Architecture](02-Design-Patterns/05-Architectural-Patterns.md) principles
- Consider [Microservices Architecture](01-System-Design/02-Microservices-Architecture.md) when needed

### See Also

- [Scalability Patterns](01-System-Design/04-Scalability-Patterns.md) - Scaling monolithic applications
- [Performance Patterns](01-System-Design/07-Performance-Patterns.md) - Optimizing monolithic performance