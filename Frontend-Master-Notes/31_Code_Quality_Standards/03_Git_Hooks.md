# Git Hooks and Automation

## Table of Contents

- [Introduction](#introduction)
- [Git Hooks Overview](#git-hooks-overview)
- [Husky Setup](#husky-setup)
- [Lint-Staged Configuration](#lint-staged-configuration)
- [Commitlint Setup](#commitlint-setup)
- [Advanced Hook Patterns](#advanced-hook-patterns)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Key Takeaways](#key-takeaways)

## Introduction

Git hooks are scripts that Git executes before or after events such as commit, push, and merge. They provide a powerful way to automate quality checks, enforce standards, and prevent bad code from entering your repository.

### Why Git Hooks Matter

```
Without Hooks:
Developer → Commit → Push → CI Fails → Fix → Commit → Push
                             ↑
                    Wasted time and CI resources

With Hooks:
Developer → Pre-commit Check → Fast Feedback → Fix → Commit → Push → CI Passes
                    ↑
            Catches issues locally
```

### Benefits

- **Immediate Feedback**: Catch issues before they reach CI/CD
- **Reduced CI Costs**: Fewer failed builds
- **Consistent Quality**: Enforce standards automatically
- **Better Commits**: Well-formatted, properly-named commits
- **Team Alignment**: Standardized workflows

## Git Hooks Overview

### Available Hooks

```bash
# Client-side hooks (run on developer machine)
.git/hooks/
├── pre-commit          # Before commit is created
├── prepare-commit-msg  # Before commit message editor
├── commit-msg          # After commit message written
├── post-commit         # After commit is created
├── pre-push            # Before push to remote
├── pre-rebase          # Before rebase
└── post-checkout       # After checkout/switch

# Server-side hooks (run on Git server)
├── pre-receive         # Before refs are updated
├── update              # During ref update
└── post-receive        # After refs are updated
```

### Hook Workflow

```
┌─────────────────────────────────────────────────┐
│  1. Developer runs: git commit                  │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│  2. pre-commit hook runs:                       │
│     - Lint staged files                         │
│     - Run tests                                 │
│     - Format code                               │
└────────────────┬────────────────────────────────┘
                 │
                 ├─ Success ──────────────────────┐
                 │                                 ▼
                 │              ┌──────────────────────────────┐
                 │              │  3. Commit message editor    │
                 │              └─────────┬────────────────────┘
                 │                        │
                 │                        ▼
                 │              ┌──────────────────────────────┐
                 │              │  4. commit-msg hook:         │
                 │              │     - Validate format        │
                 │              │     - Check conventions      │
                 │              └─────────┬────────────────────┘
                 │                        │
                 │                        ├─ Success ──────────┐
                 │                        │                     ▼
                 │                        │         ┌──────────────────┐
                 │                        │         │  5. Commit saved │
                 │                        │         └──────────────────┘
                 │                        │
                 └─ Failure ───────────────┴─ Failure
                           │                        │
                           ▼                        ▼
                  ┌──────────────────────────────────────┐
                  │  Commit aborted - fix issues         │
                  └──────────────────────────────────────┘
```

## Husky Setup

### Installation

```bash
# Install Husky
npm install --save-dev husky

# Initialize Husky (creates .husky/ directory)
npx husky-init
npm install

# Or manually
npx husky install
npm pkg set scripts.prepare="husky install"
```

### Project Structure

```
my-project/
├── .husky/
│   ├── _/                    # Husky internal files
│   │   └── husky.sh
│   ├── pre-commit            # Pre-commit hook
│   ├── commit-msg            # Commit message hook
│   ├── pre-push              # Pre-push hook
│   └── prepare-commit-msg    # Prepare commit message hook
├── .git/
├── package.json
└── src/
```

### Basic Hooks

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."
npm run lint-staged
```

```bash
# .husky/commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "📝 Validating commit message..."
npx --no -- commitlint --edit "$1"
```

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🚀 Running pre-push checks..."
npm run test:coverage
npm run build
```

### Creating Hooks Programmatically

```bash
# Add pre-commit hook
npx husky add .husky/pre-commit "npm run lint-staged"

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'

# Add pre-push hook
npx husky add .husky/pre-push "npm test"
```

### Package.json Configuration

```json
{
  "name": "my-project",
  "scripts": {
    "prepare": "husky install",
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\"",
    "format:check": "prettier --check \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\"",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "build": "tsc && vite build"
  },
  "devDependencies": {
    "husky": "^8.0.3",
    "lint-staged": "^15.2.0",
    "@commitlint/cli": "^18.4.3",
    "@commitlint/config-conventional": "^18.4.3"
  }
}
```

## Lint-Staged Configuration

### What is Lint-Staged?

Lint-staged runs linters and formatters only on files that are staged for commit, making pre-commit hooks fast and efficient.

```
Without Lint-Staged:
npm run lint → Checks ALL files (slow, unrelated failures)

With Lint-Staged:
lint-staged → Checks ONLY staged files (fast, relevant)
```

### Installation

```bash
npm install --save-dev lint-staged
```

### Configuration in package.json

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,css,scss,md}": ["prettier --write"],
    "*.{png,jpg,jpeg,gif,svg}": ["imagemin-lint-staged"]
  }
}
```

### Separate Configuration File

```javascript
// .lintstagedrc.js
module.exports = {
  // JavaScript/TypeScript files
  "*.{js,jsx,ts,tsx}": [
    "eslint --fix",
    "prettier --write",
    "git add", // Not needed in lint-staged v10+
  ],

  // Style files
  "*.{css,scss}": ["stylelint --fix", "prettier --write"],

  // JSON files
  "*.json": ["prettier --write"],

  // Markdown files
  "*.md": ["markdownlint --fix", "prettier --write"],

  // Run tests for related files
  "*.{js,jsx,ts,tsx}": [() => "npm run test:related"],
};
```

### Advanced Patterns

```javascript
// .lintstagedrc.js
const path = require("path");

module.exports = {
  // Type check TypeScript files
  "*.{ts,tsx}": () => "tsc --noEmit",

  // Run ESLint with type information
  "*.{ts,tsx}": (filenames) => {
    const files = filenames.join(" ");
    return [`eslint --fix ${files}`, `prettier --write ${files}`];
  },

  // Run tests for changed files and their dependencies
  "*.{js,jsx,ts,tsx}": (filenames) => {
    const testFiles = filenames
      .map((file) => file.replace(/\.(js|jsx|ts|tsx)$/, ".test.$1"))
      .filter((file) => require("fs").existsSync(file))
      .join(" ");

    return testFiles ? `jest ${testFiles}` : [];
  },

  // Check bundle size impact
  "src/**/*.{js,jsx,ts,tsx}": () => "npm run bundle:check",

  // Update snapshots for test files
  "*.test.{js,jsx,ts,tsx}": ["jest --updateSnapshot --bail --findRelatedTests"],

  // Optimize images
  "*.{png,jpg,jpeg,gif}": ["imagemin --out-dir=optimized"],

  // Generate documentation
  "src/**/*.{js,jsx,ts,tsx}": () => "npm run docs:generate",
};
```

### Concurrent Execution

```javascript
// .lintstagedrc.js
module.exports = {
  "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],

  // Run tasks in parallel
  "*.ts?(x)": [
    () => "tsc --noEmit", // Type check
    () => "npm run test:unit", // Unit tests
  ].map((cmd) => ({ command: cmd, concurrent: true })),
};
```

### Ignore Patterns

```javascript
// .lintstagedrc.js
module.exports = {
  "*.{js,jsx,ts,tsx}": (filenames) => {
    // Filter out generated files
    const filteredFiles = filenames.filter(
      (file) => !file.includes("generated") && !file.includes(".min."),
    );

    if (filteredFiles.length === 0) {
      return [];
    }

    return [
      `eslint --fix ${filteredFiles.join(" ")}`,
      `prettier --write ${filteredFiles.join(" ")}`,
    ];
  },
};
```

## Commitlint Setup

### What is Commitlint?

Commitlint checks if your commit messages meet the conventional commit format, ensuring consistency and enabling automated changelog generation.

### Installation

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

### Configuration

```javascript
// commitlint.config.js
module.exports = {
  extends: ["@commitlint/config-conventional"],

  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat", // New feature
        "fix", // Bug fix
        "docs", // Documentation changes
        "style", // Formatting, missing semi colons, etc.
        "refactor", // Code refactoring
        "perf", // Performance improvements
        "test", // Adding or updating tests
        "chore", // Maintenance tasks
        "ci", // CI configuration changes
        "build", // Build system changes
        "revert", // Revert previous commit
      ],
    ],
    "type-case": [2, "always", "lower-case"],
    "type-empty": [2, "never"],
    "scope-case": [2, "always", "lower-case"],
    "subject-empty": [2, "never"],
    "subject-full-stop": [2, "never", "."],
    "subject-case": [2, "always", "lower-case"],
    "header-max-length": [2, "always", 100],
    "body-leading-blank": [1, "always"],
    "body-max-line-length": [2, "always", 100],
    "footer-leading-blank": [1, "always"],
    "footer-max-line-length": [2, "always", 100],
  },
};
```

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Examples

```bash
# ✅ Valid commits
feat(auth): add login functionality
fix(api): resolve timeout issue
docs(readme): update installation instructions
style: format code with prettier
refactor(utils): simplify date formatting
perf(parser): improve parsing speed by 30%
test(auth): add unit tests for login
chore(deps): update dependencies
ci(github): add workflow for deployment
build(webpack): optimize bundle size

