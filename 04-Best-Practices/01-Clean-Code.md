# Clean Code

## Title & Summary

Clean Code is a methodology and set of principles focused on writing code that is readable, maintainable, and understandable. The core philosophy, popularized by Robert C. Martin (Uncle Bob), emphasizes that code is read far more often than it is written, making readability paramount.

## Problem Statement

Software systems often degrade over time due to:
- **Accumulated technical debt**: Quick fixes compound into complex, fragile code
- **Poor naming**: Variables and functions with unclear purposes
- **Excessive complexity**: Functions that do too much, classes that are too large
- **Inconsistent formatting**: Making code harder to scan and understand
- **Dead code**: Unused functions, commented-out blocks, obsolete features
- **Magic numbers/strings**: Hardcoded values without explanation

This degradation leads to increased bug rates, slower feature development, and higher maintenance costs.

## Solution

### Core Principles of Clean Code

**1. Meaningful Names**
```javascript
// ❌ Bad
let d = new Date();
let result = calc(d, 42);

// ✅ Good
let transactionDate = new Date();
let totalWithTax = calculateTotalWithTax(transactionDate, taxRate);
```

**2. Small Functions**
- Functions should do one thing and do it well
- Ideally less than 20 lines, often less than 10
- Name should describe what the function does

**3. No Comments (When Possible)**
```javascript
// ❌ Bad - comment explains what code does
let daysInWeek = 7; // Number of days in a week

// ✅ Good - code is self-explanatory
const DAYS_IN_WEEK = 7;
```

**4. Extract Parameters into Objects**
```javascript
// ❌ Bad - many parameters
function createRectangle(x, y, width, height, color);

// ✅ Good - parameter object
function createRectangle({ x, y, width, height, color });
```

**5. Avoid Duplication (DRY)**
- Extract repeated logic into reusable functions
- Use templates and frameworks appropriately

### Code Smells and Refactoring

| Code Smell | Indicator | Refactoring Technique |
|------------|-----------|----------------------|
| Long Method | >20-30 lines | Extract Method |
| Large Class | >500 lines | Extract Class |
| Dead Code | Unused variables/functions | Remove Dead Code |
| Duplicated Code | Similar code blocks | Extract Method/Class |
| Long Parameter List | >3-4 parameters | Preserve Whole Object |
| Feature Envy | Uses another class more than its own | Move Method |

## When to Use

**Apply Clean Code principles when:**
- Writing new features or modules
- Refactoring legacy code
- Code review process
- Onboarding new team members
- Technical debt reduction sprints

**Prioritize clean code especially for:**
- Core business logic
- Frequently modified modules
- Public APIs and libraries
- Code shared across teams

## Tradeoffs

| Tradeoff | Consideration |
|----------|---------------|
| **Time Investment** | Clean code takes more time upfront but saves time long-term |
| **Over-Engineering Risk** | Don't clean code that won't be maintained or extended |
| **Team Consistency** | Requires team buy-in and consistent enforcement |
| **Legacy Code** | May need to work with "dirty" code temporarily for business needs |
| **Testing Overhead** | Smaller functions require more tests but are easier to test |

## Implementation Example

### Before: Unclean Code
```javascript
function proc(c, a, b) {
    let res = 0;
    for (let i = 0; i < c.length; i++) {
        if (c[i].type === 'A') {
            res += a * c[i].value;
        } else if (c[i].type === 'B') {
            res += b * c[i].value;
        }
    }
    return res;
}
```

### After: Clean Code
```javascript
const TAX_RATE_A = 0.10;
const TAX_RATE_B = 0.15;

function calculateTotalWithTax(transactions, rateA, rateB) {
    const taxedAmounts = transactions.map(applyTaxRate);
    return taxedAmounts.reduce(sumAmounts, 0);
}

function applyTaxRate(transaction) {
    const rate = transaction.type === 'A' ? TAX_RATE_A : TAX_RATE_B;
    return {
        ...transaction,
        taxedAmount: transaction.value * (1 + rate)
    };
}

function sumAmounts(total, transaction) {
    return total + transaction.taxedAmount;
}
```

## Anti-Pattern

### The "It Works, Ship It" Anti-Pattern

```javascript
// ❌ Anti-pattern: Code that works but is unmaintainable
function h(d, f, g) {
    var r = [];
    for (var i = 0; i < d.length; i++) {
        if (d[i] > 0 && d[i] % 2 === 0) {
            r.push(d[i] * f + g);
        }
    }
    return r.filter(function(x) { return x > 100; });
}

// Called as: h(someArray, 42, 1337)
```

**Problems:**
- No meaningful names
- Magic numbers (42, 1337)
- Unclear purpose without reading implementation
- Cannot be understood by new team members
- Difficult to modify safely

### The "Commented Code Graveyard" Anti-Pattern

```javascript
// ❌ Anti-pattern: Leaving old code commented out
function calculateDiscount(price) {
    // return price * 0.9;
    // return price * 0.8;
    // return price - 10;
    return price * 0.85;
}
```

**Solution:** Use version control, not comments, to preserve history.

## Related Patterns

- **[SOLID Principles](./02-SOLID-Principles.md)** - Foundational principles that clean code follows
- **[Design Principles](./03-Design-Principles.md)** - DRY, KISS, YAGNI principles
- **[Naming Conventions](./04-Naming-Conventions.md)** - Specific naming guidelines
- **[Refactoring](../02-Design-Patterns/05-Architectural-Patterns.md)** - Techniques to improve code structure
- **[Code Review](../11-DevOps/01-CI-CD.md)** - Process for maintaining code quality