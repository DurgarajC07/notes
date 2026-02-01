# Accessibility Attributes & ARIA

## Core Concepts

Web accessibility ensures everyone, including people with disabilities, can perceive, understand, navigate, and interact with websites. ARIA (Accessible Rich Internet Applications) extends HTML's semantic capabilities with roles, states, and properties that communicate interface elements and dynamic changes to assistive technologies. Modern web development requires understanding ARIA roles, labels, live regions, focus management, and WCAG compliance to create inclusive experiences.

### ARIA Roles

ARIA roles define what an element is or does - whether it's a button, navigation landmark, dialog, alert, or custom widget. Roles communicate purpose to screen readers when native HTML elements are insufficient or when creating custom interactive components.

### ARIA Properties and States

ARIA properties (aria-label, aria-labelledby, aria-describedby) provide accessible names and descriptions, while states (aria-expanded, aria-hidden, aria-current, aria-disabled) communicate dynamic conditions to assistive technologies.

### Focus Management

Keyboard accessibility requires managing focus order (tabindex), trapping focus in modals, and ensuring all interactive elements are reachable and operable without a mouse.

---

## HTML/CSS/TypeScript Code Examples

### Example 1: ARIA Roles - Landmark Regions

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ARIA Landmarks Example</title>
    <style>
      * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
      }

      body {
        font-family:
          system-ui,
          -apple-system,
          sans-serif;
        line-height: 1.6;
      }

      header {
        background: #1e293b;
        color: white;
        padding: 1rem 2rem;
      }

      nav {
        background: #334155;
        padding: 1rem 2rem;
      }

      nav ul {
        list-style: none;
        display: flex;
        gap: 2rem;
      }

      nav a {
        color: white;
        text-decoration: none;
      }

      nav a:hover,
      nav a:focus {
        text-decoration: underline;
      }

      .container {
        display: grid;
        grid-template-columns: 250px 1fr 250px;
        gap: 2rem;
        padding: 2rem;
        max-width: 1400px;
        margin: 0 auto;
      }

      aside {
        background: #f1f5f9;
        padding: 1.5rem;
        border-radius: 0.5rem;
      }

      main {
        background: white;
      }

      footer {
        background: #1e293b;
        color: white;
        padding: 2rem;
        text-align: center;
        margin-top: 2rem;
      }

      h1,
      h2,
      h3 {
        margin-bottom: 1rem;
      }

      .search-form {
        margin-bottom: 1.5rem;
      }

      .search-form input {
        width: 100%;
        padding: 0.5rem;
        border: 1px solid #cbd5e1;
        border-radius: 0.25rem;
      }
    </style>
  </head>
  <body>
    <!-- Banner: Site header -->
    <header role="banner">
      <h1>Accessible Web Application</h1>
      <p>Demonstrating ARIA landmark roles</p>
    </header>

    <!-- Navigation: Main navigation -->
    <nav role="navigation" aria-label="Main navigation">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
        <li><a href="/services">Services</a></li>
        <li><a href="/contact">Contact</a></li>
      </ul>
    </nav>

    <div class="container">
      <!-- Complementary: Sidebar content -->
      <aside role="complementary" aria-label="Sidebar">
        <h2>Quick Links</h2>
        <ul>
          <li><a href="#section1">Section 1</a></li>
          <li><a href="#section2">Section 2</a></li>
          <li><a href="#section3">Section 3</a></li>
        </ul>

        <!-- Search: Search functionality -->
        <section role="search" aria-label="Site search">
          <h3>Search</h3>
          <form class="search-form">
            <label for="siteSearch" class="sr-only">Search the site</label>
            <input
              type="search"
              id="siteSearch"
              name="q"
              placeholder="Search..."
              aria-label="Search query"
            />
            <button type="submit">Search</button>
          </form>
        </section>
      </aside>

      <!-- Main: Primary content -->
      <main role="main" id="main-content">
        <article>
          <h2>Main Content Area</h2>
          <p>
            This is the primary content of the page. Screen readers can jump
            directly here using landmark navigation.
          </p>

          <section id="section1">
            <h3>Section 1</h3>
            <p>Content for section 1...</p>
          </section>

          <section id="section2">
            <h3>Section 2</h3>
            <p>Content for section 2...</p>
          </section>

          <section id="section3">
            <h3>Section 3</h3>
            <p>Content for section 3...</p>
          </section>
        </article>
      </main>

      <!-- Complementary: Related content -->
      <aside role="complementary" aria-label="Related content">
        <h2>Related Articles</h2>
        <ul>
          <li><a href="/article1">Understanding ARIA</a></li>
          <li><a href="/article2">Keyboard Navigation</a></li>
          <li><a href="/article3">Screen Reader Testing</a></li>
        </ul>
      </aside>
    </div>

    <!-- Contentinfo: Site footer -->
    <footer role="contentinfo">
      <p>&copy; 2026 Accessible Web App. All rights reserved.</p>
      <nav aria-label="Footer navigation">
        <ul
          style="list-style: none; display: flex; gap: 1rem; justify-content: center; margin-top: 1rem;"
        >
          <li><a href="/privacy" style="color: white;">Privacy Policy</a></li>
          <li><a href="/terms" style="color: white;">Terms of Service</a></li>
          <li>
            <a href="/accessibility" style="color: white;"
              >Accessibility Statement</a
            >
          </li>
        </ul>
      </nav>
    </footer>
  </body>
