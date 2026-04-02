# Naming Conventions

## Title & Summary

Naming conventions are standardized rules for naming variables, functions, classes, files, and other identifiers in code. Consistent naming improves code readability, reduces cognitive load, and enables developers to understand code structure and purpose at a glance.

## Problem Statement

Inconsistent naming leads to:
- **Reduced readability**: Time wasted deciphering what identifiers mean
- **Confusion**: Unclear whether `userId`, `user_id`, or `UserId` is correct
- **Search difficulties**: Hard to find all references to a concept
- **Team friction**: Disagreements over naming styles
- **Maintenance overhead**: Unclear relationships between related items
- **Professional appearance**: Inconsistent code looks amateurish

## Solution

### Core Naming Principles

**1. Be Descriptive and Clear**
```javascript
// ❌ Bad - Unclear meaning
let d = new Date();
let r = calc(d, 42);

// ✅ Good - Self-documenting
let transactionDate = new Date();
let totalWithTax = calculateTotalWithTax(transactionDate, taxRate);
```

**2. Use Full Words**
```javascript
// ❌ Bad - Abbreviations
let custId = 123;
let maxNumOfItems = 10;

// ✅ Good - Full words
let customerId = 123;
let maximumNumberOfItems = 10;
```

**3. Avoid Ambiguous Names**
```javascript
// ❌ Bad - What does "flag" mean?
let flag = true;
let data = [];

// ✅ Good - Clear purpose
let isPaymentVerified = true;
let transactionRecords = [];
```

### Language-Specific Conventions

**JavaScript/TypeScript**

| Element | Convention | Example |
|--------|-----------|--------|
| Variables/Functions | camelCase | `userProfile`, `calculateTotal` |
| Classes/Types | PascalCase | `UserProfile`, `PaymentProcessor` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `API_BASE_URL` |
| Files | camelCase or kebab-case | `userProfile.js`, `user-profile.ts` |
| Directories | kebab-case | `user-profiles`, `payment-gateway` |
| Private Fields | Prefix with `_` | `_internalState` |

```typescript
// JavaScript/TypeScript naming conventions
const MAX_RETRIES = 3;                    // Constant
let currentUser: UserProfile | null = null; // Variable
async function fetchUserData(id: string): Promise<UserProfile> { } // Function

class PaymentProcessor {                   // Class
    private _apiKey: string;              // Private field
    constructor(apiKey: string) { }
}

interface PaymentGateway {                // Interface
    process(payment: Payment): Promise<Result>;
}
```

**Python**

| Element | Convention | Example |
|--------|-----------|--------|
| Variables/Functions | snake_case | `user_profile`, `calculate_total` |
| Classes | PascalCase | `UserProfile`, `PaymentProcessor` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Files/Modules | snake_case | `user_profile.py` |
| Packages | snake_case | `payment_gateway` |
| Private | Single underscore prefix | `_internal_state` |
| Private (strong) | Double underscore prefix | `__private_method` |

```python
# Python naming conventions
MAX_RETRIES = 3  # Constant

def fetch_user_data(user_id: str) -> UserProfile:  # Function
    pass

class PaymentProcessor:  # Class
    def __init__(self, api_key: str):
        self._api_key = api_key  # Protected attribute
    
    def __validate_key(self):  # Private method (name mangling)
        pass
```

**Java**

| Element | Convention | Example |
|--------|-----------|--------|
| Packages | lowercase, dot-separated | `com.company.module` |
| Classes/Interfaces | PascalCase | `UserProfile`, `PaymentGateway` |
| Variables/Methods | camelCase | `userProfile`, `calculateTotal` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Files | PascalCase (matching class) | `UserProfile.java` |

```java
// Java naming conventions
package com.company.payment;

public class PaymentProcessor {
    private static final int MAX_RETRIES = 3;
    private String apiKey;
    
    public PaymentProcessor(String apiKey) {
        this.apiKey = apiKey;
    }
    
    public PaymentResult processPayment(Payment payment) { }
}
```

**Database Naming**

| Element | Convention | Example |
|--------|-----------|--------|
| Tables | snake_case, plural | `users`, `payment_transactions` |
| Columns | snake_case, singular | `user_id`, `email_address` |
| Primary Keys | `id` or `table_id` | `id`, `user_id` |
| Foreign Keys | `referenced_table_id` | `user_id`, `order_id` |
| Indexes | `idx_table_column` | `idx_users_email` |
| Constraints | `ck_table_column` | `ck_users_email_unique` |

```sql
-- Database naming conventions
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email_address VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email_address);
CREATE UNIQUE CONSTRAINT ck_users_email_unique ON users(email_address);
```

### Semantic Naming Patterns

**Boolean Variables**
Use prefixes that indicate boolean nature:
- `is` - State or condition: `isAuthenticated`, `isAvailable`
- `has` - Possession: `hasPermission`, `hasUnreadMessages`
- `can` - Capability: `canEdit`, `canDelete`
- `should` - Recommendation: `shouldRetry`, `shouldCache`
- `enable`/`disable` - Toggle: `enableNotifications`, `disableCache`

```javascript
// Boolean naming examples
let isAuthenticated = true;
let hasUnreadMessages = false;
let canEditProfile = user.role === 'admin';
let shouldRetryRequest = error.isNetworkError;
```

