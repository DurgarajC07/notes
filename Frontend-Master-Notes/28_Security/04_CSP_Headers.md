# Content Security Policy (CSP) Headers

## Core Concepts

Content Security Policy (CSP) is a powerful security layer that helps prevent Cross-Site Scripting (XSS), clickjacking, and other code injection attacks. CSP works by specifying trusted sources for content through HTTP headers or meta tags, allowing browsers to only execute or render resources from approved origins.

### Key Concepts

- **CSP Directives**: Rules that control which resources can be loaded
- **Nonce-Based CSP**: Dynamic tokens for inline scripts/styles
- **strict-dynamic**: Modern CSP that simplifies policy management
- **Report-Only Mode**: Test policies without blocking resources
- **Violation Reporting**: Monitor CSP violations for security insights
- **Defense in Depth**: CSP as part of layered security strategy

### CSP Security Benefits

- **XSS Prevention**: Block execution of untrusted scripts
- **Data Exfiltration Prevention**: Control where data can be sent
- **Clickjacking Protection**: Control framing of pages
- **Protocol Enforcement**: Upgrade insecure requests
- **Mixed Content Prevention**: Block HTTP resources on HTTPS pages

## TypeScript/JavaScript Code Examples

### Example 1: Basic CSP Header Builder

```typescript
// CSP header builder with type safety
class CSPBuilder {
  private directives: Map<CSPDirective, string[]> = new Map();

  // Add source to directive
  addSource(directive: CSPDirective, source: string | string[]): this {
    const sources = Array.isArray(source) ? source : [source];
    const existing = this.directives.get(directive) || [];
    this.directives.set(directive, [...existing, ...sources]);
    return this;
  }

  // Set default-src (fallback for all fetch directives)
  defaultSrc(...sources: string[]): this {
    return this.addSource("default-src", sources);
  }

  // Set script-src
  scriptSrc(...sources: string[]): this {
    return this.addSource("script-src", sources);
  }

  // Set style-src
  styleSrc(...sources: string[]): this {
    return this.addSource("style-src", sources);
  }

  // Set img-src
  imgSrc(...sources: string[]): this {
    return this.addSource("img-src", sources);
  }

  // Set connect-src (fetch, XHR, WebSocket)
  connectSrc(...sources: string[]): this {
    return this.addSource("connect-src", sources);
  }

  // Set font-src
  fontSrc(...sources: string[]): this {
    return this.addSource("font-src", sources);
  }

  // Set object-src (plugins)
  objectSrc(...sources: string[]): this {
    return this.addSource("object-src", sources);
  }

  // Set media-src (audio, video)
  mediaSrc(...sources: string[]): this {
    return this.addSource("media-src", sources);
  }

  // Set frame-src (iframes)
  frameSrc(...sources: string[]): this {
    return this.addSource("frame-src", sources);
  }

  // Set frame-ancestors (who can frame this page)
  frameAncestors(...sources: string[]): this {
    return this.addSource("frame-ancestors", sources);
  }

  // Set form-action
  formAction(...sources: string[]): this {
    return this.addSource("form-action", sources);
  }

  // Set base-uri
  baseUri(...sources: string[]): this {
    return this.addSource("base-uri", sources);
  }

  // Add upgrade-insecure-requests
  upgradeInsecureRequests(): this {
    this.directives.set("upgrade-insecure-requests", []);
    return this;
  }

  // Add block-all-mixed-content
  blockAllMixedContent(): this {
    this.directives.set("block-all-mixed-content", []);
    return this;
  }

  // Set report-uri
  reportUri(uri: string): this {
    return this.addSource("report-uri", [uri]);
  }

  // Set report-to
  reportTo(group: string): this {
    return this.addSource("report-to", [group]);
  }

  // Build CSP header string
  build(): string {
    const parts: string[] = [];

    for (const [directive, sources] of this.directives) {
      if (sources.length === 0) {
        // Directive without value (e.g., upgrade-insecure-requests)
        parts.push(directive);
      } else {
        parts.push(`${directive} ${sources.join(" ")}`);
      }
    }

    return parts.join("; ");
  }

  // Build for report-only mode
  buildReportOnly(): string {
    return this.build();
  }

  // Clear all directives
  clear(): void {
    this.directives.clear();
  }

  // Clone builder
  clone(): CSPBuilder {
    const cloned = new CSPBuilder();
    this.directives.forEach((sources, directive) => {
      cloned.directives.set(directive, [...sources]);
    });
    return cloned;
  }
}

type CSPDirective =
  | "default-src"
  | "script-src"
  | "script-src-elem"
  | "script-src-attr"
  | "style-src"
  | "style-src-elem"
  | "style-src-attr"
  | "img-src"
  | "font-src"
  | "connect-src"
  | "media-src"
  | "object-src"
  | "frame-src"
  | "frame-ancestors"
  | "form-action"
  | "base-uri"
  | "manifest-src"
  | "worker-src"
  | "prefetch-src"
  | "upgrade-insecure-requests"
  | "block-all-mixed-content"
  | "report-uri"
  | "report-to";

// Usage: Basic secure policy
const basicPolicy = new CSPBuilder()
  .defaultSrc("'self'")
  .scriptSrc("'self'", "https://cdn.example.com")
  .styleSrc("'self'", "https://fonts.googleapis.com")
  .imgSrc("'self'", "https:", "data:")
  .fontSrc("'self'", "https://fonts.gstatic.com")
  .connectSrc("'self'", "https://api.example.com")
  .frameSrc("'none'")
  .objectSrc("'none'")
  .upgradeInsecureRequests()
  .build();

console.log("CSP Header:", basicPolicy);
```