# ❌ Invalid commits
Add login feature                    # Missing type
feat: Add Login Feature              # Subject not lowercase
feat(Auth): add login                # Scope not lowercase
fix: resolve issue.                  # Subject ends with period
feature: new component               # Wrong type
```

### Advanced Configuration

```javascript
// commitlint.config.js
module.exports = {
  extends: ["@commitlint/config-conventional"],

  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "chore",
        "ci",
        "build",
        "revert",
      ],
    ],

    // Custom scopes for monorepo
    "scope-enum": [
      2,
      "always",
      ["web", "mobile", "api", "shared", "docs", "config"],
    ],

    // Require scope for certain types
    "scope-empty": [2, "never"],

    // Subject must match pattern
    "subject-pattern": [2, "always", /^[a-z][a-z0-9\s-]*$/],

    // Enforce issue reference in footer
    "footer-match-pattern": [2, "always", /^(Closes|Fixes|Refs):\s#\d+$/],
  },

  // Custom plugin
  plugins: [
    {
      rules: {
        "ticket-number-required": (parsed) => {
          const { subject, footer } = parsed;
          const ticketPattern = /\[JIRA-\d+\]/;

          if (!ticketPattern.test(subject) && !ticketPattern.test(footer)) {
            return [false, "Commit must reference a JIRA ticket: [JIRA-123]"];
          }

          return [true];
        },
      },
    },
  ],

  rules: {
    "ticket-number-required": [2, "always"],
  },
};
```

### Prompt for Commit Messages

```bash
# Install commitizen
npm install --save-dev commitizen cz-conventional-changelog

