# WCAG Guidelines - Web Content Accessibility Guidelines

## Table of Contents

1. [Introduction to WCAG](#introduction-to-wcag)
2. [WCAG Principles (POUR)](#wcag-principles-pour)
3. [Conformance Levels](#conformance-levels)
4. [WCAG 2.1 Success Criteria](#wcag-21-success-criteria)
5. [WCAG 2.2 New Features](#wcag-22-new-features)
6. [Implementation Examples](#implementation-examples)
7. [Compliance Testing](#compliance-testing)
8. [Legal Requirements](#legal-requirements)
9. [Best Practices](#best-practices)
10. [Key Takeaways](#key-takeaways)

---

## Introduction to WCAG

**WCAG (Web Content Accessibility Guidelines)** is the international standard for web accessibility, published by the W3C's Web Accessibility Initiative (WAI).

### WCAG Versions

```typescript
interface WCAGVersion {
  version: string;
  releaseDate: string;
  status: string;
  successCriteria: number;
  newFeatures?: string[];
}

const wcagVersions: WCAGVersion[] = [
  {
    version: "WCAG 2.0",
    releaseDate: "December 2008",
    status: "Superseded",
    successCriteria: 61,
  },
  {
    version: "WCAG 2.1",
    releaseDate: "June 2018",
    status: "Current standard",
    successCriteria: 78,
    newFeatures: [
      "Mobile accessibility",
      "Low vision requirements",
      "Cognitive and learning disabilities",
    ],
  },
  {
    version: "WCAG 2.2",
    releaseDate: "October 2023",
    status: "Latest",
    successCriteria: 86,
    newFeatures: [
      "Focus appearance",
      "Dragging movements alternative",
      "Accessible authentication",
      "Consistent help",
    ],
  },
  {
    version: "WCAG 3.0",
    releaseDate: "TBD (Draft)",
    status: "In development",
    successCriteria: 0,
    newFeatures: [
      "New scoring model",
      "More comprehensive scope",
      "Easier to understand and apply",
    ],
  },
];
```

---

## WCAG Principles (POUR)

WCAG is organized around four principles:

### 1. Perceivable

Information and user interface components must be presentable to users in ways they can perceive.

```typescript
interface PerceivableGuidelines {
  "1.1": "Text Alternatives - Provide text alternatives for non-text content";
  "1.2": "Time-based Media - Provide alternatives for time-based media";
  "1.3": "Adaptable - Create content that can be presented in different ways";
  "1.4": "Distinguishable - Make it easier to see and hear content";
}

// Example: Text alternatives
const imageExample = `
<!-- ✓ Good: Informative image -->
<img src="chart.png" alt="Sales increased 50% from Q3 to Q4">

<!-- ✓ Good: Decorative image -->
<img src="decoration.png" alt="" role="presentation">

<!-- ✓ Good: Functional image -->
<button>
  <img src="save.png" alt="Save document">
</button>
`;
```

### 2. Operable

User interface components and navigation must be operable.

```typescript
interface OperableGuidelines {
  "2.1": "Keyboard Accessible - Make all functionality available from keyboard";
  "2.2": "Enough Time - Provide users enough time to read and use content";
  "2.3": "Seizures and Physical Reactions - Do not design content that causes seizures";
  "2.4": "Navigable - Provide ways to help users navigate and find content";
  "2.5": "Input Modalities - Make it easier to operate functionality through various inputs";
}

// Example: Keyboard accessible
const keyboardExample = `
<button onclick="handleClick()" onkeypress="handleKeyPress(event)">
  Click Me
</button>

<div role="button" 
     tabindex="0"
     onclick="handleClick()"
     onkeydown="handleKeyPress(event)">
  Custom Button
</div>
`;
```

### 3. Understandable

Information and the operation of user interface must be understandable.

```typescript
interface UnderstandableGuidelines {
  "3.1": "Readable - Make text content readable and understandable";
  "3.2": "Predictable - Make web pages appear and operate in predictable ways";
  "3.3": "Input Assistance - Help users avoid and correct mistakes";
}

// Example: Input assistance
const formExample = `
<form>
  <label for="email">Email Address *</label>
  <input type="email" 
         id="email"
         required
         aria-required="true"
         aria-invalid="false"
         aria-describedby="email-error email-help">
  <div id="email-help" class="help-text">
    We'll never share your email
  </div>
  <div id="email-error" role="alert" aria-live="assertive"></div>
</form>
`;
```

### 4. Robust

Content must be robust enough to be interpreted by a wide variety of user agents, including assistive technologies.

```typescript
interface RobustGuidelines {
  "4.1": "Compatible - Maximize compatibility with current and future user agents";
}

// Example: Valid HTML and ARIA
const robustExample = `
<!-- ✓ Good: Valid ARIA and HTML -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- ✓ Good: Custom widget with proper roles -->
<div role="tablist" aria-label="Settings tabs">
  <button role="tab" 
          aria-selected="true"
          aria-controls="general-panel"
          id="general-tab">
    General
  </button>
</div>
`;
```

---

## Conformance Levels

WCAG defines three levels of conformance:

### Level A (Minimum)

```typescript
interface LevelA {
  description: "Essential accessibility features";
  impact: "Severe barriers without these";
  examples: string[];
  totalCriteria: number;
}

const levelA: LevelA = {
  description: "Minimum level of accessibility",
  impact: "If not met, assistive technology users cannot access content",
  examples: [
    "1.1.1 Non-text Content",
    "2.1.1 Keyboard",
    "3.1.1 Language of Page",
    "4.1.2 Name, Role, Value",
  ],
  totalCriteria: 30,
};

// Critical Level A examples
const levelAExamples = {
  "1.1.1": `
    <!-- Non-text Content -->
    <img src="logo.png" alt="Company Name">
  `,

  "2.1.1": `
    <!-- Keyboard access -->
    <button onclick="save()">Save</button>
    <!-- Not: <div onclick="save()">Save</div> -->
  `,

  "3.1.1": `
    <!-- Language of Page -->
    <html lang="en">
  `,

  "4.1.2": `
    <!-- Name, Role, Value -->
    <button aria-label="Close dialog" aria-pressed="false">
      X
    </button>
  `,
};
```

### Level AA (Mid-Range)

```typescript
interface LevelAA {
  description: "Removes significant barriers";
  impact: "Recommended for most organizations";
  examples: string[];
  totalCriteria: number;
}

const levelAA: LevelAA = {
  description: "Deals with common barriers for disabled users",
  impact: "Most legal requirements target Level AA",
  examples: [
    "1.4.3 Contrast (Minimum)",
    "2.4.7 Focus Visible",
    "3.2.4 Consistent Identification",
    "4.1.3 Status Messages",
  ],
  totalCriteria: 20,
};

// Important Level AA examples
const levelAAExamples = {
  "1.4.3": `
    <!-- Contrast (Minimum) - 4.5:1 for normal text -->
    <style>
      .text {
        color: #595959; /* 7:1 ratio on white */
        background: #ffffff;
      }
    </style>
  `,

  "2.4.7": `
    <!-- Focus Visible -->
    <style>
      *:focus {
        outline: 2px solid #4A90E2;
        outline-offset: 2px;
      }
    </style>
  `,

  "3.2.4": `
    <!-- Consistent Identification -->
    <!-- Save icon always means "save" across the site -->
    <button aria-label="Save document">
      <i class="icon-save"></i>
    </button>
  `,

  "4.1.3": `
    <!-- Status Messages -->
    <div role="status" aria-live="polite">
      Changes saved successfully
    </div>
  `,
};
```

### Level AAA (Highest)

```typescript
interface LevelAAA {
  description: "Highest level of accessibility";
  impact: "Difficult to satisfy for all content";
  examples: string[];
  totalCriteria: number;
}

const levelAAA: LevelAAA = {
  description: "Enhanced accessibility for users with severe disabilities",
  impact: "Often impossible to satisfy for entire sites",
  examples: [
    "1.4.6 Contrast (Enhanced)",
    "2.4.9 Link Purpose (Link Only)",
    "3.1.5 Reading Level",
    "2.2.3 No Timing",
  ],
  totalCriteria: 28,
};

// Level AAA examples
const levelAAAExamples = {
  "1.4.6": `
    <!-- Contrast (Enhanced) - 7:1 for normal text -->
    <style>
      .text {
        color: #4A4A4A; /* 7.5:1 ratio on white */
        background: #ffffff;
      }
    </style>
  `,

  "2.4.9": `
    <!-- Link Purpose (Link Only) -->
    <!-- ❌ Bad -->
    <a href="/article1">Click here</a>
    
    <!-- ✓ Good -->
    <a href="/article1">Read our accessibility guide</a>
  `,

  "3.1.5": `
    <!-- Reading Level -->
    <!-- Text should not require advanced reading ability -->
    <!-- or provide supplementary content for complex topics -->
  `,
};
```

---

## WCAG 2.1 Success Criteria

### Complete Level A Criteria

```typescript
interface SuccessCriterion {
  id: string;
  title: string;
  level: "A" | "AA" | "AAA";
  principle: "Perceivable" | "Operable" | "Understandable" | "Robust";
  description: string;
  implementation: string;
}

const levelACriteria: SuccessCriterion[] = [
  {
    id: "1.1.1",
    title: "Non-text Content",
    level: "A",
    principle: "Perceivable",
    description: "All non-text content has a text alternative",
    implementation: "Use alt attributes, aria-label, or aria-labelledby",
  },
  {
    id: "1.2.1",
    title: "Audio-only and Video-only (Prerecorded)",
    level: "A",
    principle: "Perceivable",
    description: "Provide alternative for prerecorded audio/video-only media",
    implementation: "Provide transcript or audio description",
  },
  {
    id: "1.2.2",
    title: "Captions (Prerecorded)",
    level: "A",
    principle: "Perceivable",
    description: "Captions for all prerecorded audio in synchronized media",
    implementation: 'Add <track kind="captions"> to video elements',
  },
  {
    id: "1.2.3",
    title: "Audio Description or Media Alternative (Prerecorded)",
    level: "A",
    principle: "Perceivable",
    description: "Audio description or full text alternative for video",
    implementation: "Provide audio track describing visual information",
  },
  {
    id: "1.3.1",
    title: "Info and Relationships",
    level: "A",
    principle: "Perceivable",
    description:
      "Information, structure, and relationships can be programmatically determined",
    implementation: "Use semantic HTML, headings, lists, tables properly",
  },
  {
    id: "1.3.2",
    title: "Meaningful Sequence",
    level: "A",
    principle: "Perceivable",
    description: "Correct reading sequence can be programmatically determined",
    implementation: "Ensure DOM order matches visual order",
  },
  {
    id: "1.3.3",
    title: "Sensory Characteristics",
    level: "A",
    principle: "Perceivable",
    description: "Instructions don't rely solely on sensory characteristics",
    implementation: 'Don\'t say "click the round button" - use labels',
  },
  {
    id: "1.4.1",
    title: "Use of Color",
    level: "A",
    principle: "Perceivable",
    description: "Color is not the only means of conveying information",
    implementation: "Add icons, labels, or patterns alongside color",
  },
  {
    id: "1.4.2",
    title: "Audio Control",
    level: "A",
    principle: "Perceivable",
    description: "Provide mechanism to pause, stop, or control audio volume",
    implementation: "Add play/pause controls to audio that plays automatically",
  },
  {
    id: "2.1.1",
    title: "Keyboard",
    level: "A",
    principle: "Operable",
    description: "All functionality available from keyboard",
    implementation: "Ensure tabindex, onkeydown handlers for custom widgets",
  },
  {
    id: "2.1.2",
    title: "No Keyboard Trap",
    level: "A",
    principle: "Operable",
    description: "Focus can be moved away from any component",
    implementation: "Ensure Escape key or standard navigation works everywhere",
  },
  {
    id: "2.1.4",
    title: "Character Key Shortcuts",
    level: "A",
    principle: "Operable",
    description: "If single-character shortcuts exist, they can be turned off",
    implementation: "Provide settings to disable or remap shortcuts",
  },
  {
    id: "2.2.1",
    title: "Timing Adjustable",
    level: "A",
    principle: "Operable",
    description: "Time limits can be turned off, adjusted, or extended",
    implementation: "Allow users to extend session timeout",
  },
  {
    id: "2.2.2",
    title: "Pause, Stop, Hide",
    level: "A",
    principle: "Operable",
    description: "Moving, blinking, scrolling content can be paused",
    implementation: "Add pause button to carousels and animations",
  },
  {
    id: "2.3.1",
    title: "Three Flashes or Below Threshold",
    level: "A",
    principle: "Operable",
    description: "Content does not flash more than 3 times per second",
    implementation: "Avoid rapid flashing animations",
  },
  {
    id: "2.4.1",
    title: "Bypass Blocks",
    level: "A",
    principle: "Operable",
    description: "Mechanism to bypass blocks of repeated content",
    implementation: "Add skip links at top of page",
  },
  {
    id: "2.4.2",
    title: "Page Titled",
    level: "A",
    principle: "Operable",
    description: "Web pages have descriptive titles",
    implementation: "<title>Page Name - Site Name</title>",
  },
  {
    id: "2.4.3",
    title: "Focus Order",
    level: "A",
    principle: "Operable",
    description: "Focusable components receive focus in logical order",
    implementation: "Avoid positive tabindex values, keep DOM order logical",
  },
  {
    id: "2.4.4",
    title: "Link Purpose (In Context)",
    level: "A",
    principle: "Operable",
    description: "Purpose of link can be determined from link text or context",
    implementation: 'Use descriptive link text, not "click here"',
  },
  {
    id: "2.5.1",
    title: "Pointer Gestures",
    level: "A",
    principle: "Operable",
    description:
      "Multipoint or path-based gestures have single-pointer alternative",
    implementation: "Provide buttons for pinch-zoom, swipe actions",
  },
  {
    id: "2.5.2",
    title: "Pointer Cancellation",
    level: "A",
    principle: "Operable",
    description: "Down-event doesn't trigger action (use up-event)",
    implementation: "Use onclick instead of onmousedown",
  },
  {
    id: "2.5.3",
    title: "Label in Name",
    level: "A",
    principle: "Operable",
    description: "Accessible name contains visible label text",
    implementation:
      'If button shows "Submit", aria-label should include "Submit"',
  },
  {
    id: "2.5.4",
    title: "Motion Actuation",
    level: "A",
    principle: "Operable",
    description:
      "Functionality triggered by motion has UI component alternative",
    implementation: "If shake-to-undo exists, also provide undo button",
  },
  {
    id: "3.1.1",
    title: "Language of Page",
    level: "A",
    principle: "Understandable",
    description: "Default language of page can be programmatically determined",
    implementation: '<html lang="en">',
  },
  {
    id: "3.2.1",
    title: "On Focus",
    level: "A",
    principle: "Understandable",
    description: "Receiving focus doesn't initiate change of context",
    implementation: "Don't auto-submit forms or navigate on focus",
  },
  {
    id: "3.2.2",
    title: "On Input",
    level: "A",
    principle: "Understandable",
    description:
      "Changing setting doesn't automatically cause change of context",
    implementation: "Provide submit button, don't auto-submit on selection",
  },
  {
    id: "3.3.1",
    title: "Error Identification",
    level: "A",
    principle: "Understandable",
    description: "Input errors are identified and described to user",
    implementation: "Show clear error messages near form fields",
  },
  {
    id: "3.3.2",
    title: "Labels or Instructions",
    level: "A",
    principle: "Understandable",
    description: "Labels or instructions provided for user input",
    implementation: "Every form field has visible label",
  },
  {
    id: "4.1.1",
    title: "Parsing",
    level: "A",
    principle: "Robust",
    description: "Content can be parsed unambiguously",
    implementation: "Use valid HTML, no duplicate IDs",
  },
  {
    id: "4.1.2",
    title: "Name, Role, Value",
    level: "A",
    principle: "Robust",
    description: "Name and role can be programmatically determined",
    implementation: "Use semantic HTML or proper ARIA roles",
  },
];
```

### Complete Level AA Criteria (Additional)

```typescript
const levelAACriteria: SuccessCriterion[] = [
  {
    id: "1.2.4",
    title: "Captions (Live)",
    level: "AA",
    principle: "Perceivable",
    description: "Captions provided for all live audio in synchronized media",
    implementation: "Provide real-time captions for live video streams",
  },
  {
    id: "1.2.5",
    title: "Audio Description (Prerecorded)",
    level: "AA",
    principle: "Perceivable",
    description: "Audio description for all prerecorded video",
    implementation: "Provide descriptive audio track",
  },
  {
    id: "1.3.4",
    title: "Orientation",
    level: "AA",
    principle: "Perceivable",
    description: "Content does not restrict view to single orientation",
    implementation: "Support both portrait and landscape",
  },
  {
    id: "1.3.5",
    title: "Identify Input Purpose",
    level: "AA",
    principle: "Perceivable",
    description:
      "Input fields collecting user info have autocomplete attribute",
    implementation: '<input autocomplete="email" type="email">',
  },
  {
    id: "1.4.3",
    title: "Contrast (Minimum)",
    level: "AA",
    principle: "Perceivable",
    description: "4.5:1 contrast ratio for normal text, 3:1 for large text",
    implementation: "Use sufficient color contrast for text",
  },
  {
    id: "1.4.4",
    title: "Resize Text",
    level: "AA",
    principle: "Perceivable",
    description: "Text can be resized up to 200% without loss of content",
    implementation: "Use relative units (rem, em) for font sizes",
  },
  {
    id: "1.4.5",
    title: "Images of Text",
    level: "AA",
    principle: "Perceivable",
    description: "Use actual text rather than images of text",
    implementation: "Use web fonts instead of text in images",
  },
  {
    id: "1.4.10",
    title: "Reflow",
    level: "AA",
    principle: "Perceivable",
    description: "Content reflows to 320px width without horizontal scrolling",
    implementation: "Responsive design with mobile-first approach",
  },
  {
    id: "1.4.11",
    title: "Non-text Contrast",
    level: "AA",
    principle: "Perceivable",
    description: "3:1 contrast ratio for UI components and graphics",
    implementation: "Ensure form borders, buttons have sufficient contrast",
  },
  {
    id: "1.4.12",
    title: "Text Spacing",
    level: "AA",
    principle: "Perceivable",
    description: "No loss of content when user adjusts text spacing",
    implementation: "Avoid fixed heights, allow text to reflow",
  },
  {
    id: "1.4.13",
    title: "Content on Hover or Focus",
    level: "AA",
    principle: "Perceivable",
    description:
      "Additional content on hover/focus is dismissible and hoverable",
    implementation:
      "Tooltips can be dismissed with Escape, pointer can move over them",
  },
  {
    id: "2.4.5",
    title: "Multiple Ways",
    level: "AA",
    principle: "Operable",
    description: "More than one way to locate pages within a set",
    implementation: "Provide site map, search, navigation menu",
  },
  {
    id: "2.4.6",
    title: "Headings and Labels",
    level: "AA",
    principle: "Operable",
    description: "Headings and labels describe topic or purpose",
    implementation: "Use descriptive, unique headings",
  },
  {
    id: "2.4.7",
    title: "Focus Visible",
    level: "AA",
    principle: "Operable",
    description: "Keyboard focus indicator is visible",
    implementation: "Never remove outline without replacement",
  },
  {
    id: "3.1.2",
    title: "Language of Parts",
    level: "AA",
    principle: "Understandable",
    description: "Language of passages can be programmatically determined",
    implementation: '<span lang="es">Hola</span>',
  },
  {
    id: "3.2.3",
    title: "Consistent Navigation",
    level: "AA",
    principle: "Understandable",
    description: "Navigation mechanisms repeated consistently",
    implementation: "Keep navigation menu in same order across pages",
  },
  {
    id: "3.2.4",
    title: "Consistent Identification",
    level: "AA",
    principle: "Understandable",
    description: "Components with same functionality identified consistently",
    implementation: 'Use same icon and label for "save" everywhere',
  },
  {
    id: "3.3.3",
    title: "Error Suggestion",
    level: "AA",
    principle: "Understandable",
    description: "Input errors have suggestions provided",
    implementation: "Suggest corrections for form validation errors",
  },
  {
    id: "3.3.4",
    title: "Error Prevention (Legal, Financial, Data)",
    level: "AA",
    principle: "Understandable",
    description: "Submissions can be reviewed, confirmed, or reversed",
    implementation: "Add confirmation step before final submission",
  },
  {
    id: "4.1.3",
    title: "Status Messages",
    level: "AA",
    principle: "Robust",
    description: "Status messages can be programmatically determined",
    implementation: 'Use role="status" or role="alert" for dynamic messages',
  },
];
```

---

## WCAG 2.2 New Features

```typescript
const wcag22NewCriteria: SuccessCriterion[] = [
  {
    id: "2.4.11",
    title: "Focus Not Obscured (Minimum)",
    level: "AA",
    principle: "Operable",
    description:
      "Focused element is not entirely hidden by author-created content",
    implementation: `
      // Ensure sticky headers don't cover focused elements
      element.addEventListener('focus', (e) => {
        e.target.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
      });
    `,
  },
  {
    id: "2.4.12",
    title: "Focus Not Obscured (Enhanced)",
    level: "AAA",
    principle: "Operable",
    description:
      "No part of focus indicator is hidden by author-created content",
    implementation:
      "Ensure focused element is fully visible, accounting for sticky elements",
  },
  {
    id: "2.4.13",
    title: "Focus Appearance",
    level: "AAA",
    principle: "Operable",
    description: "Focus indicator meets minimum size and contrast requirements",
    implementation: `
      /* Focus indicator must be at least 2px thick and 3:1 contrast */
      *:focus-visible {
        outline: 2px solid #0066CC;
        outline-offset: 2px;
      }
    `,
  },
  {
    id: "2.5.7",
    title: "Dragging Movements",
    level: "AA",
    principle: "Operable",
    description: "Dragging actions have single pointer alternative",
    implementation: `
      <!-- Provide buttons as alternative to drag-and-drop -->
      <div class="sortable-item">
        Item 1
        <button aria-label="Move up">↑</button>
        <button aria-label="Move down">↓</button>
      </div>
    `,
  },
  {
    id: "2.5.8",
    title: "Target Size (Minimum)",
    level: "AA",
    principle: "Operable",
    description: "Target size is at least 24x24 CSS pixels",
    implementation: `
      /* Ensure interactive elements are large enough */
      button, a, input {
        min-width: 24px;
        min-height: 24px;
      }
    `,
  },
  {
    id: "3.2.6",
    title: "Consistent Help",
    level: "A",
    principle: "Understandable",
    description: "Help mechanisms in same relative order across pages",
    implementation:
      "Place help link/button in consistent location (e.g., top-right)",
  },
  {
    id: "3.3.7",
    title: "Redundant Entry",
    level: "A",
    principle: "Understandable",
    description: "Information previously entered can be auto-populated",
    implementation: `
      <!-- Use autocomplete or remember previous entries -->
      <input type="text" 
             name="email"
             autocomplete="email">
    `,
  },
  {
    id: "3.3.8",
    title: "Accessible Authentication (Minimum)",
    level: "AA",
    principle: "Understandable",
    description: "Cognitive function test not required for authentication",
    implementation: `
      <!-- Don't require CAPTCHA or memorization -->
      <!-- Provide alternatives like:
           - Email magic link
           - SMS code
           - Biometric authentication
           - Password managers
      -->
    `,
  },
  {
    id: "3.3.9",
    title: "Accessible Authentication (Enhanced)",
    level: "AAA",
    principle: "Understandable",
    description: "No cognitive function test required for any step",
    implementation:
      "Support password managers, biometrics, or copy-paste for all auth steps",
  },
];
```

### WCAG 2.2 Implementation Example

```typescript
class WCAG22Compliant {
  // 2.4.11: Focus Not Obscured (Minimum)
  ensureFocusVisible(): void {
    document.addEventListener("focusin", (e) => {
      const element = e.target as HTMLElement;
      const stickyHeaderHeight = 80; // Height of sticky header

      const rect = element.getBoundingClientRect();
      const isObscured = rect.top < stickyHeaderHeight;

      if (isObscured) {
        window.scrollBy({
          top: rect.top - stickyHeaderHeight - 20,
          behavior: "smooth",
        });
      }
    });
  }

  // 2.5.7: Dragging Movements
  implementSortableList(): void {
    const container = document.getElementById("sortable-list")!;
    const items = Array.from(container.children);

    items.forEach((item, index) => {
      // Add drag functionality
      item.setAttribute("draggable", "true");

      // Add button alternative
      const moveUpBtn = document.createElement("button");
      moveUpBtn.textContent = "↑";
      moveUpBtn.setAttribute("aria-label", "Move up");
      moveUpBtn.onclick = () => this.moveItem(index, -1);

      const moveDownBtn = document.createElement("button");
      moveDownBtn.textContent = "↓";
      moveDownBtn.setAttribute("aria-label", "Move down");
      moveDownBtn.onclick = () => this.moveItem(index, 1);

      item.appendChild(moveUpBtn);
      item.appendChild(moveDownBtn);
    });
  }

  private moveItem(index: number, direction: number): void {
    // Implementation for moving items
  }

  // 2.5.8: Target Size (Minimum)
  ensureTargetSize(): void {
    const style = document.createElement("style");
    style.textContent = `
      button, a, input[type="checkbox"], input[type="radio"] {
        min-width: 24px;
        min-height: 24px;
        padding: 8px;
      }
      
      /* Touch targets on mobile should be larger */
      @media (hover: none) and (pointer: coarse) {
        button, a {
          min-width: 44px;
          min-height: 44px;
        }
      }
    `;
    document.head.appendChild(style);
  }

  // 3.3.7: Redundant Entry
  implementAutofill(): void {
    const form = document.querySelector("form")!;

    // Use autocomplete attributes
    const inputs = form.querySelectorAll("input");
    inputs.forEach((input) => {
      const name = input.getAttribute("name");

      const autocompleteMap: Record<string, string> = {
        email: "email",
        name: "name",
        firstName: "given-name",
        lastName: "family-name",
        phone: "tel",
        address: "street-address",
        city: "address-level2",
        zip: "postal-code",
        country: "country-name",
      };

      if (name && autocompleteMap[name]) {
        input.setAttribute("autocomplete", autocompleteMap[name]);
      }
    });
  }

  // 3.3.8: Accessible Authentication (Minimum)
  implementAccessibleAuth(): void {
    const authForm = document.getElementById("auth-form")!;

    // Provide multiple authentication methods
    const methods = [
      {
        id: "password",
        label: "Password",
        type: "password",
        autocomplete: "current-password",
        description: "Use your password manager",
      },
      {
        id: "magic-link",
        label: "Email Magic Link",
        type: "email",
        autocomplete: "email",
        description: "We'll send you a link to sign in",
      },
      {
        id: "sms",
        label: "SMS Code",
        type: "tel",
        autocomplete: "tel",
        description: "We'll text you a code",
      },
    ];

    methods.forEach((method) => {
      const wrapper = document.createElement("div");
      wrapper.innerHTML = `
        <input type="radio" 
               id="auth-${method.id}"
               name="auth-method"
               value="${method.id}">
        <label for="auth-${method.id}">
          ${method.label}
          <span class="help-text">${method.description}</span>
        </label>
      `;
      authForm.appendChild(wrapper);
    });
  }
}

// Initialize
const wcag22 = new WCAG22Compliant();
wcag22.ensureFocusVisible();
wcag22.ensureTargetSize();
```

---

## Implementation Examples

### Complete Accessible Form

```html
<form id="registration-form" novalidate>
  <h2>Registration Form</h2>

  <!-- 3.3.2: Labels or Instructions (Level A) -->
  <!-- 3.3.7: Redundant Entry (Level A WCAG 2.2) -->
  <div class="form-group">
    <label for="first-name">
      First Name
      <abbr title="required" aria-label="required">*</abbr>
    </label>
    <input
      type="text"
      id="first-name"
      name="firstName"
      required
      aria-required="true"
      aria-invalid="false"
      aria-describedby="first-name-help first-name-error"
      autocomplete="given-name"
    />
    <div id="first-name-help" class="help-text">Enter your given name</div>
    <div
      id="first-name-error"
      role="alert"
      aria-live="assertive"
      class="error hidden"
    ></div>
  </div>

  <!-- 1.3.5: Identify Input Purpose (Level AA) -->
  <div class="form-group">
    <label for="email">
      Email Address
      <abbr title="required" aria-label="required">*</abbr>
    </label>
    <input
      type="email"
      id="email"
      name="email"
      required
      aria-required="true"
      aria-invalid="false"
      aria-describedby="email-help email-error"
      autocomplete="email"
    />
    <div id="email-help" class="help-text">We'll never share your email</div>
    <div
      id="email-error"
      role="alert"
      aria-live="assertive"
      class="error hidden"
    ></div>
  </div>

  <!-- Password field with show/hide -->
  <div class="form-group">
    <label for="password">
      Password
      <abbr title="required" aria-label="required">*</abbr>
    </label>
    <div class="input-wrapper">
      <input
        type="password"
        id="password"
        name="password"
        required
        aria-required="true"
        aria-invalid="false"
        aria-describedby="password-help password-error"
        autocomplete="new-password"
        minlength="8"
      />
      <button
        type="button"
        aria-label="Show password"
        aria-pressed="false"
        onclick="togglePasswordVisibility()"
      >
        Show
      </button>
    </div>
    <div id="password-help" class="help-text">
      Must be at least 8 characters
    </div>
    <div
      id="password-error"
      role="alert"
      aria-live="assertive"
      class="error hidden"
    ></div>
  </div>

  <!-- 3.3.1: Error Identification (Level A) -->
  <!-- 3.3.3: Error Suggestion (Level AA) -->
  <div role="alert" aria-live="assertive" class="form-errors hidden">
    <h3>Please fix the following errors:</h3>
    <ul id="error-summary"></ul>
  </div>

  <!-- 3.3.4: Error Prevention (Level AA) -->
  <div class="form-actions">
    <button type="submit">Submit Registration</button>
    <button type="button" onclick="resetForm()">Clear Form</button>
  </div>

  <!-- 4.1.3: Status Messages (Level AA) -->
  <div
    role="status"
    aria-live="polite"
    aria-atomic="true"
    class="sr-only"
    id="form-status"
  ></div>
</form>
```

```typescript
class AccessibleForm {
  private form: HTMLFormElement;
  private inputs: NodeListOf<HTMLInputElement>;

  constructor(formId: string) {
    this.form = document.getElementById(formId) as HTMLFormElement;
    this.inputs = this.form.querySelectorAll("input[required]");
    this.setupValidation();
  }

  private setupValidation(): void {
    this.form.addEventListener("submit", (e) => {
      e.preventDefault();
      this.validateForm();
    });

    // Real-time validation
    this.inputs.forEach((input) => {
      input.addEventListener("blur", () => {
        this.validateField(input);
      });

      input.addEventListener("input", () => {
        // Clear error on input
        if (input.getAttribute("aria-invalid") === "true") {
          this.clearFieldError(input);
        }
      });
    });
  }

  private validateForm(): boolean {
    let isValid = true;
    const errors: string[] = [];

    this.inputs.forEach((input) => {
      if (!this.validateField(input)) {
        isValid = false;
        const label = this.form.querySelector(
          `label[for="${input.id}"]`,
        )?.textContent;
        errors.push(`${label}: ${this.getErrorMessage(input)}`);
      }
    });

    if (!isValid) {
      this.showErrorSummary(errors);
      this.announceErrors(errors.length);
    } else {
      this.submitForm();
    }

    return isValid;
  }

  private validateField(input: HTMLInputElement): boolean {
    const value = input.value.trim();

    // Required validation
    if (input.required && !value) {
      this.showFieldError(input, "This field is required");
      return false;
    }

    // Email validation
    if (input.type === "email" && value) {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      if (!emailRegex.test(value)) {
        this.showFieldError(
          input,
          "Please enter a valid email address (e.g., user@example.com)",
        );
        return false;
      }
    }

    // Password validation
    if (input.type === "password" && value) {
      if (value.length < 8) {
        this.showFieldError(
          input,
          "Password must be at least 8 characters long",
        );
        return false;
      }
    }

    this.clearFieldError(input);
    return true;
  }

  private showFieldError(input: HTMLInputElement, message: string): void {
    // Update aria-invalid
    input.setAttribute("aria-invalid", "true");
    input.classList.add("error");

    // Show error message
    const errorId = `${input.id}-error`;
    const errorDiv = document.getElementById(errorId);
    if (errorDiv) {
      errorDiv.textContent = message;
      errorDiv.classList.remove("hidden");
    }
  }

  private clearFieldError(input: HTMLInputElement): void {
    input.setAttribute("aria-invalid", "false");
    input.classList.remove("error");

    const errorId = `${input.id}-error`;
    const errorDiv = document.getElementById(errorId);
    if (errorDiv) {
      errorDiv.textContent = "";
      errorDiv.classList.add("hidden");
    }
  }

  private getErrorMessage(input: HTMLInputElement): string {
    const errorId = `${input.id}-error`;
    const errorDiv = document.getElementById(errorId);
    return errorDiv?.textContent || "Invalid input";
  }

  private showErrorSummary(errors: string[]): void {
    const summary = document.getElementById("error-summary")!;
    summary.innerHTML = errors.map((error) => `<li>${error}</li>`).join("");

    const container = summary.closest(".form-errors")!;
    container.classList.remove("hidden");

    // Focus first error
    const firstErrorInput = this.form.querySelector(
      '[aria-invalid="true"]',
    ) as HTMLElement;
    firstErrorInput?.focus();
  }

  private announceErrors(count: number): void {
    const status = document.getElementById("form-status")!;
    status.textContent = `Form has ${count} error${count > 1 ? "s" : ""}. Please review and correct.`;
  }

  private submitForm(): void {
    // Submit logic
    const status = document.getElementById("form-status")!;
    status.textContent = "Form submitted successfully!";

    // Clear form
    this.form.reset();
  }
}

// Initialize
const registrationForm = new AccessibleForm("registration-form");
```

---

## Compliance Testing

### Automated Testing Tools

```typescript
interface A11yTestingTool {
  name: string;
  type: "Browser Extension" | "Library" | "CLI" | "Service";
  coverage: string;
  wcagLevel: string[];
}

const testingTools: A11yTestingTool[] = [
  {
    name: "axe DevTools",
    type: "Browser Extension",
    coverage: "Catches ~57% of WCAG issues",
    wcagLevel: ["A", "AA", "AAA"],
  },
  {
    name: "WAVE",
    type: "Browser Extension",
    coverage: "Visual feedback for accessibility",
    wcagLevel: ["A", "AA"],
  },
  {
    name: "Lighthouse",
    type: "Browser Extension",
    coverage: "Built into Chrome DevTools",
    wcagLevel: ["A", "AA"],
  },
  {
    name: "Pa11y",
    type: "CLI",
    coverage: "Command-line testing",
    wcagLevel: ["A", "AA", "AAA"],
  },
  {
    name: "@axe-core/react",
    type: "Library",
    coverage: "Runtime testing in React",
    wcagLevel: ["A", "AA", "AAA"],
  },
  {
    name: "cypress-axe",
    type: "Library",
    coverage: "Integration with Cypress",
    wcagLevel: ["A", "AA", "AAA"],
  },
];
```

### Testing Checklist

```typescript
interface WCAGTestChecklist {
  criterion: string;
  level: "A" | "AA" | "AAA";
  automated: boolean;
  manual: boolean;
  testMethod: string;
}

const testingChecklist: WCAGTestChecklist[] = [
  {
    criterion: "1.1.1 Non-text Content",
    level: "A",
    automated: true,
    manual: true,
    testMethod:
      "Check alt attributes with tool; verify meaningfulness manually",
  },
  {
    criterion: "1.4.3 Contrast (Minimum)",
    level: "AA",
    automated: true,
    manual: false,
    testMethod: "Use color contrast checker tool",
  },
  {
    criterion: "2.1.1 Keyboard",
    level: "A",
    automated: false,
    manual: true,
    testMethod: "Navigate entire site with keyboard only",
  },
  {
    criterion: "2.4.7 Focus Visible",
    level: "AA",
    automated: true,
    manual: true,
    testMethod: "Tool checks for outline; manually verify visibility",
  },
  {
    criterion: "3.3.2 Labels or Instructions",
    level: "A",
    automated: true,
    manual: true,
    testMethod: "Tool checks for labels; manually verify clarity",
  },
  {
    criterion: "4.1.2 Name, Role, Value",
    level: "A",
    automated: true,
    manual: true,
    testMethod: "Tool checks ARIA; manually test with screen reader",
  },
];
```

---

## Legal Requirements

### Global Accessibility Laws

```typescript
interface AccessibilityLaw {
  region: string;
  law: string;
  scope: string;
  standard: string;
  enforcement: string;
}

const globalLaws: AccessibilityLaw[] = [
  {
    region: "United States",
    law: "ADA (Americans with Disabilities Act)",
    scope: "All public accommodations",
    standard: "WCAG 2.1 Level AA (commonly cited)",
    enforcement: "Department of Justice; private lawsuits",
  },
  {
    region: "United States",
    law: "Section 508",
    scope: "Federal agencies and contractors",
    standard: "WCAG 2.0 Level AA (Section 508 refresh)",
    enforcement: "Federal government",
  },
  {
    region: "European Union",
    law: "European Accessibility Act (EAA)",
    scope: "Products and services (effective June 2025)",
    standard: "EN 301 549 (references WCAG 2.1 AA)",
    enforcement: "Member states",
  },
  {
    region: "European Union",
    law: "Web Accessibility Directive",
    scope: "Public sector bodies",
    standard: "WCAG 2.1 Level AA",
    enforcement: "EU member states",
  },
  {
    region: "United Kingdom",
    law: "Equality Act 2010",
    scope: "Service providers",
    standard: "WCAG 2.1 Level AA",
    enforcement: "Equality and Human Rights Commission",
  },
  {
    region: "Canada",
    law: "Accessible Canada Act (ACA)",
    scope: "Federally regulated entities",
    standard: "WCAG 2.0 Level AA (moving to 2.1)",
    enforcement: "Canadian Accessibility Standards",
  },
  {
    region: "Australia",
    law: "Disability Discrimination Act 1992",
    scope: "All organizations providing services",
    standard: "WCAG 2.1 Level AA",
    enforcement: "Australian Human Rights Commission",
  },
  {
    region: "Japan",
    law: "JIS X 8341-3",
    scope: "Government and large corporations",
    standard: "JIS X 8341-3:2016 (harmonized with WCAG 2.0)",
    enforcement: "Voluntary with government pressure",
  },
];
```

---

## Best Practices

### Accessibility-First Development

```typescript
class AccessibilityFirstDevelopment {
  // 1. Start with semantic HTML
  static useSemanticHTML(): string {
    return `
      <!-- ✓ Good: Semantic structure -->
      <header>
        <nav aria-label="Main">
          <ul>
            <li><a href="/">Home</a></li>
          </ul>
        </nav>
      </header>
      
      <main>
        <article>
          <h1>Page Title</h1>
          <p>Content...</p>
        </article>
      </main>
      
      <footer>
        <p>&copy; 2026</p>
      </footer>
    `;
  }

  // 2. Progressive enhancement
  static progressiveEnhancement(): void {
    // Base functionality works without JavaScript
    const button = document.querySelector("button")!;

    // Enhance with JavaScript
    button.addEventListener("click", () => {
      // Enhanced functionality
    });

    // Degrade gracefully if JS fails
    button.setAttribute("formaction", "/fallback-submit");
  }

  // 3. Test early and often
  static integrateTestingEarly(): void {
    // Add accessibility tests to CI/CD
    // Run automated tools on every commit
    // Manual testing in sprint reviews
  }

  // 4. Include users with disabilities
  static includDisabledUsers(): void {
    // User testing with screen reader users
    // Feedback from users with various disabilities
    // Accessibility consultants review
  }
}
```

---

## Key Takeaways

1. **WCAG is organized by POUR principles** - Perceivable, Operable, Understandable, and Robust provide the foundation for all accessibility requirements.

2. **Three conformance levels exist** - Level A (minimum), Level AA (recommended target for most), and Level AAA (enhanced accessibility for specialized needs).

3. **Level AA is the legal standard** - Most accessibility laws (ADA, EAA, Section 508) require WCAG 2.1 Level AA compliance.

4. **WCAG 2.2 adds 9 new criteria** - Focus on mobile, authentication, focus visibility, and reducing cognitive load for users.

5. **Automated tools catch ~30-50% of issues** - Manual testing, keyboard navigation, and screen reader testing are essential for full compliance.

6. **Semantic HTML is the foundation** - Use native HTML elements before adding ARIA; proper structure satisfies many WCAG criteria automatically.

7. **Document compliance systematically** - Maintain VPAT (Voluntary Product Accessibility Template) or ACR (Accessibility Conformance Report) for enterprise clients.

8. **Test with real users when possible** - Automated tools and expert reviews are valuable, but users with disabilities provide irreplaceable insights.

9. **Accessibility is iterative** - Start with Level A, progress to AA, address issues as discovered, and continuously improve.

10. **Plan for global compliance** - Different regions have different requirements; WCAG 2.1 Level AA generally satisfies most international laws.