### Example 2: Nonce-Based CSP Implementation

```typescript
// Generate and manage nonces for inline scripts/styles
class CSPNonceManager {
  private currentNonce: string = "";
  private nonceCache: Map<string, string> = new Map();

  // Generate cryptographically secure nonce
  generateNonce(): string {
    // Use crypto API for secure random values
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);

    const nonce = btoa(String.fromCharCode(...array))
      .replace(/\+/g, "-")
      .replace(/\//g, "_")
      .replace(/=/g, "");

    this.currentNonce = nonce;
    return nonce;
  }

  // Get current nonce
  getCurrentNonce(): string {
    return this.currentNonce;
  }

  // Build CSP with nonce
  buildNoncePolicy(nonce: string): string {
    const csp = new CSPBuilder()
      .defaultSrc("'self'")
      .scriptSrc("'self'", `'nonce-${nonce}'`)
      .styleSrc("'self'", `'nonce-${nonce}'`)
      .imgSrc("'self'", "https:", "data:")
      .connectSrc("'self'", "https://api.example.com")
      .fontSrc("'self'", "https://fonts.gstatic.com")
      .objectSrc("'none'")
      .baseUri("'self'")
      .frameAncestors("'none'")
      .upgradeInsecureRequests()
      .reportUri("/csp-violation-report")
      .build();

    return csp;
  }

  // Apply nonce to inline script
  applyNonceToScript(script: HTMLScriptElement): void {
    if (!script.src && this.currentNonce) {
      script.setAttribute("nonce", this.currentNonce);
    }
  }

  // Apply nonce to inline style
  applyNonceToStyle(style: HTMLStyleElement): void {
    if (this.currentNonce) {
      style.setAttribute("nonce", this.currentNonce);
    }
  }

  // Create script with nonce
  createScriptWithNonce(code: string): HTMLScriptElement {
    const script = document.createElement("script");
    script.setAttribute("nonce", this.currentNonce);
    script.textContent = code;
    return script;
  }

  // Create style with nonce
  createStyleWithNonce(css: string): HTMLStyleElement {
    const style = document.createElement("style");
    style.setAttribute("nonce", this.currentNonce);
    style.textContent = css;
    return style;
  }

  // Rotate nonce (for page transitions)
  rotateNonce(): string {
    return this.generateNonce();
  }
}

// Express middleware for nonce-based CSP
class CSPMiddleware {
  private nonceManager = new CSPNonceManager();

  middleware() {
    return (req: any, res: any, next: any) => {
      // Generate nonce for this request
      const nonce = this.nonceManager.generateNonce();

      // Build CSP with nonce
      const csp = this.nonceManager.buildNoncePolicy(nonce);

      // Set CSP header
      res.setHeader("Content-Security-Policy", csp);

      // Make nonce available to templates
      res.locals.cspNonce = nonce;

      next();
    };
  }

  reportOnlyMiddleware() {
    return (req: any, res: any, next: any) => {
      const nonce = this.nonceManager.generateNonce();
      const csp = this.nonceManager.buildNoncePolicy(nonce);

      // Use report-only header for testing
      res.setHeader("Content-Security-Policy-Report-Only", csp);
      res.locals.cspNonce = nonce;

      next();
    };
  }
}

// Usage in HTML template
function renderHTML(nonce: string): string {
  return `
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>CSP with Nonce</title>
    
    <!-- Inline style with nonce -->
    <style nonce="${nonce}">
        body { font-family: Arial, sans-serif; }
    </style>
</head>
<body>
    <h1>CSP Nonce Example</h1>
    
    <!-- Inline script with nonce -->
    <script nonce="${nonce}">
        console.log('This script is allowed by CSP nonce');
        
        // Dynamic script injection also needs nonce
        const script = document.createElement('script');
        script.nonce = '${nonce}';
        script.textContent = 'console.log("Dynamic script with nonce")';
        document.head.appendChild(script);
    </script>
</body>
</html>
  `;
}

// Client-side nonce extraction
class ClientSideNonceManager {
  private nonce: string | null = null;

  // Extract nonce from existing script tag
  extractNonce(): string | null {
    if (this.nonce) return this.nonce;

    // Find script with nonce attribute
    const scriptWithNonce = document.querySelector("script[nonce]");
    if (scriptWithNonce) {
      this.nonce = scriptWithNonce.getAttribute("nonce");
    }

    return this.nonce;
  }

  // Get current nonce
  getNonce(): string | null {
    return this.nonce || this.extractNonce();
  }

  // Create dynamic script with nonce
  createScript(code: string): HTMLScriptElement {
    const script = document.createElement("script");
    const nonce = this.getNonce();

    if (nonce) {
      script.setAttribute("nonce", nonce);
    }

    script.textContent = code;
    return script;
  }

  // Load external script dynamically
  async loadScript(src: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const script = document.createElement("script");
      const nonce = this.getNonce();

      if (nonce) {
        script.setAttribute("nonce", nonce);
      }

      script.src = src;
      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load script: ${src}`));

      document.head.appendChild(script);
    });
  }
}

// Usage
const clientNonce = new ClientSideNonceManager();

// Add dynamic inline script
const dynamicScript = clientNonce.createScript(`
  console.log('Dynamic script with CSP nonce');
`);
document.body.appendChild(dynamicScript);

