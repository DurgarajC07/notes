# Production Deployment Checklist

## Table of Contents

- [Introduction](#introduction)
- [Pre-Deployment](#pre-deployment)
- [Build and Test](#build-and-test)
- [Environment Configuration](#environment-configuration)
- [Database Migration](#database-migration)
- [Security Hardening](#security-hardening)
- [Monitoring Setup](#monitoring-setup)
- [CDN and Caching](#cdn-and-caching)
- [Deployment Strategy](#deployment-strategy)
- [Post-Deployment](#post-deployment)
- [Rollback Plan](#rollback-plan)
- [Documentation](#documentation)
- [Key Takeaways](#key-takeaways)

## Introduction

Production deployment is the final step before your application reaches users. A comprehensive checklist ensures nothing is overlooked and provides a clear rollback path if issues arise.

### Deployment Principles

```typescript
interface DeploymentConfig {
  environment: "staging" | "production";
  strategy: "blue-green" | "canary" | "rolling";
  healthCheck: string;
  rollbackEnabled: boolean;
  autoScaling: boolean;
  monitoring: boolean;
}

const productionConfig: DeploymentConfig = {
  environment: "production",
  strategy: "blue-green",
  healthCheck: "/api/health",
  rollbackEnabled: true,
  autoScaling: true,
  monitoring: true,
};

// Deployment checklist state
interface ChecklistItem {
  id: string;
  category: string;
  task: string;
  completed: boolean;
  required: boolean;
  verifiedBy?: string;
  verifiedAt?: Date;
}
```

## Pre-Deployment

### ✅ Code Review and Testing

```typescript
// Pre-deployment verification script
import { execSync } from "child_process";

interface PreDeploymentChecks {
  codeQuality: boolean;
  tests: boolean;
  security: boolean;
  performance: boolean;
  documentation: boolean;
}

async function runPreDeploymentChecks(): Promise<PreDeploymentChecks> {
  const results: PreDeploymentChecks = {
    codeQuality: false,
    tests: false,
    security: false,
    performance: false,
    documentation: false,
  };

  console.log("🔍 Running pre-deployment checks...\n");

  // 1. Code Quality
  try {
    console.log("✓ Running ESLint...");
    execSync("npm run lint", { stdio: "inherit" });

    console.log("✓ Running TypeScript check...");
    execSync("npm run type-check", { stdio: "inherit" });

    results.codeQuality = true;
  } catch (error) {
    console.error("✗ Code quality checks failed");
    return results;
  }

  // 2. Tests
  try {
    console.log("✓ Running unit tests...");
    execSync("npm run test:unit", { stdio: "inherit" });

    console.log("✓ Running integration tests...");
    execSync("npm run test:integration", { stdio: "inherit" });

    console.log("✓ Running E2E tests...");
    execSync("npm run test:e2e", { stdio: "inherit" });

    results.tests = true;
  } catch (error) {
    console.error("✗ Tests failed");
    return results;
  }

  // 3. Security
  try {
    console.log("✓ Running security audit...");
    execSync("npm audit --production", { stdio: "inherit" });

    console.log("✓ Checking for secrets...");
    execSync("npm run check-secrets", { stdio: "inherit" });

    results.security = true;
  } catch (error) {
    console.error("✗ Security checks failed");
    return results;
  }

  // 4. Performance
  try {
    console.log("✓ Running Lighthouse CI...");
    execSync("npm run lighthouse", { stdio: "inherit" });

    console.log("✓ Analyzing bundle size...");
    execSync("npm run analyze", { stdio: "inherit" });

    results.performance = true;
  } catch (error) {
    console.error("✗ Performance checks failed");
    return results;
  }

  // 5. Documentation
  try {
    console.log("✓ Checking documentation...");
    execSync("npm run docs:check", { stdio: "inherit" });

    results.documentation = true;
  } catch (error) {
    console.error("✗ Documentation checks failed");
  }

  return results;
}

// Usage
const checks = await runPreDeploymentChecks();

if (Object.values(checks).every(Boolean)) {
  console.log("\n✅ All pre-deployment checks passed!");
  console.log("Ready to deploy to production.");
} else {
  console.log("\n❌ Pre-deployment checks failed!");
  console.log("Please fix issues before deploying.");
  process.exit(1);
}
```

### ✅ Version Control

```bash
#!/bin/bash
# pre-deploy-git-check.sh

echo "🔍 Checking Git status..."

# Ensure on correct branch
CURRENT_BRANCH=$(git branch --show-current)
if [ "$CURRENT_BRANCH" != "main" ] && [ "$CURRENT_BRANCH" != "master" ]; then
  echo "❌ Not on main/master branch (currently on $CURRENT_BRANCH)"
  exit 1
fi

# Ensure clean working directory
if [ -n "$(git status --porcelain)" ]; then
  echo "❌ Uncommitted changes detected"
  git status --short
  exit 1
fi

# Ensure up to date with remote
git fetch origin
LOCAL=$(git rev-parse @)
REMOTE=$(git rev-parse @{u})

if [ "$LOCAL" != "$REMOTE" ]; then
  echo "❌ Local branch is not up to date with remote"
  exit 1
fi

# Create deployment tag
VERSION=$(node -p "require('./package.json').version")
TAG="v$VERSION"

if git rev-parse "$TAG" >/dev/null 2>&1; then
  echo "❌ Tag $TAG already exists"
  exit 1
fi

echo "✅ Git checks passed"
echo "Creating tag: $TAG"
git tag -a "$TAG" -m "Release $VERSION"
git push origin "$TAG"

echo "✅ Ready for deployment"
```

## Build and Test

### ✅ Production Build

```typescript
// build-production.ts
import { build } from "vite";
import { execSync } from "child_process";
import { readFileSync, writeFileSync } from "fs";
import { gzipSync } from "zlib";

interface BuildMetrics {
  timestamp: string;
  version: string;
  duration: number;
  bundleSize: {
    total: number;
    js: number;
    css: number;
    assets: number;
  };
  gzipSize: {
    total: number;
  };
  chunks: Array<{
    name: string;
    size: number;
    gzipSize: number;
  }>;
}

async function buildProduction(): Promise<BuildMetrics> {
  const startTime = Date.now();

  console.log("🏗️  Building for production...\n");

  // Clean dist folder
  execSync("rm -rf dist", { stdio: "inherit" });

  // Run production build
  await build({
    mode: "production",
    build: {
      outDir: "dist",
      sourcemap: true,
      minify: "terser",
      terserOptions: {
        compress: {
          drop_console: true,
          drop_debugger: true,
        },
      },
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ["react", "react-dom"],
            ui: ["@radix-ui/react-dialog", "@radix-ui/react-dropdown-menu"],
          },
        },
      },
    },
  });

  const duration = Date.now() - startTime;

  // Analyze bundle
  const metrics = analyzeBuild();
  metrics.duration = duration;
  metrics.timestamp = new Date().toISOString();
  metrics.version = JSON.parse(readFileSync("package.json", "utf-8")).version;

  // Save build report
  writeFileSync("build-report.json", JSON.stringify(metrics, null, 2));

  console.log("\n✅ Production build complete!\n");
  console.log(`Duration: ${(duration / 1000).toFixed(2)}s`);
  console.log(`Total size: ${(metrics.bundleSize.total / 1024).toFixed(2)} KB`);
  console.log(`Gzipped: ${(metrics.gzipSize.total / 1024).toFixed(2)} KB`);

  // Check bundle size limits
  const MAX_BUNDLE_SIZE = 500 * 1024; // 500 KB
  if (metrics.bundleSize.total > MAX_BUNDLE_SIZE) {
    console.warn(
      `\n⚠️  Bundle size exceeds limit (${MAX_BUNDLE_SIZE / 1024} KB)`,
    );
  }

  return metrics;
}

function analyzeBuild(): BuildMetrics {
  const distFiles = execSync("find dist -type f", { encoding: "utf-8" })
    .split("\n")
    .filter(Boolean);

  let totalSize = 0;
  let jsSize = 0;
  let cssSize = 0;
  let assetsSize = 0;
  let totalGzipSize = 0;

  const chunks: BuildMetrics["chunks"] = [];

  distFiles.forEach((file) => {
    const content = readFileSync(file);
    const size = content.length;
    const gzipSize = gzipSync(content).length;

    totalSize += size;
    totalGzipSize += gzipSize;

    if (file.endsWith(".js")) {
      jsSize += size;
      chunks.push({
        name: file.replace("dist/", ""),
        size,
        gzipSize,
      });
    } else if (file.endsWith(".css")) {
      cssSize += size;
    } else {
      assetsSize += size;
    }
  });

  return {
    timestamp: "",
    version: "",
    duration: 0,
    bundleSize: {
      total: totalSize,
      js: jsSize,
      css: cssSize,
      assets: assetsSize,
    },
    gzipSize: {
      total: totalGzipSize,
    },
    chunks: chunks.sort((a, b) => b.size - a.size),
  };
}

// Run build
buildProduction().catch((error) => {
  console.error("❌ Build failed:", error);
  process.exit(1);
});
```

### ✅ Smoke Tests

```typescript
// smoke-tests.ts
import puppeteer from "puppeteer";

interface SmokeTest {
  name: string;
  url: string;
  test: (page: puppeteer.Page) => Promise<boolean>;
}

const smokeTests: SmokeTest[] = [
  {
    name: "Homepage loads",
    url: "/",
    test: async (page) => {
      const title = await page.title();
      return title.length > 0;
    },
  },
  {
    name: "Login page accessible",
    url: "/login",
    test: async (page) => {
      const form = await page.$("form");
      return form !== null;
    },
  },
  {
    name: "API health check",
    url: "/api/health",
    test: async (page) => {
      const content = await page.content();
      return content.includes("ok");
    },
  },
  {
    name: "Assets load correctly",
    url: "/",
    test: async (page) => {
      const failures = await page.evaluate(() => {
        return Array.from(
          document.querySelectorAll('img, script, link[rel="stylesheet"]'),
        ).filter((el) => {
          if (el instanceof HTMLImageElement) return !el.complete;
          if (el instanceof HTMLScriptElement)
            return el.getAttribute("data-error") === "true";
          if (el instanceof HTMLLinkElement) return el.sheet === null;
          return false;
        });
      });
      return failures.length === 0;
    },
  },
  {
    name: "No console errors",
    url: "/",
    test: async (page) => {
      const errors: string[] = [];
      page.on("console", (msg) => {
        if (msg.type() === "error") {
          errors.push(msg.text());
        }
      });
      await page.waitForTimeout(2000);
      return errors.length === 0;
    },
  },
];

async function runSmokeTests(baseUrl: string): Promise<boolean> {
  console.log("🔥 Running smoke tests...\n");

  const browser = await puppeteer.launch();
  const results: Array<{ name: string; passed: boolean; error?: string }> = [];

  for (const test of smokeTests) {
    const page = await browser.newPage();

    try {
      await page.goto(`${baseUrl}${test.url}`, {
        waitUntil: "networkidle2",
        timeout: 10000,
      });

      const passed = await test.test(page);
      results.push({ name: test.name, passed });

      console.log(`${passed ? "✓" : "✗"} ${test.name}`);
    } catch (error) {
      results.push({
        name: test.name,
        passed: false,
        error: error instanceof Error ? error.message : String(error),
      });
      console.log(`✗ ${test.name}: ${error}`);
    } finally {
      await page.close();
    }
  }

  await browser.close();

  const allPassed = results.every((r) => r.passed);

  console.log(
    `\n${allPassed ? "✅" : "❌"} Smoke tests ${allPassed ? "passed" : "failed"}`,
  );
  console.log(
    `Passed: ${results.filter((r) => r.passed).length}/${results.length}`,
  );

  return allPassed;
}

// Usage
const STAGING_URL = process.env.STAGING_URL || "https://staging.example.com";
const passed = await runSmokeTests(STAGING_URL);

if (!passed) {
  process.exit(1);
}
```

## Environment Configuration

### ✅ Environment Variables

```typescript
// env.ts - Type-safe environment variables
import { z } from "zod";

const envSchema = z.object({
  // Required
  NODE_ENV: z.enum(["development", "staging", "production"]),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),

  // API Keys
  STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  SENDGRID_API_KEY: z.string(),
  AWS_ACCESS_KEY_ID: z.string(),
  AWS_SECRET_ACCESS_KEY: z.string(),

  // Optional
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  RATE_LIMIT_MAX: z.coerce.number().default(100),
  RATE_LIMIT_WINDOW: z.coerce.number().default(900000),

  // Feature Flags
  FEATURE_NEW_CHECKOUT: z.coerce.boolean().default(false),
  FEATURE_BETA_UI: z.coerce.boolean().default(false),
});

export type Env = z.infer<typeof envSchema>;

// Validate environment variables
function validateEnv(): Env {
  try {
    return envSchema.parse(process.env);
  } catch (error) {
    if (error instanceof z.ZodError) {
      console.error("❌ Invalid environment variables:");
      error.errors.forEach((err) => {
        console.error(`  - ${err.path.join(".")}: ${err.message}`);
      });
    }
    process.exit(1);
  }
}

export const env = validateEnv();

// .env.production template
const envTemplate = `
# Database
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://host:6379

# Authentication
JWT_SECRET=your-secret-key-min-32-characters

# Third-party APIs
STRIPE_SECRET_KEY=sk_live_xxxxx
SENDGRID_API_KEY=SG.xxxxx
AWS_ACCESS_KEY_ID=AKIAXXXXX
AWS_SECRET_ACCESS_KEY=xxxxx

# Configuration
NODE_ENV=production
LOG_LEVEL=info
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=900000

# Feature Flags
FEATURE_NEW_CHECKOUT=false
FEATURE_BETA_UI=false
`;
```

### ✅ Configuration Management

```typescript
// config.ts
interface AppConfig {
  server: {
    port: number;
    host: string;
    cors: {
      origin: string[];
      credentials: boolean;
    };
  };
  database: {
    url: string;
    poolSize: number;
    ssl: boolean;
  };
  cache: {
    redis: {
      url: string;
      ttl: number;
    };
  };
  monitoring: {
    sentry: {
      dsn: string;
      environment: string;
      tracesSampleRate: number;
    };
    datadog: {
      apiKey: string;
      service: string;
    };
  };
  features: Record<string, boolean>;
}

const productionConfig: AppConfig = {
  server: {
    port: parseInt(env.PORT || "3000"),
    host: "0.0.0.0",
    cors: {
      origin: ["https://example.com", "https://www.example.com"],
      credentials: true,
    },
  },
  database: {
    url: env.DATABASE_URL,
    poolSize: 20,
    ssl: true,
  },
  cache: {
    redis: {
      url: env.REDIS_URL,
      ttl: 3600,
    },
  },
  monitoring: {
    sentry: {
      dsn: env.SENTRY_DSN,
      environment: "production",
      tracesSampleRate: 0.1,
    },
    datadog: {
      apiKey: env.DATADOG_API_KEY,
      service: "my-app",
    },
  },
  features: {
    newCheckout: env.FEATURE_NEW_CHECKOUT,
    betaUI: env.FEATURE_BETA_UI,
  },
};

export const config = productionConfig;
```

## Database Migration

### ✅ Migration Strategy

```typescript
// migrate.ts
import { Pool } from "pg";
import { readFileSync, readdirSync } from "fs";
import { join } from "path";

interface Migration {
  id: number;
  name: string;
  sql: string;
  appliedAt?: Date;
}

class MigrationRunner {
  private pool: Pool;

  constructor(databaseUrl: string) {
    this.pool = new Pool({ connectionString: databaseUrl });
  }

  async ensureMigrationsTable(): Promise<void> {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS migrations (
        id INTEGER PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
  }

  async getAppliedMigrations(): Promise<Set<number>> {
    const result = await this.pool.query("SELECT id FROM migrations");
    return new Set(result.rows.map((row) => row.id));
  }

  async loadMigrations(): Promise<Migration[]> {
    const migrationsDir = join(__dirname, "migrations");
    const files = readdirSync(migrationsDir)
      .filter((f) => f.endsWith(".sql"))
      .sort();

    return files.map((file) => {
      const [id, ...nameParts] = file.replace(".sql", "").split("_");
      return {
        id: parseInt(id),
        name: nameParts.join("_"),
        sql: readFileSync(join(migrationsDir, file), "utf-8"),
      };
    });
  }

  async runMigrations(dryRun = false): Promise<void> {
    console.log("🔄 Running database migrations...\n");

    await this.ensureMigrationsTable();
    const applied = await this.getAppliedMigrations();
    const migrations = await this.loadMigrations();
    const pending = migrations.filter((m) => !applied.has(m.id));

    if (pending.length === 0) {
      console.log("✅ No pending migrations");
      return;
    }

    console.log(`Found ${pending.length} pending migrations:\n`);
    pending.forEach((m) => {
      console.log(`  ${m.id}: ${m.name}`);
    });

    if (dryRun) {
      console.log("\n🔍 Dry run - no changes applied");
      return;
    }

    // Create backup
    console.log("\n📦 Creating database backup...");
    await this.createBackup();

    const client = await this.pool.connect();

    try {
      await client.query("BEGIN");

      for (const migration of pending) {
        console.log(`\n⏳ Applying: ${migration.id}_${migration.name}`);

        await client.query(migration.sql);
        await client.query(
          "INSERT INTO migrations (id, name) VALUES ($1, $2)",
          [migration.id, migration.name],
        );

        console.log(`✅ Applied: ${migration.id}_${migration.name}`);
      }

      await client.query("COMMIT");
      console.log("\n✅ All migrations applied successfully!");
    } catch (error) {
      await client.query("ROLLBACK");
      console.error("\n❌ Migration failed:", error);
      console.log("🔄 Restoring from backup...");
      await this.restoreBackup();
      throw error;
    } finally {
      client.release();
    }
  }

  async createBackup(): Promise<void> {
    const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
    const backupFile = `backup_${timestamp}.sql`;

    // Implementation depends on your database
    console.log(`Created backup: ${backupFile}`);
  }

  async restoreBackup(): Promise<void> {
    // Implementation depends on your database
    console.log("Backup restored");
  }

  async close(): Promise<void> {
    await this.pool.end();
  }
}

// Usage
const runner = new MigrationRunner(env.DATABASE_URL);

// Dry run first
await runner.runMigrations(true);

// Prompt for confirmation
const readline = require("readline").createInterface({
  input: process.stdin,
  output: process.stdout,
});

readline.question("Apply migrations? (yes/no): ", async (answer: string) => {
  if (answer.toLowerCase() === "yes") {
    await runner.runMigrations(false);
  } else {
    console.log("Migration cancelled");
  }

  await runner.close();
  readline.close();
});
```

### ✅ Migration Files

```sql
-- migrations/001_create_users_table.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/002_add_user_roles.sql
CREATE TYPE user_role AS ENUM ('user', 'admin', 'moderator');

ALTER TABLE users
ADD COLUMN role user_role DEFAULT 'user';

CREATE INDEX idx_users_role ON users(role);

-- migrations/003_create_posts_table.sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

## Security Hardening

### ✅ Security Headers

```typescript
// security.middleware.ts
import helmet from "helmet";
import { Express } from "express";

export function setupSecurityHeaders(app: Express): void {
  // Helmet - Security headers
  app.use(
    helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
          styleSrc: [
            "'self'",
            "'unsafe-inline'",
            "https://fonts.googleapis.com",
          ],
          fontSrc: ["'self'", "https://fonts.gstatic.com"],
          imgSrc: ["'self'", "data:", "https:"],
          connectSrc: ["'self'", "https://api.example.com"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
          upgradeInsecureRequests: [],
        },
      },
      hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true,
      },
      referrerPolicy: {
        policy: "strict-origin-when-cross-origin",
      },
    }),
  );

  // Additional headers
  app.use((req, res, next) => {
    res.setHeader("X-Content-Type-Options", "nosniff");
    res.setHeader("X-Frame-Options", "DENY");
    res.setHeader("X-XSS-Protection", "1; mode=block");
    res.setHeader(
      "Permissions-Policy",
      "geolocation=(), microphone=(), camera=()",
    );
    next();
  });
}

// Rate limiting
import rateLimit from "express-rate-limit";

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: "Too many requests from this IP",
  standardHeaders: true,
  legacyHeaders: false,
});

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 attempts
  skipSuccessfulRequests: true,
});
```

## Monitoring Setup

### ✅ Application Monitoring

```typescript
// monitoring.ts
import * as Sentry from "@sentry/node";
import { datadogLogs } from "@datadog/browser-logs";

// Sentry initialization
export function initSentry(): void {
  Sentry.init({
    dsn: env.SENTRY_DSN,
    environment: env.NODE_ENV,
    tracesSampleRate: 0.1,
    beforeSend(event, hint) {
      // Filter sensitive data
      if (event.request) {
        delete event.request.cookies;
        delete event.request.headers?.authorization;
      }
      return event;
    },
    integrations: [
      new Sentry.Integrations.Http({ tracing: true }),
      new Sentry.Integrations.Express({ app }),
    ],
  });
}

// Datadog logging
export function initDatadog(): void {
  datadogLogs.init({
    clientToken: env.DATADOG_CLIENT_TOKEN,
    site: "datadoghq.com",
    forwardErrorsToLogs: true,
    sessionSampleRate: 100,
    service: "my-app",
    env: env.NODE_ENV,
  });
}

// Health check endpoint
export function healthCheck(app: Express): void {
  app.get("/api/health", async (req, res) => {
    const checks = {
      status: "healthy",
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      database: await checkDatabase(),
      redis: await checkRedis(),
      memory: process.memoryUsage(),
      cpu: process.cpuUsage(),
    };

    const allHealthy = checks.database && checks.redis;

    res.status(allHealthy ? 200 : 503).json(checks);
  });
}

async function checkDatabase(): Promise<boolean> {
  try {
    await pool.query("SELECT 1");
    return true;
  } catch {
    return false;
  }
}

async function checkRedis(): Promise<boolean> {
  try {
    await redis.ping();
    return true;
  } catch {
    return false;
  }
}

// Metrics collection
import StatsD from "hot-shots";

const statsd = new StatsD({
  host: "localhost",
  port: 8125,
  prefix: "myapp.",
});

export function trackMetrics(app: Express): void {
  app.use((req, res, next) => {
    const start = Date.now();

    res.on("finish", () => {
      const duration = Date.now() - start;

      statsd.increment("requests.count", {
        method: req.method,
        route: req.route?.path,
        status: res.statusCode.toString(),
      });

      statsd.timing("requests.duration", duration, {
        method: req.method,
        route: req.route?.path,
      });
    });

    next();
  });
}
```

## CDN and Caching

### ✅ CDN Configuration

```typescript
// cdn.config.ts
export const cdnConfig = {
  provider: "cloudflare", // or 'cloudfront', 'fastly'
  domain: "cdn.example.com",
  assets: {
    images: {
      formats: ["webp", "avif", "jpeg"],
      sizes: [320, 640, 1024, 1920],
      quality: 80,
    },
    scripts: {
      minify: true,
      compress: true,
    },
    styles: {
      minify: true,
      compress: true,
    },
  },
  cacheRules: [
    {
      path: "/static/*",
      ttl: 31536000, // 1 year
      headers: {
        "Cache-Control": "public, max-age=31536000, immutable",
      },
    },
    {
      path: "/api/*",
      ttl: 0,
      headers: {
        "Cache-Control": "no-cache, no-store, must-revalidate",
      },
    },
    {
      path: "/*",
      ttl: 3600, // 1 hour
      headers: {
        "Cache-Control": "public, max-age=3600, stale-while-revalidate=86400",
      },
    },
  ],
};

// Cache headers middleware
export function setCacheHeaders(app: Express): void {
  app.use("/static", (req, res, next) => {
    res.setHeader("Cache-Control", "public, max-age=31536000, immutable");
    next();
  });

  app.use("/api", (req, res, next) => {
    res.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
    next();
  });

  app.use((req, res, next) => {
    if (req.method === "GET") {
      res.setHeader(
        "Cache-Control",
        "public, max-age=3600, stale-while-revalidate=86400",
      );
    }
    next();
  });
}
```

## Deployment Strategy

### ✅ Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

ENVIRONMENT="production"
CURRENT_ENV=$(kubectl get service app-service -o jsonpath='{.spec.selector.version}')
NEW_ENV=$([ "$CURRENT_ENV" = "blue" ] && echo "green" || echo "blue")

echo "🚀 Starting Blue-Green Deployment"
echo "Current: $CURRENT_ENV"
echo "New: $NEW_ENV"

# 1. Deploy new version
echo "\n📦 Deploying $NEW_ENV environment..."
kubectl apply -f k8s/deployment-$NEW_ENV.yaml

# 2. Wait for deployment to be ready
echo "\n⏳ Waiting for pods to be ready..."
kubectl wait --for=condition=available --timeout=300s \
  deployment/app-$NEW_ENV

# 3. Run smoke tests
echo "\n🔥 Running smoke tests on $NEW_ENV..."
SMOKE_TEST_URL="http://app-$NEW_ENV:3000"
npm run smoke-test -- --url=$SMOKE_TEST_URL

if [ $? -ne 0 ]; then
  echo "\n❌ Smoke tests failed! Aborting deployment."
  kubectl delete -f k8s/deployment-$NEW_ENV.yaml
  exit 1
fi

# 4. Switch traffic
echo "\n🔄 Switching traffic to $NEW_ENV..."
kubectl patch service app-service -p \
  "{\"spec\":{\"selector\":{\"version\":\"$NEW_ENV\"}}}"

# 5. Monitor for errors
echo "\n👀 Monitoring for 5 minutes..."
sleep 300

ERROR_RATE=$(curl -s "http://monitoring/api/error-rate?env=$NEW_ENV")
if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
  echo "\n❌ High error rate detected! Rolling back..."
  kubectl patch service app-service -p \
    "{\"spec\":{\"selector\":{\"version\":\"$CURRENT_ENV\"}}}"
  exit 1
fi

# 6. Scale down old environment
echo "\n✅ Deployment successful! Scaling down $CURRENT_ENV..."
kubectl scale deployment app-$CURRENT_ENV --replicas=1

echo "\n🎉 Blue-Green deployment complete!"
```

### ✅ Canary Deployment

```yaml
# canary-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000

---
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
        - name: app
          image: myapp:v1.0.0
          ports:
            - containerPort: 3000

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
        - name: app
          image: myapp:v1.1.0
          ports:
            - containerPort: 3000
```

## Post-Deployment

### ✅ Verification

```typescript
// post-deploy-verify.ts
interface VerificationCheck {
  name: string;
  check: () => Promise<boolean>;
  critical: boolean;
}

const verificationChecks: VerificationCheck[] = [
  {
    name: "Health check passes",
    check: async () => {
      const response = await fetch("https://api.example.com/health");
      return response.ok;
    },
    critical: true,
  },
  {
    name: "Database connection working",
    check: async () => {
      // Check database
      return true;
    },
    critical: true,
  },
  {
    name: "Authentication working",
    check: async () => {
      // Test login flow
      return true;
    },
    critical: true,
  },
  {
    name: "Critical API endpoints responding",
    check: async () => {
      const endpoints = ["/api/users", "/api/posts", "/api/products"];
      const results = await Promise.all(
        endpoints.map(async (endpoint) => {
          const response = await fetch(`https://api.example.com${endpoint}`);
          return response.ok;
        }),
      );
      return results.every(Boolean);
    },
    critical: true,
  },
  {
    name: "Static assets loading",
    check: async () => {
      const response = await fetch("https://cdn.example.com/assets/app.js");
      return response.ok;
    },
    critical: false,
  },
  {
    name: "Error rate acceptable",
    check: async () => {
      // Check error monitoring
      return true;
    },
    critical: true,
  },
];

