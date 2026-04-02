# Declarative Programming Paradigm

## Title & Summary

Declarative Programming is a paradigm that expresses the logic of computation without describing its control flow. It focuses on what the program should accomplish rather than how to accomplish it, contrasting with imperative programming's step-by-step instructions.

## Problem Statement

Declarative programming addresses challenges when:
- Imperative code becomes verbose and hard to read
- You want to separate business logic from control flow
- You need to optimize without changing code
- Configuration-driven behavior is needed
- You want to express intent clearly without implementation details

## Solution

Declarative programming provides several approaches:

**Query Languages (SQL)**: Specify what data to retrieve, not how to find it.

**Functional Programming**: Express computations as function evaluations.

**Logic Programming**: Define facts and rules, let the engine find solutions.

**Configuration as Code**: Declare desired state, let systems achieve it.

**Declarative UI**: Describe UI state, framework handles rendering.

**Constraint Programming**: Specify constraints, solver finds valid solutions.

## When to Use

- When writing database queries (SQL)
- When building UIs with frameworks like React
- When defining infrastructure (Terraform, Kubernetes)
- When expressing business rules clearly
- When you want separation of concerns between what and how

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Readability | Expresses intent clearly | May hide performance implications |
| Maintainability | Less code to maintain | Harder to debug internal workings |
| Optimization | Engine can optimize execution | Less control over execution |
| Learning | Intuitive for domain experts | Abstract thinking required |

## Implementation Example

```python
# Imperative vs Declarative: Sum of Evens

# Imperative (how)
def sum_evens_imperative(numbers):
    total = 0
    for num in numbers:
        if num % 2 == 0:
            total += num
    return total

# Declarative (what)
def sum_evens_declarative(numbers):
    return sum(filter(lambda x: x % 2 == 0, numbers))

# SQL - Declarative Query
query = """
SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.total > 100
ORDER BY orders.total DESC
"""

# React - Declarative UI
def UserProfile(user):
    return f"""
    <div>
        <h1>{user.name}</h1>
        <p>{user.email}</p>
        <button onClick={() => updateUser(user)}>Edit</button>
    </div>
    """

# Terraform - Declarative Infrastructure
terraform_config = """
resource "aws_instance" "web" {
    ami           = "ami-12345678"
    instance_type = "t2.micro"
    
    tags = {
        Name = "WebServer"
    }
}
"""

# LINQ - Declarative Query in Code
from itertools import chain

# Declarative query
result = [x for x in range(100) if x % 2 == 0 and x > 10]
```

## Anti-Pattern

**Premature Optimization**: Adding imperative optimizations defeats declarative benefits.

**Over-Abstraction**: Creating overly complex declarative expressions that are hard to understand.

**Misplaced Control Flow**: Trying to express control flow in declarative code.

## Related Patterns

- [Functional Paradigm](./02-Functional-Paradigm.md) - Related declarative approach
- [Infrastructure as Code](../09-Infrastructure/03-IaC.md) - Declarative infrastructure
- [Database Design](../08-Database-Design/01-Relational.md) - SQL as declarative language