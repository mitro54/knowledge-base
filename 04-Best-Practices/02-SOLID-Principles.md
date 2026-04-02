# SOLID Principles

## Title & Summary

SOLID is an acronym representing five fundamental principles of object-oriented design and software engineering. These principles, introduced by Robert C. Martin (Uncle Bob), help developers create maintainable, scalable, and understandable software by guiding design decisions at the class and module level.

## Problem Statement

Without design principles, software tends to suffer from:
- **Tightly coupled code**: Changes in one module break unrelated modules
- **Rigid structures**: Simple modifications require extensive changes
- **Fragile systems**: Changes cause unexpected failures elsewhere
- **Inflexible designs**: Difficult to extend without modification
- **Complex dependencies**: Hard to understand, test, and maintain
- **Code duplication**: Similar functionality scattered across the codebase

## Solution

### The Five SOLID Principles

**S - Single Responsibility Principle (SRP)**

A class should have only one reason to change.

```typescript
// ❌ Violates SRP - Multiple responsibilities
class User {
    saveToDatabase() { }
    sendEmail() { }
    validate() { }
    generateReport() { }
}

// ✅ Follows SRP - Single responsibility
class UserRepository {
    save(user: User) { }
}

class EmailService {
    send(to: string, body: string) { }
}

class UserValidator {
    validate(user: User): boolean { }
}
```

**O - Open/Closed Principle (OCP)**

Software entities should be open for extension but closed for modification.

```typescript
// ❌ Violates OCP - Modified for each new shape
class ShapeCalculator {
    calculateArea(shape: string, params: any): number {
        if (shape === 'rectangle') return params.width * params.height;
        if (shape === 'circle') return Math.PI * params.radius ** 2;
        if (shape === 'triangle') return params.base * params.height / 2;
        // Must modify this class for every new shape!
    }
}

// ✅ Follows OCP - Extended without modification
abstract class Shape {
    abstract getArea(): number;
}

class Rectangle extends Shape {
    constructor(private width: number, private height: number) { }
    getArea(): number { return this.width * this.height; }
}

class Circle extends Shape {
    constructor(private radius: number) { }
    getArea(): number { return Math.PI * this.radius ** 2; }
}

// New shapes added by extending, not modifying
```

**L - Liskov Substitution Principle (LSP)**

Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.

```typescript
// ❌ Violates LSP - Subclass breaks expected behavior
class Bird {
    fly() { console.log('Flying'); }
}

class Penguin extends Bird {
    fly() { throw new Error('Penguins cannot fly!'); }
}

// ✅ Follows LSP - Proper hierarchy
class Bird {
    move() { console.log('Moving'); }
}

class FlyingBird extends Bird {
    fly() { console.log('Flying'); }
}

class SwimmingBird extends Bird {
    swim() { console.log('Swimming'); }
}
```

**I - Interface Segregation Principle (ISP)**

Clients should not be forced to depend on interfaces they do not use.

```typescript
// ❌ Violates ISP - Fat interface
interface Worker {
    work(): void;
    eat(): void;
    sleep(): void;
}

// Human must implement eat() and sleep() even if not relevant
class Robot implements Worker {
    work() { }
    eat() { throw new Error('Robots do not eat'); }
    sleep() { throw new Error('Robots do not sleep'); }
}

// ✅ Follows ISP - Segregated interfaces
interface Workable {
    work(): void;
}

interface Eatable {
    eat(): void;
}

interface Sleepable {
    sleep(): void;
}

class Human implements Workable, Eatable, Sleepable {
    work() { }
    eat() { }
    sleep() { }
}

class Robot implements Workable {
    work() { }
}
```

**D - Dependency Inversion Principle (DIP)**

High-level modules should not depend on low-level modules. Both should depend on abstractions.

```typescript
// ❌ Violates DIP - High-level depends on low-level
class MySQLDatabase {
    query(sql: string) { }
}

class UserService {
    private db = new MySQLDatabase(); // Tight coupling!
    getUser(id: number) {
        return this.db.query(`SELECT * FROM users WHERE id = ${id}`);
    }
}

// ✅ Follows DIP - Both depend on abstraction
interface Database {
    query(sql: string): any;
}

class MySQLDatabase implements Database {
    query(sql: string) { }
}

class PostgreSQLDatabase implements Database {
    query(sql: string) { }
}

class UserService {
    constructor(private db: Database) { } // Depends on abstraction
    getUser(id: number) {
        return this.db.query(`SELECT * FROM users WHERE id = ${id}`);
    }
}
```

### Principle Interactions

