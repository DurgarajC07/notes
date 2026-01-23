# 🧪 Testing Best Practices with React Testing Library

> Testing Library encourages testing user behavior, not implementation details. This guide covers patterns, anti-patterns, and real-world testing strategies for production React applications.

---

## 📖 1. Concept Explanation

### Testing Library Philosophy

**Core Principle:** Test how users interact with your app, not internal implementation.

```javascript
// ❌ WRONG: Testing implementation
expect(component.state.count).toBe(5);

// ✅ RIGHT: Testing behavior
expect(screen.getByText(/count: 5/i)).toBeInTheDocument();
```

### Guiding Principles

1. **Query by accessibility**
2. **Test user interactions**
3. **Avoid testing implementation details**
4. **Write tests that resemble user behavior**

---

## 🧠 2. Why It Matters

### Real-World Impact

**Confidence in Refactoring:**

```javascript
// Component refactor from class to hooks
// ✅ Tests still pass (behavior unchanged)
// ❌ Would fail if testing implementation
```

**Catch Real Bugs:**

```javascript
// Test user flow: Click button → see result
// Catches bugs users would encounter
// Not just "function returns correct value"
```

**Maintainability:**

```javascript
// Change internal state structure
// Tests don't break (testing behavior, not state)
```

---

## ⚙️ 3. Query Priority

### Recommended Query Order

**1. Accessible by Everyone:**

```javascript
// getByRole - BEST
screen.getByRole("button", { name: /submit/i });
screen.getByRole("heading", { level: 1 });

// getByLabelText - Form inputs
screen.getByLabelText(/email/i);
screen.getByLabelText("Password");
```

**2. Semantic Queries:**

```javascript
// getByText - Text content
screen.getByText(/hello world/i);
screen.getByText("Welcome");

// getByDisplayValue - Form current value
screen.getByDisplayValue("John Doe");
```

**3. Test IDs (Last Resort):**

```javascript
// Only when other queries don't work
screen.getByTestId("custom-element");
```

### Query Variants

```javascript
// getBy - Throws if not found (default)
const button = screen.getByRole("button");

// queryBy - Returns null if not found
const button = screen.queryByRole("button");
expect(button).not.toBeInTheDocument();

// findBy - Async, waits for element
const button = await screen.findByRole("button");

// getAllBy - Returns array
const buttons = screen.getAllByRole("button");
expect(buttons).toHaveLength(3);
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Test user behavior
test("user can submit form", async () => {
  const user = userEvent.setup();
  render(<ContactForm />);

  await user.type(screen.getByLabelText(/email/i), "test@example.com");
  await user.click(screen.getByRole("button", { name: /submit/i }));

  expect(await screen.findByText(/success/i)).toBeInTheDocument();
});

// 2. Use userEvent over fireEvent
// ✅ BETTER: userEvent (more realistic)
await userEvent.type(input, "Hello");

// ❌ AVOID: fireEvent (too low-level)
fireEvent.change(input, { target: { value: "Hello" } });

// 3. Query by role
// ✅ GOOD
screen.getByRole("button", { name: /submit/i });

// ❌ AVOID
screen.getByTestId("submit-button");

// 4. Use findBy for async
// ✅ GOOD
expect(await screen.findByText(/loaded/i)).toBeInTheDocument();

// ❌ WRONG: Missing await
expect(screen.findByText(/loaded/i)).toBeInTheDocument();

// 5. Test accessibility
test("form is accessible", () => {
  render(<LoginForm />);

  // All inputs have labels
  expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/password/i)).toBeInTheDocument();

  // Button has accessible name
  expect(screen.getByRole("button", { name: /login/i })).toBeInTheDocument();
});
```

### DON'T ❌

```javascript
// ❌ Don't test implementation details
test("counter increments state", () => {
  const { result } = renderHook(() => useState(0));
  expect(result.current[0]).toBe(0); // Testing internal state!
});

// ✅ Test behavior instead
test("counter displays incremented value", async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole("button", { name: /increment/i }));

  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});

// ❌ Don't use container querySelector
const { container } = render(<Component />);
const element = container.querySelector(".my-class"); // Brittle!

// ✅ Use proper queries
const element = screen.getByRole("button");

// ❌ Don't test library code
test("useState works", () => {
  // Don't test React itself!
});

// ✅ Test your code
test("component uses state correctly", () => {
  // Test your component's behavior
});
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Not Waiting for Async Updates

```javascript
// ❌ WRONG: Missing await
test("loads data", () => {
  render(<DataList />);

  // This fails! Data hasn't loaded yet
  expect(screen.getByText(/item 1/i)).toBeInTheDocument();
});

