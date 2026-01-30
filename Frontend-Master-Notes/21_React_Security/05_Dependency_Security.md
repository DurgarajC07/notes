# Dependency Security

## Core Concept

Third-party dependencies introduce security vulnerabilities into your application. Regular audits, updates, and monitoring are essential to prevent supply chain attacks, malicious packages, and known CVEs.

---

## npm audit

```bash
# Check for vulnerabilities
npm audit

# Output example:
# found 3 vulnerabilities (1 moderate, 2 high) in 1200 scanned packages

# Get detailed report
npm audit --json > audit-report.json

# Fix automatically (updates package-lock.json)
npm audit fix

# Fix including breaking changes
npm audit fix --force

# Production-only audit
npm audit --production
```

---

## Continuous Monitoring

```typescript
// package.json scripts
{
  "scripts": {
    "audit": "npm audit --audit-level=moderate",
    "audit:fix": "npm audit fix",
    "preinstall": "npm audit --audit-level=high"
  }
}

// CI/CD integration (.github/workflows/security.yml)
name: Security Audit

on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run npm audit
        run: npm audit --audit-level=high
```

---

## Snyk Integration

```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Test for vulnerabilities
snyk test

# Test and monitor
snyk monitor

# Test Docker images
snyk container test myapp:latest

# Fix vulnerabilities
snyk wizard
```

---

## Snyk in CI/CD

```yaml
# .github/workflows/snyk.yml
name: Snyk Security

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

---

## Dependency Review

```typescript
// Check package before installation
// .npmrc configuration
audit-level=moderate
fund=false
save-exact=true

// Lock versions exactly (no ^ or ~)
{
  "dependencies": {
    "react": "18.2.0",  // Exact version
    "lodash": "^4.17.21" // ❌ Allows updates
  }
}

// Better approach
{
  "dependencies": {
    "react": "18.2.0",
    "lodash": "4.17.21"  // ✅ Locked version
  }
}
```

---

## Package Integrity Verification

```bash
# Verify package integrity
npm install --ignore-scripts

# Check package lock integrity
npm ci --ignore-scripts

# Use package-lock.json in version control
git add package-lock.json
```

---

## Allowlist Known Good Packages

```typescript
// allowed-packages.json
{
  "allowed": [
    "react@18.2.0",
    "react-dom@18.2.0",
    "typescript@5.0.0"
  ]
}

// validate-deps.js
const fs = require('fs');
const packageJson = require('./package.json');
const allowedPackages = require('./allowed-packages.json');

const dependencies = {
  ...packageJson.dependencies,
  ...packageJson.devDependencies
};

const unauthorized = [];

for (const [name, version] of Object.entries(dependencies)) {
  const key = `${name}@${version}`;
  if (!allowedPackages.allowed.includes(key)) {
    unauthorized.push(key);
  }
}

if (unauthorized.length > 0) {
  console.error('Unauthorized packages:', unauthorized);
  process.exit(1);
}
```

---

## Scan for Malicious Code

```typescript
// check-malicious.js
const fs = require("fs");
const path = require("path");