// Load external script
clientNonce
  .loadScript("https://cdn.example.com/library.js")
  .then(() => console.log("Script loaded"))
  .catch((err) => console.error("Script load failed:", err));
```

### Example 3: Strict-Dynamic CSP

```typescript
// Modern CSP with strict-dynamic
class StrictDynamicCSP {
  private nonceManager = new CSPNonceManager();

  // Build strict-dynamic policy
  buildStrictPolicy(nonce: string): string {
    const csp = new CSPBuilder()
      .scriptSrc(`'nonce-${nonce}'`, "'strict-dynamic'")
      .objectSrc("'none'")
      .baseUri("'none'")
      .build();

    // Note: strict-dynamic makes 'self', 'unsafe-inline', https:, etc. ignored
    // Only nonce/hash and dynamically added scripts are allowed
    return csp;
  }

  // Build with fallback for older browsers
  buildStrictPolicyWithFallback(nonce: string): string {
    const csp = new CSPBuilder()
      .scriptSrc(
        `'nonce-${nonce}'`,
        "'strict-dynamic'",
        // Fallbacks for browsers without strict-dynamic support
        "https:",
        "'unsafe-inline'",
      )
      .objectSrc("'none'")
      .baseUri("'none'")
      .build();

    return csp;
  }

  // Middleware with strict-dynamic
  middleware() {
    return (req: any, res: any, next: any) => {
      const nonce = this.nonceManager.generateNonce();
      const csp = this.buildStrictPolicyWithFallback(nonce);

      res.setHeader("Content-Security-Policy", csp);
      res.locals.cspNonce = nonce;

      next();
    };
  }
}

// Usage
const strictCSP = new StrictDynamicCSP();

// Example HTML with strict-dynamic
function renderStrictHTML(nonce: string): string {
  return `
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Strict-Dynamic CSP</title>
</head>
<body>
    <h1>Strict-Dynamic Example</h1>
    
    <!-- Only this script needs nonce -->
    <script nonce="${nonce}">
        // This script is trusted
        console.log('Trusted script with nonce');
        
        // Dynamically created scripts are automatically trusted
        // No nonce needed!
        const script = document.createElement('script');
        script.src = 'https://cdn.example.com/library.js';
        document.head.appendChild(script);
        
        // Inline dynamic scripts also work
        const inlineScript = document.createElement('script');
        inlineScript.textContent = 'console.log("Dynamic inline script")';
        document.head.appendChild(inlineScript);
    </script>
</body>
</html>
  `;
}

// Trusted Types with strict-dynamic
class TrustedTypesCSP {
  buildPolicy(): string {
    const csp = new CSPBuilder()
      .addSource("require-trusted-types-for", ["'script'"])
      .addSource("trusted-types", ["default", "myPolicy"])
      .scriptSrc("'strict-dynamic'")
      .objectSrc("'none'")
      .build();

    return csp;
  }

  // Create trusted types policy
  createTrustedTypesPolicy(): TrustedTypePolicy {
    if (!window.trustedTypes) {
      throw new Error("Trusted Types not supported");
    }

    return window.trustedTypes.createPolicy("myPolicy", {
      createHTML: (input: string) => {
        // Sanitize HTML
        return this.sanitizeHTML(input);
      },
      createScript: (input: string) => {
        // Validate script
        return this.validateScript(input);
      },
      createScriptURL: (input: string) => {
        // Validate script URL
        return this.validateScriptURL(input);
      },
    });
  }

  private sanitizeHTML(html: string): string {
    // Implement HTML sanitization
    return html;
  }

  private validateScript(script: string): string {
    // Implement script validation
    return script;
  }

  private validateScriptURL(url: string): string {
    // Validate URL is from trusted origin
    const trustedOrigins = ["https://cdn.example.com"];
    const parsedURL = new URL(url);

    if (!trustedOrigins.includes(parsedURL.origin)) {
      throw new Error(`Untrusted script URL: ${url}`);
    }

    return url;
  }
}
```

### Example 4: CSP Violation Reporter

```typescript
// Handle and report CSP violations
class CSPViolationReporter {
  private violations: CSPViolation[] = [];
  private reportEndpoint: string;
  private maxViolations = 100;

  constructor(reportEndpoint: string = "/csp-violation-report") {
    this.reportEndpoint = reportEndpoint;
    this.setupListener();
  }

  private setupListener(): void {
    document.addEventListener("securitypolicyviolation", (e) => {
      this.handleViolation(e as SecurityPolicyViolationEvent);
    });
  }

  private handleViolation(event: SecurityPolicyViolationEvent): void {
    const violation: CSPViolation = {
      documentURI: event.documentURI,
      referrer: event.referrer,
      blockedURI: event.blockedURI,
      violatedDirective: event.violatedDirective,
      effectiveDirective: event.effectiveDirective,
      originalPolicy: event.originalPolicy,
      sourceFile: event.sourceFile,
      sample: event.sample,
      disposition: event.disposition,
      statusCode: event.statusCode,
      lineNumber: event.lineNumber,
      columnNumber: event.columnNumber,
      timestamp: Date.now(),
    };

    this.violations.push(violation);

    // Trim if too many violations
    if (this.violations.length > this.maxViolations) {
      this.violations.shift();
    }

    // Report violation
    this.reportViolation(violation);

    // Log to console in development
    if (process.env.NODE_ENV === "development") {
      console.warn("CSP Violation:", violation);
    }
  }

