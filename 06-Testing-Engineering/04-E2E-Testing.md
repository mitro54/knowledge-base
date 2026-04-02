# End-to-End (E2E) Testing

## Title & Summary

**End-to-End Testing** validates complete user journeys through an application, simulating real user interactions from start to finish. E2E tests verify that all components work together correctly, including the UI, backend services, databases, and external integrations.

## Problem Statement

Software systems face end-to-end challenges:

- **User Journey Breaks**: Individual components work but user flows fail
- **Integration Gaps**: Services don't communicate correctly in production
- **UI/Backend Mismatches**: Frontend expects different data than backend provides
- **Session Management**: Authentication/authorization fails in real scenarios
- **Performance Degradation**: System works but too slow for users
- **Browser Compatibility**: Features work in some browsers but not others

## Solution

### Core Principles

**1. Simulate Real Users**
- Test actual user workflows
- Use real browsers (or browser emulation)
- Test with realistic data

**2. Test Critical Paths**
- Focus on happy paths and critical error paths
- Cover revenue-generating flows
- Test authentication and authorization

**3. Automate Where Possible**
- Use E2E testing frameworks
- Run in CI/CD pipeline
- Use headless browsers for speed

### Implementation Example

```python
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.keys import Keys

class TestUserJourney:
    
    @pytest.fixture
    def driver(self):
        """Set up browser for testing"""
        options = webdriver.ChromeOptions()
        options.add_argument("--headless")
        options.add_argument("--no-sandbox")
        driver = webdriver.Chrome(options=options)
        driver.implicitly_wait(10)
        yield driver
        driver.quit()
    
    def test_complete_purchase_journey(self, driver):
        """Test complete purchase flow from login to order confirmation"""
        base_url = "https://test.example.com"
        
        # Step 1: Login
        driver.get(f"{base_url}/login")
        WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.ID, "username"))
        )
        
        driver.find_element(By.ID, "username").send_keys("testuser")
        driver.find_element(By.ID, "password").send_keys("testpass123")
        driver.find_element(By.ID, "login-btn").click()
        
        # Verify login successful
        assert "Welcome, testuser" in driver.page_source
        
        # Step 2: Search for product
        driver.get(f"{base_url}/products")
        search_box = driver.find_element(By.NAME, "search")
        search_box.send_keys("Widget")
        search_box.send_keys(Keys.RETURN)
        
        # Verify search results
        results = driver.find_elements(By.CLASS_NAME, "product-card")
        assert len(results) > 0
        
        # Step 3: Add to cart
        driver.find_elements(By.CLASS_NAME, "add-to-cart")[0].click()
        assert "Added to cart" in driver.page_source
        
        # Step 4: View cart and checkout
        driver.get(f"{base_url}/cart")
        driver.find_element(By.ID, "checkout-btn").click()
        
        # Step 5: Enter shipping info
        driver.find_element(By.ID, "address").send_keys("123 Test St")
        driver.find_element(By.ID, "city").send_keys("Test City")
        driver.find_element(By.ID, "zip").send_keys("12345")
        
        # Step 6: Enter payment info (test card)
        driver.find_element(By.ID, "card-number").send_keys("4242424242424242")
        driver.find_element(By.ID, "expiry").send_keys("12/25")
        driver.find_element(By.ID, "cvv").send_keys("123")
        
        # Step 7: Place order
        driver.find_element(By.ID, "place-order-btn").click()
        
        # Step 8: Verify order confirmation
        WebDriverWait(driver, 10).until(
            EC.url_contains("/order-confirmation")
        )
        assert "Order Confirmed" in driver.page_source
        assert "Thank you for your purchase" in driver.page_source

class TestAuthenticationFlows:
    
    def test_password_reset_flow(self, driver):
        """Test complete password reset journey"""
        driver.get("https://test.example.com/forgot-password")
        
        # Enter email
        driver.find_element(By.ID, "email").send_keys("test@example.com")
        driver.find_element(By.ID, "reset-btn").click()
        
        # Verify confirmation message
        assert "Password reset link sent" in driver.page_source
        
        # Simulate clicking email link (in real tests, use email service mock)
        driver.get("https://test.example.com/reset-password?token=abc123")
        
        # Enter new password
        driver.find_element(By.ID, "new-password").send_keys("NewPass123!")
        driver.find_element(By.ID, "confirm-password").send_keys("NewPass123!")
        driver.find_element(By.ID, "update-password-btn").click()
        
        # Verify password updated
        assert "Password updated successfully" in driver.page_source
    
    def test_social_login_flow(self, driver):
        """Test social authentication"""
        driver.get("https://test.example.com/login")
        
        # Click Google login
        driver.find_element(By.CLASS_NAME, "google-login").click()
        
        # Handle OAuth popup (requires special setup)
        # This typically requires browser extension or special configuration
```

### Playwright E2E Tests