**Collection Variables**
Use plural names for collections:
```javascript
let users: User[];           // Multiple users
let user: User;              // Single user
let userMap: Map<string, User>;  // Map of users
let userSet: Set<User>;      // Set of users
```

**Functions**
Use verb-object pattern for actions:
```javascript
// Getters - Return values
function getUserById(id) { }
function getCartTotal() { }

// Setters - Modify state
function setUserPreference(key, value) { }
function setCartItems(items) { }

// Actions - Perform operations
function createUser(data) { }
function deleteSession(token) { }
function validateInput(input) { }

// Checks - Return boolean
function isValidEmail(email) { }
function hasPermission(user, permission) { }
```

**Classes**
Use noun names describing the entity:
```typescript
// Entity classes
class User { }
class Product { }
class Order { }

// Service classes
class UserService { }
class PaymentService { }
class EmailService { }

// Repository classes
class UserRepository { }
class OrderRepository { }

// Controller/Handler classes
class UserController { }
class PaymentHandler { }
```

## When to Use

**Apply naming conventions when:**
- Starting a new project
- Joining an existing codebase
- Creating team style guides
- Setting up linters and formatters
- Code review process

**Prioritize for:**
- Public APIs (most visible)
- Shared libraries
- Team projects
- Long-term maintenance code

## Tradeoffs

| Tradeoff | Consideration |
|----------|---------------|
| **Longer Names** | More typing but better clarity; use autocomplete |
| **Strict Enforcement** | Requires tooling but ensures consistency |
| **Legacy Code** | May need gradual migration |
| **Team Agreement** | Requires consensus; document decisions |

## Implementation Example

### Complete Project Structure with Naming

```
my-ecommerce-app/
├── src/
│   ├── controllers/           # kebab-case directory
│   │   ├── user-controller.ts   # kebab-case file
│   │   └── product-controller.ts
│   ├── services/
│   │   ├── user-service.ts
│   │   └── payment-service.ts
│   ├── repositories/
│   │   ├── user-repository.ts
│   │   └── order-repository.ts
│   ├── models/
│   │   ├── user.model.ts
│   │   └── product.model.ts
│   ├── utils/
│   │   ├── validation-utils.ts
│   │   └── date-utils.ts
│   └── constants/
│       └── app-constants.ts
├── tests/
│   ├── unit/
│   └── integration/
└── package.json
```

```typescript
// user.model.ts - Model with proper naming
export interface UserProfile {
    id: string;
    email: string;
    firstName: string;
    lastName: string;
    isActive: boolean;
    createdAt: Date;
}

export interface UserPreferences {
    theme: 'light' | 'dark';
    notificationsEnabled: boolean;
    language: string;
}

// user-service.ts - Service with proper naming
export class UserService {
    private static readonly MAX_SEARCH_RESULTS = 100;
    private userRepository: UserRepository;
    
    constructor(userRepository: UserRepository) {
        this.userRepository = userRepository;
    }
    
    async getUserById(userId: string): Promise<UserProfile | null> {
        return this.userRepository.findById(userId);
    }
    
    async createUser(userData: CreateUserData): Promise<UserProfile> {
        const isValidEmail = this.validateEmail(userData.email);
        if (!isValidEmail) {
            throw new ValidationError('Invalid email');
        }
        return this.userRepository.create(userData);
    }
    
    private validateEmail(email: string): boolean {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    }
}

// app-constants.ts - Constants file
export const APP_NAME = 'E-Commerce Platform';
export const MAX_UPLOAD_SIZE = 10 * 1024 * 1024; // 10MB
export const DEFAULT_PAGE_SIZE = 20;
export const PASSWORD_MIN_LENGTH = 8;
```

## Anti-Pattern

### The "Single Letter" Anti-Pattern

```javascript
// ❌ Anti-pattern: Single letter variable names
function c(a, b, d) {
    for (let i = 0; i < a.length; i++) {
        if (a[i] > d) {
            b.push(a[i]);
        }
    }
    return b;
}

// Called as: c(someArray, [], 100)
// What does this do?
```

**Refactored:**
```javascript
// ✅ Clear naming
function filterValuesAboveThreshold(values, filteredArray, threshold) {
    for (let i = 0; i < values.length; i++) {
        if (values[i] > threshold) {
            filteredArray.push(values[i]);
        }
    }
    return filteredArray;
}

// Called as: filterValuesAboveThreshold(numbers, [], 100)
// Clear what it does!
```

### The "Mysterious Numbers" Anti-Pattern

```javascript
// ❌ Anti-pattern: Magic numbers without meaning
if (status === 1) { }
setTimeout(callback, 5000);
if (length > 255) { }

// ✅ With named constants
if (status === USER_STATUS_ACTIVE) { }
setTimeout(callback, TIMEOUT_5_SECONDS);
if (length > MAX_USERNAME_LENGTH) { }
```

## Related Patterns

- **[Clean Code](./01-Clean-Code.md)** - Meaningful names are clean code
- **[Design Principles](./03-Design-Principles.md)** - KISS applies to naming
- **[Code Organization](./05-Code-Organization.md)** - File/directory naming
- **[Database Design](../08-Database-Design/01-Relational-Design.md)** - Database naming conventions