  private async reportViolation(violation: CSPViolation): Promise<void> {
    try {
      await fetch(this.reportEndpoint, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          "csp-report": violation,
        }),
      });
    } catch (error) {
      console.error("Failed to report CSP violation:", error);
    }
  }

  // Get violation statistics
  getStats(): ViolationStats {
    const byDirective = new Map<string, number>();
    const bySource = new Map<string, number>();

    this.violations.forEach((v) => {
      // Count by directive
      const directive = v.violatedDirective;
      byDirective.set(directive, (byDirective.get(directive) || 0) + 1);

      // Count by source
      const source = v.blockedURI;
      bySource.set(source, (bySource.get(source) || 0) + 1);
    });

    return {
      total: this.violations.length,
      byDirective: Object.fromEntries(byDirective),
      bySource: Object.fromEntries(bySource),
      recent: this.violations.slice(-10),
    };
  }

  // Get all violations
  getViolations(): CSPViolation[] {
    return [...this.violations];
  }

  // Clear violations
  clear(): void {
    this.violations = [];
  }

  // Export violations as JSON
  export(): string {
    return JSON.stringify(this.violations, null, 2);
  }
}

interface CSPViolation {
  documentURI: string;
  referrer: string;
  blockedURI: string;
  violatedDirective: string;
  effectiveDirective: string;
  originalPolicy: string;
  sourceFile: string;
  sample: string;
  disposition: string;
  statusCode: number;
  lineNumber: number;
  columnNumber: number;
  timestamp: number;
}

interface ViolationStats {
  total: number;
  byDirective: Record<string, number>;
  bySource: Record<string, number>;
  recent: CSPViolation[];
}

// Server-side violation handler
class CSPViolationHandler {
  private storage: CSPViolation[] = [];
  private readonly MAX_STORAGE = 1000;

  // Handle violation report
  async handleReport(report: any): Promise<void> {
    const violation = report["csp-report"];

    // Store violation
    this.storage.push({
      ...violation,
      timestamp: Date.now(),
    });

    // Trim storage
    if (this.storage.length > this.MAX_STORAGE) {
      this.storage.shift();
    }

    // Analyze and alert if needed
    this.analyzeViolation(violation);
  }

  private analyzeViolation(violation: any): void {
    // Check for common attack patterns
    if (this.isXSSAttempt(violation)) {
      console.error("⚠️ Potential XSS attempt detected:", violation);
      this.alertSecurity(violation);
    }

    // Check for misconfiguration
    if (this.isMisconfiguration(violation)) {
      console.warn("CSP misconfiguration detected:", violation);
    }
  }

  private isXSSAttempt(violation: any): boolean {
    const blockedURI = violation.blockedURI || "";
    const sample = violation.sample || "";

    // Check for common XSS patterns
    const xssPatterns = [/<script/i, /javascript:/i, /onerror=/i, /onclick=/i];

    return xssPatterns.some(
      (pattern) => pattern.test(blockedURI) || pattern.test(sample),
    );
  }

  private isMisconfiguration(violation: any): boolean {
    const blockedURI = violation.blockedURI || "";
    const documentURI = violation.documentURI || "";

    // Check if blocking own resources
    return blockedURI.startsWith(new URL(documentURI).origin);
  }

  private alertSecurity(violation: any): void {
    // Send alert to security team
    console.error("Security alert sent for:", violation);
  }

  // Get statistics
  getStatistics(): any {
    const byDirective = new Map<string, number>();
    const byURI = new Map<string, number>();

    this.storage.forEach((v) => {
      const directive = v.violatedDirective;
      byDirective.set(directive, (byDirective.get(directive) || 0) + 1);

      const uri = v.blockedURI;
      byURI.set(uri, (byURI.get(uri) || 0) + 1);
    });

    return {
      total: this.storage.length,
      byDirective: Object.fromEntries(byDirective),
      byURI: Object.fromEntries(byURI),
      recentViolations: this.storage.slice(-20),
    };
  }
}

// Usage
const violationReporter = new CSPViolationReporter("/api/csp-violations");

// View statistics in console
setTimeout(() => {
  const stats = violationReporter.getStats();
  console.log("CSP Violation Statistics:", stats);
}, 5000);
```

### Example 5: CSP Testing Framework

```typescript
// Test CSP policies before deployment
class CSPTester {
  private violations: string[] = [];

  // Test policy with sample resources
  async testPolicy(
    policy: string,
    testCases: CSPTestCase[],
  ): Promise<CSPTestResult> {
    this.violations = [];

    const results = await Promise.all(
      testCases.map((testCase) => this.runTestCase(policy, testCase)),
    );

    const passed = results.filter((r) => r.passed).length;
    const failed = results.filter((r) => !r.passed).length;

    return {
      policy,
      totalTests: testCases.length,
      passed,
      failed,
      testResults: results,
      violations: this.violations,
    };
  }

  private async runTestCase(
    policy: string,
    testCase: CSPTestCase,
  ): Promise<TestCaseResult> {
    try {
      const allowed = this.checkIfAllowed(policy, testCase);

      if (allowed === testCase.shouldAllow) {
        return {
          name: testCase.name,
          passed: true,
          expected: testCase.shouldAllow,
          actual: allowed,
        };
      } else {
        this.violations.push(
          `Test "${testCase.name}" failed: expected ${testCase.shouldAllow}, got ${allowed}`,
        );

        return {
          name: testCase.name,
          passed: false,
          expected: testCase.shouldAllow,
          actual: allowed,
          message: `Resource should ${testCase.shouldAllow ? "be allowed" : "be blocked"}`,
        };
      }
    } catch (error) {
      return {
        name: testCase.name,
        passed: false,
        expected: testCase.shouldAllow,
        actual: false,
        message: (error as Error).message,
      };
    }
  }