```python
import pytest
from playwright.sync_api import sync_playwright, expect

def test_product_checkout(playwright):
    """Test checkout using Playwright"""
    browser = playwright.chromium.launch(headless=True)
    page = browser.new_page()
    
    # Login
    page.goto("https://test.example.com/login")
    page.fill("#username", "testuser")
    page.fill("#password", "testpass123")
    page.click("#login-btn")
    
    # Navigate to product
    page.goto("https://test.example.com/products/123")
    expect(page).to_have_title("Widget Pro - Example Store")
    
    # Add to cart
    page.click(".add-to-cart-btn")
    expect(page.locator(".cart-count")).to_have_text("1")
    
    # Checkout
    page.click(".checkout-btn")
    page.fill("#email", "test@example.com")
    page.click(".place-order-btn")
    
    # Verify confirmation
    expect(page).to_have_url("*order-confirmation*")
    expect(page.locator("h1")).to_contain_text("Order Confirmed")
    
    browser.close()

def test_visual_regression(playwright):
    """Test visual appearance hasn't changed"""
    browser = playwright.chromium.launch(headless=True)
    page = browser.new_page()
    
    page.goto("https://test.example.com")
    
    # Take screenshot and compare to baseline
    page.screenshot(path="screenshots/homepage.png")
    
    # Use Percy or similar for visual comparison
    # expect(page).to_have_screenshot("baseline/homepage.png")
    
    browser.close()
```

### Cypress E2E Tests

```javascript
// cypress/e2e/checkout.spec.js
describe('Checkout Flow', () => {
  beforeEach(() => {
    cy.visit('/login');
    cy.get('#username').type('testuser');
    cy.get('#password').type('testpass123');
    cy.get('#login-btn').click();
  });

  it('should complete purchase', () => {
    // Add product to cart
    cy.visit('/products/123');
    cy.get('.add-to-cart').click();
    cy.contains('Added to cart').should('be.visible');
    
    // Go to cart
    cy.get('.cart-icon').click();
    cy.get('.checkout-btn').click();
    
    // Fill shipping info
    cy.get('#address').type('123 Test St');
    cy.get('#city').type('Test City');
    cy.get('#zip').type('12345');
    
    // Fill payment info
    cy.get('#card-number').type('4242424242424242');
    cy.get('#expiry').type('12/25');
    cy.get('#cvv').type('123');
    
    // Place order
    cy.get('.place-order-btn').click();
    
    // Verify confirmation
    cy.url().should('include', 'order-confirmation');
    cy.contains('Order Confirmed').should('be.visible');
    cy.contains('Thank you for your purchase').should('be.visible');
  });

  it('should show error for invalid card', () => {
    cy.visit('/cart');
    cy.get('.checkout-btn').click();
    
    // Fill with invalid card
    cy.get('#card-number').type('0000000000000000');
    cy.get('#expiry').type('12/25');
    cy.get('#cvv').type('123');
    
    cy.get('.place-order-btn').click();
    cy.contains('Invalid card number').should('be.visible');
  });
});
```

## Tradeoffs & Considerations

| Aspect | Consideration |
|--------|---------------|
| **Speed** | Slowest tests (minutes per test) |
| **Maintenance** | High - UI changes break tests |
| **Debugging** | Complex - many moving parts |
| **Cost** | Requires browser infrastructure |
| **Reliability** | Can be flaky due to timing/network |

### Test Execution Strategy

```
Development:  Run unit tests only (fast feedback)
Pre-commit:   Run unit + integration tests
Nightly:      Run full suite including E2E
Before deploy: Run critical path E2E tests
```

## Anti-Patterns

### ❌ BAD: Testing Too Much in E2E

```python
# Don't test business logic in E2E
def test_discount_calculation():
    # This should be a unit test!
    add_item_to_cart(price=100)
    apply_discount_code("SAVE10")
    assert cart_total == 90  # Test logic, not UI
```

### ❌ BAD: Hardcoded Delays

```python
# Don't use hardcoded waits
import time
time.sleep(5)  # Brittle and slow!
driver.find_element(By.ID, "result").click()
```

### ❌ BAD: Testing Third-Party Services

```python
# Don't rely on external services
def test_stripe_checkout():
    # Uses real Stripe - may fail, cost money
    proceed_to_stripe_checkout()
    complete_payment()
```

### ❌ BAD: No Error Handling

```python
# Don't ignore potential failures
driver.find_element(By.ID, "element").click()  # May not exist!
# Should use explicit waits and error handling
```

## When to Use

| Scenario | E2E Testing Approach |
|----------|----------------------|
| **Critical User Journeys** | Full E2E tests |
| **Smoke Tests** | Quick E2E verification |
| **Regression Testing** | Run on every release |
| **Browser Testing** | Test on multiple browsers |
| **Performance Testing** | Measure real user experience |

## Related Patterns

- **[Testing Pyramid](./01-Testing-Pyramid.md)** - E2E tests at the top
- **[Integration Testing](./03-Integration-Testing.md)** - Lower level testing
- **[CI-CD](../11-DevOps/01-CI-CD.md)** - Running E2E in pipelines
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Synthetic monitoring

## Verification Checklist

- [ ] Critical user journeys covered
- [ ] Tests run in headless mode for CI
- [ ] Explicit waits used instead of sleeps
- [ ] Test data is isolated and cleaned up
- [ ] Tests are idempotent
- [ ] Screenshots captured on failure
- [ ] Tests run on multiple browsers (if needed)
- [ ] Network requests are intercepted/mocked where appropriate
- [ ] Tests have meaningful names
- [ ] Test failures produce clear error messages