// ✅ CORRECT: Wait for async
test("loads data", async () => {
  render(<DataList />);

  // findBy automatically waits
  expect(await screen.findByText(/item 1/i)).toBeInTheDocument();
});

// ✅ ALTERNATIVE: waitFor
test("loads data", async () => {
  render(<DataList />);

  await waitFor(() => {
    expect(screen.getByText(/item 1/i)).toBeInTheDocument();
  });
});
```

### Mistake #2: Using Wrong Query Type

```javascript
// ❌ WRONG: getBy for element that might not exist
test("error message shown on failure", () => {
  render(<Form />);

  // Throws if not found!
  const error = screen.getByText(/error/i);
  expect(error).toBeInTheDocument();
});

// ✅ CORRECT: queryBy for optional elements
test("no error shown initially", () => {
  render(<Form />);

  // Returns null if not found
  expect(screen.queryByText(/error/i)).not.toBeInTheDocument();
});

test("error shown after invalid submission", async () => {
  const user = userEvent.setup();
  render(<Form />);

  await user.click(screen.getByRole("button", { name: /submit/i }));

  // Now it should exist
  expect(await screen.findByText(/error/i)).toBeInTheDocument();
});
```

### Mistake #3: Testing Implementation Details

```javascript
// ❌ WRONG: Testing props/state
test("prop passed correctly", () => {
  const wrapper = shallow(<Child value={5} />);
  expect(wrapper.prop("value")).toBe(5); // Implementation detail!
});

// ✅ CORRECT: Testing rendered output
test("displays value", () => {
  render(<Child value={5} />);
  expect(screen.getByText("Value: 5")).toBeInTheDocument();
});

// ❌ WRONG: Testing component methods
test("handleClick called", () => {
  const spy = jest.spyOn(Component.prototype, "handleClick");
  // Testing implementation!
});

// ✅ CORRECT: Testing user interaction result
test("clicking button shows message", async () => {
  const user = userEvent.setup();
  render(<Component />);

  await user.click(screen.getByRole("button"));

  expect(screen.getByText(/clicked/i)).toBeInTheDocument();
});
```

---

## 🧪 6. Testing Patterns

### Pattern 1: Form Testing

```javascript
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

test("user can fill and submit form", async () => {
  const user = userEvent.setup();
  const mockSubmit = jest.fn();

  render(<ContactForm onSubmit={mockSubmit} />);

  // Fill form
  await user.type(screen.getByLabelText(/name/i), "John Doe");
  await user.type(screen.getByLabelText(/email/i), "john@example.com");
  await user.type(screen.getByLabelText(/message/i), "Hello!");

  // Submit
  await user.click(screen.getByRole("button", { name: /submit/i }));

  // Verify
  expect(mockSubmit).toHaveBeenCalledWith({
    name: "John Doe",
    email: "john@example.com",
    message: "Hello!",
  });
});

