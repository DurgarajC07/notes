# End-to-End Testing in React

## Core Concept

E2E (End-to-End) tests verify that **the entire application works correctly from the user's perspective**, testing real user workflows in a real browser environment with actual network requests and navigation.

---

## Playwright vs Cypress Comparison

| Feature             | Playwright                           | Cypress                     |
| ------------------- | ------------------------------------ | --------------------------- |
| **Browser Support** | Chromium, Firefox, WebKit            | Chrome, Firefox, Edge       |
| **Language**        | JavaScript, TypeScript, Python, .NET | JavaScript, TypeScript      |
| **Test Execution**  | Parallel by default                  | Sequential (paid: parallel) |
| **Auto-waiting**    | Yes                                  | Yes                         |
| **Network Control** | Full control                         | Full control                |
| **Debugging**       | VS Code debugger, trace viewer       | Time-travel debugging       |
| **Speed**           | Fast                                 | Fast                        |
| **API Testing**     | Built-in                             | Via cy.request()            |
| **Mobile Testing**  | Device emulation                     | Viewport emulation          |

---

## Playwright Setup and Basics

### **Installation**

```bash
npm init playwright@latest
# or
npx create-playwright
```

### **Basic Test Structure**

```typescript
import { test, expect } from "@playwright/test";

test.describe("User Authentication", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("http://localhost:3000");
  });

  test("user can login successfully", async ({ page }) => {
    // Navigate to login page
    await page.click("text=Login");

    // Fill in form
    await page.fill('input[name="email"]', "user@example.com");
    await page.fill('input[name="password"]', "password123");

    // Submit form
    await page.click('button[type="submit"]');

    // Assert successful login
    await expect(page).toHaveURL(/\/dashboard/);
    await expect(page.locator("text=Welcome")).toBeVisible();
  });

  test("shows error for invalid credentials", async ({ page }) => {
    await page.click("text=Login");
    await page.fill('input[name="email"]', "wrong@example.com");
    await page.fill('input[name="password"]', "wrongpass");
    await page.click('button[type="submit"]');

    await expect(page.locator("text=Invalid credentials")).toBeVisible();
  });
});
```

### **Playwright Locators**

```typescript
import { test, expect } from "@playwright/test";

test("locator strategies", async ({ page }) => {
  await page.goto("http://localhost:3000");

  // By text
  await page.click("text=Submit");

  // By role (accessible, preferred)
  await page.click('role=button[name="Submit"]');

  // By label
  await page.fill("label=Email", "user@example.com");

  // By test ID
  await page.click('[data-testid="submit-button"]');

  // By CSS selector
  await page.click(".btn-primary");

  // By XPath
  await page.click('xpath=//button[contains(text(), "Submit")]');

  // Chaining
  const form = page.locator("form");
  await form.locator('input[name="email"]').fill("user@example.com");

  // Get by multiple selectors
  await page.locator('button:has-text("Submit")').click();
});
```

---

## Cypress Setup and Basics

### **Installation**

```bash
npm install cypress --save-dev
npx cypress open
```

### **Basic Test Structure**

```typescript
describe("User Authentication", () => {
  beforeEach(() => {
    cy.visit("http://localhost:3000");
  });

  it("user can login successfully", () => {
    // Navigate to login
    cy.contains("Login").click();

    // Fill form
    cy.get('input[name="email"]').type("user@example.com");
    cy.get('input[name="password"]').type("password123");

    // Submit
    cy.get('button[type="submit"]').click();

    // Assert
    cy.url().should("include", "/dashboard");
    cy.contains("Welcome").should("be.visible");
  });

  it("shows error for invalid credentials", () => {
    cy.contains("Login").click();
    cy.get('input[name="email"]').type("wrong@example.com");
    cy.get('input[name="password"]').type("wrongpass");
    cy.get('button[type="submit"]').click();

    cy.contains("Invalid credentials").should("be.visible");
  });
});
```

### **Cypress Commands**

