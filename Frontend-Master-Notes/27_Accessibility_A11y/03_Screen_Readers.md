# Screen Readers - Testing with Assistive Technologies

## Table of Contents

1. [Introduction to Screen Readers](#introduction-to-screen-readers)
2. [Major Screen Readers](#major-screen-readers)
3. [NVDA (Windows)](#nvda-windows)
4. [JAWS (Windows)](#jaws-windows)
5. [VoiceOver (macOS/iOS)](#voiceover-macos-ios)
6. [Testing Strategies](#testing-strategies)
7. [Common Issues and Solutions](#common-issues-and-solutions)
8. [Screen Reader Specific Patterns](#screen-reader-specific-patterns)
9. [Automated Testing](#automated-testing)
10. [Key Takeaways](#key-takeaways)

---

## Introduction to Screen Readers

**Screen readers** are assistive technologies that convert text and interface elements into synthesized speech or Braille output for users who are blind or have low vision.

### How Screen Readers Work

```typescript
interface ScreenReaderProcess {
  1: "Parse HTML DOM and accessibility tree";
  2: "Build semantic structure from ARIA and native semantics";
  3: "Expose information through platform accessibility APIs";
  4: "Convert to speech or Braille output";
  5: "Respond to user commands (keyboard/touch)";
}

interface AccessibilityTree {
  role: string;
  name: string;
  state: string[];
  properties: Record<string, any>;
  children: AccessibilityTree[];
}
```

### Global Screen Reader Statistics (2024-2026)

```typescript
const screenReaderUsage = {
  JAWS: { percentage: 40.5, platform: "Windows", cost: "Commercial" },
  NVDA: { percentage: 35.8, platform: "Windows", cost: "Free" },
  VoiceOver: { percentage: 12.7, platform: "macOS/iOS", cost: "Built-in" },
  Narrator: { percentage: 4.1, platform: "Windows", cost: "Built-in" },
  TalkBack: { percentage: 3.9, platform: "Android", cost: "Built-in" },
  Others: { percentage: 3.0, platform: "Various", cost: "Varies" },
};
```

### WCAG Requirements

- **1.3.1 Info and Relationships (Level A)**: Information, structure, and relationships must be programmatically determined
- **4.1.2 Name, Role, Value (Level A)**: All UI components must have accessible names and roles
- **4.1.3 Status Messages (Level AA)**: Status messages can be programmatically determined

---

## Major Screen Readers

### Comparison Matrix

```typescript
interface ScreenReaderComparison {
  name: string;
  platform: string;
  cost: string;
  marketShare: number;
  keyFeatures: string[];
  bestFor: string;
}

const screenReaders: ScreenReaderComparison[] = [
  {
    name: "JAWS",
    platform: "Windows",
    cost: "$90-$1695",
    marketShare: 40.5,
    keyFeatures: [
      "Most feature-rich",
      "Excellent form handling",
      "Strong web application support",
      "Custom scripts support",
    ],
    bestFor: "Professional environments, complex applications",
  },
  {
    name: "NVDA",
    platform: "Windows",
    cost: "Free",
    marketShare: 35.8,
    keyFeatures: [
      "Open source",
      "Active development",
      "Good ARIA support",
      "Community plugins",
    ],
    bestFor: "Testing, general use, budget-conscious users",
  },
  {
    name: "VoiceOver",
    platform: "macOS/iOS",
    cost: "Built-in",
    marketShare: 12.7,
    keyFeatures: [
      "Deep OS integration",
      "Excellent mobile experience",
      "Gesture support",
      "Braille display support",
    ],
    bestFor: "Apple ecosystem users, mobile testing",
  },
  {
    name: "Narrator",
    platform: "Windows 10/11",
    cost: "Built-in",
    marketShare: 4.1,
    keyFeatures: [
      "Native Windows integration",
      "Improving rapidly",
      "Basic functionality",
    ],
    bestFor: "Basic testing, Windows users",
  },
  {
    name: "TalkBack",
    platform: "Android",
    cost: "Built-in",
    marketShare: 3.9,
    keyFeatures: [
      "Native Android integration",
      "Touch gestures",
      "Good localization",
    ],
    bestFor: "Android app testing",
  },
];
```

---

## NVDA (Windows)

### Installation and Setup

```powershell
# Download from: https://www.nvaccess.org/download/
# Or install via Chocolatey:
choco install nvda

# Portable version available for testing
# No installation required
```

### Essential NVDA Commands

```typescript
interface NVDACommands {
  // Basic Navigation
  "NVDA + Down Arrow": "Say all (read from current position)";
  "NVDA + Up Arrow": "Read current line";
  "NVDA + T": "Read window title";
  "NVDA + B": "Read status bar";

  // Element Navigation
  H: "Next heading";
  "Shift + H": "Previous heading";
  "1-6": "Next heading of level 1-6";
  L: "Next list";
  I: "Next list item";
  K: "Next link";
  F: "Next form field";
  B: "Next button";
  E: "Next edit field";
  C: "Next combo box";
  R: "Next radio button";
  X: "Next checkbox";
  T: "Next table";
  G: "Next graphic";
  D: "Next landmark";

  // Forms Mode
  "NVDA + Space": "Toggle forms mode / browse mode";

  // Tables
  "Ctrl + Alt + Arrow Keys": "Navigate table cells";
  "NVDA + T": "Table title";
  "NVDA + Shift + T": "Table summary";

  // Element List
  "NVDA + F7": "Elements list (headings, links, landmarks)";

  // Control
  "NVDA + N": "NVDA menu";
  "NVDA + Q": "Quit NVDA";
  "NVDA + Ctrl + C": "Copy to clipboard (for testing)";
}
```

### NVDA Configuration for Testing

```typescript
class NVDATestSetup {
  static configure(): void {
    console.log(`
NVDA Configuration for Web Testing:

1. Open NVDA Settings (NVDA + N > Preferences > Settings)

2. Speech Settings:
   - Rate: Medium (for testing comprehension)
   - Pitch: Medium
   - Enable "Say all by paragraph"

3. Keyboard Settings:
   - Enable "Speak typed characters"
   - Enable "Speak typed words"
   - Enable "Speak command keys"

4. Browse Mode Settings:
   - Enable "Use screen layout"
   - Enable "Automatic focus mode for focus changes"
   - Table cell reading: "Row and column"

5. Document Formatting:
   ✓ Report headings
   ✓ Report links
   ✓ Report lists
   ✓ Report tables
   ✓ Report frame and iframe
   ✓ Report landmarks
   ✓ Report ARIA annotations
    `);
  }
}
```

### NVDA Testing Script

```typescript
class NVDATestRunner {
  private testResults: Map<string, boolean> = new Map();

  async runFullTest(url: string): Promise<void> {
    console.log(`Testing: ${url}`);

    await this.testPageStructure();
    await this.testHeadings();
    await this.testLandmarks();
    await this.testLinks();
    await this.testForms();
    await this.testTables();
    await this.testImages();
    await this.testARIA();

    this.printResults();
  }

  private async testPageStructure(): Promise<void> {
    console.log("\n=== Testing Page Structure ===");

    // Test: Page has title
    const hasTitle = document.title.length > 0;
    this.testResults.set("Page title exists", hasTitle);
    console.log(`Page title: ${document.title}`);

    // Test: Page has main landmark
    const hasMain = document.querySelector('main, [role="main"]') !== null;
    this.testResults.set("Main landmark exists", hasMain);

    // Test: Page has proper language
    const hasLang = document.documentElement.lang.length > 0;
    this.testResults.set("Language attribute set", hasLang);
  }

  private async testHeadings(): Promise<void> {
    console.log("\n=== Testing Headings ===");

    const headings = Array.from(
      document.querySelectorAll('h1, h2, h3, h4, h5, h6, [role="heading"]'),
    );

    console.log(`Found ${headings.length} headings`);

    // Test: Has H1
    const hasH1 = document.querySelector("h1") !== null;
    this.testResults.set("H1 exists", hasH1);

    // Test: Heading hierarchy
    let lastLevel = 0;
    let hierarchyValid = true;

    headings.forEach((heading) => {
      const level = this.getHeadingLevel(heading);
      console.log(`H${level}: ${heading.textContent?.trim()}`);

      if (level > lastLevel + 1) {
        hierarchyValid = false;
      }
      lastLevel = level;
    });

    this.testResults.set("Heading hierarchy valid", hierarchyValid);
  }

  private getHeadingLevel(element: Element): number {
    const ariaLevel = element.getAttribute("aria-level");
    if (ariaLevel) return parseInt(ariaLevel);

    const match = element.tagName.match(/H(\d)/);
    return match ? parseInt(match[1]) : 0;
  }

  private async testLandmarks(): Promise<void> {
    console.log("\n=== Testing Landmarks ===");

    const landmarks = [
      { selector: 'header, [role="banner"]', name: "Banner" },
      { selector: 'nav, [role="navigation"]', name: "Navigation" },
      { selector: 'main, [role="main"]', name: "Main" },
      { selector: 'aside, [role="complementary"]', name: "Complementary" },
      { selector: 'footer, [role="contentinfo"]', name: "Content Info" },
      { selector: '[role="search"]', name: "Search" },
    ];

    landmarks.forEach(({ selector, name }) => {
      const elements = document.querySelectorAll(selector);
      console.log(`${name}: ${elements.length} found`);

      elements.forEach((el, i) => {
        const label =
          el.getAttribute("aria-label") || el.getAttribute("aria-labelledby");
        if (label) {
          console.log(`  [${i}] Label: ${label}`);
        }
      });
    });
  }

  private async testLinks(): Promise<void> {
    console.log("\n=== Testing Links ===");

    const links = Array.from(document.querySelectorAll("a[href]"));
    console.log(`Found ${links.length} links`);

    // Test: No empty links
    const emptyLinks = links.filter((link) => {
      const text = link.textContent?.trim();
      const ariaLabel = link.getAttribute("aria-label");
      const title = link.getAttribute("title");
      return !text && !ariaLabel && !title;
    });

    this.testResults.set("No empty links", emptyLinks.length === 0);

    if (emptyLinks.length > 0) {
      console.log(`⚠️ Found ${emptyLinks.length} empty links`);
    }

    // Test: Link purpose clear
    const unclearLinks = links.filter((link) => {
      const text = link.textContent?.trim().toLowerCase();
      return text === "click here" || text === "read more" || text === "here";
    });

    if (unclearLinks.length > 0) {
      console.log(`⚠️ Found ${unclearLinks.length} unclear link texts`);
      unclearLinks.forEach((link) => {
        console.log(
          `  - "${link.textContent?.trim()}" -> ${link.getAttribute("href")}`,
        );
      });
    }
  }

  private async testForms(): Promise<void> {
    console.log("\n=== Testing Forms ===");

    const inputs = Array.from(
      document.querySelectorAll("input, select, textarea"),
    );

    console.log(`Found ${inputs.length} form controls`);

    inputs.forEach((input) => {
      const htmlInput = input as HTMLInputElement;
      const id = htmlInput.id;
      const name = htmlInput.name;
      const type = htmlInput.type;

      // Check for label
      const label =
        document.querySelector(`label[for="${id}"]`) || input.closest("label");
      const ariaLabel = input.getAttribute("aria-label");
      const ariaLabelledby = input.getAttribute("aria-labelledby");

      const hasLabel = !!(label || ariaLabel || ariaLabelledby);

      console.log(`${type} - ${name || id}: ${hasLabel ? "✓" : "✗"} labeled`);

      if (!hasLabel) {
        console.log(`  ⚠️ Missing label`);
      }
    });
  }

  private async testTables(): Promise<void> {
    console.log("\n=== Testing Tables ===");

    const tables = Array.from(document.querySelectorAll("table"));
    console.log(`Found ${tables.length} tables`);

    tables.forEach((table, i) => {
      // Check for caption
      const caption = table.querySelector("caption");
      console.log(`Table ${i + 1}:`);
      console.log(`  Caption: ${caption ? "✓" : "✗"}`);

      // Check for headers
      const headers = table.querySelectorAll("th");
      console.log(`  Headers: ${headers.length}`);

      // Check scope attributes
      const hasScope = Array.from(headers).every((th) =>
        th.hasAttribute("scope"),
      );
      console.log(`  Scope attributes: ${hasScope ? "✓" : "✗"}`);
    });
  }

  private async testImages(): Promise<void> {
    console.log("\n=== Testing Images ===");

    const images = Array.from(document.querySelectorAll("img"));
    console.log(`Found ${images.length} images`);

    images.forEach((img, i) => {
      const alt = img.getAttribute("alt");
      const ariaLabel = img.getAttribute("aria-label");
      const role = img.getAttribute("role");

      console.log(`Image ${i + 1}:`);
      console.log(`  Alt: ${alt !== null ? `"${alt}"` : "✗ MISSING"}`);
      console.log(`  Aria-label: ${ariaLabel || "none"}`);
      console.log(`  Role: ${role || "none"}`);

      // Check if decorative
      if (role === "presentation" || role === "none" || alt === "") {
        console.log(`  Type: Decorative`);
      } else {
        console.log(`  Type: Informative`);
      }
    });
  }

  private async testARIA(): Promise<void> {
    console.log("\n=== Testing ARIA ===");

    // Find all elements with ARIA attributes
    const ariaElements = Array.from(
      document.querySelectorAll(
        "[role], [aria-label], [aria-labelledby], [aria-describedby]",
      ),
    );

    console.log(`Found ${ariaElements.length} elements with ARIA`);

    ariaElements.forEach((el) => {
      const role = el.getAttribute("role");
      const label = el.getAttribute("aria-label");
      const labelledby = el.getAttribute("aria-labelledby");
      const describedby = el.getAttribute("aria-describedby");

      console.log(`${el.tagName.toLowerCase()}:`);
      if (role) console.log(`  Role: ${role}`);
      if (label) console.log(`  Label: ${label}`);
      if (labelledby) console.log(`  Labelledby: ${labelledby}`);
      if (describedby) console.log(`  Describedby: ${describedby}`);
    });
  }

  private printResults(): void {
    console.log("\n=== Test Results Summary ===");

    let passed = 0;
    let failed = 0;

    this.testResults.forEach((result, test) => {
      const status = result ? "✓ PASS" : "✗ FAIL";
      console.log(`${status}: ${test}`);
      result ? passed++ : failed++;
    });

    console.log(`\nTotal: ${passed} passed, ${failed} failed`);
  }
}

// Run tests
// const tester = new NVDATestRunner();
// tester.runFullTest(window.location.href);
```

---

## JAWS (Windows)

### JAWS Commands

```typescript
interface JAWSCommands {
  // Basic Navigation
  "Insert + Down Arrow": "Say all";
  "Insert + Up Arrow": "Read current line";
  "Insert + T": "Read window title";
  "Insert + F12": "Read time and date";

  // Element Navigation (similar to NVDA)
  H: "Next heading";
  "Shift + H": "Previous heading";
  "Insert + F6": "List headings";
  "Insert + Ctrl + L": "List links";
  "Insert + Ctrl + F": "List form fields";
  "Insert + Ctrl + B": "List buttons";
  "Insert + F5": "List form fields";
  "Insert + F7": "List links";

  // Virtual Cursor
  "Insert + Z": "Toggle virtual cursor";
  Semicolon: "JAWS Find";

  // Forms
  "Insert + F5": "Select a form field";
  "Insert + Enter": "Forms mode on/off";

  // Tables
  "Insert + Space, T": "Table navigation commands";
  "Ctrl + Alt + Arrow Keys": "Navigate table cells";

  // Settings
  "Insert + J": "JAWS window";
  "Insert + F2": "Pass through next key to application";
}
```

### JAWS Testing Considerations

```typescript
class JAWSTestConsiderations {
  static getTestingGuidelines(): string[] {
    return [
      "Test with both virtual cursor and forms mode",
      "Verify table reading with Insert + Space + T",
      "Check custom ARIA widget announcements",
      "Test with multiple verbosity levels (Insert + V)",
      "Verify page summary (Insert + F1 on a page)",
      "Test forms mode automatic activation",
      "Check list item announcements (position in list)",
      "Verify link announcements (link count, visited state)",
      "Test heading navigation and heading list",
      "Check landmark navigation and announcements",
    ];
  }

  static getJAWSSpecificTests(): Record<string, string> {
    return {
      "Forms mode":
        "Verify automatic forms mode activation for interactive elements",
      "Virtual cursor": "Test reading without forms mode for static content",
      "Place marker":
        "Test Insert + K and Insert + Shift + K for temporary markers",
      "JAWS cursor": "Test Insert + NumPad Minus for JAWS cursor mode",
      "Screen sensitive": "Verify OCR reading for problematic layouts",
      "Research It": "Test Insert + Space + R for additional information",
    };
  }
}
```

---

## VoiceOver (macOS/iOS)

### VoiceOver macOS Commands

```typescript
interface VoiceOverMacCommands {
  // Activation
  "Cmd + F5": "Toggle VoiceOver on/off";

  // Basic Navigation (VO = Ctrl + Option)
  "VO + Right Arrow": "Next item";
  "VO + Left Arrow": "Previous item";
  "VO + A": "Start reading";
  "VO + Space": "Activate item";
  "VO + Shift + Down": "Interact with item";
  "VO + Shift + Up": "Stop interacting";

  // Web Navigation
  "VO + Cmd + H": "Next heading";
  "VO + Cmd + L": "Next link";
  "VO + Cmd + J": "Next form control";
  "VO + Cmd + T": "Next table";
  "VO + Cmd + X": "Next list";
  "VO + Cmd + G": "Next graphic";

  // Rotor (Web navigation menu)
  "VO + U": "Open rotor";
  "Left/Right Arrow": "Change rotor category (in rotor)";
  "Up/Down Arrow": "Navigate items (in rotor)";

  // Landmarks
  "VO + Cmd + Shift + W": "Navigate landmarks";

  // Reading
  "VO + A": "Read from current position";
  Ctrl: "Stop reading";
  "VO + P": "Read paragraph";
  "VO + S": "Read sentence";
  "VO + W": "Read word";
  "VO + C": "Read character";

  // Tables
  "VO + Cmd + [": "Table row up";
  "VO + Cmd + ]": "Table row down";

  // Help
  "VO + H": "VoiceOver help";
  "VO + K": "Keyboard help";
}
```

### VoiceOver iOS Gestures

```typescript
interface VoiceOverIOSGestures {
  // Basic Navigation
  "Swipe right": "Next item";
  "Swipe left": "Previous item";
  "Double tap": "Activate item";
  "Two-finger swipe up": "Read all from current position";
  "Two-finger swipe down": "Read all from top";

  // Rotor
  "Two-finger rotate": "Open rotor";
  "Swipe up/down": "Move by rotor setting";

  // Adjustable controls
  "Swipe up with one finger": "Increment";
  "Swipe down with one finger": "Decrement";

  // Web navigation
  "Rotor > Headings, then swipe": "Navigate headings";
  "Rotor > Links, then swipe": "Navigate links";
  "Rotor > Form Controls, then swipe": "Navigate form controls";

  // Text input
  "Triple tap": "Select text";
  "Four-finger tap near top": "Focus address bar";

  // Screen Curtain
  "Three-finger triple tap": "Toggle screen curtain";
}
```

### VoiceOver Testing Script

```typescript
class VoiceOverTestGuide {
  static getMacOSTestSteps(): string[] {
    return [
      "1. Enable VoiceOver: Cmd + F5",
      "2. Navigate to page: Cmd + T for new tab",
      "3. Open Rotor: VO + U",
      "4. Test heading navigation: Arrow keys in Headings category",
      "5. Test landmark navigation: Switch to Landmarks category",
      "6. Test link navigation: Switch to Links category",
      "7. Test forms: VO + Cmd + J to jump to form controls",
      "8. Test tables: Navigate to table and use VO + Shift + Down to interact",
      "9. Test ARIA widgets: Listen for role announcements",
      "10. Test live regions: Trigger dynamic content changes",
    ];
  }

  static getIOSTestSteps(): string[] {
    return [
      "1. Enable VoiceOver: Settings > Accessibility > VoiceOver",
      "2. Open Safari and navigate to page",
      "3. Swipe right repeatedly to hear all elements",
      "4. Use rotor (two-finger rotate) to change navigation mode",
      "5. Test headings: Set rotor to Headings, swipe up/down",
      "6. Test landmarks: Set rotor to Landmarks, swipe up/down",
      "7. Test links: Set rotor to Links, swipe up/down",
      "8. Test forms: Double-tap inputs, use on-screen keyboard",
      "9. Test images: Listen for alt text announcements",
      "10. Test custom widgets: Verify gesture support",
    ];
  }

  static getCommonIssues(): Record<string, string> {
    return {
      "Requires interaction":
        "Some containers require VO + Shift + Down to interact",
      "Focus not visible": "VoiceOver cursor may not be visually indicated",
      "Form labels": "Must use proper label association or aria-label",
      "Button text": 'VoiceOver reads "button" after button text',
      "Link announcements": 'Says "link" before link text',
      "List announcements": 'Announces "list N items" before reading',
      "Table navigation":
        "Must interact with table first, then use Cmd + [ / ]",
      "Live regions":
        "May not interrupt current speech (depends on politeness)",
    };
  }
}
```

---

## Testing Strategies

### Comprehensive Test Plan

```typescript
interface ScreenReaderTestPlan {
  phase: string;
  screenReader: string;
  testCases: TestCase[];
}

interface TestCase {
  category: string;
  test: string;
  expectedBehavior: string;
  priority: "High" | "Medium" | "Low";
}

const comprehensiveTestPlan: ScreenReaderTestPlan[] = [
  {
    phase: "Initial Setup",
    screenReader: "All",
    testCases: [
      {
        category: "Configuration",
        test: "Install and configure screen reader",
        expectedBehavior: "Screen reader installed with optimal settings",
        priority: "High",
      },
      {
        category: "Baseline",
        test: "Test on known-accessible site (e.g., gov.uk)",
        expectedBehavior: "Understand expected behavior patterns",
        priority: "High",
      },
    ],
  },
  {
    phase: "Page Structure",
    screenReader: "NVDA/JAWS/VoiceOver",
    testCases: [
      {
        category: "Document",
        test: "Page title announces on load",
        expectedBehavior: "Screen reader announces page title",
        priority: "High",
      },
      {
        category: "Landmarks",
        test: "Navigate between landmarks",
        expectedBehavior: "All landmarks reachable and properly labeled",
        priority: "High",
      },
      {
        category: "Headings",
        test: "Navigate heading hierarchy",
        expectedBehavior: "Logical heading structure, no skipped levels",
        priority: "High",
      },
    ],
  },
  {
    phase: "Interactive Elements",
    screenReader: "All",
    testCases: [
      {
        category: "Forms",
        test: "Complete form with screen reader only",
        expectedBehavior: "All fields labeled, errors announced",
        priority: "High",
      },
      {
        category: "Buttons",
        test: "Activate buttons",
        expectedBehavior: "Purpose clear, activation confirmed",
        priority: "High",
      },
      {
        category: "Links",
        test: "Navigate and activate links",
        expectedBehavior: "Link purpose clear from text alone",
        priority: "High",
      },
    ],
  },
  {
    phase: "Dynamic Content",
    screenReader: "All",
    testCases: [
      {
        category: "Live Regions",
        test: "Trigger status messages",
        expectedBehavior: "Updates announced without interrupting",
        priority: "High",
      },
      {
        category: "Modals",
        test: "Open and close dialogs",
        expectedBehavior: "Focus trapped, purpose announced",
        priority: "High",
      },
      {
        category: "Alerts",
        test: "Trigger alert messages",
        expectedBehavior: "Alert announced immediately",
        priority: "High",
      },
    ],
  },
];
```

### Test Documentation Template

```typescript
interface TestResult {
  testId: string;
  testName: string;
  screenReader: string;
  version: string;
  browser: string;
  date: string;
  tester: string;
  result: "Pass" | "Fail" | "Partial";
  issues: Issue[];
  notes: string;
}

interface Issue {
  severity: "Critical" | "High" | "Medium" | "Low";
  description: string;
  reproduction: string;
  expectedBehavior: string;
  actualBehavior: string;
  screenshot?: string;
  audioRecording?: string;
}

class TestReporter {
  private results: TestResult[] = [];

  addResult(result: TestResult): void {
    this.results.push(result);
  }

  generateReport(): string {
    let report = "# Screen Reader Test Report\n\n";
    report += `Date: ${new Date().toLocaleDateString()}\n\n`;

    // Summary
    const passed = this.results.filter((r) => r.result === "Pass").length;
    const failed = this.results.filter((r) => r.result === "Fail").length;
    const partial = this.results.filter((r) => r.result === "Partial").length;

    report += "## Summary\n\n";
    report += `- ✓ Passed: ${passed}\n`;
    report += `- ✗ Failed: ${failed}\n`;
    report += `- ⚠ Partial: ${partial}\n`;
    report += `- Total: ${this.results.length}\n\n`;

    // Details
    report += "## Test Results\n\n";
    this.results.forEach((result) => {
      const icon =
        result.result === "Pass" ? "✓" : result.result === "Fail" ? "✗" : "⚠";

      report += `### ${icon} ${result.testName}\n\n`;
      report += `- **Screen Reader**: ${result.screenReader} ${result.version}\n`;
      report += `- **Browser**: ${result.browser}\n`;
      report += `- **Tester**: ${result.tester}\n`;
      report += `- **Date**: ${result.date}\n`;
      report += `- **Result**: ${result.result}\n\n`;

      if (result.issues.length > 0) {
        report += "#### Issues Found:\n\n";
        result.issues.forEach((issue, i) => {
          report += `${i + 1}. **${issue.severity}**: ${issue.description}\n`;
          report += `   - Expected: ${issue.expectedBehavior}\n`;
          report += `   - Actual: ${issue.actualBehavior}\n`;
          report += `   - Reproduction: ${issue.reproduction}\n\n`;
        });
      }

      if (result.notes) {
        report += `**Notes**: ${result.notes}\n\n`;
      }

      report += "---\n\n";
    });

    return report;
  }

  exportJSON(): string {
    return JSON.stringify(this.results, null, 2);
  }

  exportCSV(): string {
    let csv =
      "Test ID,Test Name,Screen Reader,Version,Browser,Date,Result,Issues\n";

    this.results.forEach((result) => {
      const issueCount = result.issues.length;
      csv += `"${result.testId}","${result.testName}","${result.screenReader}",`;
      csv += `"${result.version}","${result.browser}","${result.date}",`;
      csv += `"${result.result}",${issueCount}\n`;
    });

    return csv;
  }
}
```

---

## Common Issues and Solutions

### Issue Matrix

```typescript
interface CommonIssue {
  problem: string;
  screenReader: string[];
  impact: "Critical" | "High" | "Medium" | "Low";
  solution: string;
  example: string;
}

const commonIssues: CommonIssue[] = [
  {
    problem: "Empty links or buttons",
    screenReader: ["All"],
    impact: "Critical",
    solution: "Add aria-label or visible text",
    example: `
<!-- ❌ Bad -->
<button><i class="icon-save"></i></button>

<!-- ✓ Good -->
<button aria-label="Save document">
  <i class="icon-save" aria-hidden="true"></i>
</button>
    `,
  },
  {
    problem: "Missing form labels",
    screenReader: ["All"],
    impact: "Critical",
    solution: "Associate labels with inputs",
    example: `
<!-- ❌ Bad -->
<input type="text" placeholder="Email">

<!-- ✓ Good -->
<label for="email">Email</label>
<input type="text" id="email">
    `,
  },
  {
    problem: "Inaccessible custom widgets",
    screenReader: ["All"],
    impact: "High",
    solution: "Add proper ARIA roles and keyboard support",
    example: `
<!-- ✓ Good -->
<div role="button" 
     tabindex="0"
     aria-pressed="false"
     onkeydown="handleKeyPress(event)">
  Toggle
</div>
    `,
  },
  {
    problem: "Dynamic content not announced",
    screenReader: ["All"],
    impact: "High",
    solution: "Use ARIA live regions",
    example: `
<!-- ✓ Good -->
<div role="status" aria-live="polite" aria-atomic="true">
  <p id="status-message"></p>
</div>
    `,
  },
  {
    problem: "Keyboard trap in modal",
    screenReader: ["NVDA", "JAWS"],
    impact: "Critical",
    solution: "Implement proper focus management",
    example: `
// ✓ Good: Focus trap implementation
class Modal {
  open() {
    this.trapFocus();
    this.modal.setAttribute('aria-modal', 'true');
  }
  
  trapFocus() {
    // Trap focus within modal
  }
}
    `,
  },
  {
    problem: "Poor table structure",
    screenReader: ["All"],
    impact: "High",
    solution: "Use proper table markup with headers",
    example: `
<!-- ✓ Good -->
<table>
  <caption>Sales Data</caption>
  <thead>
    <tr>
      <th scope="col">Product</th>
      <th scope="col">Sales</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">Widget</th>
      <td>$1000</td>
    </tr>
  </tbody>
</table>
    `,
  },
  {
    problem: "Images without alt text",
    screenReader: ["All"],
    impact: "High",
    solution: "Add descriptive alt text or empty alt for decorative",
    example: `
<!-- ✓ Good: Informative -->
<img src="chart.png" alt="Sales increased 50% in Q4">

<!-- ✓ Good: Decorative -->
<img src="decoration.png" alt="" role="presentation">
    `,
  },
  {
    problem: "Skip links not working",
    screenReader: ["VoiceOver"],
    impact: "Medium",
    solution: "Ensure target has tabindex and receives focus",
    example: `
<!-- ✓ Good -->
<a href="#main" class="skip-link">Skip to main</a>
...
<main id="main" tabindex="-1">
  Content
</main>
    `,
  },
];
```

### Debugging Screen Reader Issues

```typescript
class ScreenReaderDebugger {
  static logAccessibilityTree(element: HTMLElement): void {
    const computedRole =
      element.getAttribute("role") || this.getImplicitRole(element);
    const computedName = this.getAccessibleName(element);
    const computedDescription = this.getAccessibleDescription(element);

    console.group(`Accessibility Info: ${element.tagName}`);
    console.log("Role:", computedRole);
    console.log("Name:", computedName);
    console.log("Description:", computedDescription);
    console.log("Focusable:", this.isFocusable(element));
    console.log("Hidden:", this.isAccessibilityHidden(element));
    console.groupEnd();
  }

  private static getImplicitRole(element: HTMLElement): string {
    const roleMap: Record<string, string> = {
      A: "link",
      BUTTON: "button",
      INPUT: "textbox",
      NAV: "navigation",
      MAIN: "main",
      HEADER: "banner",
      FOOTER: "contentinfo",
      ASIDE: "complementary",
      ARTICLE: "article",
      SECTION: "region",
    };

    return roleMap[element.tagName] || "";
  }

  private static getAccessibleName(element: HTMLElement): string {
    // Check aria-label
    const ariaLabel = element.getAttribute("aria-label");
    if (ariaLabel) return ariaLabel;

    // Check aria-labelledby
    const labelledby = element.getAttribute("aria-labelledby");
    if (labelledby) {
      const labelElement = document.getElementById(labelledby);
      if (labelElement) return labelElement.textContent || "";
    }

    // Check associated label
    const id = element.id;
    if (id) {
      const label = document.querySelector(`label[for="${id}"]`);
      if (label) return label.textContent || "";
    }

    // Check title
    const title = element.getAttribute("title");
    if (title) return title;

    // Check text content
    return element.textContent || "";
  }

  private static getAccessibleDescription(element: HTMLElement): string {
    const describedby = element.getAttribute("aria-describedby");
    if (describedby) {
      const descElement = document.getElementById(describedby);
      if (descElement) return descElement.textContent || "";
    }

    return "";
  }

  private static isFocusable(element: HTMLElement): boolean {
    const tabindex = element.getAttribute("tabindex");
    if (tabindex === "-1") return false;

    const focusableElements = ["A", "BUTTON", "INPUT", "SELECT", "TEXTAREA"];

    return (
      focusableElements.includes(element.tagName) ||
      tabindex === "0" ||
      element.hasAttribute("contenteditable")
    );
  }

  private static isAccessibilityHidden(element: HTMLElement): boolean {
    // Check aria-hidden
    if (element.getAttribute("aria-hidden") === "true") return true;

    // Check CSS visibility
    const style = window.getComputedStyle(element);
    if (style.display === "none" || style.visibility === "hidden") {
      return true;
    }

    // Check if element is off-screen
    const rect = element.getBoundingClientRect();
    if (rect.width === 0 && rect.height === 0) return true;

    return false;
  }

  static announceToScreenReader(
    message: string,
    priority: "polite" | "assertive" = "polite",
  ): void {
    const liveRegion = document.createElement("div");
    liveRegion.setAttribute(
      "role",
      priority === "assertive" ? "alert" : "status",
    );
    liveRegion.setAttribute("aria-live", priority);
    liveRegion.setAttribute("aria-atomic", "true");
    liveRegion.className = "sr-only";
    liveRegion.textContent = message;

    document.body.appendChild(liveRegion);

    setTimeout(() => {
      liveRegion.remove();
    }, 1000);
  }
}

// Usage
const button = document.querySelector("button")!;
ScreenReaderDebugger.logAccessibilityTree(button as HTMLElement);
```

---

## Screen Reader Specific Patterns

### NVDA-Specific Implementation

```typescript
class NVDAOptimizedWidget {
  // NVDA benefits from clear role announcements
  createButton(): HTMLElement {
    const button = document.createElement("button");
    button.textContent = "Submit";
    button.setAttribute("type", "submit");
    // NVDA will announce: "Submit button"
    return button;
  }

  // NVDA reads aria-describedby well
  createInput(): HTMLElement {
    const wrapper = document.createElement("div");
    wrapper.innerHTML = `
      <label for="username">Username</label>
      <input type="text" 
             id="username"
             aria-describedby="username-help">
      <div id="username-help" class="help-text">
        Must be 3-20 characters
      </div>
    `;
    return wrapper;
  }
}
```

### JAWS-Specific Implementation

```typescript
class JAWSOptimizedWidget {
  // JAWS provides detailed form information
  createFormField(): HTMLElement {
    const wrapper = document.createElement("div");
    wrapper.innerHTML = `
      <label for="email">
        Email Address
        <span class="required" aria-label="required">*</span>
      </label>
      <input type="email" 
             id="email"
             required
             aria-required="true"
             aria-invalid="false"
             aria-describedby="email-error email-help">
      <div id="email-help">We'll never share your email</div>
      <div id="email-error" role="alert" aria-live="assertive"></div>
    `;
    // JAWS will announce: "Email Address required edit"
    return wrapper;
  }

  // JAWS excels at table navigation
  createAccessibleTable(): HTMLElement {
    const table = document.createElement("table");
    table.innerHTML = `
      <caption>Employee Directory</caption>
      <thead>
        <tr>
          <th scope="col">Name</th>
          <th scope="col">Department</th>
          <th scope="col">Email</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <th scope="row">John Doe</th>
          <td>Engineering</td>
          <td>john@example.com</td>
        </tr>
      </tbody>
    `;
    return table;
  }
}
```

### VoiceOver-Specific Implementation

```typescript
class VoiceOverOptimizedWidget {
  // VoiceOver requires explicit interaction for some containers
  createMenu(): HTMLElement {
    const nav = document.createElement("nav");
    nav.setAttribute("aria-label", "Main navigation");
    // VoiceOver: "Main navigation, navigation landmark"

    const list = document.createElement("ul");
    list.setAttribute("role", "list"); // Explicit for VoiceOver

    ["Home", "About", "Contact"].forEach((item) => {
      const li = document.createElement("li");
      li.setAttribute("role", "listitem");
      li.innerHTML = `<a href="/${item.toLowerCase()}">${item}</a>`;
      list.appendChild(li);
    });

    nav.appendChild(list);
    return nav;
  }

  // VoiceOver rotor benefits from clear grouping
  createFormWithFieldset(): HTMLElement {
    const form = document.createElement("form");
    form.innerHTML = `
      <fieldset>
        <legend>Personal Information</legend>
        <label for="first-name">First Name</label>
        <input type="text" id="first-name">
        
        <label for="last-name">Last Name</label>
        <input type="text" id="last-name">
      </fieldset>
    `;
    return form;
  }
}
```

---

## Automated Testing

### Accessibility Testing Tools

```typescript
class AutomatedScreenReaderTesting {
  // Using axe-core for automated testing
  static async runAxeCore(): Promise<void> {
    // Assuming axe-core is loaded
    const results = await (window as any).axe.run();

    console.log("Violations found:", results.violations.length);

    results.violations.forEach((violation: any) => {
      console.group(
        `${violation.impact.toUpperCase()}: ${violation.description}`,
      );
      console.log("Help:", violation.helpUrl);
      console.log("Affected nodes:", violation.nodes.length);
      violation.nodes.forEach((node: any) => {
        console.log("- Element:", node.html);
        console.log("  Issue:", node.failureSummary);
      });
      console.groupEnd();
    });
  }

  // Simulate screen reader announcements
  static getScreenReaderText(element: HTMLElement): string {
    const role = element.getAttribute("role") || this.getImplicitRole(element);
    const name = this.getAccessibleName(element);
    const state = this.getState(element);

    // Construct what a screen reader would say
    let announcement = name;

    if (state) {
      announcement += `, ${state}`;
    }

    announcement += `, ${role}`;

    return announcement;
  }

  private static getImplicitRole(element: HTMLElement): string {
    // Implementation from earlier
    return "element";
  }

  private static getAccessibleName(element: HTMLElement): string {
    // Implementation from earlier
    return element.textContent || "";
  }

  private static getState(element: HTMLElement): string {
    const states: string[] = [];

    if (element.hasAttribute("aria-pressed")) {
      const pressed = element.getAttribute("aria-pressed") === "true";
      states.push(pressed ? "pressed" : "not pressed");
    }

    if (element.hasAttribute("aria-expanded")) {
      const expanded = element.getAttribute("aria-expanded") === "true";
      states.push(expanded ? "expanded" : "collapsed");
    }

    if (element.hasAttribute("aria-checked")) {
      const checked = element.getAttribute("aria-checked");
      states.push(
        checked === "true"
          ? "checked"
          : checked === "mixed"
            ? "mixed"
            : "not checked",
      );
    }

    if (element.hasAttribute("aria-selected")) {
      const selected = element.getAttribute("aria-selected") === "true";
      states.push(selected ? "selected" : "not selected");
    }

    return states.join(", ");
  }
}
```

---

## Key Takeaways

1. **Test with multiple screen readers** - NVDA, JAWS, and VoiceOver have different behaviors and market share; test with at least two to catch most issues.

2. **Learn keyboard commands** - Effective screen reader testing requires proficiency with navigation commands (headings, landmarks, forms, tables).

3. **Test forms mode vs browse mode** - Windows screen readers (NVDA/JAWS) switch between browse and forms mode; ensure both work correctly.

4. **Verify semantic structure** - Screen readers rely heavily on proper HTML semantics and ARIA; use heading hierarchy, landmarks, and labels consistently.

5. **Test dynamic content** - Ensure live regions announce updates appropriately; verify modals trap focus and restore it correctly.

6. **Check announcement verbosity** - Verify screen readers announce enough context (role, state, position) without being overly verbose.

7. **Test with real users when possible** - Automated tools and developer testing catch many issues, but real screen reader users find nuances you'll miss.

8. **Document platform differences** - NVDA, JAWS, and VoiceOver behave differently; document known differences and test accordingly.

9. **Use accessibility tree inspector** - Browser dev tools show the accessibility tree; use it to verify what screen readers "see" before actual testing.

10. **Record and share findings** - Document test results with audio recordings or transcripts; this helps team understand screen reader user experience.
