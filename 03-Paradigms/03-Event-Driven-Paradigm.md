# Event-Driven Programming Paradigm

## Title & Summary

Event-Driven Programming (EDP) is a paradigm where program flow is determined by events such as user actions, sensor outputs, or messages from other programs. It emphasizes asynchronous communication, event loops, and callback mechanisms for responsive, non-blocking applications.

## Problem Statement

EDP addresses challenges when:
- Blocking operations freeze user interfaces
- You need to handle multiple concurrent operations efficiently
- Systems need to respond to external stimuli in real-time
- You want loose coupling between components
- Traditional request-response models are insufficient

## Solution

EDP provides several core concepts:

**Event Loop**: Central mechanism that waits for and dispatches events or messages.

**Event Emitters**: Objects that emit named events to notify listeners.

**Event Listeners/Handlers**: Callback functions that respond to specific events.

**Event Queue**: Buffer that stores events until they can be processed.

**Asynchronous Programming**: Non-blocking operations that continue execution while waiting.

**Pub/Sub Pattern**: Publishers send events, subscribers receive events of interest.

## When to Use

- When building responsive UI applications
- When handling real-time data streams
- When you need non-blocking I/O operations
- When building microservices with loose coupling
- When implementing game loops or simulations

## Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Responsiveness | Non-blocking keeps UI responsive | Callback hell can occur |
| Scalability | Handles many concurrent connections | Debugging is harder |
| Coupling | Loose coupling between components | Event flow harder to trace |
| Complexity | Elegant for certain problems | Steeper learning curve |

## Implementation Example

```python
import asyncio

# Event Emitter Pattern
class EventEmitter:
    def __init__(self):
        self.listeners = {}
    
    def on(self, event, callback):
        if event not in self.listeners:
            self.listeners[event] = []
        self.listeners[event].append(callback)
    
    def emit(self, event, *args):
        if event in self.listeners:
            for callback in self.listeners[event]:
                callback(*args)

# Usage
emitter = EventEmitter()
emitter.on('click', lambda x: print(f'Clicked at {x}'))
emitter.emit('click', 100)  # Clicked at 100

# Async/Await Pattern
async def fetch_data(url):
    await asyncio.sleep(1)  # Simulate I/O
    return f"Data from {url}"

async def main():
    results = await asyncio.gather(
        fetch_data('url1'),
        fetch_data('url2'),
        fetch_data('url3')
    )
    print(results)

asyncio.run(main())

# Pub/Sub Pattern
class PubSub:
    def __init__(self):
        self.topics = {}
    
    def subscribe(self, topic, callback):
        if topic not in self.topics:
            self.topics[topic] = []
        self.topics[topic].append(callback)
    
    def publish(self, topic, message):
        if topic in self.topics:
            for callback in self.topics[topic]:
                callback(message)

pubsub = PubSub()
pubsub.subscribe('orders', lambda msg: print(f'Order: {msg}'))
pubsub.publish('orders', 'Order #123')  # Order: Order #123
```

## Anti-Pattern

**Callback Hell**: Nested callbacks create unreadable, hard-to-maintain code.

**Event Storm**: Emitting too many events overwhelms the system.

**Zombie Listeners**: Event listeners not properly removed cause memory leaks.

## Related Patterns

- [Concurrency Patterns](../02-Design-Patterns/04-Concurrency-Patterns.md) - Non-blocking concurrency
- [Observer Pattern](../02-Design-Patterns/03-Behavioral-Patterns.md) - Event notification
- [Event-Driven Architecture](../01-System-Design/03-Event-Driven-Architecture.md) - System-level EDP