```typescript
describe("Cypress Commands", () => {
  it("demonstrates common commands", () => {
    cy.visit("/");

    // Get elements
    cy.get(".btn").click();
    cy.get('[data-testid="submit"]').click();

    // Find within
    cy.get("form").within(() => {
      cy.get('input[name="email"]').type("user@example.com");
    });

    // Contains
    cy.contains("Submit").click();

    // Input
    cy.get('input[name="username"]')
      .type("johndoe")
      .should("have.value", "johndoe");

    // Select dropdown
    cy.get("select").select("Option 2");

    // Checkbox
    cy.get('input[type="checkbox"]').check();
    cy.get('input[type="checkbox"]').uncheck();

    // Assertions
    cy.get(".title").should("have.text", "Welcome");
    cy.get(".btn").should("be.visible");
    cy.get(".error").should("not.exist");
  });
});
```

---

## Page Object Model (POM)

### **Playwright Page Objects**

```typescript
// pages/LoginPage.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.locator('input[name="email"]');
    this.passwordInput = page.locator('input[name="password"]');
    this.submitButton = page.locator('button[type="submit"]');
    this.errorMessage = page.locator(".error-message");
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage() {
    return await this.errorMessage.textContent();
  }
}

// tests/login.spec.ts
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/LoginPage";

test("login with page object", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("user@example.com", "password123");

  await expect(page).toHaveURL(/\/dashboard/);
});
```

### **Cypress Page Objects**

```typescript
// cypress/pages/LoginPage.ts
export class LoginPage {
  visit() {
    cy.visit("/login");
  }

  get emailInput() {
    return cy.get('input[name="email"]');
  }

  get passwordInput() {
    return cy.get('input[name="password"]');
  }

  get submitButton() {
    return cy.get('button[type="submit"]');
  }

  get errorMessage() {
    return cy.get(".error-message");
  }

  login(email: string, password: string) {
    this.emailInput.type(email);
    this.passwordInput.type(password);
    this.submitButton.click();
  }
}

// cypress/e2e/login.cy.ts
import { LoginPage } from "../pages/LoginPage";

describe("Login", () => {
  const loginPage = new LoginPage();

  it("logs in successfully", () => {
    loginPage.visit();
    loginPage.login("user@example.com", "password123");

    cy.url().should("include", "/dashboard");
  });
});
```

---

## API Mocking and Stubbing

### **Playwright API Interception**

```typescript
import { test, expect } from "@playwright/test";

test("mocks API response", async ({ page }) => {
  // Mock API call
  await page.route("**/api/users", (route) => {
    route.fulfill({
      status: 200,
      contentType: "application/json",
      body: JSON.stringify([
        { id: 1, name: "John Doe" },
        { id: 2, name: "Jane Smith" },
      ]),
    });
  });

  await page.goto("/users");

  await expect(page.locator("text=John Doe")).toBeVisible();
  await expect(page.locator("text=Jane Smith")).toBeVisible();
});

test("simulates API error", async ({ page }) => {
  await page.route("**/api/users", (route) => {
    route.fulfill({
      status: 500,
      contentType: "application/json",
      body: JSON.stringify({ error: "Server error" }),
    });
  });

  await page.goto("/users");

  await expect(page.locator("text=Error loading users")).toBeVisible();
});

test("waits for API response", async ({ page }) => {
  const responsePromise = page.waitForResponse("**/api/users");

  await page.goto("/users");

  const response = await responsePromise;
  expect(response.status()).toBe(200);

  const body = await response.json();
  expect(body).toHaveLength(2);
});
```

### **Cypress API Interception**

```typescript
describe("API Mocking", () => {
  it("intercepts API calls", () => {
    cy.intercept("GET", "/api/users", {
      statusCode: 200,
      body: [
        { id: 1, name: "John Doe" },
        { id: 2, name: "Jane Smith" },
      ],
    }).as("getUsers");

    cy.visit("/users");

    cy.wait("@getUsers");

    cy.contains("John Doe").should("be.visible");
    cy.contains("Jane Smith").should("be.visible");
  });

  it("simulates slow network", () => {
    cy.intercept("GET", "/api/users", (req) => {
      req.reply({
        delay: 2000, // 2 second delay
        body: [{ id: 1, name: "John Doe" }],
      });
    });

    cy.visit("/users");

    // Loading indicator should show
    cy.get(".loading").should("be.visible");

    // Then data appears
    cy.contains("John Doe", { timeout: 3000 }).should("be.visible");
  });

  it("modifies request headers", () => {
    cy.intercept("POST", "/api/login", (req) => {
      req.headers["x-custom-header"] = "test-value";
      req.continue();
    });

    cy.visit("/login");
    cy.get('input[name="email"]').type("user@example.com");
    cy.get('button[type="submit"]').click();
  });
});
```