# Initialize commitizen
npx commitizen init cz-conventional-changelog --save-dev --save-exact
```

```json
// package.json
{
  "scripts": {
    "commit": "cz"
  },
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  }
}
```

```bash
# Instead of: git commit -m "message"
# Use: npm run commit
# Interactive prompt guides you through format
```

## Advanced Hook Patterns

### Conditional Execution

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Only run on certain branches
BRANCH=$(git symbolic-ref --short HEAD)
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "develop" ]; then
  echo "🔍 Running thorough checks on $BRANCH..."
  npm run test:all
  npm run lint
  npm run type-check
else
  echo "🔍 Running quick checks on feature branch..."
  npm run lint-staged
fi
```

### Skip Hooks When Needed

```bash
# Skip pre-commit hook
git commit --no-verify -m "emergency fix"

# Skip all hooks
git commit -n -m "message"

# Environment variable approach
SKIP_HOOKS=1 git commit -m "message"
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Allow skipping via environment variable
if [ "$SKIP_HOOKS" = "1" ]; then
  echo "⚠️  Skipping pre-commit hooks (SKIP_HOOKS=1)"
  exit 0
fi

npm run lint-staged
```

### Complex Pre-commit

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Pre-commit checks starting..."

# 1. Lint and format staged files
echo "1️⃣ Linting and formatting..."
npm run lint-staged || {
  echo "❌ Linting failed"
  exit 1
}

# 2. Type check TypeScript
echo "2️⃣ Type checking..."
npm run type-check || {
  echo "❌ Type check failed"
  exit 1
}

# 3. Run unit tests for changed files
echo "3️⃣ Running tests..."
npm run test:changed || {
  echo "❌ Tests failed"
  exit 1
}

# 4. Check for console.logs
echo "4️⃣ Checking for debug statements..."
git diff --cached --name-only | grep -E '\.(js|jsx|ts|tsx)$' | xargs grep -n 'console\.log' && {
  echo "❌ Found console.log statements"
  exit 1
}

