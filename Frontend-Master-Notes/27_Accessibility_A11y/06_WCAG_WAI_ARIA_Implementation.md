# 🎯 WCAG 2.1 & WAI-ARIA Implementation Guide

> **Complete Guide to Implementing WCAG 2.1 Standards with WAI-ARIA for Accessible Web Applications**

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [WCAG 2.1 Overview](#wcag-21-overview)
- [WAI-ARIA Fundamentals](#wai-aria-fundamentals)
- [Implementing WCAG 2.1 Level A](#implementing-wcag-21-level-a)
- [Implementing WCAG 2.1 Level AA](#implementing-wcag-21-level-aa)
- [Implementing WCAG 2.1 Level AAA](#implementing-wcag-21-level-aaa)
- [ARIA Patterns & Widgets](#aria-patterns--widgets)
- [Mobile Accessibility (WCAG 2.1)](#mobile-accessibility-wcag-21)
- [Cognitive Accessibility](#cognitive-accessibility)
- [Testing & Validation](#testing--validation)
- [Common Mistakes](#common-mistakes)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)

---

## Introduction

**WCAG 2.1** (Web Content Accessibility Guidelines 2.1) is the international standard for web accessibility. **WAI-ARIA** (Accessible Rich Internet Applications) provides additional semantics for complex web applications. Together, they form the foundation for creating inclusive digital experiences.

### Why WCAG 2.1 + WAI-ARIA?

```typescript
interface AccessibilityStandards {
  standard: string;
  purpose: string;
  focusAreas: string[];
  compliance: string[];
}

const standards: AccessibilityStandards[] = [
  {
    standard: "WCAG 2.1",
    purpose: "Comprehensive accessibility guidelines",
    focusAreas: [
      "Perceivable content",
      "Operable interfaces",
      "Understandable information",
      "Robust implementation",
    ],
    compliance: ["ADA", "Section 508", "EN 301 549", "ADA"],
  },
  {
    standard: "WAI-ARIA 1.2",
    purpose: "Bridge gap between HTML and complex widgets",
    focusAreas: [
      "Roles for custom components",
      "States and properties",
      "Live regions",
      "Dynamic content updates",
    ],
    compliance: ["Part of WCAG conformance", "Required for complex UIs"],
  },
];
```

### The Relationship

```typescript
/**
 * WCAG provides WHAT needs to be accessible
 * ARIA provides HOW to make complex patterns accessible
 */

// WCAG Requirement (2.1.1): All functionality available from keyboard
// ARIA Implementation: Roles and keyboard patterns

interface WCAGARIARelationship {
  wcagRequirement: string;
  ariaImplementation: string;
  example: string;
}

const relationships: WCAGARIARelationship[] = [
  {
    wcagRequirement: "1.3.1 Info and Relationships",
    ariaImplementation: "role, aria-labelledby, aria-describedby",
    example: "<div role='button' aria-labelledby='label-id'>",
  },
  {
    wcagRequirement: "4.1.2 Name, Role, Value",
    ariaImplementation: "All ARIA roles and properties",
    example: "<button aria-pressed='false'>Toggle</button>",
  },
  {
    wcagRequirement: "4.1.3 Status Messages",
    ariaImplementation: "aria-live, role='status', role='alert'",
    example: "<div role='status' aria-live='polite'>Saved!</div>",
  },
];
```

---

## WCAG 2.1 Overview

### POUR Principles

```typescript
/**
 * WCAG organized around 4 principles
 */
interface PourPrinciple {
  principle: string;
  description: string;
  guidelines: number;
  successCriteria: number;
  keyFocus: string[];
}

const pourPrinciples: PourPrinciple[] = [
  {
    principle: "Perceivable",
    description: "Information must be presentable to users",
    guidelines: 4,
    successCriteria: 22,
    keyFocus: [
      "Text alternatives",
      "Time-based media",
      "Adaptable content",
      "Distinguishable content",
    ],
  },
  {
    principle: "Operable",
    description: "Interface components must be operable",
    guidelines: 5,
    successCriteria: 28,
    keyFocus: [
      "Keyboard accessible",
      "Enough time",
      "Seizures and reactions",
      "Navigable",
      "Input modalities",
    ],
  },
  {
    principle: "Understandable",
    description: "Information must be understandable",
    guidelines: 3,
    successCriteria: 17,
    keyFocus: ["Readable", "Predictable", "Input assistance"],
  },
  {
    principle: "Robust",
    description: "Content must work with assistive technologies",
    guidelines: 1,
    successCriteria: 11,
    keyFocus: ["Compatible with user agents"],
  },
];
```

### WCAG 2.1 New Success Criteria

```typescript
/**
 * WCAG 2.1 adds 17 new success criteria
 * Focus on mobile, low vision, cognitive disabilities
 */
interface WCAG21NewCriteria {
  criterionId: string;
  level: "A" | "AA" | "AAA";
  name: string;
  focusArea: string;
  implementation: string;
}

const wcag21NewCriteria: WCAG21NewCriteria[] = [
  // Level A
  {
    criterionId: "1.3.4",
    level: "AA",
    name: "Orientation",
    focusArea: "Mobile",
    implementation: "No orientation lock",
  },
  {
    criterionId: "1.3.5",
    level: "AA",
    name: "Identify Input Purpose",
    focusArea: "Cognitive",
    implementation: "autocomplete attributes",
  },
  {
    criterionId: "1.4.10",
    level: "AA",
    name: "Reflow",
    focusArea: "Low Vision",
    implementation: "320px width, no horizontal scroll",
  },
  {
    criterionId: "1.4.11",
    level: "AA",
    name: "Non-text Contrast",
    focusArea: "Low Vision",
    implementation: "UI components: 3:1 contrast",
  },
  {
    criterionId: "1.4.12",
    level: "AA",
    name: "Text Spacing",
    focusArea: "Low Vision",
    implementation: "Adjust line height, spacing",
  },
  {
    criterionId: "1.4.13",
    level: "AA",
    name: "Content on Hover or Focus",
    focusArea: "Low Vision",
    implementation: "Dismissible, hoverable, persistent",
  },
  {
    criterionId: "2.1.4",
    level: "A",
    name: "Character Key Shortcuts",
    focusArea: "Motor",
    implementation: "Turn off or remap shortcuts",
  },
  {
    criterionId: "2.5.1",
    level: "A",
    name: "Pointer Gestures",
    focusArea: "Motor/Mobile",
    implementation: "Single pointer alternative",
  },
  {
    criterionId: "2.5.2",
    level: "A",
    name: "Pointer Cancellation",
    focusArea: "Motor",
    implementation: "Up-event completion",
  },
  {
    criterionId: "2.5.3",
    level: "A",
    name: "Label in Name",
    focusArea: "Speech Input",
    implementation: "Visible label in accessible name",
  },
  {
    criterionId: "2.5.4",
    level: "A",
    name: "Motion Actuation",
    focusArea: "Motor",
    implementation: "Alternative to motion",
  },
  {
    criterionId: "4.1.3",
    level: "AA",
    name: "Status Messages",
    focusArea: "Screen Readers",
    implementation: "aria-live, role=status/alert",
  },
];
```

---

## WAI-ARIA Fundamentals

### The Three Rules of ARIA

```typescript
/**
 * Critical ARIA rules - MUST follow these
 */
const ariaRules = {
  rule1: {
    text: "No ARIA is better than bad ARIA",
    explanation: "Use native HTML elements first",
    example: {
      bad: '<div role="button">Click</div>',
      good: "<button>Click</button>",
    },
  },

  rule2: {
    text: "Do not change native semantics unless absolutely necessary",
    explanation: "Don't override native HTML meaning",
    example: {
      bad: '<h1 role="button">Heading Button</h1>',
      good: "<h1><button>Button</button></h1>",
    },
  },

  rule3: {
    text: "All interactive ARIA controls must be keyboard accessible",
    explanation: "If it has a role, it must work with keyboard",
    example: {
      bad: '<div role="button" onclick="fn()">No keyboard</div>',
      good: '<div role="button" tabindex="0" onclick="fn()" onkeydown="handleKey(e)">Works</div>',
    },
  },
};
```

### ARIA Roles Categories

```typescript
/**
 * ARIA Roles organized by category
 */
interface ARIARoleCategory {
  category: string;
  roles: string[];
  usage: string;
  examples: string[];
}

const ariaCategories: ARIARoleCategory[] = [
  {
    category: "Landmark Roles",
    roles: [
      "banner",
      "main",
      "navigation",
      "complementary",
      "contentinfo",
      "form",
      "search",
      "region",
    ],
    usage: "Page structure and navigation",
    examples: ["<nav role='navigation'>", "<main role='main'>"],
  },
  {
    category: "Widget Roles",
    roles: [
      "button",
      "checkbox",
      "radio",
      "tab",
      "tablist",
      "tabpanel",
      "textbox",
      "slider",
      "spinbutton",
      "progressbar",
      "menu",
      "menuitem",
    ],
    usage: "Interactive components",
    examples: ["<div role='button'>", "<div role='tab'>"],
  },
  {
    category: "Document Structure",
    roles: [
      "article",
      "definition",
      "directory",
      "document",
      "group",
      "heading",
      "img",
      "list",
      "listitem",
      "table",
      "row",
      "cell",
    ],
    usage: "Content organization",
    examples: ["<div role='article'>", "<div role='list'>"],
  },
  {
    category: "Live Region",
    roles: ["alert", "log", "status", "timer", "marquee"],
    usage: "Dynamic content updates",
    examples: ["<div role='alert'>", "<div role='status' aria-live='polite'>"],
  },
];
```

### ARIA States and Properties

```typescript
/**
 * Key ARIA attributes for implementation
 */
interface ARIAAttribute {
  attribute: string;
  type: "state" | "property";
  values: string;
  usage: string;
  example: string;
}

const keyAriaAttributes: ARIAAttribute[] = [
  {
    attribute: "aria-label",
    type: "property",
    values: "string",
    usage: "Provides accessible name",
    example: '<button aria-label="Close dialog">X</button>',
  },
  {
    attribute: "aria-labelledby",
    type: "property",
    values: "ID reference",
    usage: "Points to labeling element",
    example: '<div role="dialog" aria-labelledby="title">',
  },
  {
    attribute: "aria-describedby",
    type: "property",
    values: "ID reference",
    usage: "Additional description",
    example: '<input aria-describedby="password-requirements">',
  },
  {
    attribute: "aria-hidden",
    type: "state",
    values: "true | false",
    usage: "Hides from accessibility tree",
    example: '<div aria-hidden="true">Decorative</div>',
  },
  {
    attribute: "aria-live",
    type: "property",
    values: "off | polite | assertive",
    usage: "Announces dynamic changes",
    example: '<div aria-live="polite">Message</div>',
  },
  {
    attribute: "aria-expanded",
    type: "state",
    values: "true | false | undefined",
    usage: "Collapsible element state",
    example: '<button aria-expanded="false">Menu</button>',
  },
  {
    attribute: "aria-pressed",
    type: "state",
    values: "true | false | mixed",
    usage: "Toggle button state",
    example: '<button aria-pressed="false">Mute</button>',
  },
  {
    attribute: "aria-checked",
    type: "state",
    values: "true | false | mixed",
    usage: "Checkbox/radio state",
    example: '<div role="checkbox" aria-checked="false">',
  },
  {
    attribute: "aria-selected",
    type: "state",
    values: "true | false | undefined",
    usage: "Tab/option selection",
    example: '<button role="tab" aria-selected="true">',
  },
  {
    attribute: "aria-invalid",
    type: "state",
    values: "true | false | grammar | spelling",
    usage: "Form validation state",
    example: '<input aria-invalid="true">',
  },
  {
    attribute: "aria-required",
    type: "property",
    values: "true | false",
    usage: "Required form field",
    example: '<input aria-required="true">',
  },
  {
    attribute: "aria-disabled",
    type: "state",
    values: "true | false",
    usage: "Disabled but visible element",
    example: '<button aria-disabled="true">Submit</button>',
  },
];
```

---

## Implementing WCAG 2.1 Level A

### 1.1.1 Non-text Content (Level A)

```typescript
/**
 * All non-text content must have text alternative
 */

// ✅ Informative Images
const informativeImage = `<img 
  src="sales-chart.png" 
  alt="Sales increased 45% from Q1 to Q2; showing steady growth trend"
/>`;

// ✅ Functional Images
const functionalImage = `<button aria-label="Search">
  <img src="search-icon.png" alt="">
</button>`;

// ✅ Decorative Images
const decorativeImage = `<img src="decorative-pattern.png" alt="" role="presentation">`;

// ✅ Complex Infographics
const complexImage = `<figure>
  <img 
    src="complex-diagram.png" 
    alt="Company organizational structure"
    aria-describedby="org-description"
  />
  <figcaption id="org-description">
    The organization has three main divisions: 
    Engineering led by Jane Doe with 50 employees,
    Sales led by John Smith with 30 employees,
    and Operations led by Alice Johnson with 20 employees.
  </figcaption>
</figure>`;

// ✅ CAPTCHAs
const accessibleCaptcha = `<div>
  <label for="captcha">Verify you're human (CAPTCHA)</label>
  <img src="captcha.png" alt="Visual CAPTCHA image">
  <input type="text" id="captcha">
  <button onclick="playAudioCaptcha()">Audio Alternative</button>
</div>`;
```

### 2.1.1 Keyboard (Level A)

```typescript
/**
 * All functionality available from keyboard
 */

// ❌ BAD: No keyboard support
class BadButton {
  render() {
    return '<div onclick="handleClick()">Click Me</div>';
  }
}

// ✅ GOOD: Full keyboard support
class AccessibleButton {
  render() {
    return `<button 
      onclick="handleClick()"
      onkeydown="handleKeyPress(event)"
      tabindex="0"
    >
      Click Me
    </button>`;
  }

  handleKeyPress(event: KeyboardEvent) {
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      this.handleClick();
    }
  }

  handleClick() {
    console.log("Button activated");
  }
}

// ✅ Custom widget with keyboard
class AccessibleDropdown {
  private element: HTMLElement;
  private isOpen = false;
  private selectedIndex = 0;
  private options: HTMLElement[] = [];

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupKeyboard();
  }

  private setupKeyboard(): void {
    const trigger = this.element.querySelector('[role="button"]')!;
    const menu = this.element.querySelector('[role="menu"]')!;
    this.options = Array.from(menu.querySelectorAll('[role="menuitem"]'));

    // Trigger keyboard
    trigger.addEventListener("keydown", (e: Event) => {
      const event = e as KeyboardEvent;
      switch (event.key) {
        case "Enter":
        case " ":
        case "ArrowDown":
          event.preventDefault();
          this.open();
          break;
        case "ArrowUp":
          event.preventDefault();
          this.open(true); // Open with last item focused
          break;
      }
    });

    // Menu keyboard navigation
    this.options.forEach((option, index) => {
      option.addEventListener("keydown", (e: Event) => {
        const event = e as KeyboardEvent;
        switch (event.key) {
          case "ArrowDown":
            event.preventDefault();
            this.focusNextOption(index);
            break;
          case "ArrowUp":
            event.preventDefault();
            this.focusPreviousOption(index);
            break;
          case "Home":
            event.preventDefault();
            this.focusOption(0);
            break;
          case "End":
            event.preventDefault();
            this.focusOption(this.options.length - 1);
            break;
          case "Escape":
            event.preventDefault();
            this.close();
            break;
          case "Enter":
          case " ":
            event.preventDefault();
            this.selectOption(index);
            break;
        }
      });
    });
  }

  private open(focusLast = false): void {
    this.isOpen = true;
    const menu = this.element.querySelector('[role="menu"]')!;
    (menu as HTMLElement).hidden = false;

    const indexToFocus = focusLast ? this.options.length - 1 : 0;
    this.focusOption(indexToFocus);
  }

  private close(): void {
    this.isOpen = false;
    const menu = this.element.querySelector('[role="menu"]')!;
    (menu as HTMLElement).hidden = true;

    const trigger = this.element.querySelector('[role="button"]')!;
    (trigger as HTMLElement).focus();
  }

  private focusOption(index: number): void {
    this.options[index].focus();
    this.selectedIndex = index;
  }

  private focusNextOption(currentIndex: number): void {
    const nextIndex = (currentIndex + 1) % this.options.length;
    this.focusOption(nextIndex);
  }

  private focusPreviousOption(currentIndex: number): void {
    const prevIndex =
      currentIndex === 0 ? this.options.length - 1 : currentIndex - 1;
    this.focusOption(prevIndex);
  }

  private selectOption(index: number): void {
    console.log("Selected:", this.options[index].textContent);
    this.close();
  }
}
```

### 3.1.1 Language of Page (Level A)

```html
<!-- ✅ HTML lang attribute -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>My Page</title>
  </head>
  <body>
    <p>This is in English</p>

    <!-- ✅ Language changes -->
    <p lang="es">Este texto está en español</p>
    <p lang="fr">Ce texte est en français</p>

    <blockquote lang="de">
      <p>Dieser Text ist auf Deutsch</p>
    </blockquote>
  </body>
</html>
```

### 4.1.2 Name, Role, Value (Level A)

```typescript
/**
 * All UI components must have name, role, and value
 * announced to assistive technologies
 */

// ✅ Native elements (automatic)
const nativeButton = `<button>Save</button>
<!-- Name: "Save", Role: "button", Value: N/A -->`;

// ✅ Custom elements need ARIA
const customToggle = `<div
  role="button"                    <!-- Role -->
  aria-label="Mute audio"          <!-- Name -->
  aria-pressed="false"             <!-- Value/State -->
  tabindex="0"
  onclick="toggleMute()"
>
  🔊
</div>`;

// ✅ Form elements
const formInput = `<label for="email">Email Address</label>
<input 
  type="email" 
  id="email"                       <!-- Name from label -->
  required 
  aria-required="true"             <!-- State -->
  aria-invalid="false"             <!-- Validation state -->
  aria-describedby="email-hint"
/>
<span id="email-hint">We'll never share your email</span>`;

// ✅ Custom checkbox
class AccessibleCheckbox {
  private element: HTMLElement;
  private checked = false;

  constructor(element: HTMLElement) {
    this.element = element;
    this.setup();
  }

  private setup(): void {
    this.element.setAttribute("role", "checkbox"); // Role
    this.element.setAttribute("aria-checked", "false"); // Value
    this.element.setAttribute("tabindex", "0");

    // Ensure accessible name
    if (
      !this.element.getAttribute("aria-label") &&
      !this.element.getAttribute("aria-labelledby")
    ) {
      console.warn("Checkbox needs accessible name!");
    }

    this.element.addEventListener("click", () => this.toggle());
    this.element.addEventListener("keydown", (e) => this.handleKey(e));
  }

  private toggle(): void {
    this.checked = !this.checked;
    this.element.setAttribute("aria-checked", String(this.checked));
    this.element.classList.toggle("checked", this.checked);

    // Announce change to screen readers
    this.announce();
  }

  private handleKey(event: KeyboardEvent): void {
    if (event.key === " " || event.key === "Enter") {
      event.preventDefault();
      this.toggle();
    }
  }

  private announce(): void {
    // Name, Role, Value are all now updated
    // Screen reader will announce: "[Name] checkbox checked/unchecked"
  }
}
```

---

## Implementing WCAG 2.1 Level AA

### 1.4.3 Contrast (Minimum) - Level AA

```typescript
/**
 * Text contrast requirements:
 * - Normal text: 4.5:1
 * - Large text (18pt+ or 14pt+ bold): 3:1
 */

// ✅ Calculate contrast ratio
function getContrastRatio(color1: string, color2: string): number {
  const luminance1 = getRelativeLuminance(color1);
  const luminance2 = getRelativeLuminance(color2);

  const lighter = Math.max(luminance1, luminance2);
  const darker = Math.min(luminance1, luminance2);

  return (lighter + 0.05) / (darker + 0.05);
}

function getRelativeLuminance(hex: string): number {
  // Convert hex to RGB
  const rgb = hexToRgb(hex);
  if (!rgb) return 0;

  // Convert to relative values
  const r = rgb.r / 255;
  const g = rgb.g / 255;
  const b = rgb.b / 255;

  // Apply gamma correction
  const rLinear = r <= 0.03928 ? r / 12.92 : Math.pow((r + 0.055) / 1.055, 2.4);
  const gLinear = g <= 0.03928 ? g / 12.92 : Math.pow((g + 0.055) / 1.055, 2.4);
  const bLinear = b <= 0.03928 ? b / 12.92 : Math.pow((b + 0.055) / 1.055, 2.4);

  // Calculate luminance
  return 0.2126 * rLinear + 0.7152 * gLinear + 0.0722 * bLinear;
}

function hexToRgb(hex: string): { r: number; g: number; b: number } | null {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  return result
    ? {
        r: parseInt(result[1], 16),
        g: parseInt(result[2], 16),
        b: parseInt(result[3], 16),
      }
    : null;
}

// ✅ Check if colors meet WCAG AA
function meetsWCAGA(
  foreground: string,
  background: string,
  isLargeText = false,
): boolean {
  const ratio = getContrastRatio(foreground, background);
  const requiredRatio = isLargeText ? 3 : 4.5;
  return ratio >= requiredRatio;
}

// Usage
const text = "#595959";
const bg = "#ffffff";

console.log(getContrastRatio(text, bg)); // 7:1
console.log(meetsWCAGAA(text, bg)); // true

// ✅ CSS implementation
const accessibleColors = `
:root {
  /* WCAG AA compliant colors on white */
  --text-primary: #212121;     /* 16.1:1 */
  --text-secondary: #595959;   /* 7:1 */
  --link-color: #0056b3;       /* 7.4:1 */
  --link-hover: #003d82;       /* 10.7:1 */
  
  /* UI components */
  --button-primary-bg: #0066cc; /* 4.7:1 on white */
  --button-primary-text: #ffffff;
  
  /* States */
  --error: #d32f2f;            /* 4.6:1 */
  --success: #2e7d32;          /* 4.6:1 */
  --warning: #f57c00;          /* 3.4:1 - only for large text */
}
`;
```

### 1.4.10 Reflow (Level AA) - WCAG 2.1

```css
/**
 * Content must reflow to 320px width without horizontal scrolling
 * (400% zoom at 1280px)
 */

/* ✅ Responsive layout that reflows */
.container {
  width: 100%;
  max-width: 1200px;
  padding: 1rem;
  margin: 0 auto;
}

/* ✅ Use responsive units */
.text {
  font-size: clamp(1rem, 2.5vw, 1.5rem);
  line-height: 1.5;
}

/* ✅ Grid that reflows */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 250px), 1fr));
  gap: 1rem;
}

/* ✅ No fixed widths that prevent reflow */
.content {
  min-width: 0; /* Allows shrinking below content size */
  overflow-wrap: break-word;
  word-wrap: break-word;
  hyphens: auto;
}

/* ❌ BAD: Fixed width prevents reflow */
.bad-container {
  width: 1000px; /* Don't do this */
  min-width: 800px; /* Don't do this */
}
```

```typescript
// ✅ Test reflow programmatically
function testReflow(): boolean {
  const viewport = window.innerWidth;
  const hasHorizontalScroll = document.body.scrollWidth > viewport;

  if (viewport <= 320 && hasHorizontalScroll) {
    console.warn("Horizontal scroll at 320px - fails reflow!");
    return false;
  }

  return true;
}
```

### 1.4.11 Non-text Contrast (Level AA) - WCAG 2.1

```css
/**
 * UI components and graphical objects need 3:1 contrast
 * against adjacent colors
 */

/* ✅ Focus indicators - 3:1 minimum */
*:focus {
  outline: 2px solid #0066cc; /* 4.7:1 on white = Pass */
  outline-offset: 2px;
}

/* ✅ Form inputs - visible boundaries */
input,
select,
textarea {
  border: 2px solid #767676; /* 4.5:1 on white = Pass */
  background: #ffffff;
}

input:focus {
  border-color: #0066cc; /* 4.7:1 = Pass */
}

/* ✅ Buttons - 3:1 contrast with background */
.button {
  background: #0066cc; /* 4.7:1 on white = Pass */
  color: #ffffff;
  border: 2px solid #0066cc;
}

/* ✅ Icons and graphics */
.icon {
  fill: #212121; /* 16.1:1 = Pass */
}

/* ✅ Charts and data visualization */
.chart-line-1 {
  stroke: #0066cc; /* 4.7:1 */
}
.chart-line-2 {
  stroke: #d32f2f; /* 4.6:1 */
}
.chart-line-3 {
  stroke: #388e3c; /* 4.5:1 */
}

/* ⚠️ Inactive/disabled - exempt  from contrast requirements */
button:disabled {
  opacity: 0.5; /* Exempt from 3:1 requirement */
}
```

### 4.1.3 Status Messages (Level AA) - WCAG 2.1

```typescript
/**
 * Status messages must be announced without focus
 * Uses aria-live regions
 */

// ✅ Success message
class StatusMessage {
  private container: HTMLElement;

  constructor() {
    this.container = this.createLiveRegion();
    document.body.appendChild(this.container);
  }

  private createLiveRegion(): HTMLElement {
    const region = document.createElement("div");
    region.setAttribute("role", "status"); // Implicit aria-live="polite"
    region.setAttribute("aria-live", "polite");
    region.setAttribute("aria-atomic", "true");
    region.className = "sr-only"; // Visually hidden
    return region;
  }

  showSuccess(message: string): void {
    this.container.textContent = message;

    // Also show visually
    this.showVisualToast(message, "success");

    // Clear after announcement
    setTimeout(() => {
      this.container.textContent = "";
    }, 1000);
  }

  showError(message: string): void {
    // Use role="alert" for errors (aria-live="assertive")
    const alert = document.createElement("div");
    alert.setAttribute("role", "alert");
    alert.textContent = message;
    document.body.appendChild(alert);

    this.showVisualToast(message, "error");

    setTimeout(() => {
      alert.remove();
    }, 5000);
  }

  private showVisualToast(message: string, type: string): void {
    // Visual toast implementation
    const toast = document.createElement("div");
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    toast.setAttribute("aria-hidden", "true"); // Don't double-announce
    document.body.appendChild(toast);

    setTimeout(() => toast.remove(), 3000);
  }
}

// Usage
const status = new StatusMessage();

// ✅ Form submission
async function handleSubmit() {
  try {
    await saveData();
    status.showSuccess("Data saved successfully");
  } catch (error) {
    status.showError("Failed to save data. Please try again.");
  }
}

// ✅ Loading states
class LoadingStatus {
  private region: HTMLElement;

  constructor() {
    this.region = document.createElement("div");
    this.region.setAttribute("role", "status");
    this.region.setAttribute("aria-live", "polite");
    this.region.setAttribute("aria-busy", "false");
    document.body.appendChild(this.region);
  }

  startLoading(message = "Loading..."): void {
    this.region.setAttribute("aria-busy", "true");
    this.region.textContent = message;
  }

  stopLoading(): void {
    this.region.setAttribute("aria-busy", "false");
    this.region.textContent = "Loading complete";

    setTimeout(() => {
      this.region.textContent = "";
    }, 1000);
  }
}
```

---

## Implementing WCAG 2.1 Level AAA

### 1.4.6 Contrast (Enhanced) - Level AAA

```typescript
/**
 * Level AAA contrast requirements:
 * - Normal text: 7:1
 * - Large text: 4.5:1
 */

function meetsWCAGAAA(
  foreground: string,
  background: string,
  isLargeText = false,
): boolean {
  const ratio = getContrastRatio(foreground, background);
  const requiredRatio = isLargeText ? 4.5 : 7;
  return ratio >= requiredRatio;
}

// ✅ AAA compliant color palette
const aaaColors = `
:root {
  /* AAA compliant on white background */
  --text-primary: #000000;     /* 21:1 */
  --text-secondary: #4a4a4a;   /* 8.6:1 */
  --link: #004085;             /* 8.6:1 */
  
  /* AAA for large text */
  --heading: #5a5a5a;          /* 7.5:1 */
}
`;
```

### 2.4.9 Link Purpose (Link Only) - Level AAA

```html
<!-- ✅ Self-explanatory links -->
<a href="/products">View all products</a>
<a href="/contact">Contact support team</a>

<!-- ❌ BAD: Ambiguous links -->
<!-- <a href="/page1">Click here</a> -->
<!-- <a href="/page2">Read more</a> -->

<!-- ✅ GOOD: Context in link text -->
<article>
  <h2>New Product Launch</h2>
  <p>We're excited to announce...</p>
  <a href="/products/new-widget"> Read more about the new widget </a>
</article>

<!-- ✅ Or use aria-label -->
<a href="/article" aria-label="Read more about the new product launch">
  Read more
</a>
```

---

## ARIA Patterns & Widgets

### Accessible Modal Dialog

```typescript
/**
 * Full ARIA modal implementation
 * Meets WCAG 2.1.1 (Keyboard), 2.4.3 (Focus Order), 2.4.7 (Focus Visible)
 */
class AccessibleModal {
  private modal: HTMLElement;
  private overlay: HTMLElement;
  private closeButton: HTMLElement;
  private focusableElements: HTMLElement[] = [];
  private previousFocus: HTMLElement | null = null;
  private firstFocusable: HTMLElement | null = null;
  private lastFocusable: HTMLElement | null = null;

  constructor(modalId: string) {
    this.modal = document.getElementById(modalId)!;
    this.overlay = document.querySelector(".modal-overlay")!;
    this.closeButton = this.modal.querySelector("[data-close]")!;

    this.setupModal();
    this.setupKeyboard();
  }

  private setupModal(): void {
    // ARIA attributes
    this.modal.setAttribute("role", "dialog");
    this.modal.setAttribute("aria-modal", "true");

    // Ensure labeling
    const title = this.modal.querySelector("h2, h3");
    if (title && !title.id) {
      title.id = `${this.modal.id}-title`;
    }
    if (title) {
      this.modal.setAttribute("aria-labelledby", title.id);
    }

    // Description
    const description = this.modal.querySelector(".modal-description, p");
    if (description && !description.id) {
      description.id = `${this.modal.id}-description`;
    }
    if (description) {
      this.modal.setAttribute("aria-describedby", description.id);
    }
  }

  private setupKeyboard(): void {
    // Escape to close
    document.addEventListener("keydown", (e) => {
      if (e.key === "Escape" && !this.modal.hidden) {
        this.close();
      }
    });

    // Tab trap
    this.modal.addEventListener("keydown", (e) => {
      if (e.key === "Tab") {
        this.handleTabKey(e);
      }
    });

    // Close button
    this.closeButton.addEventListener("click", () => this.close());
    this.overlay.addEventListener("click", () => this.close());
  }

  open(): void {
    // Store current focus
    this.previousFocus = document.activeElement as HTMLElement;

    // Show modal
    this.modal.hidden = false;
    this.overlay.hidden = false;
    document.body.classList.add("modal-open");

    // Disable page scroll
    document.body.style.overflow = "hidden";

    // Get focusable elements
    this.focusableElements = this.getFocusableElements();
    this.firstFocusable = this.focusableElements[0];
    this.lastFocusable =
      this.focusableElements[this.focusableElements.length - 1];

    // Focus first element
    setTimeout(() => {
      this.firstFocusable?.focus();
    }, 100);

    // Announce to screen readers
    this.announce("Dialog opened");
  }

  close(): void {
    // Hide modal
    this.modal.hidden = true;
    this.overlay.hidden = true;
    document.body.classList.remove("modal-open");

    // Restore scroll
    document.body.style.overflow = "";

    // Restore focus
    this.previousFocus?.focus();

    // Announce
    this.announce("Dialog closed");
  }

  private getFocusableElements(): HTMLElement[] {
    const selector =
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])';
    return Array.from(this.modal.querySelectorAll(selector));
  }

  private handleTabKey(event: KeyboardEvent): void {
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

  private announce(message: string): void {
    const announcement = document.createElement("div");
    announcement.setAttribute("role", "status");
    announcement.setAttribute("aria-live", "polite");
    announcement.className = "sr-only";
    announcement.textContent = message;
    document.body.appendChild(announcement);

    setTimeout(() => announcement.remove(), 1000);
  }
}

// HTML structure
const modalHTML = `
<div class="modal-overlay" hidden aria-hidden="true"></div>

<div id="my-modal" class="modal" hidden>
  <div class="modal-content">
    <h2 id="modal-title">Confirm Action</h2>
    <p id="modal-desc">Are you sure you want to proceed?</p>
    
    <div class="modal-actions">
      <button onclick="confirmAction()">Confirm</button>
      <button data-close>Cancel</button>
    </div>
    
    <button data-close aria-label="Close dialog" class="modal-close">
      ×
    </button>
  </div>
</div>
`;
```

### Accessible Tabs

```typescript
/**
 * Tabs following ARIA Authoring Practices
 */
class AccessibleTabs {
  private tablist: HTMLElement;
  private tabs: HTMLElement[];
  private panels: HTMLElement[];
  private selectedIndex = 0;

  constructor(container: HTMLElement) {
    this.tablist = container.querySelector('[role="tablist"]')!;
    this.tabs = Array.from(this.tablist.querySelectorAll('[role="tab"]'));
    this.panels = Array.from(container.querySelectorAll('[role="tabpanel"]'));

    this.setupTabs();
    this.setupKeyboard();
  }

  private setupTabs(): void {
    this.tabs.forEach((tab, index) => {
      // ARIA attributes
      tab.setAttribute("role", "tab");
      tab.setAttribute("tabindex", index === 0 ? "0" : "-1");
      tab.setAttribute("aria-selected", index === 0 ? "true" : "false");

      const panelId = `panel-${index}`;
      tab.setAttribute("aria-controls", panelId);
      tab.id = `tab-${index}`;

      // Panel attributes
      this.panels[index].setAttribute("role", "tabpanel");
      this.panels[index].setAttribute("tabindex", "0");
      this.panels[index].setAttribute("aria-labelledby", tab.id);
      this.panels[index].id = panelId;
      this.panels[index].hidden = index !== 0;

      // Click handler
      tab.addEventListener("click", () => this.selectTab(index));
    });
  }

  private setupKeyboard(): void {
    this.tabs.forEach((tab, index) => {
      tab.addEventListener("keydown", (e) => {
        let newIndex: number | null = null;

        switch (e.key) {
          case "ArrowLeft":
            newIndex = index === 0 ? this.tabs.length - 1 : index - 1;
            break;
          case "ArrowRight":
            newIndex = index === this.tabs.length - 1 ? 0 : index + 1;
            break;
          case "Home":
            newIndex = 0;
            break;
          case "End":
            newIndex = this.tabs.length - 1;
            break;
        }

        if (newIndex !== null) {
          e.preventDefault();
          this.selectTab(newIndex);
          this.tabs[newIndex].focus();
        }
      });
    });
  }

  private selectTab(index: number): void {
    // Deselect all
    this.tabs.forEach((tab, i) => {
      tab.setAttribute("aria-selected", "false");
      tab.setAttribute("tabindex", "-1");
      this.panels[i].hidden = true;
    });

    // Select target
    this.tabs[index].setAttribute("aria-selected", "true");
    this.tabs[index].setAttribute("tabindex", "0");
    this.panels[index].hidden = false;

    this.selectedIndex = index;

    // Announce change
    this.announce(
      `${this.tabs[index].textContent} tab selected, ${this.panels[index].querySelectorAll("*").length} items`,
    );
  }

  private announce(message: string): void {
    const status = document.createElement("div");
    status.setAttribute("role", "status");
    status.setAttribute("aria-live", "polite");
    status.className = "sr-only";
    status.textContent = message;
    document.body.appendChild(status);

    setTimeout(() => status.remove(), 1000);
  }
}
```

---

## Mobile Accessibility (WCAG 2.1)

### Touch Target Size

```typescript
/**
 * WCAG 2.5.5 (AAA): Touch targets should be at least 44x44px
 * Best practice: 48x48px minimum
 */

// ✅ CSS for touch-friendly sizes
const touchTargetsCSS = `
/* Minimum touch target sizes */
button,
a,
[role="button"],
[role="link"],
input[type="checkbox"],
input[type="radio"] {
  min-width: 44px;
  min-height: 44px;
  
  /* Adequate spacing */
  margin: 4px;
}

/* For icons */
.icon-button {
  width: 48px;
  height: 48px;
  padding: 12px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

/* Touch-friendly form controls */
input[type="text"],
input[type="email"],
select,
textarea {
  min-height: 44px;
  padding: 12px;
  font-size: 16px; /* Prevents iOS zoom */
}
`;

// ✅ Check touch target sizes
function validateTouchTargets(): void {
  const interactiveElements = document.querySelectorAll(
    'button, a, [role="button"], input, select',
  );

  interactiveElements.forEach((el) => {
    const rect = el.getBoundingClientRect();
    const { width, height } = rect;

    if (width < 44 || height < 44) {
      console.warn(`Touch target too small: ${width}x${height}px`, el);
    }
  });
}
```

### Orientation (1.3.4 - AA)

```html
<!-- ✅ Don't lock orientation -->
<meta name="viewport" content="width=device-width, initial-scale=1" />

<!-- ❌ DON'T lock to portait/landscape -->
<!-- <meta name="viewport" content="user-scalable=no, orientation=portrait"> -->
```

```css
/* ✅ Support both orientations */
@media (orientation: portrait) {
  .container {
    flex-direction: column;
  }
}

@media (orientation: landscape) {
  .container {
    flex-direction: row;
  }
}
```

### Motion Actuation (2.5.4 - A)

```typescript
/**
 * Functionality that uses device motion must have alternative
 */

// ❌ BAD: Shake-only to undo
window.addEventListener("devicemotion", (event) => {
  if (isShaking(event)) {
    undo(); // Only way to undo!
  }
});

// ✅ GOOD: Shake + button alternative
function UndoFeature() {
  // Motion alternative
  window.addEventListener("devicemotion", (event) => {
    if (isShaking(event)) {
      undo();
    }
  });

  // Always provide UI alternative
  return (
    <button onClick={undo} aria-label="Undo last action">
      Undo
    </button>
  );
}

// ✅ Respect prefers-reduced-motion
const prefersReducedMotion = window.matchMedia(
  "(prefers-reduced-motion: reduce)"
).matches;

if (!prefersReducedMotion) {
  // Only enable motion features if user hasn't opted out
  enableShakeToUndo();
}
```

---

## Cognitive Accessibility

### Clear Language (3.1.5 - AAA)

```typescript
/**
 * Use clear, simple language
 */

// ❌ BAD: Complex language
const badText = `
  Utilize the aforementioned methodology to ameliorate 
  the suboptimal performance characteristics.
`;

// ✅ GOOD: Simple language
const goodText = `
  Use this method to improve poor performance.
`;

// ✅ Provide definitions for technical terms
const technicalTerm = `
<p>
  The <dfn>API</dfn> 
  <span class="definition">(Application Programming Interface)</span>
  allows apps to communicate.
</p>
`;

// ✅ Expand abbreviations on first use
const abbreviation = `
<p>
  The 
  <abbr title="World Wide Web Consortium">W3C</abbr>
  develops web standards.
</p>
`;
```

### Consistent Navigation (3.2.3 - AA)

```typescript
/**
 * Navigation must be consistent across pages
 */

// ✅ Consistent navigation component
function MainNavigation() {
  // Same order, same labels across all pages
  const navItems = [
    { label: "Home", href: "/" },
    { label: "Products", href: "/products" },
    { label: "About", href: "/about" },
    { label: "Contact", href: "/contact" },
  ];

  return (
    <nav aria-label="Main navigation">
      <ul>
        {navItems.map((item) => (
          <li key={item.href}>
            <a href={item.href}>{item.label}</a>
          </li>
        ))}
      </ul>
    </nav>
  );
}

// ❌ DON'T change navigation order between pages
// ❌ DON'T change labels ("Products" → "Our Products")
// ❌ DON'T add/remove items on different pages
```

### Error Prevention (3.3.4 - AA)

```typescript
/**
 * Prevent errors before submission
 */

// ✅ Confirmation for important actions
function DeleteAccount() {
  const [confirmText, setConfirmText] = useState("");

  const handleDelete = () => {
    if (confirmText !== "DELETE") {
      alert("Please type DELETE to confirm");
      return;
    }

    // Show final confirmation
    if (confirm("Are you absolutely sure? This cannot be undone.")) {
      deleteAccount();
    }
  };

  return (
    <form onSubmit={handleDelete}>
      <label htmlFor="confirm">
        Type "DELETE" to confirm account deletion
      </label>
      <input
        type="text"
        id="confirm"
        value={confirmText}
        onChange={(e) => setConfirmText(e.target.value)}
        aria-required="true"
      />
      <button type="submit" disabled={confirmText !== "DELETE"}>
        Delete Account
      </button>
    </form>
  );
}

// ✅ Review before submit
function CheckoutForm() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({});

  if (step === 3) {
    // Review step
    return (
      <div>
        <h2>Review Your Order</h2>
        <Review data={formData} />
        <button onClick={() => setStep(2)}>
          Edit
        </button>
        <button onClick={submitOrder}>
          Confirm and Submit
        </button>
      </div>
    );
  }

  // Form steps...
}
```

---

## Testing & Validation

### Automated Testing

```typescript
/**
 * Automated accessibility testing with axe-core
 */
import { axe, toHaveNoViolations } from "jest-axe";

expect.extend(toHaveNoViolations);

describe("Accessibility Tests", () => {
  it("should have no accessibility violations", async () => {
    const { container } = render(<MyComponent />);
    const results = await axe(container);

    expect(results).toHaveNoViolations();
  });

  it("should have proper ARIA labels", () => {
    render(<Button />);

    const button = screen.getByRole("button");
    expect(button).toHaveAccessibleName();
  });

  it("should be keyboard navigable", () => {
    render(<Form />);

    const input = screen.getByLabelText("Email");
    input.focus();

    userEvent.tab();

    expect(screen.getByLabelText("Password")).toHaveFocus();
  });
});

// ✅ Custom accessibility checks
function checkAccessibility(element: HTMLElement): string[] {
  const issues: string[] = [];

  // Check images
  const images = element.querySelectorAll("img");
  images.forEach((img) => {
    if (!img.alt && img.alt !== "") {
      issues.push(`Image missing alt text: ${img.src}`);
    }
  });

  // Check form labels
  const inputs = element.querySelectorAll("input, select, textarea");
  inputs.forEach((input) => {
    const id = input.id;
    const label = id ? element.querySelector(`label[for="${id}"]`) : null;
    const ariaLabel = input.getAttribute("aria-label");
    const ariaLabelledby = input.getAttribute("aria-labelledby");

    if (!label && !ariaLabel && !ariaLabelledby) {
      issues.push(`Form control missing label: ${input}`);
    }
  });

  // Check heading hierarchy
  const headings = Array.from(element.querySelectorAll("h1, h2, h3, h4, h5, h6"));
  let lastLevel = 0;

  headings.forEach((heading) => {
    const level = parseInt(heading.tagName[1]);

    if (level - lastLevel > 1) {
      issues.push(`Heading hierarchy skip: ${heading.tagName} after h${lastLevel}`);
    }

    lastLevel = level;
  });

  // Check color contrast
  const textElements = element.querySelectorAll("p, span, a, button, li");
  textElements.forEach((el) => {
    const computed = window.getComputedStyle(el);
    const color = computed.color;
    const bgColor = computed.backgroundColor;

    const ratio = getContrastRatio(color, bgColor);

    if (ratio < 4.5) {
      issues.push(`Low contrast ratio: ${ratio.toFixed(2)}:1 on ${el.textContent?.substring(0, 20)}`);
    }
  });

  return issues;
}
```

### Manual Testing Checklist

```typescript
/**
 * Manual WCAG 2.1 testing checklist
 */
interface A11yTestItem {
  criterion: string;
  test: string;
  howTo: string;
  level: "A" | "AA" | "AAA";
}

const manualTests: A11yTestItem[] = [
  {
    criterion: "2.1.1 Keyboard",
    test: "All functionality works with keyboard only",
    howTo:
      "Disconnect mouse, navigate entire site with Tab, Enter, Space, Arrows",
    level: "A",
  },
  {
    criterion: "2.4.7 Focus Visible",
    test: "Focus indicator always visible",
    howTo: "Tab through all interactive elements, verify visible focus",
    level: "AA",
  },
  {
    criterion: "1.4.3 Contrast",
    test: "Text has 4.5:1 contrast minimum",
    howTo: "Use browser DevTools color picker to check contrast ratios",
    level: "AA",
  },
  {
    criterion: "1.3.1 Info and Relationships",
    test: "Screen reader reads content in logical order",
    howTo: "Use NVDA/JAWS/VoiceOver to navigate page",
    level: "A",
  },
  {
    criterion: "2.5.5 Target Size",
    test: "Touch targets at least 44x44px",
    howTo: "Measure interactive elements with DevTools",
    level: "AAA",
  },
  {
    criterion: "1.4.10 Reflow",
    test: "No horizontal scroll at 320px width",
    howTo: "Set viewport to 320px or zoom to 400%",
    level: "AA",
  },
  {
    criterion: "4.1.3 Status Messages",
    test: "Dynamic updates announced",
    howTo: "Use screen reader, trigger updates, verify announcements",
    level: "AA",
  },
];
```

---

## Common Mistakes

### Mistake 1: Using divs instead of buttons

```typescript
// ❌ BAD: div without keyboard support
const BadButton = () => (
  <div onClick={handleClick} className="button">
    Click me
  </div>
);

// ✅ GOOD: Native button
const GoodButton = () => (
  <button onClick={handleClick}>
    Click me
  </button>
);

// ✅ ACCEPTABLE: div with full ARIA and keyboard
const AriaButton = () => (
  <div
    role="button"
    tabIndex={0}
    onClick={handleClick}
    onKeyDown={(e) => {
      if (e.key === "Enter" || e.key === " ") {
        e.preventDefault();
        handleClick();
      }
    }}
    className="button"
  >
    Click me
  </div>
);
```

### Mistake 2: Missing form labels

```typescript
// ❌ BAD: No label
const BadForm = () => (
  <input type="text" placeholder="Enter email" />
);

// ✅ GOOD: Visible label
const GoodForm = () => (
  <>
    <label htmlFor="email">Email</label>
    <input type="text" id="email" />
  </>
);

// ✅ OK: aria-label (but visible label preferred)
const AriaForm = () => (
  <input type="text" aria-label="Email address" />
);
```

### Mistake 3: Low contrast

```css
/* ❌ BAD: 2.5:1 ratio */
.bad-text {
  color: #999999;
  background: #ffffff;
}

/* ✅ GOOD: 4.6:1 ratio (AA) */
.good-text {
  color: #595959;
  background: #ffffff;
}

/* ✅ BETTER: 7:1 ratio (AAA) */
.better-text {
  color: #4a4a4a;
  background: #ffffff;
}
```

### Mistake 4: Missing alt text

```html
<!-- ❌ BAD: Missing alt -->
<img src="photo.jpg" />

<!-- ✅ GOOD: Descriptive alt -->
<img src="team-photo.jpg" alt="Team of 5 developers standing in office" />

<!-- ✅ GOOD: Decorative (empty alt) -->
<img src="decoration.png" alt="" role="presentation" />

<!-- ✅ GOOD: Functional -->
<button>
  <img src="icon-save.svg" alt="Save document" />
</button>
```

### Mistake 5: Not announcing dynamic changes

```typescript
// ❌ BAD: Silent update
const BadComponent = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
};

// ✅ GOOD: Announces to screen readers
const GoodComponent = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p role="status" aria-live="polite">
        Count: {count}
      </p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
};
```

---

## Real-World Examples

### Accessible Data Table

```html
<table>
  <caption>
    Monthly Sales by Region
  </caption>
  <thead>
    <tr>
      <th scope="col">Region</th>
      <th scope="col">January</th>
      <th scope="col">February</th>
      <th scope="col">March</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">North</th>
      <td>$50,000</td>
      <td>$55,000</td>
      <td>$60,000</td>
    </tr>
    <tr>
      <th scope="row">South</th>
      <td>$45,000</td>
      <td>$48,000</td>
      <td>$52,000</td>
    </tr>
  </tbody>
</table>
```

### Accessible Form

```html
<form>
  <fieldset>
    <legend>Personal Information</legend>

    <div class="form-group">
      <label for="first-name">
        First Name
        <abbr title="required" aria-label="required">*</abbr>
      </label>
      <input
        type="text"
        id="first-name"
        required
        aria-required="true"
        aria-invalid="false"
        aria-describedby="first-name-error"
      />
      <div id="first-name-error" role="alert"></div>
    </div>

    <div class="form-group">
      <label for="email">Email</label>
      <input
        type="email"
        id="email"
        autocomplete="email"
        aria-describedby="email-hint"
      />
      <div id="email-hint" class="hint">We'll never share your email</div>
    </div>
  </fieldset>

  <button type="submit">Submit</button>
</form>
```

---

## Best Practices

### 1. Use Semantic HTML First

```html
<!-- ✅ Semantic HTML provides accessibility for free -->
<header>
  <nav>
    <ul>
      <li><a href="/">Home</a></li>
    </ul>
  </nav>
</header>

<main>
  <article>
    <h1>Heading</h1>
    <p>Content</p>
  </article>
</main>

<footer>
  <p>&copy; 2024</p>
</footer>
```

### 2. Progressive Enhancement

```typescript
// Start with HTML that works without JavaScript
// Enhance with JavaScript

// ✅ Form works without JS
// Then enhance with client-side validation
```

### 3. Test with Real Users

- Test with keyboard only
- Test with screen readers (NVDA, JAWS, VoiceOver)
- Test with zoom (up to 400%)
- Test with high contrast modes
- Test on mobile devices

### 4. Document Accessibility

```typescript
// README.md
const accessibilityDocs = `
## Accessibility

This project conforms to WCAG 2.1 Level AA.

### Features:
- Full keyboard navigation
- Screen reader support
- 4.5:1 minimum contrast
- Responsive to 320px
- No flashing content

### Known Issues:
- [Issue details and workarounds]

### Testing:
- Run \`npm run test:a11y\` for automated tests
- See [manual testing guide](./docs/a11y-testing.md)
`;
```

---

## Interview Questions

### Conceptual Questions

1. **What is WCAG 2.1 and why is it important?**
   - International accessibility standard
   - Legal requirements (ADA, Section 508)
   - Ensures inclusive design
     -17 new criteria over WCAG 2.0

2. **Explain the POUR principles.**
   - Perceivable: Users can perceive content
   - Operable: Users can operate interface
   - Understandable: Content is understandable
   - Robust: Works with assistive tech

3. **What is WAI-ARIA and when should you use it?**
   - Bridge between HTML and complex patterns
   - Use for custom widgets
   - Don't use when semantic HTML exists
   - "No ARIA is better than bad ARIA"

4. **What are the main differences between Level A, AA, and AAA?**
   - Level A: Minimum (severe barriers)
   - Level AA: Common target (significant barriers)
   - Level AAA: Enhanced (additional support)

### Technical Questions

5. **How do you implement an accessible modal?**
   - role="dialog", aria-modal="true"
   - aria-labelledby for title
   - Focus trap
   - Keyboard support (Escape, Tab trap)
   - Restore focus on close

6. **What are aria-live regions?**
   - Announce dynamic content changes
   - polite: Wait for pause
   - assertive: Interrupt immediately
   - Requires role="status" or "alert"

7. **How do you make a custom dropdown accessible?**
   - role="button" for trigger
   - role="menu" for list
   - role="menuitem" for items
   - Arrow key navigation
   - Escape to close
   - Type ahead

8. **What is the contrast ratio requirement for WCAG AA?**
   - Normal text: 4.5:1
   - Large text (18pt+ or 14pt+ bold): 3:1
   - UI components: 3:1

9. **How do you test for accessibility?**
   - Automated tools (axe, Lighthouse)
   - Keyboard navigation
   - Screen readers
   - Color contrast checkers
   - Manual WCAG audit

10. **What is a focus trap and when do you need it?**
    - Keeps focus within a component
    - Required for modals
    - Cycle through focusable elements
    - Tab and Shift+Tab loops

### Scenario Questions

11. **A designer gives you light gray text on white. What do you do?**
    - Check contrast ratio
    - If below 4.5:1, explain WCAG requirement
    - Suggest darker color that meets standards
    - Provide color palette tool

12. **User reports form not working with screen reader. How do you debug?**
    - Test with screen reader myself
    - Check form labels (for/id)
    - Verify error messages announced
    - Check validation feedback
    - Test keyboard navigation

13. **How do you make a single-page app accessible?**
    - Manage focus on route changes
    - Announce route changes with aria-live
    - Update document.title
    - Maintain keyboard navigation
    - Skip links for navigation

14. **Client wants animated carousel. Accessibility concerns?**
    - Provide pause button
    - Keyboard navigation
    - Screen reader announcements
    - Don't auto-advance faster than 5 seconds
    - Respect prefers-reduced-motion

---

## Summary

### Key Takeaways

1. **WCAG 2.1 + ARIA**: Complete accessibility solution
2. **POUR Principles**: Foundation of accessibility
3. **Semantic HTML First**: Use native elements
4. **Keyboard Support**: All functionality accessible
5. **ARIA Live Regions**: Announce dynamic updates
6. **Color Contrast**: Minimum 4.5:1 for AA
7. **Testing**: Automated + manual + real users
8. **Mobile**: Touch targets, orientation, gestures

### Implementation Checklist

- [ ] All images have alt text
- [ ] All form inputs have labels
- [ ] Keyboard navigation works
- [ ] Focus indicators visible
- [ ] Color contrast meets 4.5:1
- [ ] ARIA roles and properties correctly used
- [ ] Dynamic content announces to screen readers
- [ ] Headings in logical order
- [ ] Language attribute on html element
- [ ] No keyboard traps
- [ ] Touch targets at least 44x44px
- [ ] Responsive to 320px width
- [ ] Tested with screen reader
- [ ] Automated tests pass

### Resources

- [WCAG 2.1 Specification](https://www.w3.org/TR/WCAG21/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [WebAIM Resources](https://webaim.org/)
- [Inclusive Components](https://inclusive-components.design/)

---

**Next Steps**: Practice implementing ARIA patterns, audit existing sites, test with screen readers, and build accessible components from scratch.