</html>
```

### Example 2: ARIA Roles - Interactive Widgets

```html
<!-- Button Role -->
<div
  role="button"
  tabindex="0"
  aria-pressed="false"
  class="custom-button"
  onclick="toggleButton(this)"
  onkeydown="handleButtonKeydown(event, this)"
>
  Toggle Feature
</div>

<script>
  function toggleButton(button) {
    const pressed = button.getAttribute("aria-pressed") === "true";
    button.setAttribute("aria-pressed", String(!pressed));
    button.classList.toggle("pressed");
  }

  function handleButtonKeydown(event, button) {
    // Activate on Enter or Space
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      toggleButton(button);
    }
  }
</script>

<!-- Tab Interface -->
<div class="tabs">
  <div role="tablist" aria-label="Content sections">
    <button
      role="tab"
      aria-selected="true"
      aria-controls="panel-1"
      id="tab-1"
      tabindex="0"
    >
      Tab 1
    </button>
    <button
      role="tab"
      aria-selected="false"
      aria-controls="panel-2"
      id="tab-2"
      tabindex="-1"
    >
      Tab 2
    </button>
    <button
      role="tab"
      aria-selected="false"
      aria-controls="panel-3"
      id="tab-3"
      tabindex="-1"
    >
      Tab 3
    </button>
  </div>

  <div role="tabpanel" id="panel-1" aria-labelledby="tab-1">
    <h3>Panel 1 Content</h3>
    <p>Content for the first tab...</p>
  </div>

  <div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>
    <h3>Panel 2 Content</h3>
    <p>Content for the second tab...</p>
  </div>

  <div role="tabpanel" id="panel-3" aria-labelledby="tab-3" hidden>
    <h3>Panel 3 Content</h3>
    <p>Content for the third tab...</p>
  </div>
</div>

<!-- Alert Role -->
<div role="alert" aria-live="assertive" class="alert alert-error">
  <strong>Error:</strong> Please correct the form errors before submitting.
</div>

<!-- Status Role -->
<div role="status" aria-live="polite" aria-atomic="true">
  <p>Loading content... <span class="progress">0%</span></p>
</div>

<!-- Dialog/Modal Role -->
<div
  role="dialog"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-description"
  aria-modal="true"
  hidden
>
  <h2 id="dialog-title">Confirm Action</h2>
  <p id="dialog-description">Are you sure you want to delete this item?</p>
  <button type="button">Cancel</button>
  <button type="button">Delete</button>
</div>
```

### Example 3: aria-label vs aria-labelledby vs aria-describedby

```html
<!-- aria-label: Direct label for elements without visible text -->
<button aria-label="Close dialog">
  <svg><!-- X icon --></svg>
</button>

<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- Multiple navigation landmarks need unique labels -->
<nav aria-label="Primary navigation">
  <!-- Main nav items -->
</nav>

<footer>
  <nav aria-label="Footer navigation">
    <!-- Footer links -->
  </nav>
</footer>

<!-- aria-labelledby: Reference visible text as label -->
<section aria-labelledby="section-heading">
  <h2 id="section-heading">User Profile</h2>
  <p>Profile information goes here...</p>
</section>

<div role="dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Settings</h2>
  <!-- Dialog content -->
</div>

<!-- aria-labelledby can reference multiple elements -->
<table aria-labelledby="table-title table-caption">
  <caption id="table-caption">
    Quarterly sales figures
  </caption>
  <h3 id="table-title">Q4 2026 Results</h3>
  <!-- table content -->
</table>

<!-- aria-describedby: Additional descriptive text -->
<label for="username">Username</label>
<input
  type="text"
  id="username"
  name="username"
  aria-describedby="username-hint username-error"
  required
/>
<small id="username-hint"
  >Must be 3-15 characters, letters and numbers only</small
>
<div id="username-error" class="error" role="alert"></div>

<!-- Complex example: combining all three -->
<button aria-label="Delete user profile" aria-describedby="delete-warning">
  <svg><!-- Trash icon --></svg>
</button>
<p id="delete-warning" class="sr-only">
  This action cannot be undone. All user data will be permanently deleted.
</p>

<!-- Form with comprehensive labeling -->
<form>
  <fieldset
    aria-labelledby="contact-legend"
    aria-describedby="contact-description"
  >
    <legend id="contact-legend">Contact Information</legend>
    <p id="contact-description">
      We'll use this information to reach you about your order.
    </p>

    <div>
      <label for="email">Email Address</label>
      <input type="email" id="email" aria-describedby="email-hint" required />
      <small id="email-hint"
        >We'll never share your email with anyone else.</small
      >
    </div>
  </fieldset>
</form>
```

### Example 4: aria-hidden and Visibility

```html
<style>
  /* Visually hide but keep accessible to screen readers */
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }

  /* Hide from everyone including screen readers */
  .hidden {
    display: none;
  }

  /* Decorative images should be hidden from screen readers */
  .decorative {
    aria-hidden: true;
  }
