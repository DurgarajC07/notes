# ARIA Roles - Accessible Rich Internet Applications

## Table of Contents

1. [Introduction to ARIA](#introduction-to-aria)
2. [Widget Roles](#widget-roles)
3. [Landmark Roles](#landmark-roles)
4. [Live Regions](#live-regions)
5. [Document Structure Roles](#document-structure-roles)
6. [Abstract Roles](#abstract-roles)
7. [ARIA States and Properties](#aria-states-and-properties)
8. [Best Practices](#best-practices)
9. [Common Patterns](#common-patterns)
10. [Key Takeaways](#key-takeaways)

---

## Introduction to ARIA

**WAI-ARIA (Web Accessibility Initiative - Accessible Rich Internet Applications)** is a specification that defines ways to make web content and applications more accessible to people with disabilities, particularly those using assistive technologies like screen readers.

### The First Rule of ARIA

**"No ARIA is better than bad ARIA"**

Only use ARIA when:

- HTML semantic elements don't provide the needed semantics
- You need to add accessible behaviors to custom components
- You need to provide additional context for assistive technologies

### ARIA Role Categories

```typescript
type ARIARole =
  | WidgetRole
  | LandmarkRole
  | DocumentStructureRole
  | LiveRegionRole
  | WindowRole
  | AbstractRole;

interface ARIAAttributes {
  role: ARIARole;
  "aria-label"?: string;
  "aria-labelledby"?: string;
  "aria-describedby"?: string;
  "aria-hidden"?: boolean;
  "aria-live"?: "off" | "polite" | "assertive";
  "aria-atomic"?: boolean;
  "aria-relevant"?: string;
}
```

---

## Widget Roles

Widget roles define common interactive patterns that users expect to interact with using keyboard and assistive technologies.

### Button Role

```html
<!-- Native button (preferred) -->
<button>Click Me</button>

<!-- Custom button with ARIA -->
<div
  role="button"
  tabindex="0"
  aria-pressed="false"
  onclick="handleClick()"
  onkeydown="handleKeyDown(event)"
>
  Toggle
</div>
```

```typescript
// TypeScript implementation
interface ButtonWidget {
  role: "button";
  tabIndex: number;
  ariaPressed?: boolean;
  ariaExpanded?: boolean;
}

class CustomButton {
  private element: HTMLElement;
  private pressed: boolean = false;

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupButton();
  }

  private setupButton(): void {
    this.element.setAttribute("role", "button");
    this.element.setAttribute("tabindex", "0");
    this.element.setAttribute("aria-pressed", "false");

    this.element.addEventListener("click", () => this.toggle());
    this.element.addEventListener("keydown", (e) => this.handleKeyDown(e));
  }

  private toggle(): void {
    this.pressed = !this.pressed;
    this.element.setAttribute("aria-pressed", String(this.pressed));
    this.element.classList.toggle("pressed", this.pressed);
  }

  private handleKeyDown(event: KeyboardEvent): void {
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      this.toggle();
    }
  }
}
```

### Tab Widget Role

```html
<div class="tabs">
  <div role="tablist" aria-label="Entertainment Options">
    <button
      role="tab"
      aria-selected="true"
      aria-controls="movies-panel"
      id="movies-tab"
      tabindex="0"
    >
      Movies
    </button>
    <button
      role="tab"
      aria-selected="false"
      aria-controls="music-panel"
      id="music-tab"
      tabindex="-1"
    >
      Music
    </button>
    <button
      role="tab"
      aria-selected="false"
      aria-controls="games-panel"
      id="games-tab"
      tabindex="-1"
    >
      Games
    </button>
  </div>

  <div
    role="tabpanel"
    id="movies-panel"
    aria-labelledby="movies-tab"
    tabindex="0"
  >
    <h3>Movies Content</h3>
    <p>Browse our movie collection...</p>
  </div>

  <div
    role="tabpanel"
    id="music-panel"
    aria-labelledby="music-tab"
    hidden
    tabindex="0"
  >
    <h3>Music Content</h3>
    <p>Explore music albums...</p>
  </div>

  <div
    role="tabpanel"
    id="games-panel"
    aria-labelledby="games-tab"
    hidden
    tabindex="0"
  >
    <h3>Games Content</h3>
    <p>Discover games...</p>
  </div>
</div>
```

```typescript
class TabWidget {
  private tablist: HTMLElement;
  private tabs: HTMLElement[];
  private panels: HTMLElement[];
  private selectedIndex: number = 0;

  constructor(container: HTMLElement) {
    this.tablist = container.querySelector('[role="tablist"]')!;
    this.tabs = Array.from(this.tablist.querySelectorAll('[role="tab"]'));
    this.panels = Array.from(container.querySelectorAll('[role="tabpanel"]'));

    this.setupTabs();
  }

  private setupTabs(): void {
    this.tabs.forEach((tab, index) => {
      tab.addEventListener("click", () => this.selectTab(index));
      tab.addEventListener("keydown", (e) => this.handleKeyDown(e, index));
    });
  }

  private selectTab(index: number): void {
    // Deselect current tab
    this.tabs[this.selectedIndex].setAttribute("aria-selected", "false");
    this.tabs[this.selectedIndex].setAttribute("tabindex", "-1");
    this.panels[this.selectedIndex].hidden = true;

    // Select new tab
    this.selectedIndex = index;
    this.tabs[index].setAttribute("aria-selected", "true");
    this.tabs[index].setAttribute("tabindex", "0");
    this.tabs[index].focus();
    this.panels[index].hidden = false;
  }

  private handleKeyDown(event: KeyboardEvent, currentIndex: number): void {
    let newIndex: number | null = null;

    switch (event.key) {
      case "ArrowLeft":
        newIndex = currentIndex === 0 ? this.tabs.length - 1 : currentIndex - 1;
        break;
      case "ArrowRight":
        newIndex = currentIndex === this.tabs.length - 1 ? 0 : currentIndex + 1;
        break;
      case "Home":
        newIndex = 0;
        break;
      case "End":
        newIndex = this.tabs.length - 1;
        break;
    }

    if (newIndex !== null) {
      event.preventDefault();
      this.selectTab(newIndex);
    }
  }
}
```

### Dialog (Modal) Role

```html
<div
  role="dialog"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
  aria-modal="true"
  class="modal"
>
  <h2 id="dialog-title">Delete Confirmation</h2>
  <p id="dialog-desc">Are you sure you want to delete this item?</p>

  <div class="dialog-actions">
    <button onclick="confirmDelete()">Delete</button>
    <button onclick="closeDialog()">Cancel</button>
  </div>
</div>

<div class="modal-backdrop" aria-hidden="true"></div>
```

```typescript
class ModalDialog {
  private dialog: HTMLElement;
  private lastFocusedElement: HTMLElement | null = null;
  private focusableElements: HTMLElement[] = [];

  constructor(dialogId: string) {
    this.dialog = document.getElementById(dialogId)!;
    this.setupDialog();
  }

  open(): void {
    // Store currently focused element
    this.lastFocusedElement = document.activeElement as HTMLElement;

    // Show dialog
    this.dialog.hidden = false;
    document.body.classList.add("modal-open");

    // Get all focusable elements
    this.focusableElements = this.getFocusableElements();

    // Focus first element
    if (this.focusableElements.length > 0) {
      this.focusableElements[0].focus();
    }

    // Add event listeners
    this.dialog.addEventListener("keydown", this.handleKeyDown.bind(this));
    document.addEventListener("focus", this.trapFocus.bind(this), true);
  }

  close(): void {
    this.dialog.hidden = true;
    document.body.classList.remove("modal-open");

    // Remove event listeners
    this.dialog.removeEventListener("keydown", this.handleKeyDown.bind(this));
    document.removeEventListener("focus", this.trapFocus.bind(this), true);

    // Restore focus
    if (this.lastFocusedElement) {
      this.lastFocusedElement.focus();
    }
  }

  private setupDialog(): void {
    this.dialog.setAttribute("role", "dialog");
    this.dialog.setAttribute("aria-modal", "true");
    this.dialog.hidden = true;
  }

  private getFocusableElements(): HTMLElement[] {
    const selector =
      "a[href], button:not([disabled]), textarea:not([disabled]), " +
      'input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])';
    return Array.from(this.dialog.querySelectorAll(selector));
  }

  private trapFocus(event: FocusEvent): void {
    const target = event.target as HTMLElement;

    if (!this.dialog.contains(target)) {
      event.stopPropagation();
      if (this.focusableElements.length > 0) {
        this.focusableElements[0].focus();
      }
    }
  }

  private handleKeyDown(event: KeyboardEvent): void {
    if (event.key === "Escape") {
      this.close();
    }

    if (event.key === "Tab") {
      const firstElement = this.focusableElements[0];
      const lastElement =
        this.focusableElements[this.focusableElements.length - 1];

      if (event.shiftKey && document.activeElement === firstElement) {
        event.preventDefault();
        lastElement.focus();
      } else if (!event.shiftKey && document.activeElement === lastElement) {
        event.preventDefault();
        firstElement.focus();
      }
    }
  }
}
```

### Combobox Role

```html
<div class="combobox-container">
  <label id="combo-label" for="combo-input">Choose a fruit:</label>

  <div
    role="combobox"
    aria-expanded="false"
    aria-owns="listbox"
    aria-haspopup="listbox"
  >
    <input
      type="text"
      id="combo-input"
      aria-autocomplete="list"
      aria-controls="listbox"
      aria-labelledby="combo-label"
      autocomplete="off"
    />
  </div>

  <ul role="listbox" id="listbox" aria-labelledby="combo-label" hidden>
    <li role="option" id="option-1">Apple</li>
    <li role="option" id="option-2">Banana</li>
    <li role="option" id="option-3">Cherry</li>
    <li role="option" id="option-4">Date</li>
    <li role="option" id="option-5">Elderberry</li>
  </ul>
</div>
```

```typescript
class Combobox {
  private combobox: HTMLElement;
  private input: HTMLInputElement;
  private listbox: HTMLElement;
  private options: HTMLElement[];
  private filteredOptions: HTMLElement[] = [];
  private selectedIndex: number = -1;
  private isOpen: boolean = false;

  constructor(containerId: string) {
    const container = document.getElementById(containerId)!;
    this.combobox = container.querySelector('[role="combobox"]')!;
    this.input = container.querySelector("input")!;
    this.listbox = container.querySelector('[role="listbox"]')!;
    this.options = Array.from(this.listbox.querySelectorAll('[role="option"]'));

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.input.addEventListener("input", () => this.filterOptions());
    this.input.addEventListener("keydown", (e) => this.handleKeyDown(e));
    this.input.addEventListener("focus", () => this.open());
    this.input.addEventListener("blur", () => {
      setTimeout(() => this.close(), 200);
    });

    this.options.forEach((option, index) => {
      option.addEventListener("click", () => this.selectOption(index));
      option.addEventListener("mouseenter", () => this.highlightOption(index));
    });
  }

  private filterOptions(): void {
    const query = this.input.value.toLowerCase();

    this.filteredOptions = this.options.filter((option) => {
      const text = option.textContent?.toLowerCase() || "";
      const matches = text.includes(query);
      option.hidden = !matches;
      return matches;
    });

    this.selectedIndex = -1;
    this.updateListbox();
  }

  private open(): void {
    this.isOpen = true;
    this.listbox.hidden = false;
    this.combobox.setAttribute("aria-expanded", "true");
    this.filterOptions();
  }

  private close(): void {
    this.isOpen = false;
    this.listbox.hidden = true;
    this.combobox.setAttribute("aria-expanded", "false");
  }

  private selectOption(index: number): void {
    const option = this.filteredOptions[index];
    if (option) {
      this.input.value = option.textContent || "";
      this.close();
    }
  }

  private highlightOption(index: number): void {
    this.filteredOptions.forEach((option, i) => {
      option.setAttribute("aria-selected", String(i === index));
      option.classList.toggle("highlighted", i === index);
    });
    this.selectedIndex = index;
  }

  private handleKeyDown(event: KeyboardEvent): void {
    switch (event.key) {
      case "ArrowDown":
        event.preventDefault();
        if (!this.isOpen) {
          this.open();
        } else {
          const newIndex = Math.min(
            this.selectedIndex + 1,
            this.filteredOptions.length - 1,
          );
          this.highlightOption(newIndex);
        }
        break;

      case "ArrowUp":
        event.preventDefault();
        if (this.isOpen) {
          const newIndex = Math.max(this.selectedIndex - 1, 0);
          this.highlightOption(newIndex);
        }
        break;

      case "Enter":
        event.preventDefault();
        if (this.selectedIndex >= 0) {
          this.selectOption(this.selectedIndex);
        }
        break;

      case "Escape":
        this.close();
        break;
    }
  }

  private updateListbox(): void {
    this.input.setAttribute(
      "aria-activedescendant",
      this.selectedIndex >= 0
        ? this.filteredOptions[this.selectedIndex].id
        : "",
    );
  }
}
```

---

## Landmark Roles

Landmark roles identify regions of a page, allowing screen reader users to navigate by region.

### Common Landmark Roles

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Landmark Roles Example</title>
  </head>
  <body>
    <!-- Banner (header) -->
    <header role="banner">
      <h1>Website Title</h1>
      <nav role="navigation" aria-label="Main navigation">
        <ul>
          <li><a href="/">Home</a></li>
          <li><a href="/about">About</a></li>
          <li><a href="/contact">Contact</a></li>
        </ul>
      </nav>
    </header>

    <!-- Main content -->
    <main role="main">
      <article role="article">
        <h2>Article Title</h2>
        <p>Article content...</p>
      </article>

      <!-- Complementary content (sidebar) -->
      <aside role="complementary" aria-label="Related information">
        <h3>Related Links</h3>
        <ul>
          <li><a href="#">Link 1</a></li>
          <li><a href="#">Link 2</a></li>
        </ul>
      </aside>
    </main>

    <!-- Search -->
    <div role="search">
      <form>
        <label for="search-input">Search:</label>
        <input type="search" id="search-input" />
        <button type="submit">Search</button>
      </form>
    </div>

    <!-- Footer (contentinfo) -->
    <footer role="contentinfo">
      <p>&copy; 2026 Company Name. All rights reserved.</p>
      <nav aria-label="Footer navigation">
        <a href="/privacy">Privacy Policy</a>
        <a href="/terms">Terms of Service</a>
      </nav>
    </footer>
  </body>
</html>
```

### TypeScript Landmark Navigation Helper

```typescript
interface Landmark {
  role: string;
  label: string;
  element: HTMLElement;
}

class LandmarkNavigator {
  private landmarks: Landmark[] = [];

  constructor() {
    this.discoverLandmarks();
  }

  private discoverLandmarks(): void {
    const landmarkSelectors = [
      '[role="banner"], header',
      '[role="navigation"], nav',
      '[role="main"], main',
      '[role="complementary"], aside',
      '[role="contentinfo"], footer',
      '[role="search"]',
      '[role="form"], form',
      '[role="region"][aria-labelledby], section[aria-labelledby]',
    ];

    landmarkSelectors.forEach((selector) => {
      const elements = document.querySelectorAll(selector);
      elements.forEach((element: Element) => {
        const htmlElement = element as HTMLElement;
        this.landmarks.push({
          role: this.getLandmarkRole(htmlElement),
          label: this.getLandmarkLabel(htmlElement),
          element: htmlElement,
        });
      });
    });
  }

  private getLandmarkRole(element: HTMLElement): string {
    return element.getAttribute("role") || element.tagName.toLowerCase();
  }

  private getLandmarkLabel(element: HTMLElement): string {
    const ariaLabel = element.getAttribute("aria-label");
    if (ariaLabel) return ariaLabel;

    const ariaLabelledby = element.getAttribute("aria-labelledby");
    if (ariaLabelledby) {
      const labelElement = document.getElementById(ariaLabelledby);
      return labelElement?.textContent || "";
    }

    return this.getLandmarkRole(element);
  }

  getLandmarks(): Landmark[] {
    return this.landmarks;
  }

  navigateToLandmark(index: number): void {
    if (index >= 0 && index < this.landmarks.length) {
      this.landmarks[index].element.focus();
      this.landmarks[index].element.scrollIntoView({ behavior: "smooth" });
    }
  }

  createLandmarkMenu(): HTMLElement {
    const menu = document.createElement("nav");
    menu.setAttribute("aria-label", "Page landmarks");

    const list = document.createElement("ul");

    this.landmarks.forEach((landmark, index) => {
      const item = document.createElement("li");
      const link = document.createElement("button");
      link.textContent = `${landmark.role}: ${landmark.label}`;
      link.onclick = () => this.navigateToLandmark(index);
      item.appendChild(link);
      list.appendChild(item);
    });

    menu.appendChild(list);
    return menu;
  }
}
```

---

## Live Regions

Live regions announce dynamic content changes to screen reader users.

### ARIA Live Region Attributes

```typescript
interface LiveRegionAttributes {
  "aria-live": "off" | "polite" | "assertive";
  "aria-atomic"?: boolean;
  "aria-relevant"?: "additions" | "removals" | "text" | "all";
  "aria-busy"?: boolean;
}

type PolitenessLevel = "off" | "polite" | "assertive";
```

### Live Region Examples

```html
<!-- Status messages (polite) -->
<div role="status" aria-live="polite" aria-atomic="true">
  <p id="status-message"></p>
</div>

<!-- Alert messages (assertive) -->
<div role="alert" aria-live="assertive" aria-atomic="true">
  <p id="alert-message"></p>
</div>

<!-- Log messages -->
<div role="log" aria-live="polite" aria-atomic="false">
  <ul id="log-list"></ul>
</div>

<!-- Timer/countdown -->
<div role="timer" aria-live="off" aria-atomic="true">
  <span id="timer-display">00:00</span>
</div>

<!-- Marquee (ticker) -->
<div role="marquee" aria-live="off">
  <p id="ticker-text"></p>
</div>
```

### TypeScript Live Region Manager

```typescript
class LiveRegionManager {
  private container: HTMLElement;

  constructor(containerId: string = "live-region-container") {
    this.container = this.getOrCreateContainer(containerId);
  }

  private getOrCreateContainer(id: string): HTMLElement {
    let container = document.getElementById(id);

    if (!container) {
      container = document.createElement("div");
      container.id = id;
      container.className = "sr-only"; // Visually hidden
      document.body.appendChild(container);
    }

    return container;
  }

  announce(message: string, politeness: PolitenessLevel = "polite"): void {
    const region = this.createLiveRegion(politeness);
    region.textContent = message;

    // Remove after announcement
    setTimeout(() => {
      region.remove();
    }, 1000);
  }

  private createLiveRegion(politeness: PolitenessLevel): HTMLElement {
    const region = document.createElement("div");
    region.setAttribute(
      "role",
      politeness === "assertive" ? "alert" : "status",
    );
    region.setAttribute("aria-live", politeness);
    region.setAttribute("aria-atomic", "true");
    this.container.appendChild(region);
    return region;
  }

  announceStatus(message: string): void {
    this.announce(message, "polite");
  }

  announceAlert(message: string): void {
    this.announce(message, "assertive");
  }
}

// Usage example
const liveRegion = new LiveRegionManager();

// Status update (non-urgent)
liveRegion.announceStatus("Item added to cart");

// Alert (urgent)
liveRegion.announceAlert("Error: Payment failed. Please try again.");
```

### Advanced Live Region Implementation

```typescript
class AdvancedLiveRegion {
  private region: HTMLElement;
  private queue: string[] = [];
  private isAnnouncing: boolean = false;

  constructor(politeness: PolitenessLevel = "polite", atomic: boolean = true) {
    this.region = this.createRegion(politeness, atomic);
  }

  private createRegion(
    politeness: PolitenessLevel,
    atomic: boolean,
  ): HTMLElement {
    const region = document.createElement("div");
    region.className = "sr-only";
    region.setAttribute("aria-live", politeness);
    region.setAttribute("aria-atomic", String(atomic));
    region.setAttribute("aria-relevant", "additions text");
    document.body.appendChild(region);
    return region;
  }

  announce(message: string, delay: number = 0): void {
    if (delay > 0) {
      setTimeout(() => this.addToQueue(message), delay);
    } else {
      this.addToQueue(message);
    }
  }

  private addToQueue(message: string): void {
    this.queue.push(message);
    if (!this.isAnnouncing) {
      this.processQueue();
    }
  }

  private async processQueue(): Promise<void> {
    if (this.queue.length === 0) {
      this.isAnnouncing = false;
      return;
    }

    this.isAnnouncing = true;
    const message = this.queue.shift()!;

    // Set busy state
    this.region.setAttribute("aria-busy", "true");

    // Clear previous content
    this.region.textContent = "";

    // Brief pause for screen readers to register change
    await this.pause(100);

    // Add new content
    this.region.textContent = message;
    this.region.setAttribute("aria-busy", "false");

    // Wait before next announcement
    await this.pause(1000);

    // Process next in queue
    this.processQueue();
  }

  private pause(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  clear(): void {
    this.queue = [];
    this.region.textContent = "";
  }

  destroy(): void {
    this.region.remove();
  }
}

// Usage for form validation
class FormValidator {
  private liveRegion: AdvancedLiveRegion;

  constructor() {
    this.liveRegion = new AdvancedLiveRegion("assertive", true);
  }

  validateField(field: HTMLInputElement): boolean {
    const errors: string[] = [];

    if (field.required && !field.value) {
      errors.push(`${field.name} is required`);
    }

    if (field.type === "email" && field.value) {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!emailRegex.test(field.value)) {
        errors.push(`${field.name} must be a valid email address`);
      }
    }

    if (errors.length > 0) {
      this.liveRegion.announce(errors.join(". "));
      field.setAttribute("aria-invalid", "true");
      field.setAttribute("aria-describedby", `${field.id}-error`);
      return false;
    }

    field.setAttribute("aria-invalid", "false");
    field.removeAttribute("aria-describedby");
    return true;
  }
}
```

---

## Document Structure Roles

Document structure roles describe the structure of content on the page.

### Common Document Roles

```html
<article role="article">
  <header>
    <h2 id="article-title">Understanding ARIA</h2>
    <p role="doc-subtitle">A comprehensive guide</p>
  </header>

  <div role="group" aria-labelledby="section-1">
    <h3 id="section-1">Introduction</h3>
    <p>Content here...</p>
  </div>

  <figure role="figure" aria-labelledby="fig-caption">
    <img src="diagram.png" alt="ARIA role hierarchy" />
    <figcaption id="fig-caption">Figure 1: ARIA Role Hierarchy</figcaption>
  </figure>

  <div role="list">
    <div role="listitem">First item</div>
    <div role="listitem">Second item</div>
    <div role="listitem">Third item</div>
  </div>

  <table role="table" aria-labelledby="table-title">
    <caption id="table-title">
      ARIA Role Categories
    </caption>
    <thead role="rowgroup">
      <tr role="row">
        <th role="columnheader">Category</th>
        <th role="columnheader">Description</th>
      </tr>
    </thead>
    <tbody role="rowgroup">
      <tr role="row">
        <td role="cell">Widget</td>
        <td role="cell">Interactive elements</td>
      </tr>
      <tr role="row">
        <td role="cell">Landmark</td>
        <td role="cell">Page regions</td>
      </tr>
    </tbody>
  </table>

  <div role="separator" aria-orientation="horizontal"></div>

  <footer>
    <p role="contentinfo">
      Published: <time datetime="2026-02-01">February 1, 2026</time>
    </p>
  </footer>
</article>
```

---

## ARIA States and Properties

### Common ARIA Properties

```typescript
interface CommonARIAProperties {
  // Labeling
  "aria-label"?: string;
  "aria-labelledby"?: string;
  "aria-describedby"?: string;

  // States
  "aria-disabled"?: boolean;
  "aria-hidden"?: boolean;
  "aria-invalid"?: boolean | "grammar" | "spelling";
  "aria-selected"?: boolean;
  "aria-checked"?: boolean | "mixed";
  "aria-pressed"?: boolean | "mixed";
  "aria-expanded"?: boolean;
  "aria-current"?: "page" | "step" | "location" | "date" | "time" | boolean;

  // Relationships
  "aria-controls"?: string;
  "aria-owns"?: string;
  "aria-activedescendant"?: string;
  "aria-flowto"?: string;

  // Live regions
  "aria-live"?: "off" | "polite" | "assertive";
  "aria-atomic"?: boolean;
  "aria-relevant"?: string;
  "aria-busy"?: boolean;

  // Values
  "aria-valuemin"?: number;
  "aria-valuemax"?: number;
  "aria-valuenow"?: number;
  "aria-valuetext"?: string;
}
```

### Comprehensive Example

```html
<div class="custom-slider">
  <label id="slider-label">Volume Control</label>

  <div
    role="slider"
    tabindex="0"
    aria-labelledby="slider-label"
    aria-valuemin="0"
    aria-valuemax="100"
    aria-valuenow="50"
    aria-valuetext="50 percent"
    aria-orientation="horizontal"
  >
    <div class="slider-track">
      <div class="slider-thumb" style="left: 50%"></div>
    </div>
  </div>

  <output aria-live="off">50%</output>
</div>
```

```typescript
class Slider {
  private slider: HTMLElement;
  private thumb: HTMLElement;
  private output: HTMLElement;
  private min: number;
  private max: number;
  private value: number;

  constructor(sliderId: string) {
    this.slider = document.getElementById(sliderId)!;
    this.thumb = this.slider.querySelector(".slider-thumb")!;
    this.output = this.slider.parentElement!.querySelector("output")!;

    this.min = Number(this.slider.getAttribute("aria-valuemin")) || 0;
    this.max = Number(this.slider.getAttribute("aria-valuemax")) || 100;
    this.value = Number(this.slider.getAttribute("aria-valuenow")) || 50;

    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.slider.addEventListener("keydown", (e) => this.handleKeyDown(e));
    this.slider.addEventListener("mousedown", (e) => this.handleMouseDown(e));
  }

  private handleKeyDown(event: KeyboardEvent): void {
    let newValue = this.value;

    switch (event.key) {
      case "ArrowLeft":
      case "ArrowDown":
        newValue = Math.max(this.min, this.value - 1);
        break;
      case "ArrowRight":
      case "ArrowUp":
        newValue = Math.min(this.max, this.value + 1);
        break;
      case "PageDown":
        newValue = Math.max(this.min, this.value - 10);
        break;
      case "PageUp":
        newValue = Math.min(this.max, this.value + 10);
        break;
      case "Home":
        newValue = this.min;
        break;
      case "End":
        newValue = this.max;
        break;
      default:
        return;
    }

    event.preventDefault();
    this.setValue(newValue);
  }

  private handleMouseDown(event: MouseEvent): void {
    const rect = this.slider.getBoundingClientRect();
    const percent = (event.clientX - rect.left) / rect.width;
    const newValue = Math.round(this.min + (this.max - this.min) * percent);
    this.setValue(newValue);

    const handleMouseMove = (e: MouseEvent) => {
      const percent = (e.clientX - rect.left) / rect.width;
      const value = Math.round(this.min + (this.max - this.min) * percent);
      this.setValue(Math.max(this.min, Math.min(this.max, value)));
    };

    const handleMouseUp = () => {
      document.removeEventListener("mousemove", handleMouseMove);
      document.removeEventListener("mouseup", handleMouseUp);
    };

    document.addEventListener("mousemove", handleMouseMove);
    document.addEventListener("mouseup", handleMouseUp);
  }

  private setValue(value: number): void {
    this.value = value;
    const percent = ((value - this.min) / (this.max - this.min)) * 100;

    this.slider.setAttribute("aria-valuenow", String(value));
    this.slider.setAttribute("aria-valuetext", `${value} percent`);
    this.thumb.style.left = `${percent}%`;
    this.output.textContent = `${value}%`;
  }
}
```

---

## Best Practices

### 1. Prefer Native HTML Elements

```html
<!-- ❌ Bad: Custom button with ARIA -->
<div role="button" tabindex="0" onclick="submit()">Submit</div>

<!-- ✅ Good: Native button -->
<button onclick="submit()">Submit</button>
```

### 2. Don't Override Native Semantics

```html
<!-- ❌ Bad: Changing button to link -->
<button role="link">Click me</button>

<!-- ✅ Good: Use appropriate element -->
<a href="/page">Click me</a>
```

### 3. Ensure Keyboard Accessibility

```typescript
// ✅ Good: Complete keyboard support
function makeAccessibleButton(element: HTMLElement): void {
  element.setAttribute("role", "button");
  element.setAttribute("tabindex", "0");

  element.addEventListener("click", handleClick);
  element.addEventListener("keydown", (e) => {
    if (e.key === "Enter" || e.key === " ") {
      e.preventDefault();
      handleClick();
    }
  });
}
```

### 4. Provide Meaningful Labels

```html
<!-- ❌ Bad: Generic label -->
<button aria-label="Click">Delete</button>

<!-- ✅ Good: Descriptive label -->
<button aria-label="Delete user account">Delete</button>
```

### 5. Use ARIA States Dynamically

```typescript
class Accordion {
  togglePanel(button: HTMLElement, panel: HTMLElement): void {
    const isExpanded = button.getAttribute("aria-expanded") === "true";

    button.setAttribute("aria-expanded", String(!isExpanded));
    panel.hidden = isExpanded;
  }
}
```

### 6. Test with Assistive Technologies

```typescript
// Testing checklist
interface AccessibilityTest {
  screenReaderTest: boolean; // NVDA, JAWS, VoiceOver
  keyboardTest: boolean; // Tab navigation, shortcuts
  semanticTest: boolean; // Proper roles and labels
  contrastTest: boolean; // Color contrast ratios
}
```

---

## Common Patterns

### Accessible Dropdown Menu

```html
<nav>
  <button
    aria-expanded="false"
    aria-controls="menu-1"
    aria-haspopup="true"
    id="menu-button-1"
  >
    Products
  </button>

  <ul role="menu" id="menu-1" aria-labelledby="menu-button-1" hidden>
    <li role="none">
      <a role="menuitem" href="/product-1">Product 1</a>
    </li>
    <li role="none">
      <a role="menuitem" href="/product-2">Product 2</a>
    </li>
  </ul>
</nav>
```

### Accessible Tree View

```html
<div role="tree" aria-labelledby="tree-label">
  <h3 id="tree-label">File Explorer</h3>

  <ul role="group">
    <li role="treeitem" aria-expanded="false" aria-level="1" tabindex="0">
      <span>Documents</span>
      <ul role="group">
        <li role="treeitem" aria-level="2" tabindex="-1">File1.txt</li>
      </ul>
    </li>
  </ul>
</div>
```

---

## Key Takeaways

1. **No ARIA is better than bad ARIA** - Only use ARIA when native HTML doesn't provide the needed semantics. Incorrect ARIA can make content less accessible than no ARIA at all.

2. **Roles define what something is** - Use appropriate ARIA roles to communicate the purpose of elements to assistive technologies (button, dialog, tablist, etc.).

3. **Landmarks structure your page** - Use landmark roles (banner, navigation, main, complementary, contentinfo) to help users navigate efficiently through different sections.

4. **Live regions announce changes** - Use `aria-live` with appropriate politeness levels (polite/assertive) to announce dynamic content updates to screen reader users.

5. **States and properties provide context** - Use ARIA states (aria-expanded, aria-selected) and properties (aria-label, aria-describedby) to communicate element state and relationships.

6. **Keyboard accessibility is mandatory** - Every interactive ARIA widget must be fully keyboard accessible with appropriate focus management and keyboard shortcuts.

7. **Label everything meaningfully** - Provide clear, descriptive labels using aria-label or aria-labelledby for all interactive elements and regions.

8. **Manage focus appropriately** - When content changes dynamically (modals, tabs), ensure focus moves logically and is trapped when needed (like in modal dialogs).

9. **Test with real assistive technology** - Always test your ARIA implementations with actual screen readers (NVDA, JAWS, VoiceOver) and keyboard-only navigation.

10. **Follow established patterns** - Use well-documented ARIA authoring practices (ARIA Authoring Practices Guide) rather than inventing custom solutions.
