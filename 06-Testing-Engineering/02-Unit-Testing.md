# Unit Testing

## Title & Summary

**Unit Testing** is the practice of testing individual units of source code in isolation to verify they function correctly. A unit is the smallest testable part of an application—typically a function, method, or class. Unit tests are fast, automated, and form the foundation of the testing pyramid.

## Problem Statement

Developers face several challenges when building software:

- **Regression Bugs**: Changes break existing functionality
- **Hidden Defects**: Bugs discovered late are expensive to fix
- **Refactoring Fear**: Developers avoid improving code for fear of breaking it
- **Integration Complexity**: Debugging is harder when multiple components interact
- **Documentation Gaps**: Code behavior isn't clearly specified

## Solution

### Core Principles

**1. Isolation**
- Test one unit at a time
- Mock external dependencies (databases, APIs, file systems)
- Use test doubles: mocks, stubs, fakes, spies

**2. Automation**
- Tests run automatically as part of CI/CD
- No manual intervention required
- Fast execution (milliseconds per test)

**3. Determinism**
- Tests produce consistent results
- No reliance on external state or timing
- Same input always produces same output

### Implementation Example

```python
import unittest
from unittest.mock import Mock, patch, MagicMock
from myapp.service import UserService, OrderService
from myapp.models import User, Order

class TestUserService(unittest.TestCase):
    
    def setUp(self):
        """Set up test fixtures before each test"""
        self.user_service = UserService()
        self.mock_db = Mock()
        
    def test_create_user_success(self):
        """Test successful user creation"""
        # Arrange
        mock_user = User(id=1, name="John Doe", email="john@example.com")
        self.mock_db.save.return_value = mock_user
        
        # Act
        result = self.user_service.create_user(
            name="John Doe",
            email="john@example.com",
            db=self.mock_db
        )
        
        # Assert
        self.assertIsNotNone(result)
        self.assertEqual(result.name, "John Doe")
        self.assertEqual(result.email, "john@example.com")
        self.mock_db.save.assert_called_once()
    
    def test_create_user_duplicate_email(self):
        """Test user creation with duplicate email"""
        # Arrange
        self.mock_db.exists.return_value = True
        
        # Act & Assert
        with self.assertRaises(ValueError) as context:
            self.user_service.create_user(
                name="Jane Doe",
                email="existing@example.com",
                db=self.mock_db
            )
        
        self.assertIn("email", str(context.exception).lower())
    
    def test_get_user_not_found(self):
        """Test retrieving non-existent user"""
        # Arrange
        self.mock_db.get.return_value = None
        
        # Act
        result = self.user_service.get_user(user_id=999, db=self.mock_db)
        
        # Assert
        self.assertIsNone(result)
    
    def test_update_user_validates_email(self):
        """Test that email validation occurs on update"""
        # Arrange
        user = User(id=1, name="John", email="john@example.com")
        self.mock_db.get.return_value = user
        
        # Act & Assert
        with self.assertRaises(ValueError):
            self.user_service.update_user(
                user_id=1,
                email="invalid-email",  # Invalid format
                db=self.mock_db
            )

class TestOrderService(unittest.TestCase):
    
    @patch('myapp.service.PaymentGateway')
    def test_process_order_charges_payment(self, mock_gateway_class):
        """Test that order processing charges payment"""
        # Arrange
        mock_gateway = Mock()
        mock_gateway_class.return_value = mock_gateway
        mock_gateway.charge.return_value = {"status": "success", "transaction_id": "tx_123"}
        
        order_service = OrderService(payment_gateway=mock_gateway)
        order = Order(id=1, total=99.99, items=[{"name": "Widget", "qty": 2}])
        
        # Act
        result = order_service.process_order(order)
        
        # Assert
        mock_gateway.charge.assert_called_once_with(
            amount=99.99,
            currency="USD"
        )
        self.assertEqual(result.transaction_id, "tx_123")

if __name__ == '__main__':
    unittest.main()
```

### Test-Driven Development (TDD) Cycle

```python
# RED-PINK-GREEN-REFACTOR cycle

# 1. RED: Write a failing test
def test_calculate_discount():
    assert calculate_discount(price=100, discount_rate=0.1) == 90

# 2. PINK: Write minimal code to make it compile/run
def calculate_discount(price, discount_rate):
    pass  # Returns None, test fails

# 3. GREEN: Write minimal code to pass the test
def calculate_discount(price, discount_rate):
    return price * (1 - discount_rate)

# 4. REFACTOR: Improve code while keeping tests green
def calculate_discount(price: float, discount_rate: float) -> float:
    """Apply discount to price, ensuring result is non-negative."""
    if not 0 <= discount_rate <= 1:
        raise ValueError("Discount rate must be between 0 and 1")
    return max(0, price * (1 - discount_rate))
```

