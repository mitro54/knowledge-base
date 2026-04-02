# Structural Design Patterns

## Title & Summary

Structural design patterns explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient. They focus on class and object composition, providing ways to simplify interfaces and reduce coupling between components.

## Problem Statement

Structural complexity arises when:
- You need to combine multiple objects into a unified structure
- Interfaces between components are incompatible
- You want to add functionality to objects without modifying their structure
- You need to provide a simplified interface to a complex subsystem
- Objects need to be wrapped to add behavior or control access

## Solution

Structural patterns provide several approaches:

**Adapter Pattern**: Allows incompatible interfaces to work together by converting one interface to another.

**Bridge Pattern**: Decouples an abstraction from its implementation so they can vary independently.

**Composite Pattern**: Composes objects into tree structures to represent part-whole hierarchies.

**Decorator Pattern**: Attaches additional responsibilities to objects dynamically.

**Facade Pattern**: Provides a unified interface to a set of interfaces in a subsystem.

**Flyweight Pattern**: Shares objects to support large numbers of fine-grained objects efficiently.

**Proxy Pattern**: Provides a surrogate or placeholder for another object to control access to it.

## When to Use

- When you need to combine objects into larger structures
- When you want to add responsibilities to individual objects dynamically
- When you need to hide the complexity of a subsystem
- When you want to control access to an object
- When you need to make incompatible interfaces work together

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Flexibility | Easy to extend functionality | More objects in memory |
| Complexity | Simplifies client code | Adds indirection layers |
| Performance | Flyweight reduces memory usage | Lookup overhead for shared objects |
| Maintainability | Separates concerns | More classes to understand |

## Implementation Example

```python
# Adapter Pattern
class LegacyPaymentProcessor:
    def process_payment_old(self, amount, currency):
        return f"Processed ${amount} in {currency}"

class ModernPaymentProcessor:
    def __init__(self, legacy_processor):
        self.legacy = legacy_processor
    
    def charge(self, payment_request):
        return self.legacy.process_payment_old(
            payment_request.amount, 
            payment_request.currency
        )

# Decorator Pattern
class Coffee:
    def get_cost(self): return 5
    def get_description(self): return "Coffee"

class MilkDecorator(Coffee):
    def __init__(self, coffee):
        self.coffee = coffee
    def get_cost(self): return self.coffee.get_cost() + 2
    def get_description(self): return self.coffee.get_description() + " with milk"

# Facade Pattern
class ComputerFacade:
    def __init__(self):
        self.cpu = CPU()
        self.memory = Memory()
        self.hard_drive = HardDrive()
    
    def start(self):
        self.cpu.freeze()
        self.memory.load(0, self.hard_drive.read(0, 1024))
        self.cpu.jump(0)
        self.cpu.execute()
```

## Anti-Pattern

**Feature Creep in Decorators**: Adding too many decorators creates deep nesting that's hard to understand and debug.

**Facade Overuse**: Creating a facade for every subsystem can hide important details and make debugging difficult.

## Related Patterns

- [Creational Patterns](./01-Creational-Patterns.md) - For creating the objects to compose
- [Behavioral Patterns](./03-Behavioral-Patterns.md) - For object interaction
- [Dependency Injection](../04-Best-Practices/02-SOLID.md) - Related to interface design