# 5. Check for TODOs without ticket numbers
echo "5️⃣ Checking TODOs..."
git diff --cached --name-only | xargs grep -n 'TODO' | grep -v '\[JIRA-[0-9]\+\]' && {
  echo "❌ Found TODO without ticket number"
  exit 1
}

echo "✅ All pre-commit checks passed!"
```

### Pre-push Validation

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🚀 Pre-push checks starting..."

# 1. Run full test suite
echo "1️⃣ Running full test suite..."
npm run test:coverage || {
  echo "❌ Tests failed"
  exit 1
}

# 2. Build production bundle
echo "2️⃣ Building production bundle..."
npm run build || {
  echo "❌ Build failed"
  exit 1
}

# 3. Check bundle size
echo "3️⃣ Checking bundle size..."
npm run bundle:check || {
  echo "❌ Bundle size exceeds limit"
  exit 1
}

# 4. Security audit
echo "4️⃣ Running security audit..."
npm audit --audit-level=moderate || {
  echo "❌ Security vulnerabilities found"
  exit 1
}

echo "✅ All pre-push checks passed!"
```

### Prepare Commit Message

```bash
# .husky/prepare-commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# Only modify message if not from merge/squash
if [ -z "$COMMIT_SOURCE" ]; then
  # Get current branch name
  BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)

  # Extract ticket number from branch (e.g., feature/JIRA-123-description)
  TICKET=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+')

  if [ -n "$TICKET" ]; then
    # Prepend ticket number to commit message
    CURRENT_MSG=$(cat "$COMMIT_MSG_FILE")
    echo "[$TICKET] $CURRENT_MSG" > "$COMMIT_MSG_FILE"
  fi
fi
```

### Post-checkout Hook

```bash
# .husky/post-checkout
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Check if dependencies changed
PREV_REF=$1
NEW_REF=$2
BRANCH_CHECKOUT=$3

# Only run on branch checkout
if [ "$BRANCH_CHECKOUT" = "1" ]; then
  # Check if package.json changed
  if git diff --name-only "$PREV_REF" "$NEW_REF" | grep -q "package.json"; then
    echo "📦 package.json changed, running npm install..."
    npm install
  fi
fi
```

## Performance Optimization

### Fast Pre-commit

```javascript
// .lintstagedrc.js
module.exports = {
  // Run in parallel for speed
  "*.{js,jsx,ts,tsx}": ["eslint --fix --max-warnings=0", "prettier --write"],

  // Skip expensive operations
  "*.{js,jsx,ts,tsx}": (filenames) => {
    // Only type-check if more than 5 files changed
    if (filenames.length > 5) {
      return "tsc --noEmit";
    }
    return [];
  },
};
```

### Caching Strategy

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Enable ESLint cache
ESLINT_CACHE=1 npm run lint-staged

# Enable Prettier cache
npm run format -- --cache
```

### Selective Tests

```javascript
// scripts/test-changed.js
const { execSync } = require("child_process");

// Get changed files
const changedFiles = execSync(
  "git diff --cached --name-only --diff-filter=ACMR",
)
  .toString()
  .trim()
  .split("\n")
  .filter((file) => file.endsWith(".ts") || file.endsWith(".tsx"))
  .filter((file) => !file.includes(".test."));

if (changedFiles.length === 0) {
  console.log("No files to test");
  process.exit(0);
}

// Find related test files
const testFiles = changedFiles
  .map((file) => file.replace(/\.(ts|tsx)$/, ".test.$1"))
  .filter((file) => {
    try {
      require("fs").statSync(file);
      return true;
    } catch {
      return false;
    }
  });

if (testFiles.length > 0) {
  console.log(`Running tests for ${testFiles.length} files...`);
  execSync(`jest ${testFiles.join(" ")}`, { stdio: "inherit" });
}
```

### Debounced Hooks

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Skip if commit was recent (within 5 seconds)
LAST_COMMIT_TIME=$(git log -1 --format=%ct 2>/dev/null || echo 0)
CURRENT_TIME=$(date +%s)
TIME_DIFF=$((CURRENT_TIME - LAST_COMMIT_TIME))

if [ "$TIME_DIFF" -lt 5 ]; then
  echo "⚠️  Skipping hooks (recent commit)"
  exit 0
fi

npm run lint-staged
```

