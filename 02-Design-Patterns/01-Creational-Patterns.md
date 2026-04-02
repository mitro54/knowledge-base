# Creational Design Patterns

## Title & Summary

Creational design patterns deal with object creation mechanisms, increasing flexibility and reusability of existing code. These patterns provide ways to create objects while hiding the creation logic, making the system independent of how its objects are created, composed, and represented.

## Problem Statement

Object creation can introduce complexity into a system when:
- The system needs to be independent of how its objects are created, composed, and represented
- You need to specify a family of objects to instantiate without specifying their concrete classes
- You want to promote reuse of existing objects for performance or memory efficiency
- Object construction involves complex logic or multiple steps
- You need to control the number of instances of a class

## Solution

Creational patterns provide several approaches:

**Singleton Pattern**: Ensures a class has only one instance and provides global access to it.

**Factory Method Pattern**: Defines an interface for creating objects but lets subclasses decide which class to instantiate.

**Abstract Factory Pattern**: Creates families of related objects without specifying their concrete classes.

**Builder Pattern**: Separates construction of a complex object from its representation.

**Prototype Pattern**: Creates new objects by copying an existing prototype instance.

## When to Use

- When your system should be independent of how its products are created, composed, and represented
- When you want to promote code reuse by sharing common objects
- When object creation involves complex logic or validation
- When you need to control instance creation (e.g., single instance, limited pool)
- When you want to decouple client code from concrete product classes

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Flexibility | Easy to swap implementations | More classes to maintain |
| Memory | Prototype reduces allocation overhead | Storing prototypes consumes memory |
| Complexity | Encapsulates creation logic | Adds abstraction layers |
| Performance | Object pooling improves reuse | Synchronization overhead for pools |

## Implementation Example

```python
# Singleton Pattern
class DatabaseConnection:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if not self._initialized:
            self.connection_string = "default_connection"
            self._initialized = True

# Builder Pattern
class PizzaBuilder:
    def __init__(self):
        self.pizza = Pizza()
    
    def add_dough(self, dough_type):
        self.pizza.dough = dough_type
        return self
    
    def add_sauce(self, sauce_type):
        self.pizza.sauce = sauce_type
        return self
    
    def add_cheese(self, cheese_type):
        self.pizza.cheese = cheese_type
        return self
    
    def build(self):
        return self.pizza

class Pizza:
    def __init__(self):
        self.dough = None
        self.sauce = None
        self.cheese = None
```

## Anti-Pattern

**God Object Creation**: Creating a single utility class that instantiates all objects in the system violates separation of concerns and creates tight coupling.

**Premature Optimization**: Implementing object pooling or caching without measuring actual performance needs can add unnecessary complexity.

## Related Patterns

- [Structural Patterns](./02-Structural-Patterns.md) - For composing objects
- [Behavioral Patterns](./03-Behavioral-Patterns.md) - For object interaction
- [Dependency Injection](../04-Best-Practices/02-SOLID.md) - Related to object creation control