</style>

<!-- Hide decorative icons from screen readers -->
<button>
  <span aria-hidden="true">🔍</span>
  <span class="sr-only">Search</span>
</button>

<!-- Icon with visible text doesn't need description -->
<button>
  <svg aria-hidden="true" focusable="false">
    <!-- Search icon SVG -->
  </svg>
  Search
</button>

<!-- Hide font icons from screen readers -->
<button aria-label="Save document">
  <i class="fas fa-save" aria-hidden="true"></i>
</button>

<!-- Hide duplicate content -->
<a href="/products">
  View Products
  <svg aria-hidden="true"><!-- Arrow icon --></svg>
</a>

<!-- Decorative images -->
<img src="decoration.png" alt="" role="presentation" />
<!-- or -->
<img src="decoration.png" alt="" aria-hidden="true" />

<!-- Hidden content that should remain accessible (for reference) -->
<div hidden>
  <ul id="error-messages">
    <li id="error-required">This field is required</li>
    <li id="error-email">Please enter a valid email</li>
    <li id="error-password">Password must be at least 8 characters</li>
  </ul>
</div>

<script>
  // Toggle aria-hidden dynamically
  function toggleMenu(menuId) {
    const menu = document.getElementById(menuId);
    const isHidden = menu.getAttribute("aria-hidden") === "true";

    menu.setAttribute("aria-hidden", String(!isHidden));
    menu.hidden = !isHidden; // Also toggle HTML hidden attribute
  }
</script>

<!-- Mobile menu example -->
<nav id="mobile-menu" aria-hidden="true" hidden>
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<button
  aria-expanded="false"
  aria-controls="mobile-menu"
  onclick="toggleMobileMenu()"
>
  Menu
</button>
```

### Example 5: ARIA Live Regions

```html
<!-- Polite: announce when user is idle -->
<div aria-live="polite" aria-atomic="true" class="status-message">
  <!-- Updates announced when screen reader finishes current speech -->
</div>

<!-- Assertive: announce immediately -->
<div
  role="alert"
  aria-live="assertive"
  aria-atomic="true"
  class="error-message"
>
  <!-- Updates interrupt current screen reader speech -->
</div>

<!-- Off: don't announce (default) -->
<div aria-live="off">
  <!-- Updates not announced -->
</div>

<style>
  .live-region {
    border: 2px solid #3b82f6;
    padding: 1rem;
    margin: 1rem 0;
    border-radius: 0.5rem;
  }

  .sr-only-live {
    position: absolute;
    left: -10000px;
    width: 1px;
    height: 1px;
    overflow: hidden;
  }
</style>

<!-- Timer Example -->
<div class="live-region">
  <h3>Timer</h3>
  <div role="timer" aria-live="off" aria-atomic="true" id="timer">
    <span id="timer-display">05:00</span>
  </div>
  <button onclick="startTimer()">Start</button>
</div>

<!-- Form Status Example -->
<form id="registration-form">
  <!-- Form fields -->
  <button type="submit">Register</button>

  <div
    role="status"
    aria-live="polite"
    aria-atomic="true"
    id="form-status"
    class="sr-only-live"
  ></div>
</form>

<script>
  // Update form status
  document
    .getElementById("registration-form")
    .addEventListener("submit", async (e) => {
      e.preventDefault();

      const status = document.getElementById("form-status");
      status.textContent = "Submitting form...";

      // Simulate async submission
      setTimeout(() => {
        status.textContent = "Registration successful! Welcome aboard.";
      }, 2000);
    });
</script>

<!-- Search Results Count -->
<div class="live-region">
  <label for="search-input">Search products</label>
  <input
    type="search"
    id="search-input"
    oninput="updateSearchResults(this.value)"
  />

  <div role="status" aria-live="polite" aria-atomic="true" id="search-status">
    <span id="results-count"></span>
  </div>

  <div id="search-results"></div>
</div>

<script>
  let searchTimeout;

  function updateSearchResults(query) {
    clearTimeout(searchTimeout);

    searchTimeout = setTimeout(() => {
      // Simulate search
      const count = Math.floor(Math.random() * 50);
      const resultsCount = document.getElementById("results-count");

      if (query.length > 0) {
        resultsCount.textContent = `Found ${count} result${count !== 1 ? "s" : ""} for "${query}"`;
      } else {
        resultsCount.textContent = "";
      }
    }, 500);
  }
</script>

<!-- Loading State with aria-busy -->
<section id="content-area" aria-busy="false" aria-live="polite">
  <h2>Content</h2>
  <div id="dynamic-content">
    <!-- Content loaded here -->
  </div>
</section>

<script>
  async function loadContent() {
    const section = document.getElementById("content-area");
    const contentDiv = document.getElementById("dynamic-content");

    // Set busy state
    section.setAttribute("aria-busy", "true");
    contentDiv.textContent = "Loading...";

    // Simulate async load
    await new Promise((resolve) => setTimeout(resolve, 2000));

    // Update content
    contentDiv.textContent = "Content loaded successfully!";
    section.setAttribute("aria-busy", "false");
  }
