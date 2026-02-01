# Color Contrast - Ensuring Readability and Accessibility

## Table of Contents

1. [Introduction to Color Contrast](#introduction-to-color-contrast)
2. [WCAG Contrast Requirements](#wcag-contrast-requirements)
3. [Understanding Contrast Ratios](#understanding-contrast-ratios)
4. [Testing Tools](#testing-tools)
5. [Common Contrast Issues](#common-contrast-issues)
6. [Color Accessibility Patterns](#color-accessibility-patterns)
7. [Dynamic Contrast Adjustment](#dynamic-contrast-adjustment)
8. [Automated Contrast Checking](#automated-contrast-checking)
9. [Best Practices](#best-practices)
10. [Key Takeaways](#key-takeaways)

---

## Introduction to Color Contrast

**Color contrast** refers to the difference in luminance (brightness) between foreground content (text, icons) and background. Sufficient contrast ensures content is readable by people with:

- Low vision
- Color blindness (color vision deficiency)
- Age-related vision decline
- Situational impairments (bright sunlight, dim screens)

### Why Contrast Matters

```typescript
interface ContrastImpact {
  population: string;
  percentage: string;
  needHighContrast: boolean;
  affectedBy: string[];
}

const affectedPopulations: ContrastImpact[] = [
  {
    population: "Low Vision",
    percentage: "~4.25% of population",
    needHighContrast: true,
    affectedBy: ["Low contrast", "Small text", "Color-only differentiation"],
  },
  {
    population: "Color Blindness",
    percentage: "~8% of men, ~0.5% of women",
    needHighContrast: false,
    affectedBy: [
      "Red-green confusion",
      "Blue-yellow confusion",
      "Color-only information",
    ],
  },
  {
    population: "Age-related Vision Decline",
    percentage: "Increases with age (50% over 65)",
    needHighContrast: true,
    affectedBy: [
      "Reduced contrast sensitivity",
      "Yellowing of lens",
      "Decreased light transmission",
    ],
  },
  {
    population: "Situational Impairment",
    percentage: "100% (everyone affected sometimes)",
    needHighContrast: true,
    affectedBy: [
      "Bright sunlight glare",
      "Low battery dimming",
      "Cheap monitors",
      "Older displays",
    ],
  },
];
```

---

## WCAG Contrast Requirements

### WCAG 2.1 Contrast Success Criteria

```typescript
interface ContrastRequirement {
  criterion: string;
  level: "A" | "AA" | "AAA";
  minRatio: number;
  appliesTo: string;
  exceptions: string[];
}

const wcagContrastRequirements: ContrastRequirement[] = [
  {
    criterion: "1.4.3 Contrast (Minimum)",
    level: "AA",
    minRatio: 4.5,
    appliesTo: "Normal text (< 18pt or < 14pt bold)",
    exceptions: [
      "Large text (≥ 18pt or ≥ 14pt bold): 3:1",
      "Incidental text (inactive UI, decorative)",
      "Logotypes (text in logo)",
    ],
  },
  {
    criterion: "1.4.6 Contrast (Enhanced)",
    level: "AAA",
    minRatio: 7,
    appliesTo: "Normal text (< 18pt or < 14pt bold)",
    exceptions: [
      "Large text (≥ 18pt or ≥ 14pt bold): 4.5:1",
      "Incidental text (inactive UI, decorative)",
      "Logotypes (text in logo)",
    ],
  },
  {
    criterion: "1.4.11 Non-text Contrast",
    level: "AA",
    minRatio: 3,
    appliesTo: "UI components, graphic objects, focus indicators",
    exceptions: [
      "Inactive UI components",
      "User agent controlled (native elements)",
      "Essential representations (flags, logos)",
    ],
  },
];
```

### Text Size Definitions

```typescript
interface TextSizeCategory {
  category: "Normal" | "Large";
  minSize: string;
  minRatioAA: number;
  minRatioAAA: number;
  examples: string[];
}

const textSizeCategories: TextSizeCategory[] = [
  {
    category: "Normal",
    minSize: "< 18pt (24px) regular OR < 14pt (18.66px) bold",
    minRatioAA: 4.5,
    minRatioAAA: 7,
    examples: [
      "Body text: 16px regular",
      "Small text: 14px regular",
      "Labels: 12px regular",
    ],
  },
  {
    category: "Large",
    minSize: "≥ 18pt (24px) regular OR ≥ 14pt (18.66px) bold",
    minRatioAA: 3,
    minRatioAAA: 4.5,
    examples: [
      "Headings: 32px regular",
      "Large buttons: 18px bold",
      "Hero text: 48px regular",
    ],
  },
];
```

### CSS Implementation

```css
/* Level AA Compliant Colors */
:root {
  /* Normal Text (4.5:1 on white) */
  --text-primary: #212529; /* 16.9:1 */
  --text-secondary: #495057; /* 9.7:1 */
  --text-muted: #6c757d; /* 5.9:1 */

  /* Large Text (3:1 on white) */
  --text-large: #868e96; /* 4.5:1 - also works for normal text */

  /* Level AAA Normal Text (7:1 on white) */
  --text-aaa: #383838; /* 10.4:1 */

  /* Interactive Elements (3:1 for non-text) */
  --link-color: #0066cc; /* 4.5:1 */
  --button-primary: #0056b3; /* 5.9:1 */
  --border-focus: #4a90e2; /* 3.1:1 */

  /* Backgrounds */
  --bg-primary: #ffffff;
  --bg-secondary: #f8f9fa;
  --bg-dark: #212529;
}

/* Normal Text Styles */
body {
  color: var(--text-primary);
  background: var(--bg-primary);
  font-size: 16px;
}

.text-secondary {
  color: var(--text-secondary);
}

.text-muted {
  color: var(--text-muted);
}

/* Large Text Styles */
h1,
h2,
h3 {
  color: var(--text-large);
  font-weight: bold;
}

h1 {
  font-size: 2.5rem;
} /* 40px */
h2 {
  font-size: 2rem;
} /* 32px */
h3 {
  font-size: 1.75rem;
} /* 28px */

/* Link Styles */
a {
  color: var(--link-color);
  text-decoration: underline; /* Don't rely on color alone */
}

a:hover,
a:focus {
  color: var(--button-primary);
  text-decoration: underline;
}

/* Button Styles */
button {
  background: var(--button-primary);
  color: #ffffff;
  border: 2px solid var(--button-primary);
  padding: 0.5rem 1rem;
  font-size: 1rem;
}

/* Focus Indicator (3:1 contrast with background) */
*:focus-visible {
  outline: 3px solid var(--border-focus);
  outline-offset: 2px;
}

/* Form Elements */
input,
textarea,
select {
  border: 2px solid #495057; /* 9.7:1 */
  color: var(--text-primary);
  background: #ffffff;
}

input:focus,
textarea:focus,
select:focus {
  border-color: var(--border-focus);
  box-shadow: 0 0 0 3px rgba(74, 144, 226, 0.25);
}

/* High Contrast Mode Support */
@media (prefers-contrast: high) {
  :root {
    --text-primary: #000000;
    --link-color: #0000ee;
    --button-primary: #0000cc;
  }

  * {
    border-color: currentColor !important;
  }
}
```

---

## Understanding Contrast Ratios

### Luminance Calculation

The contrast ratio is calculated using relative luminance:

```typescript
class ContrastCalculator {
  /**
   * Calculate contrast ratio between two colors
   * Formula: (L1 + 0.05) / (L2 + 0.05)
   * where L1 is the lighter color and L2 is the darker color
   */
  static calculateContrast(color1: string, color2: string): number {
    const lum1 = this.getLuminance(color1);
    const lum2 = this.getLuminance(color2);

    const lighter = Math.max(lum1, lum2);
    const darker = Math.min(lum1, lum2);

    return (lighter + 0.05) / (darker + 0.05);
  }

  /**
   * Calculate relative luminance
   * Formula from WCAG: https://www.w3.org/TR/WCAG21/#dfn-relative-luminance
   */
  private static getLuminance(color: string): number {
    const rgb = this.hexToRgb(color);

    // Convert RGB to linear RGB
    const [r, g, b] = rgb.map((val) => {
      val = val / 255;
      return val <= 0.03928
        ? val / 12.92
        : Math.pow((val + 0.055) / 1.055, 2.4);
    });

    // Calculate luminance
    return 0.2126 * r + 0.7152 * g + 0.0722 * b;
  }

  private static hexToRgb(hex: string): [number, number, number] {
    // Remove # if present
    hex = hex.replace("#", "");

    // Convert shorthand hex to full
    if (hex.length === 3) {
      hex = hex
        .split("")
        .map((char) => char + char)
        .join("");
    }

    const r = parseInt(hex.substring(0, 2), 16);
    const g = parseInt(hex.substring(2, 4), 16);
    const b = parseInt(hex.substring(4, 6), 16);

    return [r, g, b];
  }

  /**
   * Check if contrast meets WCAG requirements
   */
  static meetsWCAG(
    color1: string,
    color2: string,
    level: "AA" | "AAA",
    isLargeText: boolean = false,
  ): { passes: boolean; ratio: number; required: number } {
    const ratio = this.calculateContrast(color1, color2);

    let required: number;
    if (level === "AA") {
      required = isLargeText ? 3 : 4.5;
    } else {
      required = isLargeText ? 4.5 : 7;
    }

    return {
      passes: ratio >= required,
      ratio: Math.round(ratio * 100) / 100,
      required,
    };
  }

  /**
   * Get readable description of contrast level
   */
  static getContrastLevel(ratio: number): string {
    if (ratio >= 7) return "AAA (Normal Text)";
    if (ratio >= 4.5) return "AA (Normal Text)";
    if (ratio >= 3) return "AA (Large Text Only)";
    return "Fail (Insufficient Contrast)";
  }

  /**
   * Find optimal text color (black or white) for given background
   */
  static getOptimalTextColor(backgroundColor: string): string {
    const blackContrast = this.calculateContrast(backgroundColor, "#000000");
    const whiteContrast = this.calculateContrast(backgroundColor, "#FFFFFF");

    return blackContrast > whiteContrast ? "#000000" : "#FFFFFF";
  }

  /**
   * Suggest color adjustments to meet WCAG requirements
   */
  static suggestColorAdjustment(
    foreground: string,
    background: string,
    targetLevel: "AA" | "AAA" = "AA",
    isLargeText: boolean = false,
  ): { original: number; suggested: string; newRatio: number } {
    const originalRatio = this.calculateContrast(foreground, background);
    const targetRatio =
      targetLevel === "AA" ? (isLargeText ? 3 : 4.5) : isLargeText ? 4.5 : 7;

    if (originalRatio >= targetRatio) {
      return {
        original: originalRatio,
        suggested: foreground,
        newRatio: originalRatio,
      };
    }

    // Try darkening or lightening foreground color
    const rgb = this.hexToRgb(foreground);
    const bgLum = this.getLuminance(background);

    // Determine direction (darker or lighter)
    const shouldDarken = bgLum > 0.5;

    let adjustedColor = foreground;
    let bestRatio = originalRatio;

    for (let i = 0; i < 100; i++) {
      const adjustment = shouldDarken ? -2.55 * i : 2.55 * i;
      const newRgb: [number, number, number] = [
        Math.max(0, Math.min(255, rgb[0] + adjustment)),
        Math.max(0, Math.min(255, rgb[1] + adjustment)),
        Math.max(0, Math.min(255, rgb[2] + adjustment)),
      ];

      const testColor = this.rgbToHex(newRgb);
      const testRatio = this.calculateContrast(testColor, background);

      if (testRatio >= targetRatio) {
        adjustedColor = testColor;
        bestRatio = testRatio;
        break;
      }

      if (testRatio > bestRatio) {
        adjustedColor = testColor;
        bestRatio = testRatio;
      }
    }

    return {
      original: originalRatio,
      suggested: adjustedColor,
      newRatio: bestRatio,
    };
  }

  private static rgbToHex(rgb: [number, number, number]): string {
    return (
      "#" +
      rgb
        .map((val) => {
          const hex = Math.round(val).toString(16);
          return hex.length === 1 ? "0" + hex : hex;
        })
        .join("")
    );
  }
}

// Usage Examples
console.log("=== Contrast Calculations ===\n");

// Example 1: Check common color combinations
const blackOnWhite = ContrastCalculator.calculateContrast("#000000", "#FFFFFF");
console.log(
  `Black on White: ${blackOnWhite.toFixed(2)}:1 - ${ContrastCalculator.getContrastLevel(blackOnWhite)}`,
);

const grayOnWhite = ContrastCalculator.calculateContrast("#767676", "#FFFFFF");
console.log(
  `Gray (#767676) on White: ${grayOnWhite.toFixed(2)}:1 - ${ContrastCalculator.getContrastLevel(grayOnWhite)}`,
);

// Example 2: Check WCAG compliance
const linkCheck = ContrastCalculator.meetsWCAG(
  "#0066cc",
  "#ffffff",
  "AA",
  false,
);
console.log(`\nLink Color Check:`);
console.log(`Ratio: ${linkCheck.ratio}:1`);
console.log(`Required: ${linkCheck.required}:1`);
console.log(`Passes: ${linkCheck.passes ? "✓" : "✗"}`);

// Example 3: Get optimal text color
const bgColor = "#3498db";
const optimalText = ContrastCalculator.getOptimalTextColor(bgColor);
console.log(`\nOptimal text color for ${bgColor}: ${optimalText}`);

// Example 4: Suggest adjustment
const adjustment = ContrastCalculator.suggestColorAdjustment(
  "#999999",
  "#FFFFFF",
  "AA",
  false,
);
console.log(`\nColor Adjustment:`);
console.log(`Original: #999999 (${adjustment.original.toFixed(2)}:1)`);
console.log(
  `Suggested: ${adjustment.suggested} (${adjustment.newRatio.toFixed(2)}:1)`,
);
```

---

## Testing Tools

### Browser-Based Tools

```typescript
interface ContrastTool {
  name: string;
  type: "Browser Extension" | "Web App" | "Dev Tools" | "CLI" | "Library";
  url: string;
  features: string[];
  cost: "Free" | "Paid" | "Freemium";
}

const contrastTools: ContrastTool[] = [
  {
    name: "Chrome DevTools",
    type: "Dev Tools",
    url: "Built into Chrome/Edge",
    features: [
      "Inspect element contrast ratio",
      "Shows WCAG level achieved",
      "Suggests AA and AAA compliant colors",
      "Color picker with contrast info",
    ],
    cost: "Free",
  },
  {
    name: "WebAIM Contrast Checker",
    type: "Web App",
    url: "https://webaim.org/resources/contrastchecker/",
    features: [
      "Simple color picker",
      "Instant WCAG pass/fail",
      "Lightness slider",
      "Provides suggestions",
    ],
    cost: "Free",
  },
  {
    name: "Contrast Ratio (Lea Verou)",
    type: "Web App",
    url: "https://contrast-ratio.com/",
    features: [
      "Live calculation",
      "Background image support",
      "Color format flexibility",
      "Bookmarklet available",
    ],
    cost: "Free",
  },
  {
    name: "Colour Contrast Analyser (CCA)",
    type: "Browser Extension",
    url: "TPGi (The Paciello Group)",
    features: [
      "Desktop application",
      "Eyedropper tool",
      "Comprehensive reports",
      "Simulates color blindness",
    ],
    cost: "Free",
  },
  {
    name: "axe DevTools",
    type: "Browser Extension",
    url: "https://www.deque.com/axe/",
    features: [
      "Automated scanning",
      "Highlights contrast issues",
      "Provides fix suggestions",
      "Integration with CI/CD",
    ],
    cost: "Freemium",
  },
  {
    name: "WAVE",
    type: "Browser Extension",
    url: "https://wave.webaim.org/",
    features: [
      "Visual feedback",
      "Contrast error highlighting",
      "Full page analysis",
      "No signup required",
    ],
    cost: "Free",
  },
  {
    name: "ColorSafe",
    type: "Web App",
    url: "http://colorsafe.co/",
    features: [
      "Generates accessible color palettes",
      "Based on WCAG guidelines",
      "Customizable base colors",
      "Export options",
    ],
    cost: "Free",
  },
  {
    name: "Accessible Colors",
    type: "Web App",
    url: "https://accessible-colors.com/",
    features: [
      "Suggests accessible alternatives",
      "Maintains hue when possible",
      "Batch color testing",
      "API available",
    ],
    cost: "Free",
  },
];
```

### Chrome DevTools Contrast Checker

```typescript
class ChromeDevToolsGuide {
  static getInstructions(): string[] {
    return [
      "1. Open Chrome DevTools (F12)",
      "2. Select Elements tab",
      "3. Click Inspect tool (Ctrl+Shift+C)",
      "4. Click on text element",
      "5. In Styles pane, find color property",
      "6. Click color swatch",
      "7. View contrast ratio information:",
      '   - Current ratio (e.g., "4.52")',
      "   - AA checkmark (4.5:1 for normal text)",
      "   - AAA checkmark (7:1 for normal text)",
      "8. Use color picker to adjust until checkmarks appear",
      "9. Suggested colors shown with lines on picker",
    ];
  }

  static getShortcuts(): Record<string, string> {
    return {
      "Ctrl+Shift+C (Cmd+Shift+C)": "Activate element inspector",
      "Ctrl+F (Cmd+F)": "Search in Elements panel",
      H: "Hide/show selected element",
      "Ctrl+[ (Cmd+[)": "Navigate to previous panel",
      "Ctrl+] (Cmd+])": "Navigate to next panel",
    };
  }
}
```

### Automated Testing Script

```typescript
class AutomatedContrastTester {
  private issues: ContrastIssue[] = [];

  interface ContrastIssue {
    element: HTMLElement;
    selector: string;
    foreground: string;
    background: string;
    ratio: number;
    required: number;
    fontSize: number;
    isBold: boolean;
  }

  async testPage(): Promise<void> {
    this.issues = [];

    // Find all text elements
    const elements = this.getAllTextElements();

    elements.forEach(element => {
      this.testElement(element);
    });

    this.reportIssues();
  }

  private getAllTextElements(): HTMLElement[] {
    const textElements: HTMLElement[] = [];
    const walker = document.createTreeWalker(
      document.body,
      NodeFilter.SHOW_ELEMENT,
      {
        acceptNode: (node) => {
          const element = node as HTMLElement;

          // Skip hidden elements
          const style = window.getComputedStyle(element);
          if (style.display === 'none' || style.visibility === 'hidden') {
            return NodeFilter.FILTER_REJECT;
          }

          // Only include elements with direct text content
          const hasText = Array.from(element.childNodes).some(
            child => child.nodeType === Node.TEXT_NODE &&
                    child.textContent?.trim()
          );

          return hasText ? NodeFilter.FILTER_ACCEPT : NodeFilter.FILTER_SKIP;
        }
      }
    );

    let currentNode: Node | null;
    while (currentNode = walker.nextNode()) {
      textElements.push(currentNode as HTMLElement);
    }

    return textElements;
  }

  private testElement(element: HTMLElement): void {
    const style = window.getComputedStyle(element);

    // Get colors
    const foreground = style.color;
    const background = this.getBackgroundColor(element);

    if (!background) return; // Can't test without background

    // Calculate ratio
    const ratio = ContrastCalculator.calculateContrast(
      this.rgbToHex(foreground),
      this.rgbToHex(background)
    );

    // Determine if large text
    const fontSize = parseFloat(style.fontSize);
    const fontWeight = style.fontWeight;
    const isBold = parseInt(fontWeight) >= 700 || fontWeight === 'bold';
    const isLargeText = fontSize >= 24 || (fontSize >= 18.66 && isBold);

    // Check compliance
    const required = isLargeText ? 3 : 4.5;

    if (ratio < required) {
      this.issues.push({
        element,
        selector: this.getSelector(element),
        foreground: this.rgbToHex(foreground),
        background: this.rgbToHex(background),
        ratio: Math.round(ratio * 100) / 100,
        required,
        fontSize,
        isBold
      });
    }
  }

  private getBackgroundColor(element: HTMLElement): string | null {
    let current: HTMLElement | null = element;

    while (current) {
      const style = window.getComputedStyle(current);
      const bg = style.backgroundColor;

      // Check if background is not transparent
      if (bg && bg !== 'rgba(0, 0, 0, 0)' && bg !== 'transparent') {
        return bg;
      }

      current = current.parentElement;
    }

    // Default to white if no background found
    return 'rgb(255, 255, 255)';
  }

  private rgbToHex(rgb: string): string {
    const match = rgb.match(/\d+/g);
    if (!match) return '#000000';

    return '#' + match.slice(0, 3).map(val => {
      const hex = parseInt(val).toString(16);
      return hex.length === 1 ? '0' + hex : hex;
    }).join('');
  }

  private getSelector(element: HTMLElement): string {
    if (element.id) return `#${element.id}`;

    let selector = element.tagName.toLowerCase();
    if (element.className) {
      selector += '.' + element.className.split(' ').join('.');
    }

    return selector;
  }

  private reportIssues(): void {
    console.group(`Contrast Issues Found: ${this.issues.length}`);

    this.issues.forEach((issue, index) => {
      console.group(`Issue ${index + 1}: ${issue.selector}`);
      console.log('Foreground:', issue.foreground);
      console.log('Background:', issue.background);
      console.log(`Contrast Ratio: ${issue.ratio}:1 (Required: ${issue.required}:1)`);
      console.log(`Font Size: ${issue.fontSize}px ${issue.isBold ? 'bold' : 'normal'}`);

      // Suggest fix
      const suggestion = ContrastCalculator.suggestColorAdjustment(
        issue.foreground,
        issue.background,
        'AA',
        issue.fontSize >= 24 || (issue.fontSize >= 18.66 && issue.isBold)
      );

      console.log(`Suggested Color: ${suggestion.suggested} (${suggestion.newRatio.toFixed(2)}:1)`);
      console.log('Element:', issue.element);
      console.groupEnd();
    });

    console.groupEnd();

    // Highlight issues on page
    this.highlightIssues();
  }

  private highlightIssues(): void {
    this.issues.forEach(issue => {
      issue.element.style.outline = '3px solid red';
      issue.element.title = `Contrast issue: ${issue.ratio}:1 (Required: ${issue.required}:1)`;
    });
  }

  exportReport(): string {
    let report = '# Contrast Accessibility Report\n\n';
    report += `Generated: ${new Date().toLocaleString()}\n`;
    report += `Issues Found: ${this.issues.length}\n\n`;

    this.issues.forEach((issue, index) => {
      report += `## Issue ${index + 1}\n\n`;
      report += `- **Element**: \`${issue.selector}\`\n`;
      report += `- **Foreground**: ${issue.foreground}\n`;
      report += `- **Background**: ${issue.background}\n`;
      report += `- **Contrast Ratio**: ${issue.ratio}:1\n`;
      report += `- **Required**: ${issue.required}:1\n`;
      report += `- **Font Size**: ${issue.fontSize}px ${issue.isBold ? 'bold' : 'normal'}\n\n`;

      const suggestion = ContrastCalculator.suggestColorAdjustment(
        issue.foreground,
        issue.background,
        'AA',
        issue.fontSize >= 24 || (issue.fontSize >= 18.66 && issue.isBold)
      );

      report += `**Suggested Fix**: Change foreground color to ${suggestion.suggested} (${suggestion.newRatio.toFixed(2)}:1)\n\n`;
      report += '---\n\n';
    });

    return report;
  }
}

// Usage
const tester = new AutomatedContrastTester();
// tester.testPage();
// console.log(tester.exportReport());
```

---

## Common Contrast Issues

### Issue Patterns

```typescript
interface CommonContrastIssue {
  issue: string;
  example: string;
  solution: string;
  code: string;
}

const commonIssues: CommonContrastIssue[] = [
  {
    issue: "Light gray text on white background",
    example: "#999999 on #FFFFFF = 2.85:1 (Fails AA)",
    solution: "Darken text to #767676 or darker",
    code: `
/* ❌ Bad */
.text-muted {
  color: #999999; /* 2.85:1 - Fails */
}

/* ✓ Good */
.text-muted {
  color: #6c757d; /* 4.5:1 - Passes AA */
}
    `,
  },
  {
    issue: "White text on light colored button",
    example: "#FFFFFF on #FFD700 = 1.16:1 (Fails)",
    solution: "Use darker button color or dark text",
    code: `
/* ❌ Bad */
.button-warning {
  background: #FFD700;
  color: #FFFFFF; /* Poor contrast */
}

/* ✓ Good Option 1: Darker background */
.button-warning {
  background: #CC9900;
  color: #FFFFFF; /* 4.54:1 */
}

/* ✓ Good Option 2: Dark text */
.button-warning {
  background: #FFD700;
  color: #000000; /* 10.29:1 */
}
    `,
  },
  {
    issue: "Placeholder text too light",
    example: "Default placeholder often fails contrast",
    solution: "Style placeholder to meet 4.5:1 ratio",
    code: `
/* ❌ Bad: Browser default */
input::placeholder {
  /* Usually rgba(0,0,0,0.4) - fails */
}

/* ✓ Good */
input::placeholder {
  color: #6c757d; /* 4.5:1 on white */
  opacity: 1;
}
    `,
  },
  {
    issue: "Disabled form controls",
    example: "Disabled state often has low contrast",
    solution: "Ensure 3:1 contrast for non-text UI",
    code: `
/* ❌ Bad */
input:disabled {
  background: #e9ecef;
  color: #adb5bd; /* 2.5:1 - Fails */
}

/* ✓ Good */
input:disabled {
  background: #e9ecef;
  color: #6c757d; /* 3.5:1 on #e9ecef */
  cursor: not-allowed;
}
    `,
  },
  {
    issue: "Link color insufficient",
    example: "Blue links often fail on colored backgrounds",
    solution: "Test links against actual background",
    code: `
/* ❌ Bad: Link on gray background */
.sidebar {
  background: #f8f9fa;
}

.sidebar a {
  color: #007bff; /* 3.78:1 on #f8f9fa - Fails */
}

/* ✓ Good */
.sidebar a {
  color: #0056b3; /* 4.53:1 on #f8f9fa */
}
    `,
  },
  {
    issue: "Form validation errors",
    example: "Red error text often fails contrast",
    solution: "Use darker red or add icon",
    code: `
/* ❌ Bad */
.error {
  color: #ff0000; /* 4.0:1 - Close but fails */
}

/* ✓ Good */
.error {
  color: #d32f2f; /* 4.53:1 - Passes */
}

/* ✓ Better: Don't rely on color alone */
.error::before {
  content: "⚠ ";
  aria-hidden: "true";
}
    `,
  },
];
```

### Visual Examples

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Contrast Issues Demonstration</title>
    <style>
      .comparison {
        display: flex;
        gap: 2rem;
        margin: 2rem 0;
      }

      .example {
        flex: 1;
        padding: 2rem;
        border: 1px solid #ccc;
      }

      .bad {
        background: #ffe0e0;
      }
      .good {
        background: #e0ffe0;
      }

      /* Issue 1: Light gray text */
      .text-fail {
        color: #999999; /* 2.85:1 */
        background: #ffffff;
      }

      .text-pass {
        color: #6c757d; /* 4.5:1 */
        background: #ffffff;
      }

      /* Issue 2: Button contrast */
      .button-fail {
        background: #ffd700;
        color: #ffffff; /* 1.16:1 */
        padding: 0.5rem 1rem;
        border: none;
      }

      .button-pass {
        background: #cc9900;
        color: #ffffff; /* 4.54:1 */
        padding: 0.5rem 1rem;
        border: none;
      }

      /* Issue 3: Links on backgrounds */
      .bg-gray {
        background: #f8f9fa;
        padding: 1rem;
      }

      .link-fail {
        color: #007bff; /* 3.78:1 on #f8f9fa */
      }

      .link-pass {
        color: #0056b3; /* 4.53:1 on #f8f9fa */
      }
    </style>
  </head>
  <body>
    <h1>Common Contrast Issues</h1>

    <!-- Issue 1 -->
    <div class="comparison">
      <div class="example bad">
        <h3>❌ Fails (2.85:1)</h3>
        <p class="text-fail">
          Light gray text on white background. This fails WCAG AA requirements.
        </p>
      </div>

      <div class="example good">
        <h3>✓ Passes (4.5:1)</h3>
        <p class="text-pass">
          Darker gray text on white background. This passes WCAG AA
          requirements.
        </p>
      </div>
    </div>

    <!-- Issue 2 -->
    <div class="comparison">
      <div class="example bad">
        <h3>❌ Fails (1.16:1)</h3>
        <button class="button-fail">Warning Button</button>
      </div>

      <div class="example good">
        <h3>✓ Passes (4.54:1)</h3>
        <button class="button-pass">Warning Button</button>
      </div>
    </div>

    <!-- Issue 3 -->
    <div class="comparison">
      <div class="example bad bg-gray">
        <h3>❌ Fails (3.78:1)</h3>
        <a href="#" class="link-fail">Link on gray background</a>
      </div>

      <div class="example good bg-gray">
        <h3>✓ Passes (4.53:1)</h3>
        <a href="#" class="link-pass">Link on gray background</a>
      </div>
    </div>
  </body>
</html>
```

---

## Color Accessibility Patterns

### Accessible Color Palettes

```typescript
interface ColorPalette {
  name: string;
  primary: string;
  secondary: string;
  success: string;
  warning: string;
  error: string;
  info: string;
  text: string;
  textSecondary: string;
  background: string;
  surface: string;
}

const accessiblePalettes: ColorPalette[] = [
  {
    name: "Light Theme (WCAG AA)",
    primary: "#0066cc", // 4.5:1 on white
    secondary: "#6c757d", // 4.5:1 on white
    success: "#28a745", // 3.0:1 on white (large text only)
    warning: "#856404", // 7.0:1 on white
    error: "#d32f2f", // 4.5:1 on white
    info: "#0c5460", // 7.5:1 on white
    text: "#212529", // 16.9:1 on white
    textSecondary: "#6c757d", // 4.5:1 on white
    background: "#ffffff",
    surface: "#f8f9fa",
  },
  {
    name: "Dark Theme (WCAG AA)",
    primary: "#66b3ff", // 5.6:1 on #212529
    secondary: "#adb5bd", // 7.5:1 on #212529
    success: "#5cb85c", // 4.6:1 on #212529
    warning: "#ffc107", // 10.3:1 on #212529
    error: "#f44336", // 4.8:1 on #212529
    info: "#5bc0de", // 6.5:1 on #212529
    text: "#ffffff", // 16.1:1 on #212529
    textSecondary: "#adb5bd", // 7.5:1 on #212529
    background: "#212529",
    surface: "#343a40",
  },
  {
    name: "High Contrast (WCAG AAA)",
    primary: "#0000EE", // 8.6:1 on white
    secondary: "#383838", // 10.4:1 on white
    success: "#006400", // 7.5:1 on white
    warning: "#664400", // 8.8:1 on white
    error: "#8B0000", // 8.3:1 on white
    info: "#00008B", // 11.1:1 on white
    text: "#000000", // 21:1 on white
    textSecondary: "#383838", // 10.4:1 on white
    background: "#ffffff",
    surface: "#f0f0f0",
  },
];
```

### CSS Custom Properties Implementation

```css
/* Accessible Color System */
:root {
  /* Primary Colors - AA Compliant */
  --color-primary-50: #e3f2fd;
  --color-primary-100: #bbdefb;
  --color-primary-200: #90caf9;
  --color-primary-300: #64b5f6;
  --color-primary-400: #42a5f5;
  --color-primary-500: #2196f3;
  --color-primary-600: #0066cc; /* 4.5:1 on white */
  --color-primary-700: #0056b3; /* 5.9:1 on white */
  --color-primary-800: #004494; /* 7.5:1 on white */
  --color-primary-900: #003366; /* 10.1:1 on white */

  /* Semantic Colors */
  --color-text-primary: #212529; /* 16.9:1 */
  --color-text-secondary: #6c757d; /* 4.5:1 */
  --color-text-disabled: #adb5bd; /* 3:1 (UI only) */

  --color-success: #28a745; /* 3:1 (large text or UI) */
  --color-warning: #856404; /* 7:1 */
  --color-error: #d32f2f; /* 4.5:1 */
  --color-info: #0c5460; /* 7.5:1 */

  /* Interactive States */
  --color-link: #0066cc; /* 4.5:1 */
  --color-link-hover: #0056b3; /* 5.9:1 */
  --color-link-visited: #800080; /* 5.9:1 */

  /* Form Elements */
  --color-input-border: #6c757d; /* 4.5:1 */
  --color-input-focus: #0066cc; /* 4.5:1 */
  --color-input-error: #d32f2f; /* 4.5:1 */

  /* Focus Indicators */
  --color-focus-outline: #4a90e2; /* 3.1:1 (non-text) */
  --focus-outline-width: 3px;
  --focus-outline-offset: 2px;
}

/* Apply Colors */
body {
  color: var(--color-text-primary);
  background: #ffffff;
}

a {
  color: var(--color-link);
  text-decoration: underline;
}

a:hover {
  color: var(--color-link-hover);
}

a:visited {
  color: var(--color-link-visited);
}

/* Form Elements */
input,
textarea,
select {
  border: 2px solid var(--color-input-border);
  color: var(--color-text-primary);
}

input:focus,
textarea:focus,
select:focus {
  border-color: var(--color-input-focus);
  outline: var(--focus-outline-width) solid var(--color-focus-outline);
  outline-offset: var(--focus-outline-offset);
}

input.error,
textarea.error,
select.error {
  border-color: var(--color-input-error);
}

/* Status Messages */
.success {
  color: var(--color-success);
  font-weight: bold;
}

.warning {
  color: var(--color-warning);
}

.error {
  color: var(--color-error);
}

.info {
  color: var(--color-info);
}

/* Dark Mode Support */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text-primary: #ffffff;
    --color-text-secondary: #adb5bd;
    --color-link: #66b3ff;
    --color-link-hover: #80c3ff;
    --color-success: #5cb85c;
    --color-warning: #ffc107;
    --color-error: #f44336;
    --color-info: #5bc0de;
  }

  body {
    background: #212529;
  }
}

/* High Contrast Mode */
@media (prefers-contrast: high) {
  :root {
    --color-text-primary: #000000;
    --color-link: #0000ee;
    --color-link-hover: #0000cc;
    --color-success: #006400;
    --color-warning: #664400;
    --color-error: #8b0000;
    --color-info: #00008b;
  }

  * {
    border-color: currentColor !important;
  }
}
```

---

## Dynamic Contrast Adjustment

### Automatic Contrast Fixer

```typescript
class DynamicContrastAdjuster {
  private targetLevel: "AA" | "AAA" = "AA";

  /**
   * Automatically fix contrast issues on the page
   */
  fixPageContrast(): void {
    const elements = this.getAllTextElements();

    elements.forEach((element) => {
      this.fixElementContrast(element);
    });
  }

  private getAllTextElements(): HTMLElement[] {
    return Array.from(document.querySelectorAll("*")).filter((el) => {
      const element = el as HTMLElement;
      const style = window.getComputedStyle(element);

      // Has visible text
      return (
        element.textContent?.trim() &&
        style.display !== "none" &&
        style.visibility !== "hidden"
      );
    }) as HTMLElement[];
  }

  private fixElementContrast(element: HTMLElement): void {
    const style = window.getComputedStyle(element);
    const foreground = style.color;
    const background = this.getBackgroundColor(element);

    if (!background) return;

    // Check if adjustment needed
    const fgHex = this.rgbToHex(foreground);
    const bgHex = this.rgbToHex(background);

    const ratio = ContrastCalculator.calculateContrast(fgHex, bgHex);

    const fontSize = parseFloat(style.fontSize);
    const isBold = parseInt(style.fontWeight) >= 700;
    const isLargeText = fontSize >= 24 || (fontSize >= 18.66 && isBold);

    const required =
      this.targetLevel === "AAA"
        ? isLargeText
          ? 4.5
          : 7
        : isLargeText
          ? 3
          : 4.5;

    if (ratio < required) {
      // Fix contrast
      const suggestion = ContrastCalculator.suggestColorAdjustment(
        fgHex,
        bgHex,
        this.targetLevel,
        isLargeText,
      );

      element.style.color = suggestion.suggested;

      // Add data attribute for debugging
      element.setAttribute("data-contrast-adjusted", "true");
      element.setAttribute("data-original-color", fgHex);
      element.setAttribute("data-new-color", suggestion.suggested);
      element.setAttribute("data-ratio-before", ratio.toFixed(2));
      element.setAttribute("data-ratio-after", suggestion.newRatio.toFixed(2));
    }
  }

  private getBackgroundColor(element: HTMLElement): string | null {
    let current: HTMLElement | null = element;

    while (current) {
      const style = window.getComputedStyle(current);
      const bg = style.backgroundColor;

      if (bg && bg !== "rgba(0, 0, 0, 0)" && bg !== "transparent") {
        return bg;
      }

      current = current.parentElement;
    }

    return "rgb(255, 255, 255)";
  }

  private rgbToHex(rgb: string): string {
    const match = rgb.match(/\d+/g);
    if (!match) return "#000000";

    return (
      "#" +
      match
        .slice(0, 3)
        .map((val) => {
          const hex = parseInt(val).toString(16);
          return hex.length === 1 ? "0" + hex : hex;
        })
        .join("")
    );
  }

  /**
   * Generate accessible color variations
   */
  generateAccessiblePalette(baseColor: string): Record<string, string> {
    const variations: Record<string, string> = {
      base: baseColor,
    };

    // Generate lighter variations
    for (let i = 1; i <= 5; i++) {
      const lightness = 50 + i * 10;
      variations[`light-${i}`] = this.adjustLightness(baseColor, lightness);
    }

    // Generate darker variations
    for (let i = 1; i <= 5; i++) {
      const lightness = 50 - i * 10;
      variations[`dark-${i}`] = this.adjustLightness(baseColor, lightness);
    }

    // Find AA and AAA compliant versions for white background
    variations["on-white-aa"] = ContrastCalculator.suggestColorAdjustment(
      baseColor,
      "#ffffff",
      "AA",
      false,
    ).suggested;

    variations["on-white-aaa"] = ContrastCalculator.suggestColorAdjustment(
      baseColor,
      "#ffffff",
      "AAA",
      false,
    ).suggested;

    // Find versions for dark background
    variations["on-dark-aa"] = ContrastCalculator.suggestColorAdjustment(
      baseColor,
      "#212529",
      "AA",
      false,
    ).suggested;

    return variations;
  }

  private adjustLightness(hex: string, lightness: number): string {
    // Convert hex to HSL, adjust lightness, convert back
    // Simplified implementation
    return hex; // Would implement full HSL conversion
  }

  /**
   * Create contrast adjustment UI toggle
   */
  createContrastToggle(): HTMLElement {
    const toggle = document.createElement("button");
    toggle.textContent = "Toggle High Contrast";
    toggle.setAttribute("aria-pressed", "false");
    toggle.style.cssText = `
      position: fixed;
      bottom: 20px;
      right: 20px;
      padding: 10px 20px;
      background: #0066cc;
      color: #ffffff;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      z-index: 10000;
    `;

    toggle.addEventListener("click", () => {
      const isPressed = toggle.getAttribute("aria-pressed") === "true";

      if (isPressed) {
        this.disableContrastAdjustment();
        toggle.setAttribute("aria-pressed", "false");
        toggle.textContent = "Enable High Contrast";
      } else {
        this.targetLevel = "AAA";
        this.fixPageContrast();
        toggle.setAttribute("aria-pressed", "true");
        toggle.textContent = "Disable High Contrast";
      }
    });

    return toggle;
  }

  private disableContrastAdjustment(): void {
    const adjusted = document.querySelectorAll(
      '[data-contrast-adjusted="true"]',
    );
    adjusted.forEach((el) => {
      const element = el as HTMLElement;
      const original = element.getAttribute("data-original-color");
      if (original) {
        element.style.color = original;
      }
      element.removeAttribute("data-contrast-adjusted");
    });
  }
}

// Usage
const adjuster = new DynamicContrastAdjuster();

// Add toggle button to page
document.body.appendChild(adjuster.createContrastToggle());

// Generate palette
const palette = adjuster.generateAccessiblePalette("#2196f3");
console.log("Accessible Palette:", palette);
```

---

## Best Practices

### Contrast Checklist

```typescript
interface ContrastBestPractice {
  practice: string;
  wcagCriterion: string;
  implementation: string;
  example: string;
}

const bestPractices: ContrastBestPractice[] = [
  {
    practice: "Test all text against actual background",
    wcagCriterion: "1.4.3",
    implementation: "Use browser DevTools or contrast checker on rendered page",
    example: "Test text on gradient backgrounds at multiple points",
  },
  {
    practice: "Don't rely on color alone",
    wcagCriterion: "1.4.1",
    implementation: "Add icons, patterns, or text labels alongside color",
    example: "Error messages should have icon AND red color",
  },
  {
    practice: "Test in different lighting conditions",
    wcagCriterion: "General",
    implementation: "View site in bright sunlight and dim lighting",
    example: "Reduce screen brightness to simulate mobile outdoors",
  },
  {
    practice: "Support user preferences",
    wcagCriterion: "General",
    implementation: "Respect prefers-contrast and prefers-color-scheme",
    example: "@media (prefers-contrast: high) { /* styles */ }",
  },
  {
    practice: "Provide high contrast mode",
    wcagCriterion: "General",
    implementation: "Offer toggle for enhanced contrast",
    example: "AAA level colors for all text when enabled",
  },
  {
    practice: "Test with actual users",
    wcagCriterion: "General",
    implementation: "Include people with low vision in user testing",
    example: "Observe where they struggle to read",
  },
  {
    practice: "Document color choices",
    wcagCriterion: "General",
    implementation: "Maintain style guide with contrast ratios",
    example: "Primary blue #0066cc - 4.5:1 on white",
  },
  {
    practice: "Automate contrast testing",
    wcagCriterion: "General",
    implementation: "Add contrast checks to CI/CD pipeline",
    example: "Fail build if new contrast issues introduced",
  },
];
```

### Implementation Workflow

```typescript
class ContrastWorkflow {
  static getDesignPhaseSteps(): string[] {
    return [
      "1. Choose base colors",
      "2. Test contrast ratios in design tool",
      "3. Create accessible color palette",
      "4. Document all color combinations",
      "5. Test with color blindness simulators",
      "6. Get feedback from users with low vision",
    ];
  }

  static getDevelopmentPhaseSteps(): string[] {
    return [
      "1. Implement colors using CSS custom properties",
      "2. Add automated contrast tests",
      "3. Test with browser DevTools",
      "4. Use axe or WAVE for automated scanning",
      "5. Manual testing in different lighting",
      "6. Test with actual assistive technology",
    ];
  }

  static getMaintenancePhaseSteps(): string[] {
    return [
      "1. Run automated tests on every commit",
      "2. Review user feedback about readability",
      "3. Test new features for contrast",
      "4. Update color palette as needed",
      "5. Regular audits with accessibility tools",
      "6. Monitor browser/OS updates affecting contrast",
    ];
  }
}
```

---

## Key Takeaways

1. **WCAG requires 4.5:1 for normal text** - Level AA mandates 4.5:1 contrast ratio for text smaller than 18pt (or 14pt bold); 3:1 for larger text.

2. **Non-text elements need 3:1** - UI components, icons, and focus indicators must have at least 3:1 contrast ratio with adjacent colors (WCAG 2.1 1.4.11).

3. **Test against actual backgrounds** - Always calculate contrast using the real background color element appears on, including gradients and images.

4. **Automated tools catch most issues** - Browser DevTools, axe, and WAVE can identify 80%+ of contrast problems automatically.

5. **AAA provides enhanced accessibility** - 7:1 ratio for normal text (AAA) benefits users with low vision significantly more than AA minimum.

6. **Don't rely on color alone** - Always combine color with icons, labels, or patterns to convey information (WCAG 1.4.1).

7. **Support user preferences** - Implement prefers-contrast and prefers-color-scheme media queries to respect user settings.

8. **Test in real conditions** - Verify contrast on actual devices in bright sunlight, dim lighting, and various screen qualities.

9. **Document color system** - Maintain accessible color palette with documented contrast ratios for all combinations used.

10. **Integrate testing early** - Add contrast checking to design tools, development workflow, and CI/CD pipeline to catch issues before production.