  private checkIfAllowed(policy: string, testCase: CSPTestCase): boolean {
    const directive = testCase.directive;
    const source = testCase.source;

    // Parse policy
    const directives = this.parsePolicy(policy);

    // Get directive rules
    const rules =
      directives.get(directive) || directives.get("default-src") || [];

    // Check if source matches any rule
    return this.matchesRule(source, rules);
  }

  private parsePolicy(policy: string): Map<string, string[]> {
    const directives = new Map<string, string[]>();

    policy.split(";").forEach((directive) => {
      const parts = directive.trim().split(/\s+/);
      if (parts.length > 0) {
        const name = parts[0];
        const sources = parts.slice(1);
        directives.set(name, sources);
      }
    });

    return directives;
  }

  private matchesRule(source: string, rules: string[]): boolean {
    // Check for 'none'
    if (rules.includes("'none'")) {
      return false;
    }

    // Check for 'self'
    if (rules.includes("'self'")) {
      try {
        const sourceURL = new URL(source);
        const currentOrigin = window.location.origin;
        if (sourceURL.origin === currentOrigin) {
          return true;
        }
      } catch {
        // Not a valid URL
      }
    }

    // Check for specific origins
    for (const rule of rules) {
      if (rule === "*") return true;
      if (rule === source) return true;
      if (this.matchesWildcard(source, rule)) return true;
    }

    return false;
  }

  private matchesWildcard(source: string, pattern: string): boolean {
    // Simple wildcard matching
    const regex = new RegExp(
      "^" + pattern.replace(/\*/g, ".*").replace(/\./g, "\\.") + "$",
    );
    return regex.test(source);
  }

  // Generate test report
  generateReport(result: CSPTestResult): string {
    const lines: string[] = [];

    lines.push("=== CSP Test Report ===");
    lines.push(`Policy: ${result.policy}`);
    lines.push(`Total Tests: ${result.totalTests}`);
    lines.push(`Passed: ${result.passed}`);
    lines.push(`Failed: ${result.failed}`);
    lines.push("");

    if (result.failed > 0) {
      lines.push("Failed Tests:");
      result.testResults
        .filter((r) => !r.passed)
        .forEach((r) => {
          lines.push(`  ❌ ${r.name}: ${r.message}`);
        });
    }

    return lines.join("\n");
  }
}

interface CSPTestCase {
  name: string;
  directive: string;
  source: string;
  shouldAllow: boolean;
}

interface TestCaseResult {
  name: string;
  passed: boolean;
  expected: boolean;
  actual: boolean;
  message?: string;
}

interface CSPTestResult {
  policy: string;
  totalTests: number;
  passed: number;
  failed: number;
  testResults: TestCaseResult[];
  violations: string[];
}

// Usage: Test e-commerce site policy
const tester = new CSPTester();

const policy = new CSPBuilder()
  .defaultSrc("'self'")
  .scriptSrc("'self'", "https://cdn.example.com")
  .styleSrc("'self'", "https://fonts.googleapis.com")
  .imgSrc("'self'", "https:", "data:")
  .connectSrc("'self'", "https://api.example.com")
  .fontSrc("'self'", "https://fonts.gstatic.com")
  .objectSrc("'none'")
  .build();

const testCases: CSPTestCase[] = [
  {
    name: "Allow self scripts",
    directive: "script-src",
    source: "https://example.com/script.js",
    shouldAllow: true,
  },
  {
    name: "Allow CDN scripts",
    directive: "script-src",
    source: "https://cdn.example.com/library.js",
    shouldAllow: true,
  },
  {
    name: "Block untrusted scripts",
    directive: "script-src",
    source: "https://evil.com/malware.js",
    shouldAllow: false,
  },
  {
    name: "Allow Google Fonts",
    directive: "style-src",
    source: "https://fonts.googleapis.com/css",
    shouldAllow: true,
  },
  {
    name: "Allow HTTPS images",
    directive: "img-src",
    source: "https://cdn.images.com/photo.jpg",
    shouldAllow: true,
  },
  {
    name: "Allow data: images",
    directive: "img-src",
    source: "data:image/png;base64,ABC123",
    shouldAllow: true,
  },
  {
    name: "Block plugins",
    directive: "object-src",
    source: "https://example.com/plugin.swf",
    shouldAllow: false,
  },
];

tester.testPolicy(policy, testCases).then((result) => {
  console.log(tester.generateReport(result));
});
```

### Example 6: Environment-Specific CSP Configuration

```typescript
// Manage CSP for different environments
class CSPConfigManager {
  private configs: Map<Environment, CSPConfig> = new Map();

  constructor() {
    this.initializeConfigs();
  }

