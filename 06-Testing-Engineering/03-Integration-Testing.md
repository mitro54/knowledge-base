# Integration Testing

## Title & Summary

**Integration Testing** verifies that different modules, services, or systems work together correctly. While unit tests verify individual components in isolation, integration tests verify the interactions between components, ensuring data flows correctly and interfaces are properly implemented.

## Problem Statement

Software systems face integration challenges:

- **Interface Mismatches**: Components don't communicate as expected
- **Data Format Issues**: JSON schemas, serialization problems
- **API Contract Violations**: Breaking changes in interfaces
- **Database Integration**: ORM issues, query problems
- **External Service Dependencies**: Third-party API failures
- **Configuration Issues**: Environment-specific problems

## Solution

### Core Principles

**1. Test Real Interactions**
- Use real databases (test instances)
- Test actual API calls (with test servers)
- Verify data persistence and retrieval

**2. Test Boundaries**
- API endpoints
- Database access layers
- Message queue consumers
- External service integrations

**3. Use Test Data**
- Seed databases with known data
- Use test fixtures
- Clean up after tests

### Implementation Example

```python
import pytest
import requests
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import Base, get_db
from app.models import User, Order

# Test database configuration
TEST_DATABASE_URL = "postgresql://test:test@localhost/test_db"
engine = create_engine(TEST_DATABASE_URL)
TestingSessionLocal = sessionmaker(bind=engine)

@pytest.fixture
def test_db():
    """Create test database"""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    yield db
    db.close()
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(test_db):
    """Create test client with test database"""
    def override_get_db():
        try:
            yield test_db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    client = TestClient(app)
    yield client
    app.dependency_overrides.clear()

class TestUserIntegration:
    
    def test_create_and_retrieve_user(self, client):
        """Test user creation and retrieval through API"""
        # Create user
        response = client.post("/api/users", json={
            "name": "John Doe",
            "email": "john@example.com"
        })
        assert response.status_code == 201
        user_data = response.json()
        user_id = user_data["id"]
        
        # Retrieve user
        response = client.get(f"/api/users/{user_id}")
        assert response.status_code == 200
        assert response.json()["email"] == "john@example.com"
    
    def test_user_order_integration(self, client):
        """Test user can create orders"""
        # Create user
        user_response = client.post("/api/users", json={
            "name": "Jane Doe",
            "email": "jane@example.com"
        })
        user_id = user_response.json()["id"]
        
        # Create order for user
        order_response = client.post("/api/orders", json={
            "user_id": user_id,
            "items": [{"product_id": 1, "quantity": 2}]
        })
        assert order_response.status_code == 201
        
        # Verify order is linked to user
        user_response = client.get(f"/api/users/{user_id}")
        assert len(user_response.json()["orders"]) == 1

class TestDatabaseIntegration:
    
    def test_user_query_with_orders(self, test_db):
        """Test database query joins work correctly"""
        from app.schemas import UserWithOrders
        
        # Create test data
        user = User(name="Test User", email="test@example.com")
        test_db.add(user)
        test_db.commit()
        test_db.refresh(user)
        
        order = Order(user_id=user.id, total=99.99)
        test_db.add(order)
        test_db.commit()
        
        # Query with join
        result = test_db.query(User).join(Order).filter(User.id == user.id).first()
        assert result is not None
        assert len(result.orders) == 1

class TestExternalServiceIntegration:
    
    @pytest.mark.integration
    def test_payment_gateway_integration(self, client):
        """Test real payment gateway integration (use test mode)"""
        # Use Stripe test mode
        response = client.post("/api/orders/test-payment", json={
            "amount": 100,
            "currency": "usd",
            "source": "tok_visa"  # Stripe test token
        })
        assert response.status_code == 200
        assert response.json()["status"] == "succeeded"
```

### Contract Testing

```python
import pytest
from pact import Consumer, Provider

# Consumer test - verifies provider meets contract
@pytest.fixture
def pact():
    pact = Consumer("OrderService").has_pact_with(Provider("PaymentService"))
    return pact

def test_payment_contract(pact):
    """Test that PaymentService meets our expectations"""
    with pact.given("a valid payment request"):
        pact.upon_receiving("a payment request").\
            upon_request("POST", "/payments").\
            with_body({"amount": 100, "currency": "USD"}).\
            will_respond_with(200, {"status": "success"})
        
        # Make actual request
        response = requests.post(
            "http://payment-service/payments",
            json={"amount": 100, "currency": "USD"}
        )
        assert response.status_code == 200
```

### Containerized Integration Tests

```python
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def postgres_container():
    """Start PostgreSQL container for tests"""
    with PostgresContainer("postgres:14") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def redis_container():
    """Start Redis container for tests"""
    with RedisContainer("redis:7") as redis:
        yield redis

def test_cache_integration(redis_container):
    """Test Redis cache integration"""
    import redis
    r = redis.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379)
    )
    r.set("test_key", "test_value")
    assert r.get("test_key") == b"test_value"
```

## Tradeoffs & Considerations

| Aspect | Consideration |
|--------|---------------|
| **Speed** | Slower than unit tests (seconds vs milliseconds) |
| **Complexity** | More setup required (databases, services) |
| **Flakiness** | More prone to timing and network issues |
| **Cost** | May require running multiple services |
| **Debugging** | Harder to isolate failures |

### Test Execution Time Comparison

```
Unit Tests:      ~5 seconds (500 tests)
Integration:     ~60 seconds (50 tests)
E2E Tests:       ~300 seconds (10 tests)
```

## Anti-Patterns

### ❌ BAD: Testing External Services in Production

```python
# Don't call production external services
def test_stripe_integration():
    stripe.api_key = "sk_live_..."  # Production key!
    charge = stripe.Charge.create(amount=1000)  # Real charge!
```

### ❌ BAD: Shared Test State

```python
# Don't share state between integration tests
def test_user_creation():
    # Creates user that affects other tests
    create_user("shared@example.com")

def test_user_update():
    # Assumes user exists from previous test
    update_user("shared@example.com")  # May fail!
```

### ❌ BAD: No Cleanup

```python
# Don't leave test data behind
def test_order_creation():
    create_order(items=1000)  # Database fills up!
    # No cleanup - next run fails
```

### ❌ BAD: Testing Implementation Details

```python
# Don't test internal database structure
def test_database_columns():
    result = db.query("DESCRIBE users").fetchall()
    assert len(result) == 5  # Breaks on schema change
```

## When to Use

| Scenario | Integration Testing Approach |
|----------|----------------------|
| **API Development** | Test endpoint interactions |
| **Database Changes** | Test queries and migrations |
| **Service Communication** | Test inter-service calls |
| **External Integrations** | Test with mock/stub services |
| **Cache Layers** | Test cache invalidation |

## Related Patterns

- **[Testing Pyramid](./01-Testing-Pyramid.md)** - Integration tests in middle layer
- **[Unit Testing](./02-Unit-Testing.md)** - Foundation for integration tests
- **[E2E Testing](./04-E2E-Testing.md)** - Higher level testing
- **[Event-Driven Architecture](../01-System-Design/03-Event-Driven-Architecture.md)** - Test event flows

## Verification Checklist

- [ ] Tests use isolated test databases
- [ ] Test data is cleaned up after tests
- [ ] External services are mocked or use test modes
- [ ] Tests are idempotent (can run multiple times)
- [ ] Tests don't depend on execution order
- [ ] Network timeouts are configured appropriately
- [ ] Test containers are properly managed
- [ ] Database migrations are applied in tests
- [ ] Test fixtures are reusable
- [ ] Integration tests run in CI/CD