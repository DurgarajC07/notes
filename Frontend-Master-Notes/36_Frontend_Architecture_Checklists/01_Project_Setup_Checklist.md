# Project Setup Checklist

## Table of Contents

- [Introduction](#introduction)
- [Repository Setup](#repository-setup)
- [Development Environment](#development-environment)
- [Build Configuration](#build-configuration)
- [Code Quality Tools](#code-quality-tools)
- [Testing Infrastructure](#testing-infrastructure)
- [CI/CD Pipeline](#cicd-pipeline)
- [Documentation](#documentation)
- [Dependencies Management](#dependencies-management)
- [Environment Configuration](#environment-configuration)
- [Security Setup](#security-setup)
- [Automation Scripts](#automation-scripts)
- [Key Takeaways](#key-takeaways)

## Introduction

A solid project setup is the foundation for maintainable, scalable frontend applications. This checklist ensures consistent configuration across projects and teams.

### Why Project Setup Matters

```typescript
interface ProjectSetupGoals {
  consistency: "Standardize configuration across projects";
  quality: "Enforce code quality from day one";
  productivity: "Reduce setup time for new developers";
  reliability: "Catch issues early in development";
  scalability: "Support growth without refactoring";
}

const setupPhilosophy = {
  automation: "Automate everything that can be automated",
  standards: "Establish conventions before writing code",
  documentation: "Document decisions and setup process",
  flexibility: "Allow customization while maintaining standards",
};
```

## Repository Setup

### ✅ Version Control Configuration

```bash
# .gitignore - Essential entries
node_modules/
.env.local
.env.development.local
.env.test.local
.env.production.local
dist/
build/
.DS_Store
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.vscode/*
!.vscode/settings.json
!.vscode/extensions.json
.idea/
*.swp
*.swo
coverage/
.nyc_output/
```

### ✅ Git Hooks Setup

```json
// package.json - Husky configuration
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm test",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{ts,tsx,js,jsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

### ✅ Commit Convention

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
        "docs", // Documentation only
        "style", // Formatting, missing semicolons, etc
        "refactor", // Code change that neither fixes nor adds feature
        "perf", // Performance improvement
        "test", // Adding missing tests
        "chore", // Maintenance
        "revert", // Revert previous commit
        "ci", // CI configuration
      ],
    ],
    "subject-case": [2, "always", "sentence-case"],
    "subject-max-length": [2, "always", 100],
  },
};

// Example commits:
// feat: add user authentication flow
// fix: resolve memory leak in data table
// docs: update API documentation for webhooks
// refactor: simplify state management in checkout
```

### ✅ Branch Protection Rules

```yaml
# Branch protection checklist
- [ ] Require pull request reviews before merging
- [ ] Require status checks to pass before merging
- [ ] Require branches to be up to date before merging
- [ ] Require linear history
- [ ] Include administrators in restrictions
- [ ] Require signed commits
- [ ] Restrict who can push to matching branches

# Branch naming convention
feature/TICKET-123-short-description
bugfix/TICKET-456-bug-description
hotfix/critical-issue-description
chore/dependency-updates
docs/update-readme
```

### ✅ Repository Templates

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Description

Brief description of changes

## Type of Change

- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change (fix or feature)
- [ ] Documentation update

## Testing

- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Screenshots (if applicable)

## Checklist

- [ ] My code follows the style guidelines
- [ ] I have performed a self-review
- [ ] I have commented my code
- [ ] I have updated documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests
- [ ] New and existing tests pass
```

## Development Environment

### ✅ Node Version Management

```bash
# .nvmrc - Lock Node version
18.19.0

# .npmrc - Configure npm
engine-strict=true
save-exact=true
audit=true
fund=false

# package.json - Engine specification
{
  "engines": {
    "node": ">=18.19.0",
    "npm": ">=9.0.0"
  }
}
```

### ✅ EditorConfig

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[*.{yml,yaml}]
indent_size = 2

[Makefile]
indent_style = tab
```

### ✅ VS Code Configuration

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "files.associations": {
    "*.css": "tailwindcss"
  },
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"],
    ["cn\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"]
  ]
}

// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.vscode-typescript-next",
    "usernamehw.errorlens",
    "christian-kohler.path-intellisense",
    "dsznajder.es7-react-js-snippets"
  ]
}
```

### ✅ Environment Setup Script

```bash
#!/bin/bash
# scripts/setup.sh - Development environment setup

echo "🚀 Setting up development environment..."

# Check Node version
if ! command -v node &> /dev/null; then
  echo "❌ Node.js is not installed"
  exit 1
fi

NODE_VERSION=$(node -v)
echo "✅ Node version: $NODE_VERSION"

# Install dependencies
echo "📦 Installing dependencies..."
npm ci

# Setup Git hooks
echo "🪝 Setting up Git hooks..."
npx husky install

# Create environment files
if [ ! -f .env.local ]; then
  echo "📝 Creating .env.local from template..."
  cp .env.example .env.local
  echo "⚠️  Please update .env.local with your credentials"
fi

# Verify setup
echo "🧪 Verifying setup..."
npm run lint
npm run type-check
npm test -- --passWithNoTests

echo "✅ Setup complete! Run 'npm run dev' to start development"
```

## Build Configuration

### ✅ Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: false,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
      "@components": path.resolve(__dirname, "./src/components"),
      "@hooks": path.resolve(__dirname, "./src/hooks"),
      "@utils": path.resolve(__dirname, "./src/utils"),
      "@types": path.resolve(__dirname, "./src/types"),
    },
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          router: ["react-router-dom"],
          ui: ["@radix-ui/react-dialog", "@radix-ui/react-dropdown-menu"],
        },
      },
    },
    chunkSizeWarningLimit: 1000,
  },
  server: {
    port: 3000,
    open: true,
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ""),
      },
    },
  },
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/test/setup.ts",
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      exclude: [
        "node_modules/",
        "src/test/",
        "**/*.spec.{ts,tsx}",
        "**/*.test.{ts,tsx}",
      ],
    },
  },
});
```

### ✅ TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,

    /* Path mapping */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@hooks/*": ["./src/hooks/*"],
      "@utils/*": ["./src/utils/*"],
      "@types/*": ["./src/types/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}

// tsconfig.node.json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

### ✅ Build Scripts

```json
// package.json - Build scripts
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "build:analyze": "npm run build && open dist/stats.html",
    "preview": "vite preview",
    "type-check": "tsc --noEmit",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,json,css,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,json,css,md}\"",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage",
    "clean": "rm -rf dist node_modules",
    "reinstall": "npm run clean && npm install",
    "validate": "npm run type-check && npm run lint && npm run test -- --run"
  }
}
```

## Code Quality Tools

### ✅ ESLint Configuration

```javascript
// .eslintrc.cjs
module.exports = {
  root: true,
  env: { browser: true, es2020: true },
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended-type-checked",
    "plugin:react/recommended",
    "plugin:react/jsx-runtime",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:import/recommended",
    "plugin:import/typescript",
    "prettier",
  ],
  ignorePatterns: ["dist", ".eslintrc.cjs"],
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module",
    project: ["./tsconfig.json", "./tsconfig.node.json"],
    tsconfigRootDir: __dirname,
  },
  plugins: ["react-refresh", "import", "jsx-a11y"],
  rules: {
    "react-refresh/only-export-components": [
      "warn",
      { allowConstantExport: true },
    ],
    "@typescript-eslint/no-unused-vars": [
      "error",
      { argsIgnorePattern: "^_", varsIgnorePattern: "^_" },
    ],
    "@typescript-eslint/consistent-type-imports": [
      "error",
      { prefer: "type-imports" },
    ],
    "import/order": [
      "error",
      {
        groups: [
          "builtin",
          "external",
          "internal",
          ["parent", "sibling"],
          "index",
        ],
        "newlines-between": "always",
        alphabetize: { order: "asc", caseInsensitive: true },
      },
    ],
    "react/prop-types": "off",
    "react/react-in-jsx-scope": "off",
  },
  settings: {
    react: {
      version: "detect",
    },
    "import/resolver": {
      typescript: {
        alwaysTryTypes: true,
        project: "./tsconfig.json",
      },
    },
  },
};
```

### ✅ Prettier Configuration

```javascript
// .prettierrc.cjs
module.exports = {
  semi: true,
  trailingComma: 'es5',
  singleQuote: true,
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  arrowParens: 'always',
  endOfLine: 'lf',
  bracketSpacing: true,
  jsxSingleQuote: false,
  plugins: ['prettier-plugin-tailwindcss'],
  overrides: [
    {
      files: '*.json',
      options: {
        printWidth: 80
      }
    }
  ]
};

// .prettierignore
dist
build
coverage
node_modules
*.min.js
*.min.css
```

### ✅ Stylelint Configuration

```javascript
// .stylelintrc.cjs
module.exports = {
  extends: [
    "stylelint-config-standard",
    "stylelint-config-tailwindcss",
    "stylelint-config-prettier",
  ],
  rules: {
    "at-rule-no-unknown": [
      true,
      {
        ignoreAtRules: ["tailwind", "apply", "layer", "config"],
      },
    ],
    "function-no-unknown": [
      true,
      {
        ignoreFunctions: ["theme", "screen"],
      },
    ],
    "selector-class-pattern": null,
    "custom-property-pattern": null,
  },
};
```

## Testing Infrastructure

### ✅ Test Setup

```typescript
// src/test/setup.ts
import "@testing-library/jest-dom";
import { expect, afterEach, vi } from "vitest";
import { cleanup } from "@testing-library/react";
import * as matchers from "@testing-library/jest-dom/matchers";

expect.extend(matchers);

// Cleanup after each test
afterEach(() => {
  cleanup();
});

// Mock window.matchMedia
Object.defineProperty(window, "matchMedia", {
  writable: true,
  value: vi.fn().mockImplementation((query) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});

// Mock IntersectionObserver
global.IntersectionObserver = class IntersectionObserver {
  constructor() {}
  disconnect() {}
  observe() {}
  takeRecords() {
    return [];
  }
  unobserve() {}
} as any;
```

### ✅ Test Utilities

```typescript
// src/test/utils.tsx
import { render, type RenderOptions } from '@testing-library/react';
import { ReactElement, ReactNode } from 'react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

interface AllTheProvidersProps {
  children: ReactNode;
}

function AllTheProviders({ children }: AllTheProvidersProps) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return (
    <BrowserRouter>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </BrowserRouter>
  );
}

export function renderWithProviders(
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, { wrapper: AllTheProviders, ...options });
}

export * from '@testing-library/react';
export { renderWithProviders as render };
```

### ✅ Testing Checklist

```typescript
// Testing standards
const testingChecklist = {
  unit: {
    coverage: "> 80% for utilities and hooks",
    conventions: [
      "One test file per source file",
      "Named *.test.ts or *.spec.ts",
      'Descriptive test names using "should"',
      "Arrange-Act-Assert pattern",
    ],
  },
  integration: {
    coverage: "> 60% for components",
    conventions: [
      "Test user interactions, not implementation",
      "Use data-testid sparingly",
      "Mock external dependencies",
      "Test error states",
    ],
  },
  e2e: {
    coverage: "Critical user flows",
    tools: ["Playwright", "Cypress"],
    conventions: [
      "Focus on happy path and critical errors",
      "Use Page Object pattern",
      "Independent tests (no shared state)",
    ],
  },
};
```

## CI/CD Pipeline

### ✅ GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run TypeScript
        run: npm run type-check

      - name: Check formatting
        run: npm run format:check

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage-final.json
          fail_ci_if_error: true

  build:
    runs-on: ubuntu-latest
    needs: [lint-and-type-check, test]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/
```

### ✅ Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build
        env:
          VITE_API_URL: ${{ secrets.API_URL }}

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
          vercel-args: "--prod"
```

## Documentation

### ✅ README Template

````markdown
# Project Name

Brief description of what this project does and who it's for.

## Features

- ✨ Feature 1
- 🚀 Feature 2
- 🔒 Feature 3

## Tech Stack

**Client:** React 18, TypeScript, TailwindCSS, React Query

**Build:** Vite

**Testing:** Vitest, Testing Library

## Getting Started

### Prerequisites

- Node.js >= 18.19.0
- npm >= 9.0.0

### Installation

```bash
# Clone the repository
git clone https://github.com/username/project.git

# Navigate to project
cd project

# Run setup script
npm run setup

# Start development server
npm run dev
```
````

## Available Scripts

- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm run preview` - Preview production build
- `npm run test` - Run tests
- `npm run lint` - Lint code
- `npm run format` - Format code

## Project Structure

```
src/
├── components/     # Reusable components
├── features/       # Feature-based modules
├── hooks/          # Custom React hooks
├── lib/            # Third-party library configurations
├── pages/          # Page components
├── styles/         # Global styles
├── types/          # TypeScript types
└── utils/          # Utility functions
```

## Environment Variables

Copy `.env.example` to `.env.local` and configure:

```env
VITE_API_URL=http://localhost:8000
VITE_AUTH_DOMAIN=your-auth-domain
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT License - see [LICENSE](LICENSE)

````

### ✅ Contributing Guide

```markdown
<!-- CONTRIBUTING.md -->
# Contributing Guide

## Development Workflow

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Make your changes
4. Run tests: `npm test`
5. Commit: `git commit -m 'feat: add amazing feature'`
6. Push: `git push origin feature/amazing-feature`
7. Open a Pull Request

## Code Style

- Follow ESLint and Prettier configurations
- Write meaningful commit messages following conventional commits
- Add tests for new features
- Update documentation

## Pull Request Process

1. Update README.md with details of changes if needed
2. Ensure all tests pass
3. Request review from maintainers
4. Address review comments
5. Squash commits before merge

## Reporting Bugs

Use GitHub Issues with bug template. Include:
- Description
- Steps to reproduce
- Expected vs actual behavior
- Screenshots if applicable
- Environment details
````

## Dependencies Management

### ✅ Package.json Structure

```json
{
  "name": "your-app",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint . --ext ts,tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,json,css,md}\"",
    "type-check": "tsc --noEmit",
    "validate": "npm run type-check && npm run lint && npm run test -- --run"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.1.5",
    "@testing-library/react": "^14.1.2",
    "@testing-library/user-event": "^14.5.1",
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@typescript-eslint/eslint-plugin": "^6.14.0",
    "@typescript-eslint/parser": "^6.14.0",
    "@vitejs/plugin-react": "^4.2.1",
    "eslint": "^8.55.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.5",
    "husky": "^8.0.3",
    "jsdom": "^23.0.1",
    "lint-staged": "^15.2.0",
    "prettier": "^3.1.1",
    "typescript": "^5.2.2",
    "vite": "^5.0.8",
    "vitest": "^1.0.4"
  }
}
```

### ✅ Dependency Update Strategy

```bash
# scripts/check-updates.sh
#!/bin/bash

echo "🔍 Checking for outdated packages..."

# Check outdated packages
npm outdated

# Check for security vulnerabilities
echo "\n🔒 Checking for security vulnerabilities..."
npm audit

# Suggest using npm-check-updates
echo "\n💡 To update dependencies interactively:"
echo "   npx npm-check-updates -i"
```

## Environment Configuration

### ✅ Environment Files

```bash
# .env.example - Template for environment variables
# Copy to .env.local and update values

# API Configuration
VITE_API_URL=http://localhost:8000
VITE_API_TIMEOUT=30000

# Authentication
VITE_AUTH_DOMAIN=auth.example.com
VITE_AUTH_CLIENT_ID=your_client_id

# Feature Flags
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_DEBUG=true

# Third-party Services
VITE_SENTRY_DSN=
VITE_GA_TRACKING_ID=

# Build Configuration
VITE_BUILD_TARGET=production
```

### ✅ Environment Types

```typescript
// src/types/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_API_TIMEOUT: string;
  readonly VITE_AUTH_DOMAIN: string;
  readonly VITE_AUTH_CLIENT_ID: string;
  readonly VITE_ENABLE_ANALYTICS: string;
  readonly VITE_ENABLE_DEBUG: string;
  readonly VITE_SENTRY_DSN?: string;
  readonly VITE_GA_TRACKING_ID?: string;
  readonly VITE_BUILD_TARGET: "development" | "staging" | "production";
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### ✅ Configuration Helper

```typescript
// src/config/env.ts
function getEnvVar(key: keyof ImportMetaEnv): string {
  const value = import.meta.env[key];
  if (value === undefined) {
    throw new Error(`Environment variable ${key} is not defined`);
  }
  return value;
}

function getBoolEnvVar(key: keyof ImportMetaEnv): boolean {
  const value = getEnvVar(key);
  return value === "true";
}

export const config = {
  api: {
    url: getEnvVar("VITE_API_URL"),
    timeout: parseInt(getEnvVar("VITE_API_TIMEOUT"), 10),
  },
  auth: {
    domain: getEnvVar("VITE_AUTH_DOMAIN"),
    clientId: getEnvVar("VITE_AUTH_CLIENT_ID"),
  },
  features: {
    analytics: getBoolEnvVar("VITE_ENABLE_ANALYTICS"),
    debug: getBoolEnvVar("VITE_ENABLE_DEBUG"),
  },
  sentry: {
    dsn: import.meta.env.VITE_SENTRY_DSN,
  },
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
} as const;
```

## Security Setup

### ✅ Security Headers

```typescript
// vite.config.ts - Security headers
export default defineConfig({
  server: {
    headers: {
      "X-Frame-Options": "DENY",
      "X-Content-Type-Options": "nosniff",
      "X-XSS-Protection": "1; mode=block",
      "Referrer-Policy": "strict-origin-when-cross-origin",
      "Permissions-Policy": "geolocation=(), microphone=(), camera=()",
    },
  },
});
```

### ✅ Content Security Policy

```html
<!-- index.html -->
<meta
  http-equiv="Content-Security-Policy"
  content="
    default-src 'self';
    script-src 'self' 'unsafe-inline' https://trusted-cdn.com;
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self' data:;
    connect-src 'self' https://api.example.com;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';
  "
/>
```

### ✅ Dependency Security

```json
// package.json - Security scripts
{
  "scripts": {
    "audit": "npm audit",
    "audit:fix": "npm audit fix",
    "audit:production": "npm audit --production",
    "check-licenses": "npx license-checker --summary"
  }
}
```

## Automation Scripts

### ✅ Project Setup Automation

```typescript
// scripts/setup.ts
import { execSync } from "child_process";
import fs from "fs";
import path from "path";

const colors = {
  green: "\x1b[32m",
  red: "\x1b[31m",
  yellow: "\x1b[33m",
  reset: "\x1b[0m",
};

function log(message: string, color: keyof typeof colors = "reset") {
  console.log(`${colors[color]}${message}${colors.reset}`);
}

function exec(command: string) {
  try {
    execSync(command, { stdio: "inherit" });
    return true;
  } catch {
    return false;
  }
}

async function setup() {
  log("🚀 Starting project setup...", "green");

  // Check Node version
  const nodeVersion = process.version;
  log(`✅ Node version: ${nodeVersion}`, "green");

  // Install dependencies
  log("\n📦 Installing dependencies...", "yellow");
  if (!exec("npm ci")) {
    log("❌ Failed to install dependencies", "red");
    process.exit(1);
  }

  // Setup Git hooks
  log("\n🪝 Setting up Git hooks...", "yellow");
  exec("npx husky install");

  // Create .env.local if not exists
  const envExample = ".env.example";
  const envLocal = ".env.local";

  if (fs.existsSync(envExample) && !fs.existsSync(envLocal)) {
    log("\n📝 Creating .env.local...", "yellow");
    fs.copyFileSync(envExample, envLocal);
    log("⚠️  Please update .env.local with your credentials", "yellow");
  }

  // Run validation
  log("\n🧪 Running validation...", "yellow");
  if (!exec("npm run validate")) {
    log("❌ Validation failed", "red");
    process.exit(1);
  }

  log("\n✅ Setup complete!", "green");
  log('Run "npm run dev" to start development', "green");
}

setup().catch((error) => {
  log(`❌ Setup failed: ${error.message}`, "red");
  process.exit(1);
});
```

### ✅ Pre-commit Validation

```bash
#!/bin/bash
# scripts/pre-commit.sh

echo "🔍 Running pre-commit checks..."

# Stage files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(ts|tsx)$')

if [ -z "$STAGED_FILES" ]; then
  echo "✅ No TypeScript files to check"
  exit 0
fi

# Type check
echo "📝 Type checking..."
npm run type-check || exit 1

# Lint
echo "🔍 Linting..."
npm run lint || exit 1

# Format check
echo "💅 Checking formatting..."
npm run format:check || exit 1

# Test affected files
echo "🧪 Testing..."
npm test -- --run --changed || exit 1

echo "✅ Pre-commit checks passed"
```

## Key Takeaways

1. **Automate Everything**: Setup scripts, Git hooks, and CI/CD reduce manual errors and improve consistency

2. **Enforce Quality Early**: ESLint, Prettier, TypeScript strict mode catch issues before they reach production

3. **Document Thoroughly**: README, contributing guide, and inline documentation help onboarding and maintenance

4. **Standardize Configuration**: Shared configs across projects reduce cognitive load and improve developer experience

5. **Version Control Everything**: .nvmrc, package-lock.json, and environment templates ensure reproducible builds

6. **Test Infrastructure First**: Setting up testing early makes TDD natural and prevents "we'll add tests later" syndrome

7. **Security by Default**: Content Security Policy, security headers, and dependency audits protect from day one

8. **Use Type Safety**: TypeScript strict mode and typed environment variables prevent runtime errors

9. **Optimize Build Process**: Code splitting, tree shaking, and bundle analysis keep application performant

10. **Continuous Validation**: CI/CD pipeline catches issues automatically, freeing developers to focus on features

---

**Quick Start Checklist**:

```bash
# Essential setup steps
□ Initialize Git repository
□ Add .gitignore
□ Setup Node version (.nvmrc)
□ Configure TypeScript (strict mode)
□ Setup ESLint + Prettier
□ Add pre-commit hooks (Husky)
□ Configure build tool (Vite)
□ Setup testing (Vitest + Testing Library)
□ Add CI/CD pipeline (GitHub Actions)
□ Create documentation (README)
□ Setup environment configuration
□ Add security headers
□ Configure path aliases
□ Setup VS Code workspace
□ Add automation scripts

# Ready to code! 🚀
```