  private initializeConfigs(): void {
    // Development configuration
    this.configs.set("development", {
      reportOnly: true,
      directives: {
        "default-src": ["'self'"],
        "script-src": [
          "'self'",
          "'unsafe-eval'", // For dev tools
          "'unsafe-inline'", // For HMR
          "localhost:*",
          "ws://localhost:*",
        ],
        "style-src": ["'self'", "'unsafe-inline'"],
        "img-src": ["'self'", "data:", "blob:", "*"],
        "connect-src": ["'self'", "ws://localhost:*", "localhost:*", "*"],
        "font-src": ["'self'", "data:"],
        "report-uri": ["/csp-dev-report"],
      },
    });

    // Staging configuration
    this.configs.set("staging", {
      reportOnly: true,
      directives: {
        "default-src": ["'self'"],
        "script-src": ["'self'", "https://cdn-staging.example.com"],
        "style-src": ["'self'", "https://fonts.googleapis.com"],
        "img-src": ["'self'", "https:", "data:"],
        "connect-src": [
          "'self'",
          "https://api-staging.example.com",
          "wss://ws-staging.example.com",
        ],
        "font-src": ["'self'", "https://fonts.gstatic.com"],
        "object-src": ["'none'"],
        "frame-ancestors": ["'none'"],
        "base-uri": ["'self'"],
        "form-action": ["'self'"],
        "upgrade-insecure-requests": [],
        "report-uri": ["/csp-staging-report"],
      },
    });

    // Production configuration
    this.configs.set("production", {
      reportOnly: false,
      directives: {
        "default-src": ["'self'"],
        "script-src": ["'self'", "https://cdn.example.com"],
        "style-src": ["'self'", "https://fonts.googleapis.com"],
        "img-src": ["'self'", "https:", "data:"],
        "connect-src": [
          "'self'",
          "https://api.example.com",
          "wss://ws.example.com",
        ],
        "font-src": ["'self'", "https://fonts.gstatic.com"],
        "object-src": ["'none'"],
        "frame-ancestors": ["'none'"],
        "base-uri": ["'self'"],
        "form-action": ["'self'"],
        "upgrade-insecure-requests": [],
        "block-all-mixed-content": [],
        "report-uri": ["/csp-report"],
        "report-to": ["csp-endpoint"],
      },
    });
  }

  // Get configuration for environment
  getConfig(env: Environment): CSPConfig {
    const config = this.configs.get(env);
    if (!config) {
      throw new Error(`No CSP config for environment: ${env}`);
    }
    return config;
  }

  // Build CSP header for environment
  buildHeader(
    env: Environment,
    additionalOptions?: Partial<CSPConfig>,
  ): string {
    const config = this.getConfig(env);
    const merged = this.mergeConfig(config, additionalOptions);

    const builder = new CSPBuilder();

    // Add all directives
    Object.entries(merged.directives).forEach(([directive, sources]) => {
      builder.addSource(directive as CSPDirective, sources);
    });

    return builder.build();
  }

  private mergeConfig(
    base: CSPConfig,
    override?: Partial<CSPConfig>,
  ): CSPConfig {
    if (!override) return base;

    return {
      reportOnly: override.reportOnly ?? base.reportOnly,
      directives: {
        ...base.directives,
        ...override.directives,
      },
    };
  }

  // Get header name based on report-only mode
  getHeaderName(env: Environment): string {
    const config = this.getConfig(env);
    return config.reportOnly
      ? "Content-Security-Policy-Report-Only"
      : "Content-Security-Policy";
  }

  // Middleware factory
  middleware(env: Environment) {
    return (req: any, res: any, next: any) => {
      const headerName = this.getHeaderName(env);
      const headerValue = this.buildHeader(env);

      res.setHeader(headerName, headerValue);
      next();
    };
  }
}

type Environment = "development" | "staging" | "production";

interface CSPConfig {
  reportOnly: boolean;
  directives: Record<string, string[]>;
}

// Usage
const cspConfig = new CSPConfigManager();

// Get appropriate config
const env = (process.env.NODE_ENV as Environment) || "development";
const cspHeader = cspConfig.buildHeader(env);

console.log(`CSP for ${env}:`, cspHeader);

// Use as Express middleware
app.use(cspConfig.middleware(env));
```

### Example 7: CSP for SPA (Single Page Application)

```typescript
// CSP configuration for modern SPA
class SPACSPManager {
  private nonceManager = new CSPNonceManager();

  // Build SPA-friendly CSP
  buildSPAPolicy(nonce: string): string {
    const csp = new CSPBuilder()
      .defaultSrc("'self'")
      .scriptSrc(
        "'self'",
        `'nonce-${nonce}'`,
        "'strict-dynamic'",
        // Fallback for older browsers
        "https:",
        "'unsafe-inline'",
      )
      .styleSrc("'self'", "'unsafe-inline'") // Inline styles common in SPA
      .imgSrc("'self'", "https:", "data:", "blob:")
      .connectSrc("'self'", "https://api.example.com", "wss://ws.example.com")
      .fontSrc("'self'", "https://fonts.gstatic.com", "data:")
      .mediaSrc("'self'", "blob:")
      .workerSrc("'self'", "blob:") // For Web Workers
      .objectSrc("'none'")
      .frameAncestors("'none'")
      .baseUri("'self'")
      .formAction("'self'")
      .upgradeInsecureRequests()
      .reportUri("/csp-report")
      .build();

    return csp;
  }

  // Handle dynamic module imports
  setupModuleImports(): void {
    // Module imports need special handling with CSP
    const nonce = this.nonceManager.getCurrentNonce();

    // Monkey patch import to add nonce
    const originalImport = window.import;
    if (originalImport) {
      (window as any).import = function (specifier: string) {
        const script = document.createElement("script");
        script.type = "module";
        script.src = specifier;
        if (nonce) {
          script.setAttribute("nonce", nonce);
        }
        document.head.appendChild(script);
      };
    }
  }

