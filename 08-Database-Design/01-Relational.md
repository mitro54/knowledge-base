# Relational Database Design

## Title & Summary

**Relational Database Design** is the systematic approach to organizing data into tables (relations) with well-defined relationships, constraints, and normalization principles. It provides a mathematical foundation for data storage, retrieval, and manipulation using SQL (Structured Query Language).

## Problem Statement

Without proper relational design, databases suffer from:

- **Data Redundancy**: Same data stored multiple times, wasting space and causing inconsistency
- **Update Anomalies**: Updating one record requires updating many others
- **Insertion Anomalies**: Cannot insert certain data without other unrelated data
- **Deletion Anomalies**: Deleting data unintentionally removes other needed information
- **Query Inefficiency**: Poor schema design leads to complex, slow queries
- **Data Integrity Issues**: Inconsistent or invalid data due to lack of constraints

**Example of Poor Design:**
```sql
-- ANTI-PATTERN: Single table with redundancy
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    product_category VARCHAR(50),
    quantity INT,
    order_date DATE
);

-- Problems:
-- 1. Customer data repeated for each order
-- 2. Product data repeated for each order
-- 3. Updating customer email requires updating all their orders
-- 4. Cannot store customer without an order
```

## Solution

### Normalization Principles

**First Normal Form (1NF)** - Atomic values only
```sql
-- ANTI-PATTERN: Non-atomic values
CREATE TABLE orders_bad (
    order_id INT,
    products VARCHAR(500),  -- "Widget, Gadget, Gizmo"
    quantities VARCHAR(50)   -- "2, 1, 3"
);

-- PATTERN: Atomic values
CREATE TABLE orders_1nf (
    order_id INT PRIMARY KEY,
    product_id INT,
    quantity INT,
    -- Each cell contains exactly one value
);
```

**Second Normal Form (2NF)** - No partial dependencies
```sql
-- ANTI-PATTERN: Partial dependency
CREATE TABLE order_items_bad (
    order_id INT,
    product_id INT,
    quantity INT,
    product_name VARCHAR(100),  -- Depends only on product_id, not composite key
    PRIMARY KEY (order_id, product_id)
);

-- PATTERN: Remove partial dependencies
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100)  -- Now depends only on product_id
);

CREATE TABLE order_items_2nf (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Third Normal Form (3NF)** - No transitive dependencies
```sql
-- ANTI-PATTERN: Transitive dependency
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- Depends on department_id, not employee_id
    department_location VARCHAR(100) -- Also depends on department_id
);

-- PATTERN: Remove transitive dependencies
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);

