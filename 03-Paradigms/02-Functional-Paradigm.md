# Functional Programming Paradigm

## Title & Summary

Functional Programming (FP) is a paradigm that treats computation as evaluation of mathematical functions, avoiding changing state and mutable data. It emphasizes pure functions, immutability, higher-order functions, and function composition.

## Problem Statement

FP addresses challenges when:
- Mutable state causes bugs that are hard to track down
- Side effects make code unpredictable and hard to test
- You need to parallelize computation safely
- Code becomes difficult to reason about due to hidden state
- You want to compose small functions into complex behavior

## Solution

FP provides several core concepts:

**Pure Functions**: Functions that return the same output for the same input and have no side effects.

**Immutability**: Data cannot be changed after creation, preventing unexpected mutations.

**First-Class Functions**: Functions can be passed as arguments, returned from functions, and assigned to variables.

**Higher-Order Functions**: Functions that take other functions as arguments or return functions.

**Function Composition**: Combining simple functions to create complex behavior.

**Recursion**: Using functions calling themselves instead of loops for iteration.

## When to Use

- When you need predictable, testable code
- When parallelizing computation across cores/machines
- When working with data transformations (ETL, data pipelines)
- When building reactive systems
- When you want to avoid shared mutable state bugs

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Predictability | Pure functions are deterministic | May require refactoring existing code |
| Parallelization | No shared state enables safe parallelism | Recursion can cause stack overflow |
| Testing | Easy to test in isolation | Learning curve for developers |
| Performance | Immutable data enables optimizations | Memory overhead for new objects |

## Implementation Example

```python
# Pure Function
def add(x, y):
    return x + y  # Always same output, no side effects

# Higher-Order Function
def apply_twice(func, value):
    return func(func(value))

result = apply_twice(lambda x: x * 2, 5)  # 20

# Function Composition
def compose(f, g):
    return lambda x: f(g(x))

add_then_double = compose(lambda x: x * 2, lambda x: x + 1)
result = add_then_double(5)  # 12

# Map, Filter, Reduce
numbers = [1, 2, 3, 4, 5]

squared = list(map(lambda x: x ** 2, numbers))  # [1, 4, 9, 16, 25]
evens = list(filter(lambda x: x % 2 == 0, numbers))  # [2, 4]
total = reduce(lambda acc, x: acc + x, numbers, 0)  # 15

# Immutability with dataclasses
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: float
    y: float

p1 = Point(1, 2)
p2 = Point(p1.x + 1, p1.y + 1)  # Creates new point, doesn't modify p1
```

## Anti-Pattern

**Impure Functions in FP**: Mixing side effects with pure functions defeats FP benefits.

**Overuse of Recursion**: Deep recursion without tail-call optimization causes stack overflow.

**Premature Abstraction**: Creating overly generic higher-order functions that are hard to understand.

## Related Patterns

- [OOP Paradigm](./01-OOP-Paradigm.md) - Alternative paradigm
- [Hybrid Paradigm](./05-Hybrid-Paradigm.md) - Combining approaches
- [Concurrency Patterns](../02-Design-Patterns/04-Concurrency-Patterns.md) - FP enables safe concurrency