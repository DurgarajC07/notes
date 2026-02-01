# Prettier Setup and Configuration

## Table of Contents

- [Introduction](#introduction)
- [Installation and Setup](#installation-and-setup)
- [Configuration Options](#configuration-options)
- [ESLint Integration](#eslint-integration)
- [Editor Integration](#editor-integration)
- [Formatting Strategies](#formatting-strategies)
- [Language-Specific Configuration](#language-specific-configuration)
- [CI/CD Integration](#cicd-integration)
- [Migration to Prettier](#migration-to-prettier)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)

## Introduction

Prettier is an opinionated code formatter that enforces a consistent style by parsing your code and re-printing it with its own rules. It removes all original styling and ensures that all outputted code conforms to a consistent style.

### Why Prettier?

```javascript
// Before Prettier - Inconsistent formatting
const user = { name: "John", age: 30, email: "john@example.com" };
function getData(id) {
  return fetch(`/api/${id}`).then((res) => res.json());
}

// After Prettier - Consistent formatting
const user = {
  name: "John",
  age: 30,
  email: "john@example.com",
};

function getData(id) {
  return fetch(`/api/${id}`).then((res) => res.json());
}
```

### Benefits

- **Zero Configuration**: Works out of the box with sensible defaults
- **Editor Integration**: Format on save across all editors
- **Consistent Style**: No debates about formatting in code reviews
- **Time Saving**: Stop manually formatting code
- **Easy Adoption**: Minimal configuration required

### Prettier vs ESLint

```
┌─────────────────────────────────────────┐
│           Code Quality Tools             │
├─────────────────┬───────────────────────┤
│    ESLint       │      Prettier          │
├─────────────────┼───────────────────────┤
│ Code Quality    │ Code Formatting        │
│ Bug Detection   │ Style Consistency      │
│ Best Practices  │ Whitespace/Indentation │
│ Custom Rules    │ Opinionated            │
│ Configurable    │ Minimal Config         │
└─────────────────┴───────────────────────┘
```

## Installation and Setup

### Basic Installation

```bash
# Install Prettier
npm install --save-dev prettier

# Install ESLint integration plugins
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

### Project Structure

```
my-project/
├── .prettierrc.json           # Configuration
├── .prettierignore            # Ignore patterns
├── .editorconfig              # Editor settings
├── package.json
└── src/
```

### Initial Configuration

```json
// .prettierrc.json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

### Prettier Ignore File

```
# .prettierignore
# Dependencies
node_modules
package-lock.json
yarn.lock

# Build outputs
dist
build
.next
out

# Coverage
coverage
.nyc_output

# Generated files
*.min.js
*.min.css

# Documentation
docs/generated

# Configuration files
pnpm-lock.yaml
```

### Package.json Scripts

```json
{
  "scripts": {
    "format": "prettier --write \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\"",
    "format:check": "prettier --check \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\"",
    "format:staged": "prettier --write $(git diff --cached --name-only --diff-filter=ACMR | grep -E '\\.(js|jsx|ts|tsx|json|css|scss|md)$')"
  }
}
```

## Configuration Options

### Complete Configuration Reference

```javascript
// .prettierrc.js
module.exports = {
  // Formatting Options

  // Print width - line length that the formatter will wrap on
  printWidth: 80,

  // Tab width - spaces per indentation level
  tabWidth: 2,

  // Use tabs instead of spaces
  useTabs: false,

  // Semicolons at the end of statements
  semi: true,

  // Use single quotes instead of double quotes
  singleQuote: true,

  // Quote properties in objects
  // "as-needed" - only add quotes when required
  // "consistent" - if any property requires quotes, quote all
  // "preserve" - respect input
  quoteProps: "as-needed",

  // Use single quotes in JSX
  jsxSingleQuote: false,

  // Trailing commas where valid in ES5
  // "es5" - trailing commas where valid in ES5
  // "none" - no trailing commas
  // "all" - trailing commas wherever possible
  trailingComma: "es5",

  // Spaces between brackets in object literals
  bracketSpacing: true,

  // Put > of multi-line JSX element at end of last line
  bracketSameLine: false,

  // Arrow function parentheses
  // "always" - always include parens
  // "avoid" - omit parens when possible
  arrowParens: "always",

  // Range formatting
  rangeStart: 0,
  rangeEnd: Infinity,

  // Parser - automatically inferred
  parser: undefined,

  // File path for parser inference
  filepath: undefined,

  // Require pragma in file to format
  requirePragma: false,

  // Insert pragma at top of file
  insertPragma: false,

  // Prose wrap
  // "always" - wrap prose if it exceeds printWidth
  // "never" - do not wrap prose
  // "preserve" - wrap prose as-is
  proseWrap: "preserve",

  // HTML whitespace sensitivity
  // "css" - respect CSS display property
  // "strict" - whitespace is significant
  // "ignore" - whitespace is insignificant
  htmlWhitespaceSensitivity: "css",

  // Vue files script and style tags indentation
  vueIndentScriptAndStyle: false,

  // End of line
  // "lf" - Line Feed only (\n)
  // "crlf" - Carriage Return + Line Feed (\r\n)
  // "cr" - Carriage Return only (\r)
  // "auto" - maintain existing line endings
  endOfLine: "lf",

  // Embedded language formatting
  embeddedLanguageFormatting: "auto",

  // Single attribute per line in HTML, Vue and JSX
  singleAttributePerLine: false,
};
```

### Configuration File Formats

```javascript
// .prettierrc.js (JavaScript)
module.exports = {
  semi: true,
  singleQuote: true,
};

// .prettierrc.json (JSON)
{
  "semi": true,
  "singleQuote": true
}

// .prettierrc.yaml (YAML)
semi: true
singleQuote: true

// .prettierrc.toml (TOML)
semi = true
singleQuote = true

// package.json (embedded)
{
  "prettier": {
    "semi": true,
    "singleQuote": true
  }
}
```

### Language-Specific Overrides

```javascript
// .prettierrc.js
module.exports = {
  // Default settings
  semi: true,
  singleQuote: true,
  printWidth: 80,

  // Override settings for specific file patterns
  overrides: [
    {
      files: "*.json",
      options: {
        printWidth: 120,
        trailingComma: "none",
      },
    },
    {
      files: "*.md",
      options: {
        proseWrap: "always",
        printWidth: 80,
      },
    },
    {
      files: "*.css",
      options: {
        singleQuote: false,
      },
    },
    {
      files: ["*.yml", "*.yaml"],
      options: {
        tabWidth: 2,
      },
    },
    {
      files: "legacy/**/*.js",
      options: {
        semi: false,
        trailingComma: "none",
      },
    },
  ],
};
```

## ESLint Integration

### Why Integrate ESLint with Prettier?

```
Without Integration:
ESLint ───> Formatting Rules ────> Conflicts with Prettier
               ↓
         Code Review Noise

With Integration:
ESLint ───> Code Quality Rules ───> No Conflicts
Prettier ─> Formatting Rules ─────> Consistent Style
```

### Installation

```bash
# Install integration packages
npm install --save-dev eslint-config-prettier eslint-plugin-prettier

# Or using Yarn
yarn add --dev eslint-config-prettier eslint-plugin-prettier
```

### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    // Add Prettier config last to override formatting rules
    "plugin:prettier/recommended", // Enables eslint-plugin-prettier and displays prettier errors as ESLint errors
  ],

  plugins: ["prettier"],

  rules: {
    // Turn off rules that conflict with Prettier
    "prettier/prettier": "error",
    "arrow-body-style": "off",
    "prefer-arrow-callback": "off",
  },
};
```

### Manual Integration (Alternative)

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    "eslint:recommended",
    // ... other configs
    "prettier", // Turn off conflicting ESLint rules
  ],

  plugins: ["prettier"],

  rules: {
    "prettier/prettier": [
      "error",
      {
        // Inline Prettier options (overrides .prettierrc)
        semi: true,
        singleQuote: true,
        printWidth: 80,
      },
    ],
  },
};
```

### Resolving Conflicts

```javascript
// Check for conflicts
npx eslint-config-prettier 'src/**/*.{js,jsx,ts,tsx}'

// Example output:
// The following rules are enabled but conflict with Prettier:
// - indent
// - quotes
// - semi
```

### Combined Script

```json
{
  "scripts": {
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "format": "prettier --write \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\"",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix",
    "check": "npm run lint && npm run format:check",
    "format:check": "prettier --check \"src/**/*.{js,jsx,ts,tsx,json,css,scss,md}\""
  }
}
```

## Editor Integration

### VS Code Configuration

```json
// .vscode/settings.json
{
  // Enable format on save
  "editor.formatOnSave": true,

  // Set Prettier as default formatter
  "editor.defaultFormatter": "esbenp.prettier-vscode",

  // Enable format on paste
  "editor.formatOnPaste": true,

  // Format on save for specific languages
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "[javascriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.formatOnSave": true
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[css]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[scss]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[markdown]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },

  // Prettier-specific settings
  "prettier.requireConfig": true, // Only format if config file exists
  "prettier.useEditorConfig": true, // Respect .editorconfig

  // ESLint integration
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },

  // Additional settings
  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "files.trimTrailingWhitespace": true
}
```

### Workspace Settings

```json
// .vscode/extensions.json - Recommended extensions
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "editorconfig.editorconfig"
  ]
}
```

### EditorConfig Integration

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{js,jsx,ts,tsx,json,css,scss,md}]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[*.yml]
indent_size = 2

[Makefile]
indent_style = tab
```

### JetBrains IDEs (WebStorm, IntelliJ)

```
1. Install Prettier Plugin:
   File → Settings → Plugins → Search "Prettier" → Install

2. Configure Prettier:
   File → Settings → Languages & Frameworks → JavaScript → Prettier
   - Prettier package: {project}/node_modules/prettier
   - ✓ On 'Reformat Code' action
   - ✓ On save

3. File Watcher (Alternative):
   File → Settings → Tools → File Watchers → Add → Prettier
   - File type: JavaScript, TypeScript, JSX, TSX
   - Program: $ProjectFileDir$/node_modules/.bin/prettier
   - Arguments: --write $FilePathRelativeToProjectRoot$
   - Output paths: $FilePathRelativeToProjectRoot$
```

### Vim/Neovim

```vim
" .vimrc or init.vim
" Install vim-prettier plugin
Plug 'prettier/vim-prettier', {
  \ 'do': 'yarn install',
  \ 'for': ['javascript', 'typescript', 'css', 'json', 'markdown'] }

" Format on save
let g:prettier#autoformat = 1
let g:prettier#autoformat_require_pragma = 0

" Custom config
let g:prettier#config#semi = 'true'
let g:prettier#config#single_quote = 'true'
let g:prettier#config#trailing_comma = 'es5'
```

## Formatting Strategies

### Pre-commit Hook

```bash
# Install husky and lint-staged
npm install --save-dev husky lint-staged

# Initialize husky
npx husky-init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["prettier --write", "eslint --fix"],
    "*.{json,css,scss,md}": ["prettier --write"]
  },
  "scripts": {
    "prepare": "husky install"
  }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
```

### Format on CI

```yaml
# .github/workflows/format-check.yml
name: Format Check

on: [push, pull_request]

jobs:
  format:
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

      - name: Check formatting
        run: npm run format:check
```

### Batch Formatting

```javascript
// scripts/format-batch.js
const { execSync } = require("child_process");
const fs = require("fs");
const path = require("path");

const BATCH_SIZE = 50;
const extensions = [".js", ".jsx", ".ts", ".tsx"];

function getAllFiles(dir, fileList = []) {
  const files = fs.readdirSync(dir);

  files.forEach((file) => {
    const filePath = path.join(dir, file);
    const stat = fs.statSync(filePath);

    if (stat.isDirectory()) {
      if (!file.includes("node_modules") && !file.includes("dist")) {
        getAllFiles(filePath, fileList);
      }
    } else {
      const ext = path.extname(file);
      if (extensions.includes(ext)) {
        fileList.push(filePath);
      }
    }
  });

  return fileList;
}

function formatInBatches(files) {
  console.log(
    `📝 Formatting ${files.length} files in batches of ${BATCH_SIZE}...\n`,
  );

  for (let i = 0; i < files.length; i += BATCH_SIZE) {
    const batch = files.slice(i, i + BATCH_SIZE);
    const fileArgs = batch.join(" ");

    console.log(
      `Batch ${Math.floor(i / BATCH_SIZE) + 1}/${Math.ceil(files.length / BATCH_SIZE)}`,
    );

    try {
      execSync(`prettier --write ${fileArgs}`, {
        stdio: "inherit",
      });
    } catch (error) {
      console.error(`Error formatting batch: ${error.message}`);
    }
  }

  console.log("\n✅ Formatting complete!");
}

const files = getAllFiles("src");
formatInBatches(files);
```

### Selective Formatting

```bash
# Format only changed files
prettier --write $(git diff --name-only --diff-filter=ACMR "*.js" "*.jsx" "*.ts" "*.tsx")

# Format only staged files
prettier --write $(git diff --cached --name-only --diff-filter=ACMR "*.js" "*.jsx" "*.ts" "*.tsx")

# Format files changed in last N commits
prettier --write $(git diff HEAD~5 --name-only --diff-filter=ACMR "*.js" "*.jsx" "*.ts" "*.tsx")
```

## Language-Specific Configuration

### JavaScript/TypeScript

```javascript
// .prettierrc.js
module.exports = {
  // JavaScript/TypeScript settings
  semi: true,
  singleQuote: true,
  trailingComma: "es5",
  arrowParens: "always",

  overrides: [
    {
      files: "*.ts",
      options: {
        parser: "typescript",
      },
    },
    {
      files: "*.tsx",
      options: {
        parser: "typescript",
        jsxSingleQuote: false,
      },
    },
  ],
};
```

### React/JSX

```javascript
module.exports = {
  // JSX-specific settings
  jsxSingleQuote: false,
  bracketSameLine: false, // Previously jsxBracketSameLine

  overrides: [
    {
      files: ["*.jsx", "*.tsx"],
      options: {
        printWidth: 100,
      },
    },
  ],
};
```

### CSS/SCSS

```javascript
module.exports = {
  overrides: [
    {
      files: ["*.css", "*.scss"],
      options: {
        singleQuote: false,
        printWidth: 100,
      },
    },
  ],
};
```

### JSON

```javascript
module.exports = {
  overrides: [
    {
      files: "*.json",
      options: {
        printWidth: 120,
        trailingComma: "none",
        tabWidth: 2,
      },
    },
    {
      files: "package.json",
      options: {
        printWidth: 100,
        tabWidth: 2,
      },
    },
  ],
};
```

### Markdown

```javascript
module.exports = {
  overrides: [
    {
      files: "*.md",
      options: {
        proseWrap: "always",
        printWidth: 80,
        tabWidth: 2,
      },
    },
  ],
};
```

### YAML

```javascript
module.exports = {
  overrides: [
    {
      files: ["*.yml", "*.yaml"],
      options: {
        tabWidth: 2,
        singleQuote: false,
      },
    },
  ],
};
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/prettier.yml
name: Prettier Check

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  prettier:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run Prettier check
        run: npm run format:check

      - name: Annotate code with Prettier errors
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            const { execSync } = require('child_process');
            const output = execSync('npx prettier --check "src/**/*.{js,jsx,ts,tsx}"', {
              encoding: 'utf-8',
              stdio: 'pipe'
            }).toString();

            console.log('Files with formatting issues:');
            console.log(output);
```

### Auto-fix PR

```yaml
# .github/workflows/prettier-fix.yml
name: Auto Format

on:
  pull_request:
    branches: [main, develop]

jobs:
  format:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.head_ref }}
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Run Prettier
        run: npm run format

      - name: Commit changes
        uses: stefanzweifel/git-auto-commit-action@v4
        with:
          commit_message: "style: auto-format with Prettier"
          file_pattern: "*.js *.jsx *.ts *.tsx *.json *.css *.scss *.md"
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test

prettier:
  stage: test
  image: node:18
  cache:
    paths:
      - node_modules/
  before_script:
    - npm ci
  script:
    - npm run format:check
  only:
    - merge_requests
    - main
```

### Pre-push Hook

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Check formatting before push
npm run format:check || {
  echo "❌ Formatting check failed. Please run 'npm run format' and commit the changes."
  exit 1
}
```

## Migration to Prettier

### Step-by-Step Migration

```javascript
// Step 1: Install Prettier
// npm install --save-dev prettier

// Step 2: Create minimal config
// .prettierrc.json
{
  "semi": true,
  "singleQuote": true
}

// Step 3: Run on one directory first
// npx prettier --write "src/utils/**/*.js"

// Step 4: Review changes, adjust config if needed

// Step 5: Format entire codebase
// npx prettier --write "src/**/*.{js,jsx,ts,tsx}"

// Step 6: Commit formatted code
// git add -A
// git commit -m "chore: format codebase with Prettier"
```

### Gradual Adoption Script

```javascript
// scripts/gradual-format.js
const { execSync } = require("child_process");
const fs = require("fs");

const directories = [
  "src/utils",
  "src/components/common",
  "src/components/features",
  "src/hooks",
  "src/pages",
  "src/services",
];

directories.forEach((dir, index) => {
  console.log(`\n[${index + 1}/${directories.length}] Formatting ${dir}...`);

  try {
    execSync(`prettier --write "${dir}/**/*.{js,jsx,ts,tsx}"`, {
      stdio: "inherit",
    });

    console.log(`✅ ${dir} formatted successfully`);

    // Create commit for this directory
    execSync(`git add ${dir}`, { stdio: "inherit" });
    execSync(`git commit -m "chore: format ${dir} with Prettier"`, {
      stdio: "inherit",
    });
  } catch (error) {
    console.error(`❌ Error formatting ${dir}:`, error.message);
  }
});

console.log("\n✅ All directories formatted!");
```

### Large Codebase Strategy

```javascript
// scripts/progressive-format.js
const { execSync } = require("child_process");
const fs = require("fs");
const path = require("path");

// Phase 1: Config files
console.log("Phase 1: Formatting config files...");
execSync('prettier --write "*.{js,json}"', { stdio: "inherit" });

// Phase 2: New features (strict enforcement)
console.log("\nPhase 2: Formatting new features...");
execSync('prettier --write "src/features/**/*.{js,jsx,ts,tsx}"', {
  stdio: "inherit",
});

// Phase 3: Tests
console.log("\nPhase 3: Formatting tests...");
execSync('prettier --write "src/**/*.{test,spec}.{js,jsx,ts,tsx}"', {
  stdio: "inherit",
});

// Phase 4: Legacy (warn only)
console.log("\nPhase 4: Checking legacy code (no changes)...");
try {
  execSync('prettier --check "src/legacy/**/*.js"', { stdio: "inherit" });
} catch {
  console.log("⚠️  Legacy code has formatting issues (not auto-fixing)");
}
```

## Troubleshooting

### Common Issues and Solutions

```javascript
// Issue 1: Prettier and ESLint conflicts
// Solution: Use eslint-config-prettier
{
  "extends": [
    "eslint:recommended",
    "prettier" // Must be last
  ]
}

// Issue 2: Format on save not working
// Solution: Check VS Code settings
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "prettier.requireConfig": true // Requires .prettierrc
}

// Issue 3: Slow formatting on large files
// Solution: Add to .prettierignore
node_modules
*.min.js
dist
build

// Issue 4: Different formatting in CI vs local
// Solution: Lock Prettier version in package.json
{
  "devDependencies": {
    "prettier": "3.1.0" // Exact version
  }
}

// Issue 5: Conflicting config files
// Solution: Use one config file type
// Delete: .prettierrc, .prettierrc.json, .prettierrc.yaml
// Keep only: .prettierrc.js (for comments and logic)
```

### Debugging Prettier

```javascript
// scripts/debug-prettier.js
const prettier = require("prettier");
const fs = require("fs");

async function debugPrettier(filePath) {
  // Read file
  const source = fs.readFileSync(filePath, "utf-8");

  // Get config
  const config = await prettier.resolveConfig(filePath);
  console.log("Resolved config:", JSON.stringify(config, null, 2));

  // Get file info
  const fileInfo = await prettier.getFileInfo(filePath);
  console.log("File info:", fileInfo);

  // Check if file would be formatted
  const formatted = await prettier.format(source, {
    ...config,
    filepath: filePath,
  });

  const wouldFormat = source !== formatted;
  console.log(`Would format: ${wouldFormat}`);

  if (wouldFormat) {
    console.log("\nDifferences:");
    console.log("Lines in original:", source.split("\n").length);
    console.log("Lines in formatted:", formatted.split("\n").length);
  }
}

// Usage: node scripts/debug-prettier.js src/App.tsx
debugPrettier(process.argv[2]);
```

### Performance Optimization

```json
{
  "scripts": {
    "format": "prettier --write --cache \"src/**/*.{js,jsx,ts,tsx}\"",
    "format:check": "prettier --check --cache \"src/**/*.{js,jsx,ts,tsx}\""
  }
}
```

```bash
# Use --cache flag for faster subsequent runs
prettier --write --cache "src/**/*.js"

# Cache is stored in node_modules/.cache/prettier/.prettier-cache

# Clear cache if needed
rm -rf node_modules/.cache/prettier
```

## Best Practices

### 1. Minimal Configuration

```javascript
// ❌ Over-configured
module.exports = {
  printWidth: 80, // Default
  tabWidth: 2, // Default
  useTabs: false, // Default
  semi: true, // Default
  // ... 20 more options
};

// ✅ Only configure what differs from defaults
module.exports = {
  singleQuote: true,
  trailingComma: "es5",
};
```

### 2. Team Consistency

```javascript
// Share config across projects
// prettier-config-myteam/index.js
module.exports = {
  singleQuote: true,
  trailingComma: 'es5',
  printWidth: 100,
};

// Consumer project
// package.json
{
  "prettier": "prettier-config-myteam"
}
```

### 3. Integration with Tools

```json
{
  "scripts": {
    "lint": "eslint .",
    "format": "prettier --write .",
    "check": "npm run lint && npm run format:check",
    "fix": "npm run lint:fix && npm run format"
  }
}
```

### 4. Ignore Generated Files

```
# .prettierignore
coverage/
dist/
build/
*.min.js
*.bundle.js
public/assets/
```

### 5. Document Decisions

```javascript
// .prettierrc.js
module.exports = {
  // We use single quotes for consistency with our backend Python code
  singleQuote: true,

  // 100 characters works better with our wide monitors
  printWidth: 100,

  // ES5 trailing commas for better git diffs
  trailingComma: "es5",
};
```

## Key Takeaways

1. **Opinionated by Design**: Prettier intentionally offers minimal configuration to eliminate debates about code style. Accept its defaults and focus on writing code.

2. **ESLint Complementarity**: Use Prettier for formatting and ESLint for code quality. Install `eslint-config-prettier` to disable conflicting rules.

3. **Editor Integration**: Enable format-on-save in your editor for automatic formatting. This eliminates manual formatting entirely.

4. **Pre-commit Hooks**: Use lint-staged and Husky to format code before commits. This ensures only formatted code reaches your repository.

5. **CI/CD Validation**: Run `prettier --check` in CI to catch any unformatted code. Fail builds if formatting is inconsistent.

6. **Language Overrides**: Use overrides for language-specific settings. Different file types may need different print widths or quote styles.

7. **Minimal Config**: Only configure options that differ from Prettier's defaults. Over-configuration defeats the purpose of an opinionated formatter.

8. **Gradual Migration**: For large codebases, format incrementally by directory. Commit each batch separately for easier review and potential rollback.

9. **Cache for Performance**: Use `--cache` flag for faster formatting on large projects. Prettier caches file hashes to skip unchanged files.

10. **Team Agreement**: Establish formatter settings as team conventions early. Once agreed upon, stop debating formatting in code reviews and let Prettier handle it automatically.