</script>
```

### Example 6: aria-expanded and aria-controls (Collapsible Content)

```html
<style>
  .accordion-item {
    border: 1px solid #e5e7eb;
    margin-bottom: 0.5rem;
    border-radius: 0.5rem;
    overflow: hidden;
  }

  .accordion-button {
    width: 100%;
    padding: 1rem;
    background: #f9fafb;
    border: none;
    text-align: left;
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .accordion-button:hover,
  .accordion-button:focus {
    background: #f3f4f6;
  }

  .accordion-button[aria-expanded="true"] {
    background: #e5e7eb;
  }

  .accordion-icon {
    transition: transform 0.2s;
  }

  .accordion-button[aria-expanded="true"] .accordion-icon {
    transform: rotate(180deg);
  }

  .accordion-panel {
    padding: 1rem;
    background: white;
  }

  .accordion-panel[hidden] {
    display: none;
  }
</style>

<div class="accordion">
  <!-- Accordion Item 1 -->
  <div class="accordion-item">
    <h3>
      <button
        class="accordion-button"
        aria-expanded="false"
        aria-controls="panel-1"
        id="accordion-button-1"
      >
        <span>What is ARIA?</span>
        <span class="accordion-icon" aria-hidden="true">▼</span>
      </button>
    </h3>
    <div
      id="panel-1"
      role="region"
      aria-labelledby="accordion-button-1"
      class="accordion-panel"
      hidden
    >
      <p>
        ARIA (Accessible Rich Internet Applications) is a set of attributes that
        define ways to make web content and applications more accessible to
        people with disabilities.
      </p>
    </div>
  </div>

  <!-- Accordion Item 2 -->
  <div class="accordion-item">
    <h3>
      <button
        class="accordion-button"
        aria-expanded="false"
        aria-controls="panel-2"
        id="accordion-button-2"
      >
        <span>Why use ARIA?</span>
        <span class="accordion-icon" aria-hidden="true">▼</span>
      </button>
    </h3>
    <div
      id="panel-2"
      role="region"
      aria-labelledby="accordion-button-2"
      class="accordion-panel"
      hidden
    >
      <p>
        ARIA helps assistive technologies understand dynamic content, custom
        widgets, and complex interactions that HTML alone cannot adequately
        describe.
      </p>
    </div>
  </div>

  <!-- Accordion Item 3 -->
  <div class="accordion-item">
    <h3>
      <button
        class="accordion-button"
        aria-expanded="false"
        aria-controls="panel-3"
        id="accordion-button-3"
      >
        <span>When should I use ARIA?</span>
        <span class="accordion-icon" aria-hidden="true">▼</span>
      </button>
    </h3>
    <div
      id="panel-3"
      role="region"
      aria-labelledby="accordion-button-3"
      class="accordion-panel"
      hidden
    >
      <p>
        Use ARIA when native HTML elements don't provide adequate semantics,
        when creating custom widgets, or when you need to communicate dynamic
        changes to screen readers.
      </p>
    </div>
  </div>
</div>

<script>
  class Accordion {
    constructor(selector) {
      this.accordion = document.querySelector(selector);
      this.buttons = this.accordion.querySelectorAll(".accordion-button");
      this.attachListeners();
    }

    attachListeners() {
      this.buttons.forEach((button) => {
        button.addEventListener("click", () => this.toggle(button));
      });
    }

    toggle(button) {
      const expanded = button.getAttribute("aria-expanded") === "true";
      const panel = document.getElementById(
        button.getAttribute("aria-controls"),
      );

      // Update aria-expanded
      button.setAttribute("aria-expanded", String(!expanded));

      // Toggle panel visibility
      panel.hidden = expanded;

      // Optional: Close other panels (for exclusive accordion)
      // this.closeOthers(button);
    }

    closeOthers(currentButton) {
      this.buttons.forEach((button) => {
        if (button !== currentButton) {
          button.setAttribute("aria-expanded", "false");
          const panel = document.getElementById(
            button.getAttribute("aria-controls"),
          );
          panel.hidden = true;
        }
      });
    }
  }

  // Initialize
  new Accordion(".accordion");
</script>

<!-- Dropdown Menu Example -->
<div class="dropdown">
  <button
    aria-expanded="false"
    aria-controls="dropdown-menu"
    aria-haspopup="true"
    id="dropdown-button"
  >
    Options <span aria-hidden="true">▼</span>
  </button>

  <ul id="dropdown-menu" role="menu" aria-labelledby="dropdown-button" hidden>
    <li role="none">
      <a href="#" role="menuitem">Edit</a>
    </li>
    <li role="none">
      <a href="#" role="menuitem">Delete</a>
    </li>
    <li role="none">
      <a href="#" role="menuitem">Share</a>
    </li>
  </ul>
</div>
```

### Example 7: aria-current for Navigation

```html
<style>
  .breadcrumb {
    display: flex;
    list-style: none;
    gap: 0.5rem;
    padding: 1rem 0;
  }

  .breadcrumb li:not(:last-child)::after {
    content: "/";
    margin-left: 0.5rem;
    color: #6b7280;
  }

  .breadcrumb [aria-current="page"] {
    color: #1f2937;
    font-weight: 600;
    text-decoration: none;
    pointer-events: none;
  }

  .pagination {
    display: flex;
    list-style: none;
    gap: 0.5rem;
  }

  .pagination a {
    padding: 0.5rem 1rem;
    border: 1px solid #e5e7eb;
    border-radius: 0.25rem;
    text-decoration: none;
    color: #1f2937;
  }

  .pagination [aria-current="page"] {
    background: #3b82f6;
    color: white;
    border-color: #3b82f6;
    pointer-events: none;
  }

  .nav-primary a {
    padding: 0.5rem 1rem;
    text-decoration: none;
    color: #1f2937;
  }

  .nav-primary [aria-current="page"] {
    font-weight: 600;
    border-bottom: 2px solid #3b82f6;
    color: #3b82f6;
  }
</style>

<!-- Breadcrumb Navigation -->
<nav aria-label="Breadcrumb">
  <ol class="breadcrumb">
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/products/electronics">Electronics</a></li>
    <li>
      <a href="/products/electronics/laptops" aria-current="page"> Laptops </a>
    </li>
  </ol>
</nav>

<!-- Primary Navigation with Current Page -->
<nav aria-label="Main navigation" class="nav-primary">
  <ul style="list-style: none; display: flex; gap: 1rem;">
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
    <li>
      <a href="/services" aria-current="page">Services</a>
    </li>
    <li><a href="/contact">Contact</a></li>
  </ul>
</nav>

<!-- Pagination -->
<nav aria-label="Pagination">
  <ul class="pagination">
    <li>
      <a href="?page=1" aria-label="Go to previous page">Previous</a>
    </li>
    <li>
      <a href="?page=1" aria-label="Go to page 1">1</a>
    </li>
    <li>
      <a href="?page=2" aria-label="Go to page 2">2</a>
    </li>
    <li>
      <a href="?page=3" aria-current="page" aria-label="Current page, page 3"
        >3</a
      >
    </li>
    <li>
      <a href="?page=4" aria-label="Go to page 4">4</a>
    </li>
    <li>
      <a href="?page=5" aria-label="Go to page 5">5</a>
    </li>
    <li>
      <a href="?page=4" aria-label="Go to next page">Next</a>
    </li>
  </ul>
</nav>

<!-- Step Indicator -->
<nav aria-label="Progress">
  <ol class="steps" style="list-style: none; display: flex; gap: 1rem;">
    <li>
      <a href="/checkout/cart" aria-current="step">
        <span aria-hidden="true">✓</span> Cart
      </a>
    </li>
    <li>
      <a href="/checkout/shipping" aria-current="step">
        <span aria-hidden="true">2</span> Shipping
      </a>
    </li>
    <li>
      <a href="/checkout/payment">
        <span aria-hidden="true">3</span> Payment
      </a>
    </li>
    <li>
      <a href="/checkout/confirm">
        <span aria-hidden="true">4</span> Confirm
      </a>
    </li>
  </ol>
</nav>

<!-- Date Picker with Current Date -->
<div role="application" aria-label="Calendar">
  <table role="grid" aria-label="March 2026">
    <thead>
      <tr>
        <th abbr="Sunday">Sun</th>
        <th abbr="Monday">Mon</th>
        <th abbr="Tuesday">Tue</th>
        <!-- ... -->
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><button tabindex="-1">1</button></td>
        <td><button tabindex="-1">2</button></td>
        <td>
          <button tabindex="0" aria-current="date">3</button>
        </td>
        <!-- ... -->
      </tr>
    </tbody>
  </table>
</div>

<!-- Other aria-current values -->
<!--
  aria-current="page" - Current page in navigation
  aria-current="step" - Current step in process
  aria-current="location" - Current location in breadcrumb/tree
  aria-current="date" - Current date in calendar
  aria-current="time" - Current time
  aria-current="true" - Generic current item
-->
```

### Example 8: Focus Management and Keyboard Navigation

```typescript
interface FocusableElement extends HTMLElement {
  focus(): void;
}

class FocusManager {
  private lastFocusedElement: HTMLElement | null = null;

  // Save current focus
  saveFocus(): void {
    this.lastFocusedElement = document.activeElement as HTMLElement;
  }

  // Restore previously focused element
  restoreFocus(): void {
    if (this.lastFocusedElement && this.lastFocusedElement.focus) {
      this.lastFocusedElement.focus();
    }
  }

  // Get all focusable elements within container
  getFocusableElements(container: HTMLElement): HTMLElement[] {
    const selector = [
      "a[href]",
      "button:not([disabled])",
      "textarea:not([disabled])",
      "input:not([disabled])",
      "select:not([disabled])",
      '[tabindex]:not([tabindex="-1"])',
    ].join(", ");

    return Array.from(container.querySelectorAll<HTMLElement>(selector));
  }

  // Focus first element in container
  focusFirst(container: HTMLElement): void {
    const focusable = this.getFocusableElements(container);
    if (focusable.length > 0) {
      focusable[0].focus();
    }
  }

  // Focus last element in container
  focusLast(container: HTMLElement): void {
    const focusable = this.getFocusableElements(container);
    if (focusable.length > 0) {
      focusable[focusable.length - 1].focus();
    }
  }

  // Move focus to next element
  focusNext(container: HTMLElement): void {
    const focusable = this.getFocusableElements(container);
    const currentIndex = focusable.indexOf(
      document.activeElement as HTMLElement,
    );

    if (currentIndex < focusable.length - 1) {
      focusable[currentIndex + 1].focus();
    } else {
      focusable[0].focus(); // Wrap to first
    }
  }

  // Move focus to previous element
  focusPrevious(container: HTMLElement): void {
    const focusable = this.getFocusableElements(container);
    const currentIndex = focusable.indexOf(
      document.activeElement as HTMLElement,
    );

    if (currentIndex > 0) {
      focusable[currentIndex - 1].focus();
    } else {
      focusable[focusable.length - 1].focus(); // Wrap to last
    }
  }
}

// Focus Trap for Modals
class FocusTrap {
  private container: HTMLElement;
  private focusableElements: HTMLElement[];
  private firstFocusable: HTMLElement;
  private lastFocusable: HTMLElement;
  private previousFocus: HTMLElement | null = null;

  constructor(containerSelector: string) {
    this.container = document.querySelector(containerSelector)!;
    this.focusableElements = this.getFocusableElements();
    this.firstFocusable = this.focusableElements[0];
    this.lastFocusable =
      this.focusableElements[this.focusableElements.length - 1];
  }

  private getFocusableElements(): HTMLElement[] {
    const selector =
      'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])';
    return Array.from(this.container.querySelectorAll<HTMLElement>(selector));
  }

  activate(): void {
    this.previousFocus = document.activeElement as HTMLElement;

    // Focus first element
    this.firstFocusable.focus();

    // Add keyboard listener
    document.addEventListener("keydown", this.handleKeydown);
  }

  deactivate(): void {
    document.removeEventListener("keydown", this.handleKeydown);

    // Restore focus
    if (this.previousFocus && this.previousFocus.focus) {
      this.previousFocus.focus();
    }
  }

  private handleKeydown = (e: KeyboardEvent): void => {
    if (e.key === "Tab") {
      if (e.shiftKey) {
        // Shift + Tab
        if (document.activeElement === this.firstFocusable) {
          e.preventDefault();
          this.lastFocusable.focus();
        }
      } else {
        // Tab
        if (document.activeElement === this.lastFocusable) {
          e.preventDefault();
          this.firstFocusable.focus();
        }
      }
    }

    // Close on Escape
    if (e.key === "Escape") {
      this.deactivate();
      // Trigger close event or callback
      this.container.dispatchEvent(new CustomEvent("close"));
    }
  };
}

// Usage Example
const focusManager = new FocusManager();

// Open modal
function openModal() {
  const modal = document.getElementById("modal")!;

  // Save current focus
  focusManager.saveFocus();

  // Show modal
  modal.hidden = false;
  modal.setAttribute("aria-hidden", "false");

  // Trap focus
  const trap = new FocusTrap("#modal");
  trap.activate();

  // Store trap for cleanup
  (modal as any)._focusTrap = trap;
}

// Close modal
function closeModal() {
  const modal = document.getElementById("modal")!;
  const trap = (modal as any)._focusTrap;

  // Release focus trap
  if (trap) {
    trap.deactivate();
  }

  // Hide modal
  modal.hidden = true;
  modal.setAttribute("aria-hidden", "true");

  // Restore focus
  focusManager.restoreFocus();
}
```

```html
<!-- Modal with Focus Trap -->
<button onclick="openModal()">Open Modal</button>

<div
  id="modal"
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  hidden
  style="position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); background: white; padding: 2rem; border-radius: 0.5rem; box-shadow: 0 10px 30px rgba(0,0,0,0.3);"
>
  <h2 id="modal-title">Modal Title</h2>
  <p>This is a modal dialog with focus trap.</p>

  <form>
    <label for="modal-input">Name:</label>
    <input type="text" id="modal-input" />

    <button type="submit">Submit</button>
    <button type="button" onclick="closeModal()">Cancel</button>
  </form>

  <button
    onclick="closeModal()"
    aria-label="Close modal"
    style="position: absolute; top: 1rem; right: 1rem;"
  >
    ×
  </button>
</div>

<div
  id="modal-backdrop"
  hidden
  style="position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.5);"
></div>
```

### Example 9: Skip Links

```html
<style>
  /* Skip link hidden by default */
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
    text-decoration: none;
    z-index: 100;
  }

  /* Show skip link when focused */
  .skip-link:focus {
    top: 0;
  }

  /* Focus indicator */
  :focus {
    outline: 2px solid #3b82f6;
    outline-offset: 2px;
  }

  /* Skip to specific sections */
  .skip-links {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1000;
  }

  .skip-links a {
    position: absolute;
    left: -10000px;
    width: 1px;
    height: 1px;
    overflow: hidden;
  }

  .skip-links a:focus {
    position: static;
    width: auto;
    height: auto;
    background: #1e293b;
    color: white;
    padding: 1rem 2rem;
    display: inline-block;
    text-decoration: none;
    font-weight: 500;
  }