async function runPostDeploymentVerification(): Promise<boolean> {
  console.log("🔍 Running post-deployment verification...\n");

  const results: Array<{ name: string; passed: boolean; critical: boolean }> =
    [];

  for (const check of verificationChecks) {
    try {
      const passed = await check.check();
      results.push({ name: check.name, passed, critical: check.critical });
      console.log(`${passed ? "✓" : "✗"} ${check.name}`);
    } catch (error) {
      results.push({
        name: check.name,
        passed: false,
        critical: check.critical,
      });
      console.log(`✗ ${check.name}: ${error}`);
    }
  }

  const criticalFailures = results.filter((r) => !r.passed && r.critical);

  if (criticalFailures.length > 0) {
    console.log("\n❌ Critical checks failed:");
    criticalFailures.forEach((f) => console.log(`  - ${f.name}`));
    return false;
  }

  console.log("\n✅ All critical checks passed!");
  return true;
}
```

## Rollback Plan

### ✅ Automated Rollback

```bash
#!/bin/bash
# rollback.sh

set -e

echo "🔄 Starting rollback..."

# 1. Get previous version
PREVIOUS_VERSION=$(git describe --abbrev=0 --tags $(git rev-list --tags --skip=1 --max-count=1))

echo "Rolling back to: $PREVIOUS_VERSION"

