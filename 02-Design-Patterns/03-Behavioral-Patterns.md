# Behavioral Design Patterns

## Title & Summary

Behavioral design patterns focus on communication between objects, providing mechanisms for objects to interact flexibly and efficiently. These patterns address object communication, responsibility delegation, and algorithm encapsulation.

## Problem Statement

Behavioral complexity emerges when:
- Objects need to communicate without tight coupling
- You want to encapsulate algorithms as interchangeable objects
- Objects need to respond to events or state changes
- You need to traverse complex object structures uniformly
- You want to define skeleton algorithms with customizable steps

## Solution

Behavioral patterns provide several approaches:

**Command Pattern**: Encapsulates a request as an object, enabling queuing, logging, and undo operations.

**Observer Pattern**: Defines one-to-many dependency where objects are notified of state changes.

**Strategy Pattern**: Defines a family of algorithms, encapsulates each, and makes them interchangeable.

**Iterator Pattern**: Provides a way to access elements of a collection sequentially without exposing its representation.

**Template Method Pattern**: Defines skeleton of an algorithm, deferring some steps to subclasses.

**Visitor Pattern**: Represents an operation to be performed on elements of an object structure.

**Chain of Responsibility Pattern**: Passes requests along a chain of handlers.

**State Pattern**: Allows an object to alter its behavior when its internal state changes.

## When to Use

- When you need loose coupling between objects
- When you want to encapsulate varying algorithms
- When objects need to react to events or state changes
- When you need to traverse collections uniformly
- When you want to define algorithm skeletons with customizable steps

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Flexibility | Easy to swap algorithms | More classes to maintain |
| Coupling | Loose coupling between objects | Indirect communication harder to trace |
| Extensibility | Open for extension | Can lead to pattern proliferation |
| Performance | Strategy enables optimization | Object creation overhead |

## Implementation Example

```python
# Observer Pattern
class Subject:
    def __init__(self):
        self.observers = []
    
    def attach(self, observer):
        self.observers.append(observer)
    
    def notify(self, data):
        for observer in self.observers:
            observer.update(data)

class Observer:
    def update(self, data):
        pass

# Strategy Pattern
class PaymentStrategy:
    def pay(self, amount): pass

class CreditCardStrategy(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} with Credit Card"

class PayPalStrategy(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} with PayPal"

class PaymentContext:
    def __init__(self, strategy):
        self.strategy = strategy
    def set_strategy(self, strategy):
        self.strategy = strategy
    def execute_payment(self, amount):
        return self.strategy.pay(amount)

# Command Pattern
class Command:
    def execute(self): pass

class LightOnCommand(Command):
    def __init__(self, light):
        self.light = light
    def execute(self):
        self.light.on()
```

## Anti-Pattern

**Observer Overuse**: Creating observers for every state change creates notification storms and performance issues.

**Strategy Proliferation**: Creating a strategy for every minor variation adds unnecessary complexity.

## Related Patterns

- [Creational Patterns](./01-Creational-Patterns.md) - For creating strategy/command objects
- [Structural Patterns](./02-Structural-Patterns.md) - For composing behavioral objects
- [Event-Driven Paradigm](../03-Paradigms/03-Event-Driven.md) - Related to Observer pattern