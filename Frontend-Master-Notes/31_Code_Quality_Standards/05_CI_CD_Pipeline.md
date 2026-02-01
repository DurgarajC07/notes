# CI/CD Pipeline for Frontend

## Table of Contents

- [Introduction](#introduction)
- [Pipeline Architecture](#pipeline-architecture)
- [GitHub Actions Setup](#github-actions-setup)
- [Testing in CI](#testing-in-ci)
- [Build and Deployment](#build-and-deployment)
- [Environment Management](#environment-management)
- [Performance Monitoring](#performance-monitoring)
- [Security Scanning](#security-scanning)
- [Optimization Strategies](#optimization-strategies)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)

## Introduction

CI/CD (Continuous Integration/Continuous Deployment) automates the process of testing, building, and deploying code. For frontend applications, this means catching bugs early, ensuring consistent builds, and deploying with confidence.

### Why CI/CD Matters

```
Without CI/CD:
Developer → Manual Testing → Manual Build → Manual Deploy → Hope it Works
                    ↑
        Time-consuming, error-prone, inconsistent

With CI/CD:
Developer → Push → Automated Tests → Automated Build → Automated Deploy → Confidence
                           ↑
                  Fast, reliable, consistent
```

### Benefits

- **Early Bug Detection**: Catch issues before they reach production
- **Consistent Builds**: Same process every time
- **Faster Releases**: Deploy multiple times per day
- **Reduced Risk**: Small, frequent changes easier to debug
- **Team Confidence**: Automated testing provides safety net

## Pipeline Architecture

### Complete CI/CD Pipeline

```
┌────────────────────────────────────────────────────────────────┐
│                      DEVELOPER WORKFLOW                         │
└───────────────────────┬────────────────────────────────────────┘
                        │
                        ▼
        ┌──────────────────────────────┐
        │   1. COMMIT & PUSH           │
        │   git push origin feature    │
        └────────────┬─────────────────┘
                     │
                     ▼
        ┌─────────────────────────────────────────┐
        │   2. CI PIPELINE TRIGGERED              │
        │   - Lint & Format Check                 │
        │   - Type Check                          │
        │   - Unit Tests                          │
        │   - Integration Tests                   │
        └────────────┬────────────────────────────┘
                     │
                     ├─ Success ─────────────────┐
                     │                            ▼
                     │                ┌──────────────────────┐
                     │                │  3. BUILD            │
                     │                │  - Compile assets    │
                     │                │  - Optimize          │
                     │                │  - Generate bundle   │
                     │                └─────────┬────────────┘
                     │                          │
                     │                          ▼
                     │                ┌──────────────────────┐
                     │                │  4. QUALITY CHECKS   │
                     │                │  - Bundle size       │
                     │                │  - Performance       │
                     │                │  - Security scan     │
                     │                └─────────┬────────────┘
                     │                          │
                     │                          ├─ Success ──┐
                     │                          │            ▼
                     │                          │   ┌─────────────────┐
                     │                          │   │  5. DEPLOY      │
                     │                          │   │  Dev/Staging    │
                     │                          │   └────────┬────────┘
                     │                          │            │
                     │                          │            ▼
                     │                          │   ┌─────────────────┐
                     │                          │   │  6. E2E TESTS   │
                     │                          │   │  Production-like│
                     │                          │   └────────┬────────┘
                     │                          │            │
                     │                          │            ├─ Success
                     │                          │            │
                     ├─ Failure ───────────────┼────────────┼───┐
                     │                          │            │   │
                     │                          └─ Failure ──┤   │
                     │                                       │   │
                     ▼                                       ▼   ▼
        ┌──────────────────────────────┐      ┌─────────────────────┐
        │  NOTIFY DEVELOPER            │      │  7. PRODUCTION      │
        │  - Email/Slack               │      │  - Blue/Green       │
        │  - Show logs                 │      │  - Rollback ready   │
        │  - Suggest fixes             │      └─────────────────────┘
        └──────────────────────────────┘
```

### Pipeline Stages Overview

```yaml
stages:
  - lint: # Code quality checks
  - test: # Unit and integration tests
  - build: # Compile and bundle
  - analyze: # Bundle size, performance
  - security: # Dependency and code scanning
  - deploy-dev: # Deploy to development
  - e2e: # End-to-end tests
  - deploy-prod: # Deploy to production (manual)
```

## GitHub Actions Setup

### Basic Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check formatting
        run: npm run format:check

      - name: Type check
        run: npm run type-check
```

### Complete CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: "18"
  CACHE_NAME: cache-node-modules

jobs:
  # Job 1: Install dependencies (run once, cache for other jobs)
  install:
    name: Install Dependencies
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Cache node_modules
        id: cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-${{ env.CACHE_NAME }}-${{ hashFiles('**/package-lock.json') }}

      - name: Install dependencies
        if: steps.cache.outputs.cache-hit != 'true'
        run: npm ci

  # Job 2: Lint and format
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    needs: install

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-${{ env.CACHE_NAME }}-${{ hashFiles('**/package-lock.json') }}

      - name: Run ESLint
        run: npm run lint -- --max-warnings=0

      - name: Check Prettier
        run: npm run format:check

      - name: Type check
        run: npm run type-check

  # Job 3: Unit tests
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: install

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-${{ env.CACHE_NAME }}-${{ hashFiles('**/package-lock.json') }}

      - name: Run tests
        run: npm test -- --coverage --maxWorkers=2

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          fail_ci_if_error: true

  # Job 4: Build
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Restore cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-${{ env.CACHE_NAME }}-${{ hashFiles('**/package-lock.json') }}

      - name: Build
        run: npm run build
        env:
          NODE_ENV: production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/
          retention-days: 7

  # Job 5: Bundle analysis
  analyze:
    name: Bundle Analysis
    runs-on: ubuntu-latest
    needs: build

    steps:
      - uses: actions/checkout@v3

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/

      - name: Analyze bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Matrix Testing (Multiple Versions)

```yaml
# .github/workflows/test-matrix.yml
name: Test Matrix

on: [push, pull_request]

jobs:
  test:
    name: Test Node ${{ matrix.node-version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
      fail-fast: false

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

## Testing in CI

### Jest Configuration for CI

```javascript
// jest.config.ci.js
module.exports = {
  ...require("./jest.config"),

  // Use fewer workers in CI (limited resources)
  maxWorkers: 2,

  // Enable coverage
  collectCoverage: true,
  coverageReporters: ["json", "lcov", "text", "clover"],

  // Fail if coverage drops below thresholds
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },

  // CI-specific settings
  ci: true,
  bail: true, // Stop on first failure
  verbose: true,
};
```

### Parallel Test Execution

```yaml
# .github/workflows/parallel-tests.yml
name: Parallel Tests

on: [push]

jobs:
  test:
    name: Test Chunk ${{ matrix.chunk }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        chunk: [1, 2, 3, 4]

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests (chunk ${{ matrix.chunk }})
        run: |
          npm test -- \
            --shard=${{ matrix.chunk }}/4 \
            --coverage
```

### E2E Tests with Playwright

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Start server
        run: npm run preview &

      - name: Wait for server
        run: npx wait-on http://localhost:4173

      - name: Run E2E tests
        run: npx playwright test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

### Visual Regression Testing

```yaml
# .github/workflows/visual-regression.yml
name: Visual Regression

on: [pull_request]

jobs:
  visual-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Build Storybook
        run: npm run build-storybook

      - name: Run Chromatic
        uses: chromaui/action@v1
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          storybookBuildDir: storybook-static
          exitOnceUploaded: true
```

## Build and Deployment

### Multi-Stage Build

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main, develop]

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build for production
        run: npm run build
        env:
          NODE_ENV: production
          VITE_API_URL: ${{ secrets.PROD_API_URL }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: production-build
          path: dist/

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'

    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v3

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: production-build
          path: dist/

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: "--prod"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: github.ref == 'refs/heads/main'

    environment:
      name: production
      url: https://example.com

    steps:
      - uses: actions/checkout@v3

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: production-build
          path: dist/

      - name: Deploy to Netlify
        uses: nwtgck/actions-netlify@v2
        with:
          publish-dir: "./dist"
          production-deploy: true
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
```

### Docker Build and Push

```yaml
# .github/workflows/docker.yml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: myorg/myapp
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### AWS S3 Deployment

```yaml
# .github/workflows/deploy-s3.yml
name: Deploy to S3

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: |
          aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }} \
            --delete \
            --cache-control "public, max-age=31536000"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

## Environment Management

### Environment Variables

```yaml
# .github/workflows/env-management.yml
name: Environment Management

on:
  push:
    branches: [main, develop, staging]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set environment variables
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "ENV_NAME=production" >> $GITHUB_ENV
            echo "API_URL=${{ secrets.PROD_API_URL }}" >> $GITHUB_ENV
            echo "SENTRY_DSN=${{ secrets.PROD_SENTRY_DSN }}" >> $GITHUB_ENV
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "ENV_NAME=staging" >> $GITHUB_ENV
            echo "API_URL=${{ secrets.STAGING_API_URL }}" >> $GITHUB_ENV
            echo "SENTRY_DSN=${{ secrets.STAGING_SENTRY_DSN }}" >> $GITHUB_ENV
          else
            echo "ENV_NAME=development" >> $GITHUB_ENV
            echo "API_URL=${{ secrets.DEV_API_URL }}" >> $GITHUB_ENV
            echo "SENTRY_DSN=${{ secrets.DEV_SENTRY_DSN }}" >> $GITHUB_ENV
          fi

      - name: Build with environment
        run: npm run build
        env:
          VITE_ENV: ${{ env.ENV_NAME }}
          VITE_API_URL: ${{ env.API_URL }}
          VITE_SENTRY_DSN: ${{ env.SENTRY_DSN }}
```

### Deployment Environments

```yaml
# Using GitHub Environments
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Deploy
        run: ./deploy.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    needs: deploy-staging

    steps:
      - name: Deploy
        run: ./deploy.sh production
```

## Performance Monitoring

### Bundle Size Tracking

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size Check

on: [pull_request]

jobs:
  size:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Check bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          build_script: build
```

```javascript
// .size-limit.js
module.exports = [
  {
    name: "Main Bundle",
    path: "dist/assets/index-*.js",
    limit: "150 KB",
    webpack: false,
  },
  {
    name: "Vendor Bundle",
    path: "dist/assets/vendor-*.js",
    limit: "200 KB",
    webpack: false,
  },
  {
    name: "Total JS",
    path: "dist/assets/*.js",
    limit: "350 KB",
    webpack: false,
  },
];
```

### Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/about
          budgetPath: ./budget.json
          uploadArtifacts: true
          temporaryPublicStorage: true
```

```json
// budget.json
{
  "budgets": [
    {
      "path": "/*",
      "timings": [
        {
          "metric": "interactive",
          "budget": 3000,
          "tolerance": 500
        },
        {
          "metric": "first-contentful-paint",
          "budget": 1800,
          "tolerance": 300
        }
      ],
      "resourceSizes": [
        {
          "resourceType": "script",
          "budget": 300
        },
        {
          "resourceType": "total",
          "budget": 500
        }
      ]
    }
  ]
}
```

## Security Scanning

### Dependency Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 0 * * 0" # Weekly on Sunday

jobs:
  dependency-scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Run npm audit
        run: npm audit --audit-level=moderate
        continue-on-error: true

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

### Code Scanning

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 12 * * 1" # Weekly on Monday at noon

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write

    steps:
      - uses: actions/checkout@v3

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript

      - name: Autobuild
        uses: github/codeql-action/autobuild@v2

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

### Secret Scanning

```yaml
# .github/workflows/secrets.yml
name: Secret Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
```

## Optimization Strategies

### Caching Strategy

```yaml
# Optimized caching workflow
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      # Cache npm packages
      - name: Cache npm
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      # Cache node_modules
      - name: Cache node_modules
        id: cache-node-modules
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-node-modules-${{ hashFiles('**/package-lock.json') }}

      - name: Install dependencies
        if: steps.cache-node-modules.outputs.cache-hit != 'true'
        run: npm ci

      # Cache build output
      - name: Cache build
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
```

### Parallel Jobs

```yaml
# Run jobs in parallel
jobs:
  # All these run simultaneously
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  test-unit:
    runs-on: ubuntu-latest
    steps: [...]

  test-integration:
    runs-on: ubuntu-latest
    steps: [...]

  # This waits for all above to complete
  build:
    needs: [lint, test-unit, test-integration]
    runs-on: ubuntu-latest
    steps: [...]
```

### Conditional Execution

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      # Only run tests if relevant files changed
      - name: Check for changes
        id: changes
        run: |
          if git diff --name-only HEAD~1 | grep -E '\.(js|jsx|ts|tsx)$'; then
            echo "run=true" >> $GITHUB_OUTPUT
          else
            echo "run=false" >> $GITHUB_OUTPUT
          fi

      - name: Run tests
        if: steps.changes.outputs.run == 'true'
        run: npm test
```

## Troubleshooting

### Common Issues and Solutions

```yaml
# Issue 1: Flaky tests
# Solution: Retry failed tests
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Run tests with retry
        uses: nick-invision/retry@v2
        with:
          timeout_minutes: 10
          max_attempts: 3
          command: npm test

# Issue 2: Out of memory
# Solution: Increase Node memory
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Build with more memory
        run: NODE_OPTIONS="--max-old-space-size=4096" npm run build

# Issue 3: Slow builds
# Solution: Use build matrix
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v3
      - run: npm test -- --shard=${{ matrix.shard }}/4

# Issue 4: Secrets not available in PR from forks
# Solution: Use pull_request_target (with caution)
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  safe-checkout:
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}
```

### Debugging Workflows

```yaml
# Enable debug logging
jobs:
  debug:
    runs-on: ubuntu-latest

    steps:
      - name: Enable debug
        run: echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV

      - name: Debug information
        run: |
          echo "Event: ${{ github.event_name }}"
          echo "Ref: ${{ github.ref }}"
          echo "SHA: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          env | sort
```

### Notification on Failure

```yaml
# .github/workflows/notify.yml
jobs:
  notify:
    runs-on: ubuntu-latest
    if: failure()
    needs: [lint, test, build]

    steps:
      - name: Send Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "Build failed!"
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

      - name: Create GitHub issue
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `CI Failed: ${context.workflow}`,
              body: `Build failed on ${context.ref}\nRun: ${context.runId}`,
              labels: ['bug', 'ci']
            })
```

## Key Takeaways

1. **Automate Everything**: Automate linting, testing, building, and deployment. Manual processes are error-prone and slow. CI/CD should be fully automated from commit to production.

2. **Fail Fast**: Run quick checks (linting, type-checking) before expensive ones (E2E tests). Cancel redundant runs when new commits are pushed.

3. **Parallel Execution**: Run independent jobs in parallel to reduce pipeline time. Test different chunks simultaneously for faster feedback.

4. **Smart Caching**: Cache dependencies, build outputs, and test results. Proper caching can reduce build times from 10 minutes to 2 minutes.

5. **Environment Isolation**: Use separate environments (dev, staging, production) with distinct configurations. Never deploy untested code directly to production.

6. **Comprehensive Testing**: Include unit tests, integration tests, E2E tests, and visual regression tests. Each catches different types of bugs.

7. **Security First**: Scan dependencies regularly, check for secrets in code, and run static analysis. Security issues are easier to fix early.

8. **Monitor Performance**: Track bundle size, Lighthouse scores, and Core Web Vitals. Prevent performance regressions before they reach production.

9. **Clear Failure Messages**: Provide actionable error messages. Tell developers exactly what failed and how to fix it. Include links to logs and documentation.

10. **Continuous Improvement**: Regularly review and optimize your pipeline. Remove unnecessary steps, update dependencies, and learn from failures. A good CI/CD pipeline evolves with your team's needs.