---

## Authentication and Session Handling

### **Playwright Authentication**

```typescript
// global-setup.ts
import { chromium, FullConfig } from "@playwright/test";

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto("http://localhost:3000/login");
  await page.fill('input[name="email"]', "admin@example.com");
  await page.fill('input[name="password"]', "admin123");
  await page.click('button[type="submit"]');

  // Wait for auth cookie
  await page.waitForURL("**/dashboard");

  // Save storage state
  await page.context().storageState({ path: "auth.json" });

  await browser.close();
}

export default globalSetup;

// playwright.config.ts
import { PlaywrightTestConfig } from "@playwright/test";

const config: PlaywrightTestConfig = {
  globalSetup: require.resolve("./global-setup"),
  use: {
    storageState: "auth.json", // Reuse auth state
  },
};

export default config;

// tests/dashboard.spec.ts
test("accesses protected route", async ({ page }) => {
  // Already authenticated via global setup
  await page.goto("/dashboard");

  await expect(page.locator("text=Dashboard")).toBeVisible();
});
```

### **Cypress Custom Commands for Auth**

```typescript
// cypress/support/commands.ts
Cypress.Commands.add("login", (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit("/login");
    cy.get('input[name="email"]').type(email);
    cy.get('input[name="password"]').type(password);
    cy.get('button[type="submit"]').click();
    cy.url().should("include", "/dashboard");
  });
});

// cypress/e2e/dashboard.cy.ts
describe("Dashboard", () => {
  beforeEach(() => {
    cy.login("user@example.com", "password123");
  });

  it("shows user dashboard", () => {
    cy.visit("/dashboard");
    cy.contains("Welcome").should("be.visible");
  });
});
```

---

## Visual Regression Testing

### **Playwright Screenshots**

```typescript
import { test, expect } from "@playwright/test";

test("visual regression", async ({ page }) => {
  await page.goto("/");

  // Full page screenshot
  await expect(page).toHaveScreenshot("homepage.png");

  // Element screenshot
  const header = page.locator("header");
  await expect(header).toHaveScreenshot("header.png");

  // With options
  await expect(page).toHaveScreenshot("homepage-dark.png", {
    fullPage: true,
    maxDiffPixels: 100,
  });
});
```

### **Cypress Visual Testing**

```typescript
describe("Visual Regression", () => {
  it("matches homepage snapshot", () => {
    cy.visit("/");
    cy.matchImageSnapshot("homepage");
  });

  it("matches component snapshot", () => {
    cy.visit("/");
    cy.get(".hero").matchImageSnapshot("hero-section");
  });
});
```

---

## Testing Complex Workflows

### **E-commerce Checkout Flow**

```typescript
// Playwright
test("complete checkout flow", async ({ page }) => {
  await page.goto("/products");

  // Add products to cart
  await page.click('button:has-text("Add to Cart")');
  await page.click('[data-testid="cart-icon"]');

  // Verify cart
  await expect(page.locator(".cart-item")).toHaveCount(1);

  // Proceed to checkout
  await page.click("text=Checkout");

  // Fill shipping info
  await page.fill('input[name="name"]', "John Doe");
  await page.fill('input[name="address"]', "123 Main St");
  await page.fill('input[name="city"]', "Springfield");
  await page.click('button:has-text("Continue")');

  // Fill payment info
  await page.fill('input[name="cardNumber"]', "4242424242424242");
  await page.fill('input[name="expiry"]', "12/25");
  await page.fill('input[name="cvv"]', "123");

  // Place order
  await page.click('button:has-text("Place Order")');

  // Verify success
  await expect(page).toHaveURL(/\/order-confirmation/);
  await expect(page.locator("text=Order placed successfully")).toBeVisible();
});
```

### **Multi-Step Form**

```typescript
// Cypress
describe("Multi-Step Form", () => {
  it("completes all steps", () => {
    cy.visit("/signup");

    // Step 1: Personal Info
    cy.get('input[name="firstName"]').type("John");
    cy.get('input[name="lastName"]').type("Doe");
    cy.get('input[name="email"]').type("john@example.com");
    cy.contains("Next").click();

    // Step 2: Address
    cy.url().should("include", "/step2");
    cy.get('input[name="street"]').type("123 Main St");
    cy.get('input[name="city"]').type("Springfield");
    cy.contains("Next").click();

    // Step 3: Review and Submit
    cy.url().should("include", "/review");
    cy.contains("John Doe").should("be.visible");
    cy.contains("123 Main St").should("be.visible");
    cy.contains("Submit").click();

    // Success
    cy.url().should("include", "/success");
    cy.contains("Registration complete").should("be.visible");
  });

  it("can navigate back", () => {
    cy.visit("/signup");

    cy.get('input[name="firstName"]').type("John");
    cy.contains("Next").click();

    cy.contains("Back").click();

    cy.get('input[name="firstName"]').should("have.value", "John");
  });
});
```