</style>

<body>
  <!-- Skip Links at the very top -->
  <div class="skip-links">
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <a href="#main-navigation" class="skip-link">Skip to navigation</a>
    <a href="#search" class="skip-link">Skip to search</a>
    <a href="#footer" class="skip-link">Skip to footer</a>
  </div>

  <header>
    <h1>Website Title</h1>
  </header>

  <nav id="main-navigation" aria-label="Main navigation">
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
      <li><a href="/services">Services</a></li>
      <li><a href="/contact">Contact</a></li>
    </ul>
  </nav>

  <div class="search-container">
    <form role="search" id="search">
      <label for="search-input">Search</label>
      <input type="search" id="search-input" />
      <button type="submit">Search</button>
    </form>
  </div>

  <main id="main-content" tabindex="-1">
    <h2>Main Content</h2>
    <p>Content goes here...</p>
  </main>

  <footer id="footer">
    <p>&copy; 2026 Company Name</p>
  </footer>
</body>

<script>
  // Ensure skip link targets can receive focus
  document.querySelectorAll('[id]').forEach(target => {
    if (!target.hasAttribute('tabindex')) {
      target.setAttribute('tabindex', '-1');
    }
  });

  // Smooth scroll for skip links
  document.querySelectorAll('.skip-link').forEach(link => {
    link.addEventListener('click', (e) => {
      e.preventDefault();
      const target = document.querySelector(link.getAttribute('href')!);
      if (target) {
        target.scrollIntoView({ behavior: 'smooth' });
        (target as HTMLElement).focus();
      }
    });
  });