// Suspicious patterns
const suspiciousPatterns = [
  /eval\(/gi,
  /Function\(/gi,
  /process\.env/gi,
  /child_process/gi,
  /fs\.readFileSync/gi,
  /XMLHttpRequest/gi,
  /btoa\(/gi,
  /atob\(/gi,
];

function scanFile(filePath) {
  const content = fs.readFileSync(filePath, "utf-8");
  const findings = [];

  suspiciousPatterns.forEach((pattern) => {
    if (pattern.test(content)) {
      findings.push({
        file: filePath,
        pattern: pattern.toString(),
        match: content.match(pattern)?.[0],
      });
    }
  });

  return findings;
}

function scanDirectory(dir) {
  const files = fs.readdirSync(dir);
  let allFindings = [];

  files.forEach((file) => {
    const fullPath = path.join(dir, file);
    const stat = fs.statSync(fullPath);

    if (stat.isDirectory() && !file.startsWith(".")) {
      allFindings = [...allFindings, ...scanDirectory(fullPath)];
    } else if (file.endsWith(".js")) {
      allFindings = [...allFindings, ...scanFile(fullPath)];
    }
  });

  return allFindings;
}

// Scan node_modules
const findings = scanDirectory("./node_modules");
if (findings.length > 0) {
  console.warn("Suspicious code found:", findings);
}
```

---

## License Compliance

```bash
# Install license checker
npm install -g license-checker

# Check licenses
license-checker --summary

# Exclude specific licenses
license-checker --exclude "MIT, Apache-2.0"

# Generate report
license-checker --json > licenses.json
```

---

## Automated Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    commit-message:
      prefix: "chore"
      include: "scope"
```

---

## Renovate Bot Configuration

```json
// renovate.json
{
  "extends": ["config:base"],
  "schedule": ["before 4am on Monday"],
  "labels": ["dependencies"],
  "assignees": ["security-team"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security", "urgent"],
    "assignees": ["security-lead"]
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "automerge": false
    },
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["test", "lint"]
    }
  ]
}
```

---

## Runtime Dependency Monitoring

```typescript
// Monitor for unexpected dependencies at runtime
class DependencyMonitor {
  private allowedModules = new Set(["react", "react-dom", "lodash"]);

  checkImports() {
    if (typeof require !== "undefined") {
      const originalRequire = require;

      (global as any).require = (id: string) => {
        const baseName = id.split("/")[0];

        if (!this.allowedModules.has(baseName)) {
          console.warn(`Unexpected module loaded: ${id}`);
          // Send to monitoring service
        }

        return originalRequire(id);
      };
    }
  }
}

// Initialize in app entry
if (process.env.NODE_ENV === "production") {
  new DependencyMonitor().checkImports();
}
```

---

## Subresource Integrity for CDN

```typescript
// Generate SRI hashes for CDN dependencies
import crypto from "crypto";
import https from "https";

async function generateSRI(url: string): Promise<string> {
  return new Promise((resolve, reject) => {
    https.get(url, (response) => {
      const hash = crypto.createHash("sha384");

      response.on("data", (chunk) => {
        hash.update(chunk);
      });

      response.on("end", () => {
        const integrity = `sha384-${hash.digest("base64")}`;
        resolve(integrity);
      });

      response.on("error", reject);
    });
  });
}

// Usage
const integrity = await generateSRI("https://cdn.example.com/library.js");
console.log(`<script src="..." integrity="${integrity}"></script>`);
```

---

## Security Headers for Dependencies

```typescript
// Next.js config
module.exports = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "Content-Security-Policy",
            value: [
              "default-src 'self'",
              "script-src 'self' https://trusted-cdn.com",
              "style-src 'self' 'unsafe-inline' https://trusted-cdn.com",
              "require-sri-for script style",
            ].join("; "),
          },
        ],
      },
    ];
  },
};
```

---

## Real-World: Dependency Security Pipeline

```typescript
// scripts/security-check.js
const { execSync } = require("child_process");
const fs = require("fs");

class SecurityPipeline {
  async run() {
    console.log("🔒 Starting security checks...\n");

    try {
      await this.auditDependencies();
      await this.checkLicenses();
      await this.scanMaliciousCode();
      await this.verifyIntegrity();

      console.log("\n✅ All security checks passed!");
      process.exit(0);
    } catch (error) {
      console.error("\n❌ Security checks failed:", error);
      process.exit(1);
    }
  }

  async auditDependencies() {
    console.log("📦 Auditing dependencies...");

    try {
      execSync("npm audit --audit-level=high", { stdio: "inherit" });
    } catch (error) {
      throw new Error("npm audit failed");
    }
  }

  async checkLicenses() {
    console.log("📜 Checking licenses...");

    const output = execSync("npx license-checker --json").toString();
    const licenses = JSON.parse(output);

    const disallowed = ["GPL", "AGPL", "LGPL"];
    const violations = [];

    for (const [pkg, info] of Object.entries(licenses)) {
      const license = (info as any).licenses;
      if (disallowed.some((d) => license.includes(d))) {
        violations.push({ pkg, license });
      }
    }

    if (violations.length > 0) {
      console.error("License violations:", violations);
      throw new Error("License check failed");
    }
  }

  async scanMaliciousCode() {
    console.log("🔍 Scanning for malicious code...");

    // Run custom malicious code scanner
    // Implementation similar to check-malicious.js above
  }

  async verifyIntegrity() {
    console.log("🔐 Verifying package integrity...");

    try {
      execSync("npm ci --ignore-scripts", { stdio: "inherit" });
    } catch (error) {
      throw new Error("Integrity verification failed");
    }
  }
}

// Run pipeline
new SecurityPipeline().run();
```

---

## Best Practices

✅ **Run `npm audit` regularly**  
✅ **Use Snyk or Dependabot** for automation  
✅ **Lock dependency versions** with exact versions  
✅ **Review dependency updates** before merging  
✅ **Scan for malicious code** in node_modules  
✅ **Check licenses** for compliance  
✅ **Use SRI for CDN resources**  
❌ **Don't ignore security warnings**  
❌ **Don't install random packages** without research  
❌ **Don't skip dependency audits** in CI/CD

---

## Key Takeaways

1. **Third-party dependencies are attack vectors**
2. **Regular audits prevent vulnerabilities**
3. **Automated tools catch issues early**
4. **Lock versions to prevent supply chain attacks**
5. **Monitor runtime imports** in production
6. **License compliance is critical**
7. **Security is a continuous process**