```
SRP → Creates focused classes
  ↓
OCP → Allows extension without modification
  ↓
LSP → Ensures substitutions work correctly
  ↓
ISP → Creates focused interfaces
  ↓
DIP → Connects via abstractions
```

## When to Use

**Apply SOLID principles when:**
- Designing new classes and modules
- Refactoring legacy code
- Experiencing difficulty with testing
- Facing challenges extending functionality
- Noticing tight coupling between modules
- Planning for long-term maintenance

**Prioritize for:**
- Core domain objects
- Service layers
- Repository patterns
- Plugin architectures
- Library development

## Tradeoffs

| Tradeoff | Consideration |
|----------|---------------|
| **Increased Complexity** | More classes/interfaces initially; pays off long-term |
| **Learning Curve** | Team must understand principles to apply correctly |
| **Over-Engineering Risk** | Simple scripts don't need full SOLID treatment |
| **Performance Overhead** | Abstractions add minimal overhead but improve maintainability |
| **Upfront Investment** | Takes more time initially but reduces technical debt |

## Implementation Example

### Complete Example: Payment Processing System

```typescript
// SRP: Each class has single responsibility
// ISP: Small, focused interfaces
interface PaymentProcessor {
    processPayment(amount: number, currency: string): PaymentResult;
}

interface PaymentGateway {
    charge(token: string, amount: number): boolean;
}

interface PaymentRepository {
    save(payment: Payment): void;
    findById(id: string): Payment | null;
}

// OCP: Open for extension (new payment methods)
// LSP: All processors can be substituted
abstract class BasePaymentProcessor implements PaymentProcessor {
    constructor(
        protected gateway: PaymentGateway,
        protected repository: PaymentRepository
    ) { }

    abstract processPayment(amount: number, currency: string): PaymentResult;
}

class CreditCardProcessor extends BasePaymentProcessor {
    processPayment(amount: number, currency: string): PaymentResult {
        const token = this.generateToken();
        const success = this.gateway.charge(token, amount);
        if (success) {
            this.repository.save({ amount, currency, type: 'credit_card' });
        }
        return { success };
    }

    private generateToken(): string { return 'token'; }
}

class PayPalProcessor extends BasePaymentProcessor {
    processPayment(amount: number, currency: string): PaymentResult {
        // PayPal-specific logic
        return { success: true };
    }
}

// DIP: Depends on abstractions, not concrete implementations
class OrderService {
    constructor(
        private paymentProcessor: PaymentProcessor,
        private orderRepository: OrderRepository
    ) { }

    checkout(order: Order): CheckoutResult {
        const payment = this.paymentProcessor.processPayment(
            order.total,
            order.currency
        );
        if (payment.success) {
            this.orderRepository.complete(order);
        }
        return payment;
    }
}
```

## Anti-Pattern

### The "God Class" Anti-Pattern

```typescript
// ❌ Anti-pattern: Violates SRP, OCP, LSP, ISP, DIP
class Application {
    private db = new MySQLDatabase();
    private email = new EmailService();
    private logger = new Logger();

    processOrder(order: Order) {
        // Validation
        if (!order.items || order.items.length === 0) {
            throw new Error('No items');
        }

        // Database operations
        const user = this.db.query('SELECT * FROM users WHERE id = ?', order.userId);
        const inventory = this.db.query('SELECT * FROM inventory WHERE product_id IN (?)', order.items);

        // Business logic
        let total = 0;
        for (const item of order.items) {
            total += item.price * item.quantity;
        }

        // Payment processing (multiple methods hardcoded)
        if (order.paymentMethod === 'credit_card') {
            // Credit card logic inline
        } else if (order.paymentMethod === 'paypal') {
            // PayPal logic inline
        }

        // Email sending
        this.email.send(user.email, 'Order confirmed', `Total: ${total}`);

        // Logging
        this.logger.log(`Order ${order.id} processed`);
    }
}
```

**Violations:**
- **SRP**: Does everything - validation, DB, business logic, payment, email, logging
- **OCP**: Must modify to add new payment methods
- **LSP**: Cannot substitute parts
- **ISP**: One massive interface
- **DIP**: Depends on concrete implementations

## Related Patterns

- **[Clean Code](./01-Clean-Code.md)** - Complementary principles for code quality
- **[Design Principles](./03-Design-Principles.md)** - DRY, KISS, YAGNI work with SOLID
- **[Creational Patterns](../02-Design-Patterns/01-Creational-Patterns.md)** - Factory pattern supports DIP
- **[Structural Patterns](../02-Design-Patterns/02-Structural-Patterns.md)** - Adapter, Facade support OCP
- **[Dependency Injection](../02-Design-Patterns/01-Creational-Patterns.md)** - Implementation of DIP