## Troubleshooting

### Common Issues

```bash
# Issue 1: Hooks not executing
# Solution: Ensure hooks are executable
chmod +x .husky/pre-commit
chmod +x .husky/commit-msg

# Issue 2: npx not found
# Solution: Use full path
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

/usr/local/bin/npx lint-staged

# Issue 3: Wrong Node version
# Solution: Use .nvmrc
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

if [ -f ".nvmrc" ]; then
  nvm use
fi

npm run lint-staged

# Issue 4: Hooks run in CI
# Solution: Skip in CI environment
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

if [ "$CI" = "true" ]; then
  echo "Skipping hooks in CI"
  exit 0
fi

npm run lint-staged
```

### Debugging Hooks

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Enable debug mode
set -x

echo "Current directory: $(pwd)"
echo "Node version: $(node --version)"
echo "npm version: $(npm --version)"
echo "Git status:"
git status

npm run lint-staged

# Disable debug mode
set +x
```

### Hook Execution Log

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

LOG_FILE=".husky/hooks.log"

{
  echo "================================"
  echo "Pre-commit hook executed"
  echo "Date: $(date)"
  echo "Branch: $(git branch --show-current)"
  echo "User: $(git config user.name)"
  echo "================================"

  npm run lint-staged 2>&1

  echo "Exit code: $?"
  echo ""
} >> "$LOG_FILE"
```

## Best Practices

### 1. Start Simple

```bash
# .husky/pre-commit - Start here
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint-staged
```

```javascript
// .lintstagedrc.js - Minimal config
module.exports = {
  "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
};
```

### 2. Provide Escape Hatches

```bash
# Allow skipping when necessary
git commit --no-verify -m "emergency fix"

# Document in README.md
# To skip pre-commit hooks in emergencies:
# git commit --no-verify -m "your message"
```

### 3. Fast Feedback

```javascript
// Optimize for speed
module.exports = {
  // Only lint changed files
  "*.{js,jsx,ts,tsx}": ["eslint --fix"],

  // Skip expensive type-checking
  // Run in CI instead
};
```

### 4. Clear Error Messages

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint-staged || {
  echo ""
  echo "❌ Pre-commit checks failed!"
  echo ""
  echo "To fix:"
  echo "  1. Run: npm run lint:fix"
  echo "  2. Review the changes"
  echo "  3. Stage the fixed files: git add ."
  echo "  4. Try committing again"
  echo ""
  echo "To skip (emergencies only):"
  echo "  git commit --no-verify"
  echo ""
  exit 1
}
```

### 5. Team Documentation

````markdown
# Git Hooks

This project uses git hooks to maintain code quality.

## Pre-commit

- Runs ESLint and Prettier on staged files
- Runs tests for changed files
- Usually takes 5-10 seconds

## Commit-msg

- Validates commit message format
- Enforces conventional commits

## Pre-push

- Runs full test suite
- Creates production build
- Checks bundle size
- Usually takes 1-2 minutes

## Skipping Hooks

Only skip hooks in emergencies:

```bash
git commit --no-verify
```
````

## Key Takeaways

1. **Automated Quality**: Git hooks automate code quality checks, catching issues before they reach CI/CD. This saves time and reduces failed builds.

2. **Husky for Management**: Husky simplifies git hook management by storing hooks in version control. All team members automatically get the same hooks.

3. **Lint-Staged for Speed**: Run linters only on staged files using lint-staged. This makes pre-commit hooks fast, even in large codebases.

4. **Commitlint for Consistency**: Enforce conventional commit format with commitlint. This enables automated changelog generation and better git history.

5. **Progressive Enhancement**: Start with simple hooks and add complexity gradually. Don't overwhelm developers with slow, comprehensive checks initially.

6. **Escape Hatches**: Provide ways to skip hooks when necessary (--no-verify). Document when and why this should be used.

7. **Fast Feedback**: Optimize hooks for speed. Move expensive checks (full test suite, type-checking) to pre-push or CI.

8. **Clear Communication**: Provide helpful error messages when hooks fail. Tell developers exactly how to fix issues.

9. **Environment Awareness**: Skip hooks in CI environments. Hooks are for local development, not CI/CD pipelines.

10. **Team Alignment**: Document your git hooks in README. Ensure all team members understand what runs, when it runs, and why it matters. Consider using commitizen for interactive commit message prompts.