</script>
```

### Example 10: Comprehensive Accessible Form

```html
<form class="accessible-form" novalidate>
  <fieldset>
    <legend>Personal Information</legend>

    <!-- Text Input with Error -->
    <div class="form-group">
      <label for="fullName">
        Full Name
        <span aria-label="required" class="required">*</span>
      </label>
      <input
        type="text"
        id="fullName"
        name="fullName"
        aria-required="true"
        aria-invalid="false"
        aria-describedby="fullName-hint fullName-error"
        required
      />
      <small id="fullName-hint">First and last name</small>
      <div
        id="fullName-error"
        role="alert"
        aria-live="polite"
        class="error-message"
      ></div>
    </div>

    <!-- Email with Validation -->
    <div class="form-group">
      <label for="email">
        Email Address
        <span aria-label="required" class="required">*</span>
      </label>
      <input
        type="email"
        id="email"
        name="email"
        aria-required="true"
        aria-invalid="false"
        aria-describedby="email-hint email-error"
        autocomplete="email"
        required
      />
      <small id="email-hint">We'll never share your email</small>
      <div
        id="email-error"
        role="alert"
        aria-live="polite"
        class="error-message"
      ></div>
    </div>

    <!-- Radio Group -->
    <fieldset
      role="radiogroup"
      aria-labelledby="gender-legend"
      aria-required="true"
    >
      <legend id="gender-legend">Gender</legend>
      <div>
        <input type="radio" id="gender-male" name="gender" value="male" />
        <label for="gender-male">Male</label>
      </div>
      <div>
        <input type="radio" id="gender-female" name="gender" value="female" />
        <label for="gender-female">Female</label>
      </div>
      <div>
        <input type="radio" id="gender-other" name="gender" value="other" />
        <label for="gender-other">Other</label>
      </div>
      <div>
        <input
          type="radio"
          id="gender-prefer-not"
          name="gender"
          value="prefer-not"
        />
        <label for="gender-prefer-not">Prefer not to say</label>
      </div>
    </fieldset>

    <!-- Checkbox Group -->
    <fieldset aria-labelledby="interests-legend">
      <legend id="interests-legend">Interests (select all that apply)</legend>
      <div>
        <input
          type="checkbox"
          id="interest-tech"
          name="interests"
          value="tech"
        />
        <label for="interest-tech">Technology</label>
      </div>
      <div>
        <input
          type="checkbox"
          id="interest-sports"
          name="interests"
          value="sports"
        />
        <label for="interest-sports">Sports</label>
      </div>
      <div>
        <input
          type="checkbox"
          id="interest-arts"
          name="interests"
          value="arts"
        />
        <label for="interest-arts">Arts</label>
      </div>
    </fieldset>

    <!-- Select Dropdown -->
    <div class="form-group">
      <label for="country">
        Country
        <span aria-label="required" class="required">*</span>
      </label>
      <select
        id="country"
        name="country"
        aria-required="true"
        aria-invalid="false"
        aria-describedby="country-error"
        required
      >
        <option value="">Select a country</option>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
        <option value="au">Australia</option>
      </select>
      <div
        id="country-error"
        role="alert"
        aria-live="polite"
        class="error-message"
      ></div>
    </div>

    <!-- Textarea -->
    <div class="form-group">
      <label for="comments"> Additional Comments </label>
      <textarea
        id="comments"
        name="comments"
        rows="4"
        aria-describedby="comments-hint"
        maxlength="500"
      ></textarea>
      <small id="comments-hint">
        <span id="char-count">0</span>/500 characters
      </small>
    </div>

    <!-- Terms Agreement -->
    <div class="form-group">
      <input
        type="checkbox"
        id="terms"
        name="terms"
        aria-required="true"
        aria-describedby="terms-error"
        required
      />
      <label for="terms">
        I agree to the <a href="/terms">Terms and Conditions</a>
        <span aria-label="required" class="required">*</span>
      </label>
      <div
        id="terms-error"
        role="alert"
        aria-live="polite"
        class="error-message"
      ></div>
    </div>
  </fieldset>

  <!-- Form Actions -->
  <div class="form-actions">
    <button type="submit">Submit</button>
    <button type="reset">Reset</button>
  </div>

  <!-- Form Status -->
  <div
    id="form-status"
    role="status"
    aria-live="polite"
    aria-atomic="true"
    class="sr-only"
  ></div>
