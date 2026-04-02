# Design Principles

## Title & Summary

Design principles are fundamental guidelines that help developers make architectural and implementation decisions. Beyond SOLID, several key principles—DRY, KISS, YAGNI, GRASP, and others—provide additional guidance for creating maintainable, efficient, and understandable software systems.

## Problem Statement

Software development involves countless decisions. Without guiding principles:
- **Code duplication** leads to maintenance nightmares
- **Over-engineering** creates unnecessary complexity
- **Premature optimization** wastes development time
- **Feature creep** bloats systems with unused functionality
- **Poor separation of concerns** creates tangled dependencies
- **Inconsistent design** makes codebases unpredictable

## Solution

### Core Design Principles

**DRY - Don't Repeat Yourself**

Every piece of knowledge must have a single, unambiguous representation.

```javascript
// ❌ Violates DRY - Repeated logic
function formatPhoneNumberUS(number) {
    return `+1 (${number.slice(0, 3)}) ${number.slice(3, 6)}-${number.slice(6)}`;
}

function formatPhoneNumberCA(number) {
    return `+1 (${number.slice(0, 3)}) ${number.slice(3, 6)}-${number.slice(6)}`;
}

function formatPhoneNumberMX(number) {
    return `+52 (${number.slice(0, 2)}) ${number.slice(2, 7)}-${number.slice(7)}`;
}

// ✅ Follows DRY - Extracted common logic
function formatPhoneNumber(number, countryCode) {
    const format = countryCode === 'MX' 
        ? `+${countryCode} (${number.slice(0, 2)}) ${number.slice(2, 7)}-${number.slice(7)}`
        : `+1 (${number.slice(0, 3)}) ${number.slice(3, 6)}-${number.slice(6)}`;
    return format;
}
```

**KISS - Keep It Simple, Stupid**

Simplicity should be a primary goal; complexity should be avoided unless necessary.

```javascript
// ❌ Violates KISS - Overly complex
function calculateDiscount(price, customerType, date, quantity, promotionCode) {
    const baseDiscount = customerType === 'VIP' ? 0.1 : 0.05;
    const seasonalMultiplier = isHolidaySeason(date) ? 1.5 : 1.0;
    const bulkDiscount = quantity > 10 ? 0.02 * Math.log(quantity) : 0;
    const promoDiscount = promotionCodes[promotionCode]?.discount || 0;
    return Math.min(0.5, baseDiscount * seasonalMultiplier + bulkDiscount + promoDiscount);
}

// ✅ Follows KISS - Simple and clear
function calculateDiscount(price, customerType, quantity) {
    let discount = customerType === 'VIP' ? 0.1 : 0.05;
    if (quantity > 10) discount += 0.02;
    return Math.min(0.5, discount);
}
```

**YAGNI - You Aren't Gonna Need It**

Don't add functionality until you have a concrete need for it.

```javascript
// ❌ Violates YAGNI - Anticipating future needs
class User {
    constructor(name, email, phone, address, ssn, passportId, driverLicense) {
        this.name = name;
        this.email = email;
        // All these might be needed someday...
        this.phone = phone;
        this.address = address;
        this.ssn = ssn;
        this.passportId = passportId;
        this.driverLicense = driverLicense;
    }
}

// ✅ Follows YAGNI - Only what's needed now
class User {
    constructor(name, email) {
        this.name = name;
        this.email = email;
    }
}
// Add fields when there's an actual requirement
```

**GRASP - General Responsibility Assignment Software Patterns**

A set of principles for assigning responsibilities in object-oriented design:

| Pattern | Description |
|--------|-------------|
| **Information Expert** | Assign responsibility to the class with needed information |
| **Creator** | Class A creates B if A stores B, contains B, or knows about B |
| **High Cohesion** | Keep related things together; classes should have focused purpose |
| **Low Coupling** | Minimize dependencies between classes |
| **Controller** | Non-UI class handles system use case requests |
| **Polymorphism** | Use polymorphism instead of conditional logic |
| **Pure Fabrication** | Create class solely to achieve low coupling/high cohesion |
| **Indirection** | Use intermediary to reduce coupling |

```javascript
// GRASP: Information Expert + Creator
class Order {
    constructor(items) {
        this.items = items;
        this.lineItems = items.map(item => new OrderLineItem(item));
    }
    
    getTotal() {
        return this.lineItems.reduce((sum, item) => sum + item.subtotal, 0);
    }
}

class OrderLineItem {
    constructor({ product, quantity }) {
        this.product = product;
        this.quantity = quantity;
    }
    
    get subtotal() {
        return this.product.price * this.quantity;
    }
}
```

**Additional Principles**

**Tell, Don't Ask**
```javascript
// ❌ Ask - Client knows internal details
const account = new Account(100);
if (account.getBalance() >= 10) {
    account.setBalance(account.getBalance() - 10);
}

// ✅ Tell - Object handles its own logic
const account = new Account(100);
account.withdraw(10);
```

**Law of Demeter**
Objects should only talk to their immediate friends.

```javascript
// ❌ Violates Law of Demeter - Too many dots
const total = order.getCustomer().getAddress().getCity().getName();

// ✅ Follows Law of Demeter - Only immediate friends
const city = order.getShippingCity();
```