### Test Doubles

```python
from unittest.mock import Mock, MagicMock, PropertyMock

# Mock - Basic test double
mock_db = Mock()
mock_db.query.return_value = [{"id": 1, "name": "Test"}]

# MagicMock - Mock with automatic attributes
magic_mock = MagicMock()
magic_mock.method().attribute  # All automatically created

# Spy - Track calls to real function
from unittest.mock import patch

with patch('myapp.service.send_email') as mock_send:
    send_email("user@example.com", "Hello")
    mock_send.assert_called_once_with("user@example.com", "Hello")

# Fake - Working implementation with test-friendly constraints
class InMemoryDatabase:
    """Fake database that stores data in memory"""
    def __init__(self):
        self.data = {}
    
    def save(self, key, value):
        self.data[key] = value
    
    def get(self, key):
        return self.data.get(key)

# Stub - Returns predefined values
class StubPaymentGateway:
    def charge(self, amount):
        return {"status": "success", "transaction_id": "test_123"}
```

## Tradeoffs & Considerations

| Aspect | Consideration |
|--------|---------------|
| **Coverage vs. Value** | 100% coverage doesn't mean good tests |
| **Test Speed** | More tests = slower feedback loop |
| **Maintenance** | Tests need refactoring too |
| **False Security** | Passing tests don't guarantee correctness |
| **Over-Mocking** | Can hide integration issues |

### Code Coverage Example

```python
# coverage.py report
Name                           Stmts   Branch  Part   Cover
------------------------------------------------------------
myapp/service.py                 120      45      5     96%
myapp/models.py                  85      20      0     100%
myapp/utils.py                   45      12      2     95%
------------------------------------------------------------
TOTAL                            250      77      7     97%
```

## Anti-Patterns

### ❌ BAD: Testing Implementation Details

```python
# Don't test private methods or internal state
def test_private_method():
    service = UserService()
    result = service._internal_helper()  # Testing implementation!
    assert result == expected

# Don't assert on private attributes
def test_internal_state():
    service = UserService()
    service.process()
    assert service._cache.size == 10  # Testing implementation!
```

### ❌ BAD: Testing External Dependencies

```python
# Don't test third-party libraries
def test_requests_library():
    response = requests.get("https://api.example.com")
    assert response.status_code == 200  # Tests external service!

# Don't test database directly in unit tests
def test_database_query():
    conn = get_database_connection()  # Real database!
    result = conn.query("SELECT * FROM users")
    assert len(result) > 0
```

### ❌ BAD: Flaky Tests

```python
# Don't rely on timing
def test_async_operation():
    start = time.time()
    async_operation()
    elapsed = time.time() - start
    assert elapsed < 1.0  # May fail on slow systems!

# Don't use random values without seeding
def test_random_behavior():
    result = random_function()
    assert result == expected  # Non-deterministic!
```

### ❌ BAD: Too Much Setup

```python
# Don't create complex test data
def test_simple_operation():
    # 50 lines of setup for a simple test
    user = create_user_with_all_relations()
    order = create_order_with_items(user)
    shipment = create_shipment(order)
    # ... more setup
    result = simple_function(order.id)
    assert result == expected
```

## When to Use

| Scenario | Unit Testing Approach |
|----------|----------------------|
| **New Feature** | Write tests first (TDD) |
| **Bug Fix** | Write test that reproduces bug, then fix |
| **Refactoring** | Ensure tests pass before and after |
| **Complex Logic** | Test all branches and edge cases |
| **Public API** | Test contract and behavior |

## Related Patterns

- **[Testing Pyramid](./01-Testing-Pyramid.md)** - Unit tests form the base
- **[Integration Testing](./03-Integration-Testing.md)** - Test component interactions
- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Test error paths
- **[Clean Code](../04-Best-Practices/01-Clean-Code.md)** - Readable tests

## Verification Checklist

- [ ] Tests are isolated (no external dependencies)
- [ ] Tests are fast (run in milliseconds)
- [ ] Tests are deterministic (same result every time)
- [ ] Tests follow Arrange-Act-Assert pattern
- [ ] Test names describe expected behavior
- [ ] One assertion per test (generally)
- [ ] Mocks used appropriately
- [ ] Edge cases covered
- [ ] Error paths tested
- [ ] Tests run in CI/CD pipeline