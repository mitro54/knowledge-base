# Object-Oriented Programming Paradigm

## Title & Summary

Object-Oriented Programming (OOP) is a programming paradigm based on the concept of objects containing data and methods. It emphasizes encapsulation, inheritance, polymorphism, and abstraction to create modular, reusable, and maintainable code.

## Problem Statement

OOP addresses challenges when:
- Code becomes difficult to maintain as systems grow
- You need to model real-world entities and relationships
- Code duplication leads to maintenance nightmares
- You need to extend functionality without modifying existing code
- State management becomes complex with global variables

## Solution

OOP provides four fundamental principles:

**Encapsulation**: Bundling data and methods that operate on that data within objects, hiding internal state.

**Inheritance**: Creating new classes based on existing ones, promoting code reuse and hierarchical relationships.

**Polymorphism**: Allowing objects of different types to be treated as objects of a common type.

**Abstraction**: Hiding complex implementation details and exposing only essential features.

## When to Use

- When modeling real-world entities with state and behavior
- When you need code reuse through inheritance
- When building large systems requiring modularity
- When you need to hide implementation details
- When working with GUI frameworks or enterprise applications

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Reusability | Inheritance promotes code sharing | Deep hierarchies are fragile |
| Maintainability | Encapsulation limits change impact | More boilerplate code |
| Flexibility | Polymorphism enables extensibility | Runtime overhead |
| Learning Curve | Intuitive for domain modeling | Complex inheritance chains |

## Implementation Example

```python
# Encapsulation
class BankAccount:
    def __init__(self, balance):
        self.__balance = balance  # Private attribute
    
    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount
    
    def get_balance(self):
        return self.__balance

# Inheritance
class Animal:
    def speak(self):
        pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

# Polymorphism
def make_animal_speak(animal):
    print(animal.speak())

make_animal_speak(Dog())  # Woof!
make_animal_speak(Cat())  # Meow!

# Abstraction
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    def area(self):
        return self.width * self.height
```

## Anti-Pattern

**God Object**: A single class that knows too much or does too much, violating Single Responsibility.

**Inheritance Abuse**: Using inheritance for simple code reuse instead of composition.

**Feature Envy**: Methods that use more data from other classes than their own.

## Related Patterns

- [Functional Paradigm](./02-Functional-Paradigm.md) - Alternative paradigm
- [SOLID Principles](../04-Best-Practices/02-SOLID.md) - OOP design principles
- [Creational Patterns](../02-Design-Patterns/01-Creational-Patterns.md) - OOP patterns