  // Handle lazy-loaded chunks (Webpack/Vite)
  configureBundler(): BundlerConfig {
    return {
      // Webpack configuration
      webpack: {
        output: {
          crossOriginLoading: "anonymous",
          // Add nonce to dynamically loaded scripts
          chunkLoadingGlobal: "webpackJsonp",
        },
      },
      // Vite configuration
      vite: {
        build: {
          // Enable module preload
          modulePreload: true,
          rollupOptions: {
            output: {
              // Add nonce through plugin
            },
          },
        },
      },
    };
  }
}

interface BundlerConfig {
  webpack: any;
  vite: any;
}

// React-specific CSP helper
class ReactCSPHelper {
  private nonceManager = new CSPNonceManager();

  // Add nonce to React inline styles
  addNonceToInlineStyles(): void {
    const nonce = this.nonceManager.getCurrentNonce();

    // Hook into React's style injection
    if (typeof window !== "undefined" && nonce) {
      const originalCreateElement = document.createElement;

      document.createElement = function (tagName: string, ...args: any[]): any {
        const element = originalCreateElement.apply(document, [
          tagName,
          ...args,
        ]);

        if (tagName.toLowerCase() === "style") {
          element.setAttribute("nonce", nonce);
        }

        return element;
      };
    }
  }

  // Configure for styled-components
  getStyledComponentsConfig(nonce: string): any {
    return {
      // Add nonce to all style tags
      stylisPlugins: [],
      nonce,
    };
  }

  // Configure for emotion
  getEmotionConfig(nonce: string): any {
    return {
      key: "custom",
      nonce,
    };
  }
}

// Vue-specific CSP helper
class VueCSPHelper {
  configureVue(nonce: string): any {
    return {
      // Configure Vue to use nonce
      config: {
        // Add nonce to dynamically created elements
        optionMergeStrategies: {},
      },
      nonce,
    };
  }
}
```

### Example 8: CSP Hash-Based Policies

```typescript
// Use hashes instead of nonces for static content
class CSPHashManager {
  // Generate SHA-256 hash of script/style content
  async generateHash(
    content: string,
    algorithm: "sha256" | "sha384" | "sha512" = "sha256",
  ): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(content);

    const hashBuffer = await crypto.subtle.digest(
      algorithm.toUpperCase(),
      data,
    );
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashBase64 = btoa(String.fromCharCode(...hashArray));

    return `'${algorithm}-${hashBase64}'`;
  }

  // Generate hash for inline script
  async hashScript(script: string): Promise<string> {
    return this.generateHash(script, "sha256");
  }

  // Generate hash for inline style
  async hashStyle(style: string): Promise<string> {
    return this.generateHash(style, "sha256");
  }

  // Build CSP with hashes
  async buildHashedPolicy(
    scripts: string[],
    styles: string[],
  ): Promise<string> {
    const scriptHashes = await Promise.all(
      scripts.map((s) => this.hashScript(s)),
    );

    const styleHashes = await Promise.all(styles.map((s) => this.hashStyle(s)));

    const csp = new CSPBuilder()
      .defaultSrc("'self'")
      .scriptSrc("'self'", ...scriptHashes)
      .styleSrc("'self'", ...styleHashes)
      .imgSrc("'self'", "https:", "data:")
      .connectSrc("'self'", "https://api.example.com")
      .fontSrc("'self'", "https://fonts.gstatic.com")
      .objectSrc("'none'")
      .baseUri("'self'")
      .frameAncestors("'none'")
      .build();

    return csp;
  }

  // Calculate hash of external script
  async hashExternalScript(url: string): Promise<string> {
    try {
      const response = await fetch(url);
      const content = await response.text();
      return this.hashScript(content);
    } catch (error) {
      throw new Error(`Failed to fetch script from ${url}: ${error}`);
    }
  }

  // Generate CSP meta tag with hashes
  async generateMetaTag(scripts: string[], styles: string[]): Promise<string> {
    const policy = await this.buildHashedPolicy(scripts, styles);
    return `<meta http-equiv="Content-Security-Policy" content="${policy}">`;
  }
}

// Build-time hash generator
class BuildTimeHashGenerator {
  private hashManager = new CSPHashManager();
  private hashes: Map<string, string> = new Map();

  // Process HTML file and generate hashes
  async processHTML(html: string): Promise<ProcessedHTML> {
    const scriptHashes: string[] = [];
    const styleHashes: string[] = [];

    // Extract inline scripts
    const scriptRegex = /<script>([\s\S]*?)<\/script>/g;
    let match;

    while ((match = scriptRegex.exec(html)) !== null) {
      const content = match[1].trim();
      if (content) {
        const hash = await this.hashManager.hashScript(content);
        scriptHashes.push(hash);
        this.hashes.set(content, hash);
      }
    }

    // Extract inline styles
    const styleRegex = /<style>([\s\S]*?)<\/style>/g;

    while ((match = styleRegex.exec(html)) !== null) {
      const content = match[1].trim();
      if (content) {
        const hash = await this.hashManager.hashStyle(content);
        styleHashes.push(hash);
        this.hashes.set(content, hash);
      }
    }

    // Build CSP
    const policy = await this.hashManager.buildHashedPolicy(
      Array.from(new Set(scriptHashes)),
      Array.from(new Set(styleHashes)),
    );

    // Inject CSP meta tag
    const metaTag = `<meta http-equiv="Content-Security-Policy" content="${policy}">`;
    const processedHTML = html.replace("</head>", `${metaTag}\n</head>`);

    return {
      html: processedHTML,
      policy,
      scriptHashes,
      styleHashes,
    };
  }