test("shows validation errors", async () => {
  const user = userEvent.setup();
  render(<ContactForm />);

  // Submit without filling
  await user.click(screen.getByRole("button", { name: /submit/i }));

  // Check errors
  expect(await screen.findByText(/name is required/i)).toBeInTheDocument();
  expect(screen.getByText(/email is required/i)).toBeInTheDocument();
});
```

### Pattern 2: Async Data Loading

```javascript
import { rest } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  rest.get("/api/users", (req, res, ctx) => {
    return res(
      ctx.json([
        { id: 1, name: "Alice" },
        { id: 2, name: "Bob" },
      ]),
    );
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("loads and displays users", async () => {
  render(<UserList />);

  // Loading state
  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  // Wait for data
  expect(await screen.findByText("Alice")).toBeInTheDocument();
  expect(screen.getByText("Bob")).toBeInTheDocument();

  // Loading state gone
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});

test("handles error state", async () => {
  // Override handler to return error
  server.use(
    rest.get("/api/users", (req, res, ctx) => {
      return res(ctx.status(500));
    }),
  );

  render(<UserList />);

  expect(await screen.findByText(/error loading users/i)).toBeInTheDocument();
});
```

### Pattern 3: Component Interaction

```javascript
test("todo list functionality", async () => {
  const user = userEvent.setup();
  render(<TodoList />);

  // Add todo
  const input = screen.getByPlaceholderText(/add todo/i);
  await user.type(input, "Buy milk");
  await user.keyboard("{Enter}");

  // Verify added
  expect(screen.getByText("Buy milk")).toBeInTheDocument();
  expect(input).toHaveValue("");

  // Toggle todo
  const checkbox = screen.getByRole("checkbox", { name: /buy milk/i });
  await user.click(checkbox);

  expect(checkbox).toBeChecked();
  expect(screen.getByText("Buy milk")).toHaveStyle({
    textDecoration: "line-through",
  });

  // Delete todo
  const deleteButton = screen.getByRole("button", { name: /delete buy milk/i });
  await user.click(deleteButton);

  expect(screen.queryByText("Buy milk")).not.toBeInTheDocument();
});
```

### Pattern 4: Mocking Context

```javascript
const AuthContext = createContext(null);

function renderWithAuth(ui, { user = null } = {}) {
  return render(
    <AuthContext.Provider value={{ user }}>{ui}</AuthContext.Provider>,
  );
}

test("shows user name when logged in", () => {
  renderWithAuth(<Profile />, {
    user: { name: "Alice", email: "alice@example.com" },
  });

  expect(screen.getByText("Welcome, Alice!")).toBeInTheDocument();
});

test("shows login button when logged out", () => {
  renderWithAuth(<Profile />, { user: null });

  expect(screen.getByRole("button", { name: /login/i })).toBeInTheDocument();
  expect(screen.queryByText(/welcome/i)).not.toBeInTheDocument();
});
```

---

## 🏗️ 7. Real-World Examples

### Example 1: E-commerce Cart

```javascript
describe("Shopping Cart", () => {
  test("user can add items to cart", async () => {
    const user = userEvent.setup();
    render(<ProductPage />);

    // Add to cart
    await user.click(screen.getByRole("button", { name: /add to cart/i }));

    // Cart badge updates
    expect(screen.getByText("1")).toBeInTheDocument(); // Cart count

    // Success message
    expect(await screen.findByText(/added to cart/i)).toBeInTheDocument();
  });

  test("user can remove items from cart", async () => {
    const user = userEvent.setup();
    render(<Cart items={[{ id: 1, name: "Product 1", price: 10 }]} />);

    await user.click(screen.getByRole("button", { name: /remove product 1/i }));

    expect(screen.queryByText("Product 1")).not.toBeInTheDocument();
    expect(screen.getByText(/cart is empty/i)).toBeInTheDocument();
  });

  test("calculates total price correctly", () => {
    render(
      <Cart
        items={[
          { id: 1, name: "Item 1", price: 10, quantity: 2 },
          { id: 2, name: "Item 2", price: 15, quantity: 1 },
        ]}
      />,
    );

    expect(screen.getByText("Total: $35.00")).toBeInTheDocument();
  });
});
```

### Example 2: Search with Debounce

```javascript
test("searches with debounce", async () => {
  const user = userEvent.setup();
  const mockSearch = jest.fn();

  render(<SearchBar onSearch={mockSearch} />);

  const input = screen.getByPlaceholderText(/search/i);

  // Type quickly
  await user.type(input, "react");

  // Should not call immediately
  expect(mockSearch).not.toHaveBeenCalled();

  // Wait for debounce (300ms)
  await waitFor(
    () => {
      expect(mockSearch).toHaveBeenCalledWith("react");
    },
    { timeout: 500 },
  );

  // Should only call once (debounced)
  expect(mockSearch).toHaveBeenCalledTimes(1);
});
```

---

## ❓ 8. Interview Questions

### Q1: Why use Testing Library over Enzyme?

**Answer:**

**Testing Library encourages testing behavior, not implementation.**

| Testing Library          | Enzyme                      |
| ------------------------ | --------------------------- |
| Queries by accessibility | Queries by props/state      |
| Tests user behavior      | Tests implementation        |
| Refactor-friendly        | Breaks on refactors         |
| No shallow rendering     | Shallow rendering available |

**Example:**

```javascript
// ❌ Enzyme: Tests implementation
const wrapper = shallow(<Counter />);
expect(wrapper.state("count")).toBe(0);

// ✅ Testing Library: Tests behavior
render(<Counter />);
expect(screen.getByText("Count: 0")).toBeInTheDocument();
```

**Benefits:**

- Tests what users see
- More maintainable
- Catches real bugs
- Encourages accessibility

---

### Q2: When to use getBy vs queryBy vs findBy?

**Answer:**

| Query     | When to Use             | Returns             |
| --------- | ----------------------- | ------------------- |
| `getBy`   | Element should exist    | Throws if not found |
| `queryBy` | Element might not exist | `null` if not found |
| `findBy`  | Element appears async   | Promise (waits)     |

**Examples:**

```javascript
// getBy - Element must exist
const button = screen.getByRole("button"); // Throws if not found

// queryBy - Optional element
const error = screen.queryByText(/error/i); // null if not found
expect(error).not.toBeInTheDocument();

// findBy - Async element
const message = await screen.findByText(/success/i); // Waits up to 1s
expect(message).toBeInTheDocument();
```

**Async testing:**

```javascript
test("async data loads", async () => {
  render(<UserProfile id="1" />);

  // ❌ WRONG: Doesn't wait
  expect(screen.getByText("Alice")).toBeInTheDocument(); // Fails!

  // ✅ CORRECT: Waits for element
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```

---

### Q3: How do you test user interactions?

**Answer:**

**Use `@testing-library/user-event` for realistic interactions.**

```javascript
import userEvent from "@testing-library/user-event";

test("user interactions", async () => {
  const user = userEvent.setup();
  render(<Form />);

  // Type in input
  await user.type(screen.getByLabelText(/email/i), "test@example.com");

  // Click button
  await user.click(screen.getByRole("button", { name: /submit/i }));

  // Keyboard navigation
  await user.tab(); // Focus next element
  await user.keyboard("{Enter}"); // Press Enter

  // Select dropdown
  await user.selectOptions(screen.getByLabelText(/country/i), "USA");

  // Upload file
  const file = new File(["content"], "test.png", { type: "image/png" });
  await user.upload(screen.getByLabelText(/upload/i), file);
});
```

**Why userEvent over fireEvent:**

```javascript
// ❌ fireEvent: Too low-level
fireEvent.change(input, { target: { value: "test" } });
// Doesn't trigger focus, input, keydown, keyup events

// ✅ userEvent: Realistic
await userEvent.type(input, "test");
// Triggers all events a real user would
```

---

### Q4: How do you mock API calls in tests?

**Answer:**

**Use MSW (Mock Service Worker) for realistic API mocking.**

```javascript
import { rest } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  rest.get("/api/users/:id", (req, res, ctx) => {
    const { id } = req.params;
    return res(ctx.json({ id, name: "Alice" }));
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("loads user data", async () => {
  render(<UserProfile id="1" />);

  expect(await screen.findByText("Alice")).toBeInTheDocument();
});

test("handles errors", async () => {
  server.use(
    rest.get("/api/users/:id", (req, res, ctx) => {
      return res(ctx.status(500));
    }),
  );

  render(<UserProfile id="1" />);

  expect(await screen.findByText(/error/i)).toBeInTheDocument();
});
```

**Benefits:**

- Works at network level
- Same handlers for tests and development
- Realistic error scenarios

---

## 🎯 Summary

**Testing Library principles:**

- ✅ Query by accessibility (`getByRole`, `getByLabelText`)
- ✅ Test user behavior, not implementation
- ✅ Use `userEvent` for interactions
- ✅ `findBy` for async elements
- ✅ `queryBy` for optional elements
- ✅ Mock API with MSW
- ✅ Test accessibility

**Anti-patterns:**

- ❌ Testing props/state directly
- ❌ Using `container.querySelector`
- ❌ Testing library code
- ❌ Shallow rendering
- ❌ Testing implementation details

Write tests users would run! 🧪