</form>
```

---

## Real-World Usage

### E-commerce Product Listing

Accessible product cards with proper ARIA labels, keyboard navigation, and screen reader announcements for filters, sorting, and pagination.

### Complex Data Tables

Use ARIA grid roles, row/column headers, and cell relationships (`aria-rowindex`, `aria-colindex`) for sortable, filterable tables with screen reader support.

### Single Page Applications (SPAs)

Implement route change announcements with ARIA live regions, manage focus when navigating, and update document title for each view.

### Custom Select/Combobox

Build accessible custom dropdowns using `role="combobox"`, `aria-expanded`, `aria-controls`, and proper keyboard navigation (Arrow keys, Enter, Escape).

---

## Production Patterns

### ARIA Pattern Library

```typescript
// Reusable accessible components
class AccessibleModal {
  // Focus trap, ARIA attributes, keyboard navigation
}

class AccessibleTooltip {
  // aria-describedby, focus management
}

class AccessibleTabs {
  // role="tablist", keyboard arrows, aria-selected
}
```

### Automated Accessibility Testing

```typescript
// Integration with axe-core or similar
import { axe } from "axe-core";

async function runA11yTests() {
  const results = await axe.run();
  if (results.violations.length > 0) {
    console.error("Accessibility violations:", results.violations);
  }
}
```

---

## Best Practices

### 1. **Use Semantic HTML First**

Native HTML elements have built-in accessibility. Only use ARIA when HTML is insufficient.

### 2. **Don't Override Native Semantics**

Never change native roles: `<button role="heading">` is wrong. Use correct HTML elements.

### 3. **All Interactive Elements Must Be Keyboard Accessible**

Ensure `tabindex`, Enter/Space activation, and logical tab order for all custom widgets.

### 4. **Provide Text Alternatives**

Every image, icon, and non-text content needs `alt` text or `aria-label`.

### 5. **Test with Actual Screen Readers**

NVDA (Windows), JAWS (Windows), VoiceOver (Mac/iOS), TalkBack (Android).

### 6. **Don't Rely on Color Alone**

Use patterns, icons, or text in addition to color for status/meaning (WCAG 1.4.1).

### 7. **Ensure Sufficient Color Contrast**

WCAG AA requires 4.5:1 for normal text, 3:1 for large text. Use contrast checking tools.

### 8. **Make Focus Indicators Visible**

Never remove focus outlines without providing alternative visible focus indicators.

### 9. **Keep ARIA Markup Updated**

Dynamic content changes must update ARIA states (`aria-expanded`, `aria-invalid`, etc.).

### 10. **Document Keyboard Shortcuts**

Provide visible documentation of keyboard navigation patterns for complex widgets.

---

## 10 Key Takeaways

1. **Semantic HTML Beats ARIA Every Time**: Native HTML elements like `<button>`, `<nav>`, `<main>` have built-in accessibility - only add ARIA when HTML semantics are insufficient.

2. **ARIA Roles Define Purpose**: Roles (`button`, `navigation`, `dialog`, `alert`) tell assistive technologies what an element is, especially for custom widgets without native HTML equivalents.

3. **Three Label Types Serve Different Needs**: `aria-label` provides direct labels, `aria-labelledby` references visible text, `aria-describedby` adds supplementary descriptions.

4. **Live Regions Announce Dynamic Changes**: Use `aria-live="polite"` for non-critical updates and `aria-live="assertive"` for urgent messages that should interrupt screen readers.

5. **aria-expanded and aria-controls Work Together**: Collapsible content requires `aria-expanded` on the trigger and `aria-controls` pointing to the panel ID for screen reader comprehension.

6. **aria-current Indicates Active State**: Use `aria-current="page"` for navigation, `aria-current="step"` for processes, and other values to mark the current item in sets.

7. **Focus Management Is Critical for Modals**: Implement focus traps with Tab/Shift+Tab cycling, save/restore previous focus, and close on Escape for accessible dialogs.

8. **Skip Links Improve Keyboard Navigation**: Hidden links that appear on focus let keyboard users jump past repetitive content directly to main content, navigation, or search.

9. **aria-hidden Removes Elements from Accessibility Tree**: Use `aria-hidden="true"` for decorative icons and duplicate content, but never hide interactive or meaningful elements.

10. **Test with Real Assistive Technologies**: Automated tools catch only 30-50% of accessibility issues - manual testing with screen readers (NVDA, JAWS, VoiceOver) is mandatory for production.