---

## Mobile and Responsive Testing

### **Playwright Device Emulation**

```typescript
import { test, devices } from "@playwright/test";

test.describe("Mobile Tests", () => {
  test.use({
    ...devices["iPhone 13"],
  });

  test("renders mobile menu", async ({ page }) => {
    await page.goto("/");

    await expect(page.locator(".mobile-menu-button")).toBeVisible();
    await expect(page.locator(".desktop-menu")).not.toBeVisible();
  });
});

test.describe("Tablet Tests", () => {
  test.use({
    ...devices["iPad Pro"],
  });

  test("renders tablet layout", async ({ page }) => {
    await page.goto("/");
    // Test tablet-specific layout
  });
});
```

### **Cypress Viewport Testing**

```typescript
describe("Responsive Tests", () => {
  it("mobile view", () => {
    cy.viewport("iphone-x");
    cy.visit("/");

    cy.get(".mobile-menu-button").should("be.visible");
    cy.get(".desktop-menu").should("not.be.visible");
  });

  it("tablet view", () => {
    cy.viewport("ipad-2");
    cy.visit("/");

    // Test tablet layout
  });

  it("desktop view", () => {
    cy.viewport(1920, 1080);
    cy.visit("/");

    cy.get(".desktop-menu").should("be.visible");
    cy.get(".mobile-menu-button").should("not.be.visible");
  });
});
```

---

## Performance Testing

### **Playwright Performance Metrics**

```typescript
test("measures page load time", async ({ page }) => {
  const start = Date.now();
  await page.goto("/");
  const loadTime = Date.now() - start;

  expect(loadTime).toBeLessThan(3000); // < 3 seconds

  // Performance metrics
  const metrics = await page.evaluate(() => {
    const nav = performance.getEntriesByType(
      "navigation",
    )[0] as PerformanceNavigationTiming;
    return {
      domContentLoaded: nav.domContentLoadedEventEnd - nav.fetchStart,
      loadComplete: nav.loadEventEnd - nav.fetchStart,
      firstPaint: performance.getEntriesByName("first-paint")[0]?.startTime,
    };
  });

  console.log("Performance metrics:", metrics);
  expect(metrics.domContentLoaded).toBeLessThan(2000);
});
```

---

## Best Practices

### ✅ DO: Use Data Attributes for Stable Selectors

```typescript
// Good - Stable selector
await page.click('[data-testid="submit-button"]');

// Bad - Brittle selector
await page.click(".btn.btn-primary.mt-4");
```

### ✅ DO: Use Page Objects

```typescript
// Good - Reusable and maintainable
const loginPage = new LoginPage(page);
await loginPage.login("user@example.com", "password123");
```

### ✅ DO: Test Critical User Paths

```typescript
// Good - Tests important workflow
test("user can complete purchase", async ({ page }) => {
  // Browse → Add to Cart → Checkout → Payment → Confirmation
});
```

### ❌ DON'T: Test Everything in E2E

```typescript
// Bad - Unit test concern
test("validates email format", async ({ page }) => {
  // This should be a unit test
});
```

### ✅ DO: Run E2E Tests in Parallel

```typescript
// playwright.config.ts
export default {
  workers: 4, // Run 4 tests in parallel
};
```

---

## Key Takeaways

1. **E2E tests verify complete user workflows** in real browsers
2. **Playwright supports multiple browsers** (Chromium, Firefox, WebKit)
3. **Cypress provides excellent DX** with time-travel debugging
4. **Page Object Model** improves maintainability and reusability
5. **Mock APIs** for predictable test environments
6. **Authenticate once** and reuse sessions across tests
7. **Visual regression** catches unintended UI changes
8. **Device emulation** tests responsive designs
9. **Keep E2E tests focused** on critical user paths
10. **Run E2E in CI/CD** but keep execution time reasonable