**Composition Over Inheritance**
```javascript
// ❌ Inheritance - Rigid hierarchy
class Bird { fly() { } }
class Penguin extends Bird { fly() { throw Error(); } }

// ✅ Composition - Flexible combination
class Flyable { fly() { } }
class Bird { constructor(flyBehavior) { this.flyBehavior = flyBehavior; } }
class Penguin extends Bird { constructor() { super(null); } }
```

## When to Use

| Principle | Best Applied When |
|----------|------------------|
| **DRY** | Seeing repeated code patterns, building utilities |
| **KISS** | Designing APIs, writing algorithms, debugging |
| **YAGNI** | Planning features, designing data models, estimating |
| **GRASP** | Assigning responsibilities, designing classes |
| **Tell Don't Ask** | Designing object interactions |
| **Law of Demeter** | Reducing coupling, refactoring |
| **Composition Over Inheritance** | Designing extensible systems |

## Tradeoffs

| Principle | Tradeoff | Consideration |
|----------|---------|---------------|
| **DRY** | Abstraction complexity | Sometimes duplication is clearer than complex abstraction |
| **KISS** | May miss optimization | Simple isn't always optimal; balance needed |
| **YAGNI** | May need refactoring | Adding later may require more work than adding now |
| **Composition** | More objects | More objects to manage but more flexible |

## Implementation Example

### E-Commerce Cart Using Multiple Principles

```javascript
// KISS: Simple, focused classes
// YAGNI: Only fields we need now
class CartItem {
    constructor(product, quantity) {
        this.product = product;
        this.quantity = quantity;
    }
    
    get subtotal() {
        return this.product.price * this.quantity;
    }
}

// DRY: Reusable discount calculation
// KISS: Simple discount types
class Discount {
    constructor(type, value) {
        this.type = type; // 'percentage' or 'fixed'
        this.value = value;
    }
    
    apply(subtotal) {
        return this.type === 'percentage' 
            ? subtotal * (1 - this.value / 100)
            : Math.max(0, subtotal - this.value);
    }
}

// GRASP: Information Expert - Cart knows its items
// Tell Don't Ask - Cart handles its own logic
class ShoppingCart {
    constructor() {
        this.items = [];
        this.discounts = [];
    }
    
    addItem(product, quantity) {
        const existing = this.items.find(i => i.product.id === product.id);
        if (existing) {
            existing.quantity += quantity;
        } else {
            this.items.push(new CartItem(product, quantity));
        }
    }
    
    removeItem(productId) {
        this.items = this.items.filter(i => i.product.id !== productId);
    }
    
    addDiscount(discount) {
        this.discounts.push(discount);
    }
    
    get subtotal() {
        return this.items.reduce((sum, item) => sum + item.subtotal, 0);
    }
    
    get total() {
        let total = this.subtotal;
        // DRY: Reuse discount.apply()
        this.discounts.forEach(discount => {
            total = discount.apply(total);
        });
        return total;
    }
    
    get itemCount() {
        return this.items.reduce((sum, item) => sum + item.quantity, 0);
    }
}

// Usage - KISS: Simple API
const cart = new ShoppingCart();
cart.addItem({ id: 1, name: 'Widget', price: 29.99 }, 2);
cart.addItem({ id: 2, name: 'Gadget', price: 49.99 }, 1);
cart.addDiscount(new Discount('percentage', 10));

console.log(cart.total); // 74.97 (after 10% discount)
```

## Anti-Pattern

### The "Just in Case" Anti-Pattern

```javascript
// ❌ Anti-pattern: Violates YAGNI, KISS
class Product {
    constructor(
        id, name, description, price, sku, upc, isbn,
        weight, dimensions, color, size, material,
        manufacturer, origin, warranty, 
        tags, categories, attributes, metadata,
        seoTitle, seoDescription, seoKeywords,
        relatedProducts, upsellProducts, crossSellProducts
    ) {
        // All these fields "just in case" we need them
        this.id = id;
        this.name = name;
        // ... 30 more assignments
    }
    
    // Methods for features we might build
    getSeoUrl() { /* Not used yet */ }
    getRelatedProducts() { /* Not used yet */ }
    calculateShipping() { /* Not used yet */ }
}
```

**Problems:**
- Violates YAGNI - Adding features not yet needed
- Violates KISS - Overly complex for current needs
- Harder to understand and maintain
- Slows down development

### The "Copy-Paste" Anti-Pattern

```javascript
// ❌ Anti-pattern: Violates DRY
function validateEmail(email) {
    if (!email || email.length < 5) return false;
    if (!email.includes('@')) return false;
    if (!email.includes('.')) return false;
    return true;
}

function validateContactEmail(email) {
    if (!email || email.length < 5) return false;
    if (!email.includes('@')) return false;
    if (!email.includes('.')) return false;
    return true;
}

function validateReplyToEmail(email) {
    if (!email || email.length < 5) return false;
    if (!email.includes('@')) return false;
    if (!email.includes('.')) return false;
    return true;
}
```

**Solution:** Use one `validateEmail` function everywhere.

## Related Patterns

- **[SOLID Principles](./02-SOLID-Principles.md)** - Complementary design principles
- **[Clean Code](./01-Clean-Code.md)** - Code-level application of principles
- **[OOP Paradigm](../03-Paradigms/01-OOP-Paradigm.md)** - GRASP applies to OOP
- **[Functional Paradigm](../03-Paradigms/02-Functional-Paradigm.md)** - KISS applies especially well
- **[Code Organization](./05-Code-Organization.md)** - Principles guide structure