  // Get hash for content
  getHash(content: string): string | undefined {
    return this.hashes.get(content);
  }

  // Export hashes
  exportHashes(): Record<string, string> {
    return Object.fromEntries(this.hashes);
  }
}

interface ProcessedHTML {
  html: string;
  policy: string;
  scriptHashes: string[];
  styleHashes: string[];
}

// Usage in build tool
async function processBuildFiles(): Promise<void> {
  const generator = new BuildTimeHashGenerator();

  const html = `
<!DOCTYPE html>
<html>
<head>
    <title>Hash-based CSP</title>
    <style>
        body { font-family: Arial; }
    </style>
</head>
<body>
    <h1>Hello</h1>
    <script>
        console.log('This script will be hashed');
    </script>
</body>
</html>
  `;

  const result = await generator.processHTML(html);

  console.log("Processed HTML with CSP:");
  console.log(result.html);
  console.log("\nCSP Policy:");
  console.log(result.policy);
}
```

## Real-World Usage

### E-Commerce Site CSP

```typescript
// Complete CSP setup for e-commerce
class ECommerceCSP {
  private nonceManager = new CSPNonceManager();
  private configManager = new CSPConfigManager();

  buildPolicy(nonce: string): string {
    return new CSPBuilder()
      .defaultSrc("'self'")
      .scriptSrc(
        "'self'",
        `'nonce-${nonce}'`,
        "'strict-dynamic'",
        "https://checkout.stripe.com",
        "https://js.stripe.com",
        "https://www.google-analytics.com",
        "https://www.googletagmanager.com",
        // Fallbacks
        "https:",
        "'unsafe-inline'",
      )
      .styleSrc("'self'", "'unsafe-inline'", "https://fonts.googleapis.com")
      .imgSrc("'self'", "https:", "data:", "https://www.google-analytics.com")
      .connectSrc(
        "'self'",
        "https://api.example.com",
        "https://api.stripe.com",
        "https://www.google-analytics.com",
      )
      .fontSrc("'self'", "https://fonts.gstatic.com")
      .frameSrc(
        "'self'",
        "https://checkout.stripe.com",
        "https://js.stripe.com",
      )
      .objectSrc("'none'")
      .baseUri("'self'")
      .frameAncestors("'none'")
      .formAction("'self'", "https://checkout.stripe.com")
      .upgradeInsecureRequests()
      .reportUri("/csp-violations")
      .build();
  }

  middleware() {
    return (req: any, res: any, next: any) => {
      const nonce = this.nonceManager.generateNonce();
      const policy = this.buildPolicy(nonce);

      res.setHeader("Content-Security-Policy", policy);
      res.locals.cspNonce = nonce;

      next();
    };
  }
}
```

## Production Patterns

### 1. **Start with Report-Only**

Deploy policies in report-only mode first to identify issues

### 2. **Use Nonces for Dynamic Content**

Generate unique nonces per request for inline scripts/styles

### 3. **Implement strict-dynamic**

Modern CSP that simplifies policy management

### 4. **Monitor Violations**

Track CSP violations to identify attacks and misconfigurations

### 5. **Environment-Specific Policies**

Different policies for dev, staging, production

## Best Practices

1. **Start with strict policy** and relax as needed
2. **Use nonces instead of 'unsafe-inline'**
3. **Implement strict-dynamic** for modern browsers
4. **Test with Content-Security-Policy-Report-Only** first
5. **Monitor violations** to detect attacks
6. **Block object-src** to prevent Flash/plugin exploits
7. **Set frame-ancestors** to prevent clickjacking
8. **Use upgrade-insecure-requests** to enforce HTTPS
9. **Avoid 'unsafe-eval'** except in development
10. **Document policy decisions** for team
11. **Version control CSP policies**
12. **Automate CSP testing** in CI/CD pipeline
13. **Use CSP hashes** for static inline content
14. **Implement report endpoints** for violations
15. **Regularly audit and update** policies

## 10 Key Takeaways

1. **CSP is the most effective defense against XSS** - properly configured CSP blocks execution of injected malicious scripts
2. **Nonce-based CSP is superior to unsafe-inline** - cryptographically secure random nonces allow inline scripts while blocking injected code
3. **strict-dynamic simplifies modern CSP** - eliminates need for whitelisting hosts, trusts dynamically loaded scripts from trusted sources
4. **Report-only mode is essential for testing** - deploy policies in report-only first to identify issues without breaking functionality
5. **CSP violations indicate attacks or misconfigurations** - monitor violation reports to detect security threats and policy issues
6. **object-src 'none' prevents plugin exploits** - blocks Flash and other plugins that historically had many vulnerabilities
7. **frame-ancestors prevents clickjacking** - controls which sites can iframe your content, preventing UI redressing attacks
8. **upgrade-insecure-requests enforces HTTPS** - automatically upgrades HTTP requests to HTTPS on HTTPS pages
9. **CSP is defense in depth, not silver bullet** - combine with input validation, output encoding, and other security measures
10. **Environment-specific policies reduce friction** - relaxed policies in development, strict in production balances security and developer experience