# 2. Checkout previous version
git checkout $PREVIOUS_VERSION

# 3. Restore database
echo "📦 Restoring database backup..."
psql $DATABASE_URL < backups/latest.sql

# 4. Deploy previous version
echo "🚀 Deploying previous version..."
./deploy.sh

# 5. Verify rollback
echo "✓ Running verification..."
npm run post-deploy-verify

if [ $? -eq 0 ]; then
  echo "✅ Rollback successful!"
else
  echo "❌ Rollback verification failed!"
  exit 1
fi
```

## Documentation

### ✅ Deployment Documentation

````markdown
# Deployment Guide

## Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Code review approved
- [ ] Security audit complete
- [ ] Performance benchmarks met
- [ ] Documentation updated
- [ ] Changelog updated
- [ ] Version bumped
- [ ] Git tag created

## Deployment Steps

1. **Pre-deployment checks**
   ```bash
   npm run pre-deploy-check
   ```
````

2. **Build production bundle**

   ```bash
   npm run build
   ```

3. **Run database migrations**

   ```bash
   npm run migrate
   ```

4. **Deploy application**

   ```bash
   ./deploy.sh production
   ```

5. **Run smoke tests**

   ```bash
   npm run smoke-test
   ```

6. **Monitor for issues**
   - Check error rates
   - Monitor performance
   - Watch logs

## Rollback Procedure

If issues are detected:

1. **Immediate rollback**

   ```bash
   ./rollback.sh
   ```

2. **Verify rollback**

   ```bash
   npm run post-deploy-verify
   ```

3. **Investigate issues**
   - Check logs
   - Review metrics
   - Identify root cause

## Emergency Contacts

- On-call engineer: [Phone]
- DevOps team: [Slack channel]
- Database admin: [Contact]

````

## Key Takeaways

1. **Pre-Deployment**: Run comprehensive checks - tests, linting, security audit, performance benchmarks

2. **Build Process**: Optimize production build, analyze bundle size, generate source maps

3. **Environment**: Validate environment variables, type-safe configuration, secrets management

4. **Database**: Test migrations in staging, create backups, have rollback plan

5. **Security**: Configure security headers, rate limiting, HTTPS enforcement, CSP

6. **Monitoring**: Set up error tracking (Sentry), logging (Datadog), health checks, metrics

7. **CDN**: Configure caching strategy, optimize static assets, set proper cache headers

8. **Deployment**: Choose strategy (blue-green, canary, rolling), automate process, zero-downtime

9. **Verification**: Run smoke tests, check critical paths, monitor error rates

10. **Rollback**: Automated rollback script, database backups, clear procedure, practice regularly

---

**Production Deployment Checklist**:

```bash
✅ Pre-Deployment
□ All tests passing (unit, integration, E2E)
□ Code review completed
□ Security audit passed
□ Performance benchmarks met
□ Environment variables validated
□ Database migration tested
□ Backup created
□ Changelog updated
□ Version tagged

✅ Deployment
□ Build successful
□ Migrations applied
□ Smoke tests passed
□ Health checks green
□ SSL certificates valid
□ CDN configured
□ Monitoring active
□ Logs streaming

✅ Post-Deployment
□ Critical paths verified
□ Error rate acceptable
□ Performance metrics good
□ Database connections healthy
□ Authentication working
□ API endpoints responding
□ Static assets loading
□ No console errors

✅ Monitoring
□ Sentry/error tracking active
□ Log aggregation working
□ Metrics collection running
□ Alerts configured
□ On-call engineer notified
□ Status page updated

✅ Rollback Plan
□ Rollback procedure documented
□ Previous version tagged
□ Database backup available
□ Rollback script tested
□ Team notified of deployment

# Ready for production! 🚀
````
