# Keyboard Navigation - Full Keyboard Accessibility

## Table of Contents

1. [Introduction to Keyboard Navigation](#introduction-to-keyboard-navigation)
2. [Tab Order and Focus Management](#tab-order-and-focus-management)
3. [Focus Indicators](#focus-indicators)
4. [Focus Trap](#focus-trap)
5. [Roving Tabindex](#roving-tabindex)
6. [Keyboard Shortcuts](#keyboard-shortcuts)
7. [Skip Links](#skip-links)
8. [Common Patterns](#common-patterns)
9. [Testing Strategies](#testing-strategies)
10. [Key Takeaways](#key-takeaways)

---

## Introduction to Keyboard Navigation

**Keyboard accessibility** is essential for users who:

- Cannot use a mouse (motor disabilities)
- Use screen readers
- Prefer keyboard navigation for efficiency
- Use alternative input devices (switch controls, mouth sticks)

### WCAG Requirements

- **2.1.1 Keyboard (Level A)**: All functionality must be available via keyboard
- **2.1.2 No Keyboard Trap (Level A)**: Focus can move away from any component
- **2.4.7 Focus Visible (Level AA)**: Keyboard focus indicator is visible
- **2.4.3 Focus Order (Level A)**: Focus order is logical and intuitive

### Standard Keyboard Interactions

```typescript
interface KeyboardInteractions {
  Tab: "Move focus forward";
  "Shift+Tab": "Move focus backward";
  Enter: "Activate buttons, links, form submissions";
  Space: "Activate buttons, checkboxes, toggle states";
  "Arrow Keys": "Navigate within composite widgets";
  Escape: "Close dialogs, cancel operations";
  Home: "Move to first item";
  End: "Move to last item";
  "Page Up": "Scroll up or previous section";
  "Page Down": "Scroll down or next section";
}
```

---

## Tab Order and Focus Management

### Natural Tab Order

HTML elements are naturally focusable in DOM order:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Tab Order Example</title>
  </head>
  <body>
    <!-- Focus order: 1, 2, 3, 4, 5, 6 -->
    <header>
      <a href="/">Home</a>
      <!-- 1 -->
      <a href="/about">About</a>
      <!-- 2 -->
    </header>

    <main>
      <input type="text" placeholder="Search" />
      <!-- 3 -->
      <button>Search</button>
      <!-- 4 -->

      <article>
        <a href="/article">Read more</a>
        <!-- 5 -->
      </article>
    </main>

    <footer>
      <button>Subscribe</button>
      <!-- 6 -->
    </footer>
  </body>
</html>
```

### Tabindex Attribute

```typescript
type TabIndexValue =
  | -1 // Programmatically focusable, not in tab order
  | 0 // Focusable and in natural tab order
  | number; // Positive values (antipattern - avoid!)

interface TabIndexUsage {
  "-1": "Manage focus programmatically, exclude from tab order";
  "0": "Make non-interactive elements focusable and keyboard accessible";
  positive: "AVOID - disrupts natural tab order and confuses users";
}
```

### Good Tabindex Examples

```html
<!-- ✅ Good: Make custom widget focusable -->
<div role="button" tabindex="0" onclick="handleClick()">Custom Button</div>

<!-- ✅ Good: Programmatic focus management -->
<div tabindex="-1" id="alert-container">Alert message appears here</div>

<!-- ✅ Good: Skip link -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<!-- ❌ Bad: Positive tabindex -->
<button tabindex="1">First</button>
<!-- Disrupts natural order -->
<input tabindex="2" />
<!-- Don't do this -->
```

### TypeScript Focus Manager

```typescript
class FocusManager {
  private focusableSelectors = [
    "a[href]",
    "button:not([disabled])",
    "input:not([disabled])",
    "select:not([disabled])",
    "textarea:not([disabled])",
    '[tabindex]:not([tabindex="-1"])',
    "audio[controls]",
    "video[controls]",
    '[contenteditable]:not([contenteditable="false"])',
  ].join(", ");

  getFocusableElements(container: HTMLElement = document.body): HTMLElement[] {
    return Array.from(
      container.querySelectorAll(this.focusableSelectors),
    ).filter((el) => this.isVisible(el)) as HTMLElement[];
  }

  private isVisible(element: Element): boolean {
    const htmlElement = element as HTMLElement;
    return (
      !!(
        htmlElement.offsetWidth ||
        htmlElement.offsetHeight ||
        htmlElement.getClientRects().length
      ) && window.getComputedStyle(htmlElement).visibility !== "hidden"
    );
  }

  getFirstFocusable(container: HTMLElement): HTMLElement | null {
    const elements = this.getFocusableElements(container);
    return elements.length > 0 ? elements[0] : null;
  }

  getLastFocusable(container: HTMLElement): HTMLElement | null {
    const elements = this.getFocusableElements(container);
    return elements.length > 0 ? elements[elements.length - 1] : null;
  }

  moveFocusToElement(element: HTMLElement | null): void {
    if (element) {
      // Make programmatically focusable if needed
      if (!element.hasAttribute("tabindex")) {
        element.setAttribute("tabindex", "-1");
      }
      element.focus();

      // Scroll into view if needed
      element.scrollIntoView({ block: "nearest", behavior: "smooth" });
    }
  }

  getFocusableIndex(container: HTMLElement, element: HTMLElement): number {
    const elements = this.getFocusableElements(container);
    return elements.indexOf(element);
  }

  moveFocusNext(container: HTMLElement): void {
    const elements = this.getFocusableElements(container);
    const currentIndex = elements.indexOf(
      document.activeElement as HTMLElement,
    );
    const nextIndex = (currentIndex + 1) % elements.length;
    this.moveFocusToElement(elements[nextIndex]);
  }

  moveFocusPrevious(container: HTMLElement): void {
    const elements = this.getFocusableElements(container);
    const currentIndex = elements.indexOf(
      document.activeElement as HTMLElement,
    );
    const prevIndex =
      currentIndex <= 0 ? elements.length - 1 : currentIndex - 1;
    this.moveFocusToElement(elements[prevIndex]);
  }
}

// Usage
const focusManager = new FocusManager();

// Get all focusable elements in main content
const mainContent = document.querySelector("main")!;
const focusableElements = focusManager.getFocusableElements(mainContent);
console.log("Focusable elements:", focusableElements);

// Move focus to first element
focusManager.moveFocusToElement(focusManager.getFirstFocusable(mainContent));
```

---

## Focus Indicators

### Default Focus Styles

```css
/* ❌ Bad: Removing focus indicator */
*:focus {
  outline: none; /* Never do this without replacement! */
}

/* ✅ Good: Custom focus indicator */
*:focus {
  outline: 2px solid #4a90e2;
  outline-offset: 2px;
}

/* ✅ Better: Different styles for mouse vs keyboard */
*:focus {
  outline: none;
}

*:focus-visible {
  outline: 3px solid #4a90e2;
  outline-offset: 2px;
  border-radius: 4px;
}
```

### Comprehensive Focus Styles

```css
/* Base focus styles */
:focus {
  outline: 2px solid transparent;
  box-shadow:
    0 0 0 2px #ffffff,
    0 0 0 4px #4a90e2;
}

/* High contrast mode support */
@media (prefers-contrast: high) {
  :focus {
    outline: 3px solid currentColor;
    outline-offset: 3px;
  }
}

/* Reduced motion support */
@media (prefers-reduced-motion: reduce) {
  :focus {
    transition: none;
  }
}

/* Focus within (parent highlighting) */
.form-group:focus-within {
  background-color: #f0f8ff;
  border-color: #4a90e2;
}

/* Button focus styles */
button:focus-visible,
a:focus-visible {
  outline: 3px solid #4a90e2;
  outline-offset: 2px;
}

/* Input focus styles */
input:focus,
textarea:focus,
select:focus {
  border-color: #4a90e2;
  box-shadow: 0 0 0 3px rgba(74, 144, 226, 0.2);
}

/* Skip link focus */
.skip-link:focus {
  clip: auto;
  height: auto;
  width: auto;
  position: static;
  margin: 0;
}
```

### TypeScript Focus Indicator Manager

```typescript
class FocusIndicatorManager {
  private usingMouse: boolean = false;

  constructor() {
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    // Detect mouse usage
    document.addEventListener("mousedown", () => {
      this.usingMouse = true;
      document.body.classList.add("using-mouse");
      document.body.classList.remove("using-keyboard");
    });

    // Detect keyboard usage
    document.addEventListener("keydown", (e) => {
      if (e.key === "Tab") {
        this.usingMouse = false;
        document.body.classList.add("using-keyboard");
        document.body.classList.remove("using-mouse");
      }
    });

    // Enhanced focus tracking
    document.addEventListener(
      "focus",
      (e) => {
        this.handleFocus(e.target as HTMLElement);
      },
      true,
    );

    document.addEventListener(
      "blur",
      (e) => {
        this.handleBlur(e.target as HTMLElement);
      },
      true,
    );
  }

  private handleFocus(element: HTMLElement): void {
    // Add focus class to parent containers
    let parent = element.parentElement;
    while (parent) {
      parent.classList.add("has-focus-within");
      parent = parent.parentElement;
    }

    // Announce focus change to screen readers
    this.announceFocusChange(element);
  }

  private handleBlur(element: HTMLElement): void {
    // Remove focus class from parent containers
    let parent = element.parentElement;
    while (parent) {
      parent.classList.remove("has-focus-within");
      parent = parent.parentElement;
    }
  }

  private announceFocusChange(element: HTMLElement): void {
    const label = this.getAccessibleLabel(element);
    if (label) {
      const announcement = `Focused on ${label}`;
      // Use live region to announce (non-intrusive)
      this.announce(announcement, "polite");
    }
  }

  private getAccessibleLabel(element: HTMLElement): string {
    return (
      element.getAttribute("aria-label") ||
      element.getAttribute("title") ||
      (element as HTMLInputElement).placeholder ||
      element.textContent?.trim() ||
      ""
    );
  }

  private announce(message: string, politeness: "polite" | "assertive"): void {
    const liveRegion =
      document.getElementById("focus-announcer") || this.createLiveRegion();
    liveRegion.setAttribute("aria-live", politeness);
    liveRegion.textContent = message;

    setTimeout(() => {
      liveRegion.textContent = "";
    }, 1000);
  }

  private createLiveRegion(): HTMLElement {
    const region = document.createElement("div");
    region.id = "focus-announcer";
    region.className = "sr-only";
    region.setAttribute("aria-live", "polite");
    region.setAttribute("aria-atomic", "true");
    document.body.appendChild(region);
    return region;
  }
}

// Initialize
const focusIndicatorManager = new FocusIndicatorManager();
```

---

## Focus Trap

Focus trap confines keyboard navigation within a specific container (essential for modal dialogs).

### Basic Focus Trap Implementation

```typescript
class FocusTrap {
  private container: HTMLElement;
  private focusableElements: HTMLElement[] = [];
  private firstFocusable: HTMLElement | null = null;
  private lastFocusable: HTMLElement | null = null;
  private previouslyFocused: HTMLElement | null = null;

  constructor(container: HTMLElement) {
    this.container = container;
  }

  activate(): void {
    // Store currently focused element
    this.previouslyFocused = document.activeElement as HTMLElement;

    // Get focusable elements
    this.updateFocusableElements();

    // Focus first element
    if (this.firstFocusable) {
      this.firstFocusable.focus();
    }

    // Add event listeners
    this.container.addEventListener("keydown", this.handleKeyDown);
    document.addEventListener("focus", this.handleFocus, true);
  }

  deactivate(): void {
    // Remove event listeners
    this.container.removeEventListener("keydown", this.handleKeyDown);
    document.removeEventListener("focus", this.handleFocus, true);

    // Restore previous focus
    if (this.previouslyFocused) {
      this.previouslyFocused.focus();
    }
  }

  private updateFocusableElements(): void {
    const selector = [
      "a[href]",
      "button:not([disabled])",
      "input:not([disabled])",
      "select:not([disabled])",
      "textarea:not([disabled])",
      '[tabindex]:not([tabindex="-1"])',
    ].join(", ");

    this.focusableElements = Array.from(
      this.container.querySelectorAll(selector),
    );

    this.firstFocusable = this.focusableElements[0] || null;
    this.lastFocusable =
      this.focusableElements[this.focusableElements.length - 1] || null;
  }

  private handleKeyDown = (event: KeyboardEvent): void => {
    if (event.key === "Tab") {
      if (event.shiftKey) {
        // Shift + Tab
        if (document.activeElement === this.firstFocusable) {
          event.preventDefault();
          this.lastFocusable?.focus();
        }
      } else {
        // Tab
        if (document.activeElement === this.lastFocusable) {
          event.preventDefault();
          this.firstFocusable?.focus();
        }
      }
    }
  };

  private handleFocus = (event: FocusEvent): void => {
    const target = event.target as HTMLElement;

    // If focus moves outside container, bring it back
    if (!this.container.contains(target)) {
      event.stopPropagation();
      this.firstFocusable?.focus();
    }
  };
}

// Usage in Modal
class Modal {
  private modal: HTMLElement;
  private focusTrap: FocusTrap;

  constructor(modalId: string) {
    this.modal = document.getElementById(modalId)!;
    this.focusTrap = new FocusTrap(this.modal);
  }

  open(): void {
    this.modal.hidden = false;
    this.modal.setAttribute("aria-modal", "true");
    document.body.style.overflow = "hidden";
    this.focusTrap.activate();
  }

  close(): void {
    this.modal.hidden = true;
    document.body.style.overflow = "";
    this.focusTrap.deactivate();
  }
}
```

### Advanced Focus Trap with Inert

```typescript
class AdvancedFocusTrap {
  private container: HTMLElement;
  private focusTrap: FocusTrap;
  private inertElements: HTMLElement[] = [];

  constructor(container: HTMLElement) {
    this.container = container;
    this.focusTrap = new FocusTrap(container);
  }

  activate(): void {
    // Make everything outside container inert
    this.setInertOutsideContainer();

    // Activate focus trap
    this.focusTrap.activate();
  }

  deactivate(): void {
    // Restore inert elements
    this.restoreInertElements();

    // Deactivate focus trap
    this.focusTrap.deactivate();
  }

  private setInertOutsideContainer(): void {
    const siblings = Array.from(document.body.children) as HTMLElement[];

    siblings.forEach((sibling) => {
      if (sibling !== this.container && !this.container.contains(sibling)) {
        if (!sibling.hasAttribute("inert")) {
          sibling.setAttribute("inert", "");
          sibling.setAttribute("aria-hidden", "true");
          this.inertElements.push(sibling);
        }
      }
    });
  }

  private restoreInertElements(): void {
    this.inertElements.forEach((element) => {
      element.removeAttribute("inert");
      element.removeAttribute("aria-hidden");
    });
    this.inertElements = [];
  }
}
```

---

## Roving Tabindex

Roving tabindex is a pattern where only one element in a group is in the tab order at a time.

### Concept

```typescript
interface RovingTabindexPattern {
  concept: 'Only one item in a group has tabindex="0"';
  benefit: "Users can Tab to group, then use arrows to navigate within";
  example: "Toolbar buttons, Radio groups, Tab lists, Tree views";
}
```

### Radio Group Implementation

```html
<div role="radiogroup" aria-labelledby="size-label">
  <span id="size-label">T-Shirt Size:</span>

  <div role="radio" aria-checked="true" tabindex="0" data-value="small">
    Small
  </div>

  <div role="radio" aria-checked="false" tabindex="-1" data-value="medium">
    Medium
  </div>

  <div role="radio" aria-checked="false" tabindex="-1" data-value="large">
    Large
  </div>
</div>
```

```typescript
class RadioGroup {
  private container: HTMLElement;
  private radios: HTMLElement[];
  private selectedIndex: number = 0;

  constructor(container: HTMLElement) {
    this.container = container;
    this.radios = Array.from(container.querySelectorAll('[role="radio"]'));

    this.setupEventListeners();
    this.setSelectedIndex(0);
  }

  private setupEventListeners(): void {
    this.radios.forEach((radio, index) => {
      radio.addEventListener("click", () => this.selectRadio(index));
      radio.addEventListener("keydown", (e) => this.handleKeyDown(e, index));
    });
  }

  private selectRadio(index: number): void {
    // Deselect current
    this.radios[this.selectedIndex].setAttribute("aria-checked", "false");
    this.radios[this.selectedIndex].setAttribute("tabindex", "-1");

    // Select new
    this.selectedIndex = index;
    this.radios[index].setAttribute("aria-checked", "true");
    this.radios[index].setAttribute("tabindex", "0");
    this.radios[index].focus();

    // Emit change event
    this.emitChange();
  }

  private handleKeyDown(event: KeyboardEvent, currentIndex: number): void {
    let newIndex: number | null = null;

    switch (event.key) {
      case "ArrowDown":
      case "ArrowRight":
        newIndex = (currentIndex + 1) % this.radios.length;
        break;
      case "ArrowUp":
      case "ArrowLeft":
        newIndex =
          currentIndex === 0 ? this.radios.length - 1 : currentIndex - 1;
        break;
      case "Home":
        newIndex = 0;
        break;
      case "End":
        newIndex = this.radios.length - 1;
        break;
      case " ":
        event.preventDefault();
        this.selectRadio(currentIndex);
        return;
    }

    if (newIndex !== null) {
      event.preventDefault();
      this.selectRadio(newIndex);
    }
  }

  private setSelectedIndex(index: number): void {
    this.radios.forEach((radio, i) => {
      radio.setAttribute("aria-checked", String(i === index));
      radio.setAttribute("tabindex", i === index ? "0" : "-1");
    });
    this.selectedIndex = index;
  }

  private emitChange(): void {
    const value = this.radios[this.selectedIndex].getAttribute("data-value");
    const event = new CustomEvent("radiogroup-change", {
      detail: { value, index: this.selectedIndex },
    });
    this.container.dispatchEvent(event);
  }
}
```

### Toolbar Implementation

```html
<div role="toolbar" aria-label="Text formatting">
  <button tabindex="0" aria-label="Bold" data-action="bold">
    <strong>B</strong>
  </button>
  <button tabindex="-1" aria-label="Italic" data-action="italic">
    <em>I</em>
  </button>
  <button tabindex="-1" aria-label="Underline" data-action="underline">
    <u>U</u>
  </button>

  <div role="separator" aria-orientation="vertical"></div>

  <button tabindex="-1" aria-label="Align left" data-action="align-left">
    Left
  </button>
  <button tabindex="-1" aria-label="Align center" data-action="align-center">
    Center
  </button>
  <button tabindex="-1" aria-label="Align right" data-action="align-right">
    Right
  </button>
</div>
```

```typescript
class Toolbar {
  private toolbar: HTMLElement;
  private buttons: HTMLElement[];
  private focusedIndex: number = 0;

  constructor(toolbarId: string) {
    this.toolbar = document.getElementById(toolbarId)!;
    this.buttons = Array.from(this.toolbar.querySelectorAll("button"));

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.buttons.forEach((button, index) => {
      button.addEventListener("click", () => this.handleAction(button));
      button.addEventListener("keydown", (e) => this.handleKeyDown(e, index));
      button.addEventListener("focus", () => this.updateFocusedIndex(index));
    });
  }

  private handleAction(button: HTMLElement): void {
    const action = button.getAttribute("data-action");
    console.log(`Action: ${action}`);

    // Toggle pressed state for toggle buttons
    const pressed = button.getAttribute("aria-pressed");
    if (pressed !== null) {
      button.setAttribute("aria-pressed", String(pressed !== "true"));
    }
  }

  private handleKeyDown(event: KeyboardEvent, currentIndex: number): void {
    let newIndex: number | null = null;

    switch (event.key) {
      case "ArrowRight":
        newIndex = (currentIndex + 1) % this.buttons.length;
        break;
      case "ArrowLeft":
        newIndex =
          currentIndex === 0 ? this.buttons.length - 1 : currentIndex - 1;
        break;
      case "Home":
        newIndex = 0;
        break;
      case "End":
        newIndex = this.buttons.length - 1;
        break;
    }

    if (newIndex !== null) {
      event.preventDefault();
      this.moveFocus(newIndex);
    }
  }

  private moveFocus(index: number): void {
    // Update tabindex
    this.buttons[this.focusedIndex].setAttribute("tabindex", "-1");
    this.buttons[index].setAttribute("tabindex", "0");

    // Move focus
    this.buttons[index].focus();

    // Update tracked index
    this.focusedIndex = index;
  }

  private updateFocusedIndex(index: number): void {
    this.focusedIndex = index;
  }
}
```

---

## Keyboard Shortcuts

### Implementing Custom Shortcuts

```typescript
class KeyboardShortcutManager {
  private shortcuts: Map<string, () => void> = new Map();
  private enabled: boolean = true;

  constructor() {
    this.setupEventListeners();
  }

  register(keys: string, callback: () => void, description?: string): void {
    this.shortcuts.set(this.normalizeKeys(keys), callback);
  }

  unregister(keys: string): void {
    this.shortcuts.delete(this.normalizeKeys(keys));
  }

  private normalizeKeys(keys: string): string {
    return keys.toLowerCase().replace(/\s+/g, "").split("+").sort().join("+");
  }

  private setupEventListeners(): void {
    document.addEventListener("keydown", (e) => {
      if (!this.enabled) return;

      // Don't trigger shortcuts when typing in inputs
      const target = e.target as HTMLElement;
      if (target.matches("input, textarea, select, [contenteditable]")) {
        return;
      }

      const keys = this.getKeyCombination(e);
      const callback = this.shortcuts.get(keys);

      if (callback) {
        e.preventDefault();
        callback();
      }
    });
  }

  private getKeyCombination(event: KeyboardEvent): string {
    const parts: string[] = [];

    if (event.ctrlKey) parts.push("ctrl");
    if (event.altKey) parts.push("alt");
    if (event.shiftKey) parts.push("shift");
    if (event.metaKey) parts.push("meta");

    // Add main key
    const key = event.key.toLowerCase();
    if (!["control", "alt", "shift", "meta"].includes(key)) {
      parts.push(key);
    }

    return parts.sort().join("+");
  }

  enable(): void {
    this.enabled = true;
  }

  disable(): void {
    this.enabled = false;
  }

  getShortcuts(): Map<string, () => void> {
    return new Map(this.shortcuts);
  }
}

// Usage
const shortcuts = new KeyboardShortcutManager();

// Register shortcuts
shortcuts.register("ctrl+s", () => {
  console.log("Save triggered");
  // Save logic here
});

shortcuts.register("ctrl+shift+f", () => {
  console.log("Search triggered");
  // Open search modal
});

shortcuts.register("?", () => {
  // Show keyboard shortcuts help
  console.log("Show shortcuts");
});
```

### Shortcut Help Modal

```html
<div
  role="dialog"
  aria-labelledby="shortcuts-title"
  aria-modal="true"
  id="shortcuts-modal"
  hidden
>
  <h2 id="shortcuts-title">Keyboard Shortcuts</h2>

  <table>
    <thead>
      <tr>
        <th>Action</th>
        <th>Shortcut</th>
      </tr>
    </thead>
    <tbody id="shortcuts-list">
      <!-- Dynamically populated -->
    </tbody>
  </table>

  <button onclick="closeShortcutsModal()">Close</button>
</div>
```

```typescript
interface ShortcutInfo {
  keys: string;
  description: string;
  category?: string;
}

class ShortcutHelpModal {
  private modal: HTMLElement;
  private shortcuts: ShortcutInfo[] = [];

  constructor() {
    this.modal = document.getElementById("shortcuts-modal")!;
    this.setupShortcuts();
  }

  private setupShortcuts(): void {
    this.shortcuts = [
      { keys: "Ctrl+S", description: "Save document", category: "File" },
      { keys: "Ctrl+O", description: "Open file", category: "File" },
      { keys: "Ctrl+Z", description: "Undo", category: "Edit" },
      { keys: "Ctrl+Y", description: "Redo", category: "Edit" },
      { keys: "Ctrl+F", description: "Find", category: "Search" },
      {
        keys: "Ctrl+Shift+F",
        description: "Find in files",
        category: "Search",
      },
      { keys: "?", description: "Show keyboard shortcuts", category: "Help" },
    ];
  }

  show(): void {
    this.renderShortcuts();
    this.modal.hidden = false;

    const firstButton = this.modal.querySelector("button");
    firstButton?.focus();
  }

  hide(): void {
    this.modal.hidden = true;
  }

  private renderShortcuts(): void {
    const tbody = this.modal.querySelector("#shortcuts-list")!;
    tbody.innerHTML = "";

    // Group by category
    const grouped = this.groupByCategory();

    Object.entries(grouped).forEach(([category, shortcuts]) => {
      // Add category header
      const headerRow = document.createElement("tr");
      headerRow.innerHTML = `
        <td colspan="2"><strong>${category}</strong></td>
      `;
      tbody.appendChild(headerRow);

      // Add shortcuts
      shortcuts.forEach((shortcut) => {
        const row = document.createElement("tr");
        row.innerHTML = `
          <td>${shortcut.description}</td>
          <td><kbd>${shortcut.keys}</kbd></td>
        `;
        tbody.appendChild(row);
      });
    });
  }

  private groupByCategory(): Record<string, ShortcutInfo[]> {
    return this.shortcuts.reduce(
      (acc, shortcut) => {
        const category = shortcut.category || "General";
        if (!acc[category]) {
          acc[category] = [];
        }
        acc[category].push(shortcut);
        return acc;
      },
      {} as Record<string, ShortcutInfo[]>,
    );
  }
}
```

---

## Skip Links

Skip links allow keyboard users to bypass repetitive content.

### Implementation

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Skip Links Example</title>
    <style>
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

      .skip-link:focus {
        top: 0;
      }
    </style>
  </head>
  <body>
    <!-- Skip links -->
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <a href="#navigation" class="skip-link">Skip to navigation</a>
    <a href="#search" class="skip-link">Skip to search</a>

    <header>
      <nav id="navigation">
        <!-- Navigation menu -->
      </nav>
    </header>

    <main id="main-content" tabindex="-1">
      <!-- Main content -->
    </main>

    <aside id="search">
      <!-- Search functionality -->
    </aside>
  </body>
</html>
```

```typescript
class SkipLinkManager {
  private skipLinks: HTMLElement[];

  constructor() {
    this.skipLinks = Array.from(document.querySelectorAll(".skip-link"));
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.skipLinks.forEach((link) => {
      link.addEventListener("click", (e) => {
        e.preventDefault();
        const targetId = link.getAttribute("href")?.substring(1);
        if (targetId) {
          this.skipToTarget(targetId);
        }
      });
    });
  }

  private skipToTarget(targetId: string): void {
    const target = document.getElementById(targetId);

    if (target) {
      // Make focusable if needed
      if (!target.hasAttribute("tabindex")) {
        target.setAttribute("tabindex", "-1");
      }

      // Focus and scroll
      target.focus();
      target.scrollIntoView({ behavior: "smooth" });

      // Announce to screen readers
      const announcement = `Skipped to ${targetId.replace(/-/g, " ")}`;
      this.announce(announcement);
    }
  }

  private announce(message: string): void {
    const liveRegion = document.createElement("div");
    liveRegion.setAttribute("role", "status");
    liveRegion.setAttribute("aria-live", "polite");
    liveRegion.className = "sr-only";
    liveRegion.textContent = message;

    document.body.appendChild(liveRegion);

    setTimeout(() => {
      liveRegion.remove();
    }, 1000);
  }
}

// Initialize
const skipLinkManager = new SkipLinkManager();
```

---

## Common Patterns

### Accessible Accordion

```typescript
class AccessibleAccordion {
  private accordion: HTMLElement;
  private headers: HTMLElement[];
  private panels: HTMLElement[];

  constructor(accordionId: string) {
    this.accordion = document.getElementById(accordionId)!;
    this.headers = Array.from(
      this.accordion.querySelectorAll('[role="button"]'),
    );
    this.panels = Array.from(
      this.accordion.querySelectorAll('[role="region"]'),
    );

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.headers.forEach((header, index) => {
      header.addEventListener("click", () => this.togglePanel(index));
      header.addEventListener("keydown", (e) => this.handleKeyDown(e, index));
    });
  }

  private togglePanel(index: number): void {
    const isExpanded =
      this.headers[index].getAttribute("aria-expanded") === "true";
    this.headers[index].setAttribute("aria-expanded", String(!isExpanded));
    this.panels[index].hidden = isExpanded;
  }

  private handleKeyDown(event: KeyboardEvent, index: number): void {
    switch (event.key) {
      case "ArrowDown":
        event.preventDefault();
        const nextIndex = (index + 1) % this.headers.length;
        this.headers[nextIndex].focus();
        break;

      case "ArrowUp":
        event.preventDefault();
        const prevIndex = index === 0 ? this.headers.length - 1 : index - 1;
        this.headers[prevIndex].focus();
        break;

      case "Home":
        event.preventDefault();
        this.headers[0].focus();
        break;

      case "End":
        event.preventDefault();
        this.headers[this.headers.length - 1].focus();
        break;
    }
  }
}
```

### Accessible Dropdown

```typescript
class AccessibleDropdown {
  private trigger: HTMLElement;
  private menu: HTMLElement;
  private items: HTMLElement[];
  private isOpen: boolean = false;
  private selectedIndex: number = -1;

  constructor(dropdownId: string) {
    const container = document.getElementById(dropdownId)!;
    this.trigger = container.querySelector("[aria-haspopup]")!;
    this.menu = container.querySelector('[role="menu"]')!;
    this.items = Array.from(this.menu.querySelectorAll('[role="menuitem"]'));

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.trigger.addEventListener("click", () => this.toggle());
    this.trigger.addEventListener("keydown", (e) =>
      this.handleTriggerKeyDown(e),
    );

    this.items.forEach((item, index) => {
      item.addEventListener("click", () => this.selectItem(index));
      item.addEventListener("keydown", (e) => this.handleMenuKeyDown(e, index));
    });

    document.addEventListener("click", (e) => {
      if (
        !this.trigger.contains(e.target as Node) &&
        !this.menu.contains(e.target as Node)
      ) {
        this.close();
      }
    });
  }

  private toggle(): void {
    this.isOpen ? this.close() : this.open();
  }

  private open(): void {
    this.isOpen = true;
    this.menu.hidden = false;
    this.trigger.setAttribute("aria-expanded", "true");

    // Focus first item
    if (this.items.length > 0) {
      this.items[0].focus();
      this.selectedIndex = 0;
    }
  }

  private close(): void {
    this.isOpen = false;
    this.menu.hidden = true;
    this.trigger.setAttribute("aria-expanded", "false");
    this.trigger.focus();
    this.selectedIndex = -1;
  }

  private selectItem(index: number): void {
    const value = this.items[index].textContent;
    console.log("Selected:", value);
    this.close();
  }

  private handleTriggerKeyDown(event: KeyboardEvent): void {
    switch (event.key) {
      case "Enter":
      case " ":
      case "ArrowDown":
        event.preventDefault();
        this.open();
        break;
    }
  }

  private handleMenuKeyDown(event: KeyboardEvent, index: number): void {
    switch (event.key) {
      case "Escape":
        event.preventDefault();
        this.close();
        break;

      case "ArrowDown":
        event.preventDefault();
        const nextIndex = (index + 1) % this.items.length;
        this.items[nextIndex].focus();
        this.selectedIndex = nextIndex;
        break;

      case "ArrowUp":
        event.preventDefault();
        const prevIndex = index === 0 ? this.items.length - 1 : index - 1;
        this.items[prevIndex].focus();
        this.selectedIndex = prevIndex;
        break;

      case "Home":
        event.preventDefault();
        this.items[0].focus();
        this.selectedIndex = 0;
        break;

      case "End":
        event.preventDefault();
        const lastIndex = this.items.length - 1;
        this.items[lastIndex].focus();
        this.selectedIndex = lastIndex;
        break;

      case "Enter":
      case " ":
        event.preventDefault();
        this.selectItem(index);
        break;
    }
  }
}
```

---

## Testing Strategies

### Manual Keyboard Testing Checklist

```typescript
interface KeyboardTestChecklist {
  canReachAllInteractiveElements: boolean;
  tabOrderIsLogical: boolean;
  focusIndicatorsVisible: boolean;
  noKeyboardTraps: boolean;
  shortcutsWork: boolean;
  arrowNavigationWorks: boolean;
  escapeClosesModals: boolean;
  enterActivatesButtons: boolean;
  spaceTogglesCheckboxes: boolean;
}

class KeyboardTester {
  runTests(): KeyboardTestChecklist {
    return {
      canReachAllInteractiveElements: this.testReachability(),
      tabOrderIsLogical: this.testTabOrder(),
      focusIndicatorsVisible: this.testFocusIndicators(),
      noKeyboardTraps: this.testKeyboardTraps(),
      shortcutsWork: this.testShortcuts(),
      arrowNavigationWorks: this.testArrowNavigation(),
      escapeClosesModals: this.testEscape(),
      enterActivatesButtons: this.testEnter(),
      spaceTogglesCheckboxes: this.testSpace(),
    };
  }

  // Test implementations...
  private testReachability(): boolean {
    // Logic to test if all interactive elements are reachable
    return true;
  }

  private testTabOrder(): boolean {
    // Logic to verify logical tab order
    return true;
  }

  private testFocusIndicators(): boolean {
    // Logic to check focus visibility
    return true;
  }

  private testKeyboardTraps(): boolean {
    // Logic to detect keyboard traps
    return true;
  }

  private testShortcuts(): boolean {
    // Logic to test keyboard shortcuts
    return true;
  }

  private testArrowNavigation(): boolean {
    // Logic to test arrow key navigation
    return true;
  }

  private testEscape(): boolean {
    // Logic to test Escape key functionality
    return true;
  }

  private testEnter(): boolean {
    // Logic to test Enter key activation
    return true;
  }

  private testSpace(): boolean {
    // Logic to test Space key toggling
    return true;
  }
}
```

---

## Key Takeaways

1. **All functionality must be keyboard accessible** - Every interactive element must be reachable and operable using only keyboard (WCAG 2.1.1 Level A).

2. **Maintain logical tab order** - Follow natural DOM order; avoid positive tabindex values as they disrupt navigation flow and confuse users.

3. **Provide visible focus indicators** - Always show clear visual feedback for focused elements (WCAG 2.4.7 Level AA); never remove outlines without replacement.

4. **Implement focus traps for modals** - When dialogs open, trap focus within them and restore focus to trigger element when closed.

5. **Use roving tabindex for grouped controls** - In toolbars, radio groups, and lists, only one item should be in tab order; use arrows to navigate within group.

6. **Add skip links for efficiency** - Provide skip navigation links at page top to let keyboard users bypass repetitive content.

7. **Support standard keyboard patterns** - Follow established conventions: Enter/Space activate, Escape closes, arrows navigate, Home/End jump to start/end.

8. **Handle keyboard shortcuts carefully** - Don't override browser shortcuts; provide help documentation; disable shortcuts in text inputs.

9. **Test with keyboard only** - Disconnect mouse and navigate entire application using only keyboard to identify accessibility gaps.

10. **Combine keyboard support with ARIA** - Keyboard navigation and ARIA go hand-in-hand; interactive ARIA roles require complete keyboard support to be accessible.