CREATE TABLE employees_3nf (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

### Complete Normalized Schema

```sql
-- Properly normalized e-commerce schema

-- Customers table
CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    category_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- Categories table
CREATE TABLE categories (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    parent_category_id INT,
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- Orders table
CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(10, 2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Order items table (junction table)
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Indexes for query performance
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
CREATE INDEX idx_products_category_id ON products(category_id);
```

### Constraints for Data Integrity

```sql
-- Comprehensive constraints example
CREATE TABLE accounts (
    account_id INT PRIMARY KEY AUTO_INCREMENT,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    account_holder VARCHAR(100) NOT NULL,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    account_type ENUM('checking', 'savings', 'credit') NOT NULL,
    status ENUM('active', 'frozen', 'closed') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Check constraints
    CONSTRAINT chk_balance_non_negative CHECK (balance >= 0),
    CONSTRAINT chk_account_number_format CHECK (account_number REGEXP '^[A-Z0-9]{10,20}$'),
    
    -- Unique constraints
    CONSTRAINT uq_account_holder_email UNIQUE (account_holder)
);

-- Foreign key with referential actions
CREATE TABLE transactions (
    transaction_id INT PRIMARY KEY AUTO_INCREMENT,
    from_account_id INT NOT NULL,
    to_account_id INT NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Foreign keys with cascade actions
    CONSTRAINT fk_from_account 
        FOREIGN KEY (from_account_id) 
        REFERENCES accounts(account_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    
    CONSTRAINT fk_to_account 
        FOREIGN KEY (to_account_id) 
        REFERENCES accounts(account_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    
    -- Check constraint
    CONSTRAINT chk_amount_positive CHECK (amount > 0),
    
    -- Self-referential check (cannot transfer to self)
    CONSTRAINT chk_different_accounts CHECK (from_account_id != to_account_id)
);
```

## When to Use

**Use Relational Design when:**
- Data has clear relationships and structure
- ACID transactions are required
- Complex queries with joins are needed
- Data integrity is critical
- Schema can be defined upfront
- Consistency is more important than horizontal scalability

**Consider NoSQL when:**
- Schema needs to evolve frequently
- Horizontal scalability is primary concern
- Data is highly unstructured
- Read-heavy with simple queries
- Eventual consistency is acceptable

## Tradeoffs

| Aspect | Relational | NoSQL |
|--------|-----------|-------|
| **Consistency** | Strong (ACID) | Eventual (BASE) |
| **Schema** | Rigid, defined upfront | Flexible, dynamic |
| **Queries** | Complex joins supported | Simple, denormalized |
| **Scalability** | Vertical (scale-up) | Horizontal (scale-out) |
| **Transactions** | Multi-row, multi-table | Single document/key |
| **Learning Curve** | Steeper (SQL, normalization) | Shallower |

## Anti-Pattern

### ❌ Relational Design Anti-Patterns

**1. Denormalization Without Reason**
```sql
-- ANTI-PATTERN: Storing redundant data without justification
CREATE TABLE orders_denormalized (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Redundant!
    customer_email VARCHAR(255), -- Redundant!
    product_id INT,
    product_name VARCHAR(255),   -- Redundant!
    product_price DECIMAL(10,2)  -- Redundant!
);

-- PATTERN: Normalize, denormalize only for performance with caching
CREATE TABLE orders_normalized (
    order_id INT PRIMARY KEY,
    customer_id INT,
    product_id INT,
    quantity INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**2. Using VARCHAR for Everything**
```sql
-- ANTI-PATTERN: Inappropriate data types
CREATE TABLE users_bad (
    user_id INT,
    username VARCHAR(255),
    email VARCHAR(255),
    age VARCHAR(10),        -- Should be INT
    balance VARCHAR(50),    -- Should be DECIMAL
    is_active VARCHAR(5),   -- Should be BOOLEAN
    created_at VARCHAR(50)  -- Should be TIMESTAMP
);

-- PATTERN: Appropriate data types
CREATE TABLE users_good (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL,
    age INT CHECK (age >= 0 AND age <= 150),
    balance DECIMAL(15, 2) DEFAULT 0.00,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**3. Missing Constraints**
```sql
-- ANTI-PATTERN: No constraints, garbage data allowed
CREATE TABLE products_no_constraints (
    product_id INT,
    name VARCHAR(255),
    price DECIMAL(10,2),
    stock INT
);

-- Allows: NULL names, negative prices, negative stock

-- PATTERN: Comprehensive constraints
CREATE TABLE products_with_constraints (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    sku VARCHAR(50) UNIQUE NOT NULL
);
```

**4. Exposing Predictable IDs to Users (IDOR vulnerability)**
```sql
-- ANTI-PATTERN: Using internal predictable IDs for public APIs
CREATE TABLE users_vulnerable (
    user_id INT AUTO_INCREMENT PRIMARY KEY,  -- Predictable, easy to guess
    username VARCHAR(100)
);
-- If API is /users/5, attacker will try /users/6

-- PATTERN: Use internal sequential IDs for fast joins, but UUIDs for public exposure
CREATE TABLE users_secure (
    user_id INT AUTO_INCREMENT PRIMARY KEY,          -- Fast, sequential for internal JOINs
    public_id UUID DEFAULT GEN_RANDOM_UUID() UNIQUE, -- Safe, unguessable for APIs
    username VARCHAR(100)
);
-- Internal relations use `user_id`. APIs use `public_id` (/users/a1b2c3d4...).
```

## Examples

### Complex Query Example

```sql
-- Find top 10 customers by total spending in last 30 days
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    COUNT(o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_spent
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND o.status != 'cancelled'
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
ORDER BY total_spent DESC
LIMIT 10;
```

### Transaction Example

```sql
-- Safe money transfer with ACID guarantees
START TRANSACTION;

-- Check source account balance
SELECT balance FROM accounts WHERE account_id = ? FOR UPDATE;

-- Debit source account
UPDATE accounts 
SET balance = balance - ? 
WHERE account_id = ? AND balance >= ?;

-- Credit destination account
UPDATE accounts 
SET balance = balance + ? 
WHERE account_id = ?;

-- Record transaction
INSERT INTO transactions (from_account_id, to_account_id, amount)
VALUES (?, ?, ?);

-- Verify transfer
SELECT 
    (SELECT balance FROM accounts WHERE account_id = ?) +
    (SELECT balance FROM accounts WHERE account_id = ?) AS total_balance;

COMMIT;
```

## Related Patterns

- **[NoSQL Design](./02-NoSQL.md)** - Alternative data models
- **[Indexing](./03-Indexing.md)** - Query performance optimization
- **[Query Optimization](./04-Query-Optimization.md)** - Efficient data retrieval
- **[Database Migration](./05-Migration.md)** - Schema evolution

## References

- **Book**: "Database Design for Mere Mortals" - Michael J. Hernandez
- **Book**: "SQL Performance Explained" - Markus Winand
- **Paper**: "A Relational Model of Data for Large Shared Data Banks" - E.F. Codd, 1970
- **Standard**: SQL:2023 - ISO/IEC 9075