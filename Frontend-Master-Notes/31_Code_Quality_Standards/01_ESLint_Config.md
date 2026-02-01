# ESLint Configuration

## Table of Contents

- [Introduction](#introduction)
- [Core Configuration](#core-configuration)
- [Essential Rules](#essential-rules)
- [TypeScript Integration](#typescript-integration)
- [Popular Plugins](#popular-plugins)
- [Custom Rules](#custom-rules)
- [React and JSX Rules](#react-and-jsx-rules)
- [Performance Considerations](#performance-considerations)
- [Migration Strategies](#migration-strategies)
- [Best Practices](#best-practices)
- [Key Takeaways](#key-takeaways)

## Introduction

ESLint is the most popular JavaScript linting tool, providing static code analysis to identify problematic patterns and enforce code quality standards. It's highly configurable and extensible through plugins and custom rules.

### Why ESLint Matters

```javascript
// Without ESLint - Problematic code passes silently
function calculateTotal(items) {
  let total = 0;
  for (var i = 0; i < items.length; i++) {
    total += items[i].price;
  }
  return total;
}

// With ESLint - Catches issues immediately
// ❌ Unexpected var, use let or const instead (no-var)
// ❌ Missing semicolon (semi)
// ❌ Missing radix parameter (radix)
```

### ESLint Architecture

```
┌─────────────────────────────────────────┐
│           Source Code                    │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Parser (Espree/Babel)            │
│    Converts code to AST                  │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Rules Engine                     │
│    Applies configured rules to AST       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         Formatter                        │
│    Outputs errors/warnings               │
└─────────────────────────────────────────┘
```

## Core Configuration

### Basic .eslintrc.js Setup

```javascript
// .eslintrc.js
module.exports = {
  // Environment defines global variables
  env: {
    browser: true,
    es2021: true,
    node: true,
    jest: true,
  },

  // Extends base configurations
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:import/recommended",
    "plugin:import/typescript",
  ],

  // Parser for TypeScript
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module",
    ecmaFeatures: {
      jsx: true,
    },
    project: "./tsconfig.json",
  },

  // Plugins provide additional rules
  plugins: [
    "react",
    "react-hooks",
    "@typescript-eslint",
    "jsx-a11y",
    "import",
    "prettier",
  ],

  // Rule configurations
  rules: {
    // Your custom rules here
  },

  // Settings for plugins
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

  // Files to ignore
  ignorePatterns: [
    "node_modules/",
    "dist/",
    "build/",
    "coverage/",
    "*.config.js",
  ],
};
```

### Advanced Multi-Environment Configuration

```javascript
// .eslintrc.js - Complex project setup
module.exports = {
  root: true, // Stop looking for parent configs

  env: {
    browser: true,
    es2021: true,
    node: true,
  },

  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
  ],

  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: 2021,
    sourceType: "module",
    project: "./tsconfig.json",
    tsconfigRootDir: __dirname,
  },

  plugins: ["@typescript-eslint"],

  // Override configurations for specific file patterns
  overrides: [
    {
      // Test files have relaxed rules
      files: ["**/*.test.ts", "**/*.spec.ts"],
      env: {
        jest: true,
      },
      rules: {
        "@typescript-eslint/no-explicit-any": "off",
        "@typescript-eslint/no-non-null-assertion": "off",
      },
    },
    {
      // Configuration files use Node.js
      files: ["*.config.js", "*.config.ts"],
      env: {
        node: true,
      },
      rules: {
        "@typescript-eslint/no-var-requires": "off",
      },
    },
    {
      // Scripts can use console
      files: ["scripts/**/*"],
      rules: {
        "no-console": "off",
      },
    },
  ],
};
```

## Essential Rules

### Code Quality Rules

```javascript
module.exports = {
  rules: {
    // Possible Errors
    "no-console": ["warn", { allow: ["warn", "error"] }],
    "no-debugger": "error",
    "no-dupe-args": "error",
    "no-dupe-keys": "error",
    "no-duplicate-case": "error",
    "no-empty": ["error", { allowEmptyCatch: false }],
    "no-ex-assign": "error",
    "no-extra-boolean-cast": "error",
    "no-func-assign": "error",
    "no-unreachable": "error",
    "valid-typeof": ["error", { requireStringLiterals: true }],

    // Best Practices
    curly: ["error", "all"],
    "default-case": "error",
    "dot-notation": ["error", { allowKeywords: true }],
    eqeqeq: ["error", "always", { null: "ignore" }],
    "no-alert": "warn",
    "no-caller": "error",
    "no-case-declarations": "error",
    "no-else-return": ["error", { allowElseIf: false }],
    "no-empty-function": [
      "error",
      {
        allow: ["arrowFunctions", "constructors"],
      },
    ],
    "no-eval": "error",
    "no-extend-native": "error",
    "no-extra-bind": "error",
    "no-fallthrough": "error",
    "no-floating-decimal": "error",
    "no-implicit-coercion": "error",
    "no-implied-eval": "error",
    "no-iterator": "error",
    "no-labels": "error",
    "no-lone-blocks": "error",
    "no-loop-func": "error",
    "no-multi-spaces": "error",
    "no-new": "error",
    "no-new-func": "error",
    "no-new-wrappers": "error",
    "no-param-reassign": ["error", { props: true }],
    "no-proto": "error",
    "no-return-assign": ["error", "always"],
    "no-return-await": "error",
    "no-self-compare": "error",
    "no-sequences": "error",
    "no-throw-literal": "error",
    "no-unused-expressions": [
      "error",
      {
        allowShortCircuit: true,
        allowTernary: true,
      },
    ],
    "no-useless-call": "error",
    "no-useless-concat": "error",
    "no-useless-return": "error",
    "prefer-promise-reject-errors": ["error", { allowEmptyReject: true }],
    radix: "error",
    "require-await": "error",
    yoda: "error",

    // Variables
    "no-delete-var": "error",
    "no-shadow": "off", // Use TypeScript version
    "no-undef": "error",
    "no-unused-vars": "off", // Use TypeScript version
    "no-use-before-define": "off", // Use TypeScript version

    // ES6
    "arrow-body-style": ["error", "as-needed"],
    "arrow-spacing": ["error", { before: true, after: true }],
    "constructor-super": "error",
    "no-class-assign": "error",
    "no-const-assign": "error",
    "no-dupe-class-members": "error",
    "no-duplicate-imports": "error",
    "no-new-symbol": "error",
    "no-this-before-super": "error",
    "no-useless-computed-key": "error",
    "no-useless-constructor": "off", // Use TypeScript version
    "no-useless-rename": "error",
    "no-var": "error",
    "object-shorthand": ["error", "always"],
    "prefer-arrow-callback": ["error", { allowNamedFunctions: false }],
    "prefer-const": ["error", { destructuring: "all" }],
    "prefer-destructuring": [
      "error",
      {
        array: true,
        object: true,
      },
      {
        enforceForRenamedProperties: false,
      },
    ],
    "prefer-rest-params": "error",
    "prefer-spread": "error",
    "prefer-template": "error",
    "rest-spread-spacing": ["error", "never"],
    "template-curly-spacing": ["error", "never"],
  },
};
```

### Style Rules

```javascript
module.exports = {
  rules: {
    // Note: Many style rules should be handled by Prettier
    // Only include rules that affect code logic or aren't handled by Prettier

    "array-bracket-spacing": ["error", "never"],
    "block-spacing": ["error", "always"],
    "brace-style": ["error", "1tbs", { allowSingleLine: true }],
    "comma-dangle": [
      "error",
      {
        arrays: "always-multiline",
        objects: "always-multiline",
        imports: "always-multiline",
        exports: "always-multiline",
        functions: "always-multiline",
      },
    ],
    "comma-spacing": ["error", { before: false, after: true }],
    "comma-style": ["error", "last"],
    "computed-property-spacing": ["error", "never"],
    "eol-last": ["error", "always"],
    "func-call-spacing": ["error", "never"],
    indent: [
      "error",
      2,
      {
        SwitchCase: 1,
        VariableDeclarator: 1,
        outerIIFEBody: 1,
        MemberExpression: 1,
        FunctionDeclaration: { parameters: 1, body: 1 },
        FunctionExpression: { parameters: 1, body: 1 },
        CallExpression: { arguments: 1 },
        ArrayExpression: 1,
        ObjectExpression: 1,
        ImportDeclaration: 1,
        flatTernaryExpressions: false,
        ignoreComments: false,
      },
    ],
    "key-spacing": ["error", { beforeColon: false, afterColon: true }],
    "keyword-spacing": ["error", { before: true, after: true }],
    "linebreak-style": ["error", "unix"],
    "max-len": [
      "error",
      {
        code: 100,
        tabWidth: 2,
        ignoreUrls: true,
        ignoreStrings: true,
        ignoreTemplateLiterals: true,
        ignoreRegExpLiterals: true,
      },
    ],
    "no-multiple-empty-lines": ["error", { max: 2, maxEOF: 0 }],
    "no-trailing-spaces": "error",
    "object-curly-spacing": ["error", "always"],
    "padded-blocks": ["error", "never"],
    quotes: [
      "error",
      "single",
      { avoidEscape: true, allowTemplateLiterals: true },
    ],
    semi: ["error", "always"],
    "semi-spacing": ["error", { before: false, after: true }],
    "space-before-blocks": "error",
    "space-before-function-paren": [
      "error",
      {
        anonymous: "always",
        named: "never",
        asyncArrow: "always",
      },
    ],
    "space-in-parens": ["error", "never"],
    "space-infix-ops": "error",
    "space-unary-ops": ["error", { words: true, nonwords: false }],
  },
};
```

## TypeScript Integration

### TypeScript-Specific Rules

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
  ],

  parser: "@typescript-eslint/parser",
  parserOptions: {
    project: "./tsconfig.json",
  },

  plugins: ["@typescript-eslint"],

  rules: {
    // TypeScript-specific rules
    "@typescript-eslint/adjacent-overload-signatures": "error",
    "@typescript-eslint/array-type": ["error", { default: "array-simple" }],
    "@typescript-eslint/await-thenable": "error",
    "@typescript-eslint/ban-ts-comment": [
      "error",
      {
        "ts-expect-error": "allow-with-description",
        "ts-ignore": true,
        "ts-nocheck": true,
        "ts-check": false,
        minimumDescriptionLength: 3,
      },
    ],
    "@typescript-eslint/ban-types": [
      "error",
      {
        types: {
          Object: {
            message: "Use object instead",
            fixWith: "object",
          },
          String: {
            message: "Use string instead",
            fixWith: "string",
          },
          Boolean: {
            message: "Use boolean instead",
            fixWith: "boolean",
          },
          Number: {
            message: "Use number instead",
            fixWith: "number",
          },
          Function: {
            message: "Use specific function type instead",
          },
        },
      },
    ],
    "@typescript-eslint/consistent-type-assertions": [
      "error",
      {
        assertionStyle: "as",
        objectLiteralTypeAssertions: "allow-as-parameter",
      },
    ],
    "@typescript-eslint/consistent-type-definitions": ["error", "interface"],
    "@typescript-eslint/consistent-type-imports": [
      "error",
      {
        prefer: "type-imports",
      },
    ],
    "@typescript-eslint/explicit-function-return-type": [
      "error",
      {
        allowExpressions: true,
        allowTypedFunctionExpressions: true,
        allowHigherOrderFunctions: true,
      },
    ],
    "@typescript-eslint/explicit-member-accessibility": [
      "error",
      {
        accessibility: "explicit",
        overrides: {
          constructors: "no-public",
        },
      },
    ],
    "@typescript-eslint/member-ordering": [
      "error",
      {
        default: [
          "static-field",
          "instance-field",
          "constructor",
          "static-method",
          "instance-method",
        ],
      },
    ],
    "@typescript-eslint/method-signature-style": ["error", "property"],
    "@typescript-eslint/naming-convention": [
      "error",
      {
        selector: "default",
        format: ["camelCase"],
      },
      {
        selector: "variable",
        format: ["camelCase", "UPPER_CASE", "PascalCase"],
      },
      {
        selector: "parameter",
        format: ["camelCase"],
        leadingUnderscore: "allow",
      },
      {
        selector: "memberLike",
        modifiers: ["private"],
        format: ["camelCase"],
        leadingUnderscore: "require",
      },
      {
        selector: "typeLike",
        format: ["PascalCase"],
      },
      {
        selector: "interface",
        format: ["PascalCase"],
        custom: {
          regex: "^I[A-Z]",
          match: false,
        },
      },
    ],
    "@typescript-eslint/no-base-to-string": "error",
    "@typescript-eslint/no-confusing-non-null-assertion": "error",
    "@typescript-eslint/no-dynamic-delete": "error",
    "@typescript-eslint/no-empty-interface": [
      "error",
      {
        allowSingleExtends: true,
      },
    ],
    "@typescript-eslint/no-explicit-any": [
      "error",
      {
        fixToUnknown: true,
        ignoreRestArgs: false,
      },
    ],
    "@typescript-eslint/no-extra-non-null-assertion": "error",
    "@typescript-eslint/no-floating-promises": [
      "error",
      {
        ignoreVoid: true,
      },
    ],
    "@typescript-eslint/no-for-in-array": "error",
    "@typescript-eslint/no-inferrable-types": [
      "error",
      {
        ignoreParameters: false,
        ignoreProperties: false,
      },
    ],
    "@typescript-eslint/no-invalid-void-type": "error",
    "@typescript-eslint/no-misused-new": "error",
    "@typescript-eslint/no-misused-promises": [
      "error",
      {
        checksVoidReturn: true,
      },
    ],
    "@typescript-eslint/no-namespace": "error",
    "@typescript-eslint/no-non-null-asserted-optional-chain": "error",
    "@typescript-eslint/no-non-null-assertion": "warn",
    "@typescript-eslint/no-redundant-type-constituents": "error",
    "@typescript-eslint/no-require-imports": "error",
    "@typescript-eslint/no-this-alias": "error",
    "@typescript-eslint/no-unnecessary-boolean-literal-compare": "error",
    "@typescript-eslint/no-unnecessary-condition": "error",
    "@typescript-eslint/no-unnecessary-type-arguments": "error",
    "@typescript-eslint/no-unnecessary-type-assertion": "error",
    "@typescript-eslint/no-unsafe-argument": "error",
    "@typescript-eslint/no-unsafe-assignment": "error",
    "@typescript-eslint/no-unsafe-call": "error",
    "@typescript-eslint/no-unsafe-member-access": "error",
    "@typescript-eslint/no-unsafe-return": "error",
    "@typescript-eslint/no-unused-vars": [
      "error",
      {
        argsIgnorePattern: "^_",
        varsIgnorePattern: "^_",
      },
    ],
    "@typescript-eslint/no-var-requires": "error",
    "@typescript-eslint/prefer-as-const": "error",
    "@typescript-eslint/prefer-for-of": "error",
    "@typescript-eslint/prefer-function-type": "error",
    "@typescript-eslint/prefer-includes": "error",
    "@typescript-eslint/prefer-nullish-coalescing": "error",
    "@typescript-eslint/prefer-optional-chain": "error",
    "@typescript-eslint/prefer-readonly": "error",
    "@typescript-eslint/prefer-reduce-type-parameter": "error",
    "@typescript-eslint/prefer-string-starts-ends-with": "error",
    "@typescript-eslint/promise-function-async": "error",
    "@typescript-eslint/require-array-sort-compare": [
      "error",
      {
        ignoreStringArrays: false,
      },
    ],
    "@typescript-eslint/restrict-plus-operands": "error",
    "@typescript-eslint/restrict-template-expressions": [
      "error",
      {
        allowNumber: true,
        allowBoolean: false,
        allowAny: false,
      },
    ],
    "@typescript-eslint/strict-boolean-expressions": [
      "error",
      {
        allowString: false,
        allowNumber: false,
        allowNullableObject: false,
      },
    ],
    "@typescript-eslint/switch-exhaustiveness-check": "error",
    "@typescript-eslint/triple-slash-reference": "error",
    "@typescript-eslint/unbound-method": [
      "error",
      {
        ignoreStatic: true,
      },
    ],
    "@typescript-eslint/unified-signatures": "error",

    // Extension rules (replace base ESLint rules)
    "default-param-last": "off",
    "@typescript-eslint/default-param-last": "error",

    "dot-notation": "off",
    "@typescript-eslint/dot-notation": "error",

    "no-array-constructor": "off",
    "@typescript-eslint/no-array-constructor": "error",

    "no-dupe-class-members": "off",
    "@typescript-eslint/no-dupe-class-members": "error",

    "no-empty-function": "off",
    "@typescript-eslint/no-empty-function": "error",

    "no-implied-eval": "off",
    "@typescript-eslint/no-implied-eval": "error",

    "no-loss-of-precision": "off",
    "@typescript-eslint/no-loss-of-precision": "error",

    "no-redeclare": "off",
    "@typescript-eslint/no-redeclare": "error",

    "no-shadow": "off",
    "@typescript-eslint/no-shadow": "error",

    "no-throw-literal": "off",
    "@typescript-eslint/no-throw-literal": "error",

    "no-unused-expressions": "off",
    "@typescript-eslint/no-unused-expressions": "error",

    "no-unused-vars": "off",
    // Already configured above

    "no-use-before-define": "off",
    "@typescript-eslint/no-use-before-define": [
      "error",
      {
        functions: false,
        classes: true,
        variables: true,
      },
    ],

    "no-useless-constructor": "off",
    "@typescript-eslint/no-useless-constructor": "error",

    "require-await": "off",
    "@typescript-eslint/require-await": "error",

    "no-return-await": "off",
    "@typescript-eslint/return-await": ["error", "in-try-catch"],
  },
};
```

## Popular Plugins

### React and JSX

```javascript
module.exports = {
  extends: [
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
  ],

  plugins: ["react", "react-hooks", "jsx-a11y"],

  settings: {
    react: {
      version: "detect",
    },
  },

  rules: {
    // React rules
    "react/boolean-prop-naming": [
      "error",
      {
        rule: "^(is|has)[A-Z]([A-Za-z0-9]?)+",
      },
    ],
    "react/button-has-type": "error",
    "react/default-props-match-prop-types": "error",
    "react/destructuring-assignment": ["error", "always"],
    "react/display-name": "off",
    "react/function-component-definition": [
      "error",
      {
        namedComponents: "arrow-function",
        unnamedComponents: "arrow-function",
      },
    ],
    "react/hook-use-state": "error",
    "react/jsx-boolean-value": ["error", "never"],
    "react/jsx-curly-brace-presence": [
      "error",
      {
        props: "never",
        children: "never",
      },
    ],
    "react/jsx-fragments": ["error", "syntax"],
    "react/jsx-no-constructed-context-values": "error",
    "react/jsx-no-leaked-render": [
      "error",
      {
        validStrategies: ["ternary"],
      },
    ],
    "react/jsx-no-useless-fragment": "error",
    "react/jsx-pascal-case": "error",
    "react/no-array-index-key": "warn",
    "react/no-danger": "warn",
    "react/no-unstable-nested-components": "error",
    "react/prop-types": "off", // Use TypeScript instead
    "react/react-in-jsx-scope": "off", // Not needed in React 17+
    "react/self-closing-comp": "error",

    // React Hooks rules
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",

    // Accessibility rules
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-has-content": "error",
    "jsx-a11y/anchor-is-valid": "error",
    "jsx-a11y/aria-activedescendant-has-tabindex": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-proptypes": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/click-events-have-key-events": "error",
    "jsx-a11y/heading-has-content": "error",
    "jsx-a11y/html-has-lang": "error",
    "jsx-a11y/img-redundant-alt": "error",
    "jsx-a11y/interactive-supports-focus": "error",
    "jsx-a11y/label-has-associated-control": "error",
    "jsx-a11y/media-has-caption": "warn",
    "jsx-a11y/mouse-events-have-key-events": "error",
    "jsx-a11y/no-access-key": "error",
    "jsx-a11y/no-autofocus": "warn",
    "jsx-a11y/no-distracting-elements": "error",
    "jsx-a11y/no-interactive-element-to-noninteractive-role": "error",
    "jsx-a11y/no-noninteractive-element-interactions": "error",
    "jsx-a11y/no-noninteractive-tabindex": "error",
    "jsx-a11y/no-redundant-roles": "error",
    "jsx-a11y/no-static-element-interactions": "error",
    "jsx-a11y/role-has-required-aria-props": "error",
    "jsx-a11y/role-supports-aria-props": "error",
    "jsx-a11y/scope": "error",
    "jsx-a11y/tabindex-no-positive": "error",
  },
};
```

### Import Plugin

```javascript
module.exports = {
  extends: ["plugin:import/recommended", "plugin:import/typescript"],

  plugins: ["import"],

  settings: {
    "import/resolver": {
      typescript: {
        alwaysTryTypes: true,
        project: "./tsconfig.json",
      },
      node: {
        extensions: [".js", ".jsx", ".ts", ".tsx"],
      },
    },
    "import/parsers": {
      "@typescript-eslint/parser": [".ts", ".tsx"],
    },
  },

  rules: {
    "import/no-unresolved": "error",
    "import/named": "error",
    "import/default": "error",
    "import/namespace": "error",
    "import/no-absolute-path": "error",
    "import/no-dynamic-require": "error",
    "import/no-self-import": "error",
    "import/no-cycle": ["error", { maxDepth: 1 }],
    "import/no-useless-path-segments": "error",
    "import/export": "error",
    "import/no-named-as-default": "error",
    "import/no-named-as-default-member": "error",
    "import/no-deprecated": "warn",
    "import/no-mutable-exports": "error",
    "import/no-unused-modules": [
      "error",
      {
        unusedExports: true,
      },
    ],
    "import/first": "error",
    "import/exports-last": "error",
    "import/no-duplicates": "error",
    "import/no-namespace": "error",
    "import/extensions": [
      "error",
      "ignorePackages",
      {
        js: "never",
        jsx: "never",
        ts: "never",
        tsx: "never",
      },
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
          "object",
          "type",
        ],
        pathGroups: [
          {
            pattern: "react",
            group: "external",
            position: "before",
          },
          {
            pattern: "@/**",
            group: "internal",
          },
        ],
        pathGroupsExcludedImportTypes: ["react"],
        "newlines-between": "always",
        alphabetize: {
          order: "asc",
          caseInsensitive: true,
        },
      },
    ],
    "import/newline-after-import": "error",
    "import/prefer-default-export": "off",
    "import/no-default-export": "error",
  },
};
```

## Custom Rules

### Creating a Custom Rule

```javascript
// eslint-rules/no-console-log.js
module.exports = {
  meta: {
    type: "suggestion",
    docs: {
      description: "Disallow console.log statements",
      category: "Best Practices",
      recommended: true,
    },
    fixable: "code",
    schema: [],
    messages: {
      unexpectedConsoleLog: "Unexpected console.log statement.",
    },
  },

  create(context) {
    return {
      CallExpression(node) {
        // Check if it's console.log
        if (
          node.callee.type === "MemberExpression" &&
          node.callee.object.name === "console" &&
          node.callee.property.name === "log"
        ) {
          context.report({
            node,
            messageId: "unexpectedConsoleLog",
            fix(fixer) {
              // Auto-fix: replace console.log with console.debug
              return fixer.replaceText(node.callee.property, "debug");
            },
          });
        }
      },
    };
  },
};
```

### Advanced Custom Rule with Options

```javascript
// eslint-rules/require-data-testid.js
module.exports = {
  meta: {
    type: "problem",
    docs: {
      description: "Require data-testid on interactive elements",
      category: "Testing",
      recommended: true,
    },
    fixable: null,
    schema: [
      {
        type: "object",
        properties: {
          elements: {
            type: "array",
            items: { type: "string" },
            default: ["button", "input", "select", "textarea"],
          },
          attributeName: {
            type: "string",
            default: "data-testid",
          },
        },
        additionalProperties: false,
      },
    ],
    messages: {
      missingTestId: "{{element}} element missing {{attributeName}} attribute",
    },
  },

  create(context) {
    const options = context.options[0] || {};
    const elements = options.elements || [
      "button",
      "input",
      "select",
      "textarea",
    ];
    const attributeName = options.attributeName || "data-testid";

    return {
      JSXOpeningElement(node) {
        const elementName = node.name.name;

        if (elements.includes(elementName)) {
          const hasTestId = node.attributes.some((attr) => {
            return (
              attr.type === "JSXAttribute" && attr.name.name === attributeName
            );
          });

          if (!hasTestId) {
            context.report({
              node,
              messageId: "missingTestId",
              data: {
                element: elementName,
                attributeName,
              },
            });
          }
        }
      },
    };
  },
};
```

### Loading Custom Rules

```javascript
// .eslintrc.js
const path = require("path");

module.exports = {
  plugins: ["local"],
  rules: {
    "local/no-console-log": "error",
    "local/require-data-testid": [
      "error",
      {
        elements: ["button", "input", "a"],
        attributeName: "data-testid",
      },
    ],
  },

  // Register local plugin
  settings: {
    "local-rules": {
      "no-console-log": require("./eslint-rules/no-console-log"),
      "require-data-testid": require("./eslint-rules/require-data-testid"),
    },
  },
};
```

### Custom Rule Testing

```javascript
// eslint-rules/__tests__/no-console-log.test.js
const { RuleTester } = require("eslint");
const rule = require("../no-console-log");

const ruleTester = new RuleTester({
  parserOptions: {
    ecmaVersion: 2021,
    sourceType: "module",
  },
});

ruleTester.run("no-console-log", rule, {
  valid: [
    'console.error("error")',
    'console.warn("warning")',
    'console.debug("debug")',
    'logger.log("message")',
  ],

  invalid: [
    {
      code: 'console.log("test")',
      errors: [{ messageId: "unexpectedConsoleLog" }],
      output: 'console.debug("test")',
    },
    {
      code: 'console.log("value:", value)',
      errors: [{ messageId: "unexpectedConsoleLog" }],
      output: 'console.debug("value:", value)',
    },
  ],
});
```

## React and JSX Rules

### Comprehensive React Configuration

```javascript
// .eslintrc.react.js
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:@typescript-eslint/recommended",
  ],

  plugins: ["react", "react-hooks", "jsx-a11y", "@typescript-eslint"],

  parserOptions: {
    ecmaFeatures: {
      jsx: true,
    },
  },

  settings: {
    react: {
      version: "detect",
    },
  },

  rules: {
    // JSX-specific rules
    "react/jsx-key": [
      "error",
      {
        checkFragmentShorthand: true,
        checkKeyMustBeforeSpread: true,
      },
    ],
    "react/jsx-no-duplicate-props": "error",
    "react/jsx-no-undef": "error",
    "react/jsx-uses-react": "off", // Not needed in React 17+
    "react/jsx-uses-vars": "error",

    // Component rules
    "react/no-deprecated": "error",
    "react/no-direct-mutation-state": "error",
    "react/no-find-dom-node": "error",
    "react/no-is-mounted": "error",
    "react/no-render-return-value": "error",
    "react/no-string-refs": "error",
    "react/no-unescaped-entities": "error",
    "react/no-unknown-property": "error",
    "react/require-render-return": "error",

    // Performance rules
    "react/no-children-prop": "error",
    "react/no-danger-with-children": "error",
    "react/no-multi-comp": ["error", { ignoreStateless: true }],
    "react/no-redundant-should-component-update": "error",
    "react/no-this-in-sfc": "error",
    "react/no-typos": "error",
    "react/no-unused-prop-types": "error",
    "react/no-unused-state": "error",
    "react/prefer-es6-class": ["error", "always"],
    "react/prefer-stateless-function": "error",
    "react/style-prop-object": "error",
    "react/void-dom-elements-no-children": "error",
  },
};
```

## Performance Considerations

### Caching Strategy

```javascript
// .eslintrc.js
module.exports = {
  // Enable caching to speed up subsequent runs
  cache: true,
  cacheLocation: ".eslintcache",
  cacheStrategy: "content", // or 'metadata'

  // ... rest of config
};
```

### Performance Optimization Script

```javascript
// scripts/lint-performance.js
const { ESLint } = require("eslint");
const fs = require("fs");

async function measureLintPerformance() {
  const eslint = new ESLint({
    cache: true,
    cacheLocation: ".eslintcache",
  });

  console.log("🔍 Starting ESLint performance test...\n");

  // First run (cold cache)
  console.time("Cold cache");
  await eslint.lintFiles(["src/**/*.{js,jsx,ts,tsx}"]);
  console.timeEnd("Cold cache");

  // Second run (warm cache)
  console.time("Warm cache");
  await eslint.lintFiles(["src/**/*.{js,jsx,ts,tsx}"]);
  console.timeEnd("Warm cache");

  // Cache size
  const stats = fs.statSync(".eslintcache");
  console.log(`\nCache size: ${(stats.size / 1024).toFixed(2)} KB`);
}

measureLintPerformance().catch(console.error);
```

### Selective Linting

```json
// package.json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "lint:changed": "eslint $(git diff --name-only --diff-filter=ACMR HEAD | grep -E '\\.(js|jsx|ts|tsx)$' | xargs)",
    "lint:staged": "lint-staged",
    "lint:ci": "eslint . --max-warnings=0 --format=json --output-file=eslint-report.json"
  }
}
```

## Migration Strategies

### Gradual Rule Adoption

```javascript
// .eslintrc.migration.js
module.exports = {
  extends: ["eslint:recommended"],

  rules: {
    // Phase 1: Critical errors only
    "no-undef": "error",
    "no-unused-vars": "error",
    "no-console": "warn",

    // Phase 2: Best practices (start as warnings)
    eqeqeq: "warn",
    curly: "warn",
    "no-var": "warn",

    // Phase 3: Style rules (disabled initially)
    // 'indent': 'off',
    // 'quotes': 'off',
    // 'semi': 'off',
  },

  // Override for new files
  overrides: [
    {
      files: ["src/new-feature/**/*"],
      rules: {
        // Enforce all rules on new code
        eqeqeq: "error",
        curly: "error",
        "no-var": "error",
        indent: ["error", 2],
        quotes: ["error", "single"],
        semi: ["error", "always"],
      },
    },
  ],
};
```

### Legacy Code Handling

```javascript
// .eslintrc.js
module.exports = {
  extends: ["eslint:recommended"],

  overrides: [
    {
      // Relaxed rules for legacy code
      files: ["src/legacy/**/*"],
      rules: {
        "@typescript-eslint/no-explicit-any": "off",
        "@typescript-eslint/ban-ts-comment": "off",
        "no-console": "off",
      },
    },
    {
      // Strict rules for new code
      files: ["src/features/**/*", "src/components/**/*"],
      rules: {
        "@typescript-eslint/no-explicit-any": "error",
        "@typescript-eslint/explicit-function-return-type": "error",
        "no-console": "error",
      },
    },
  ],
};
```

## Best Practices

### 1. Start with Recommended Configs

```javascript
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
  ],
  // Then customize gradually
};
```

### 2. Use Shareable Configs

```javascript
// eslint-config-mycompany/index.js
module.exports = {
  extends: ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
  rules: {
    // Company-wide standards
  },
};

// Consumer's .eslintrc.js
module.exports = {
  extends: ["mycompany"],
};
```

### 3. Document Custom Rules

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    // ❌ Bad: No explanation
    "max-len": ["error", 100],

    // ✅ Good: Documented
    // Enforce 100 character line length for readability on code reviews
    // Exception: Allow longer strings and URLs
    "max-len": [
      "error",
      {
        code: 100,
        ignoreUrls: true,
        ignoreStrings: true,
      },
    ],
  },
};
```

### 4. Use Error Levels Appropriately

```javascript
module.exports = {
  rules: {
    // 'off' (0): Rule is disabled
    "no-console": "off",

    // 'warn' (1): Warning (doesn't fail CI)
    "no-console": "warn",

    // 'error' (2): Error (fails CI)
    "no-console": "error",
  },
};
```

### 5. Integration with CI/CD

```yaml
# .github/workflows/lint.yml
name: Lint
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint -- --max-warnings=0
```

## Key Takeaways

1. **Start Simple, Iterate**: Begin with recommended configs, then customize based on team needs. Don't overwhelm with rules initially.

2. **TypeScript Integration**: Use `@typescript-eslint/parser` and TypeScript-specific rules for type-safe linting. Disable base ESLint rules that conflict.

3. **Plugin Ecosystem**: Leverage plugins like `eslint-plugin-react`, `eslint-plugin-import`, and `eslint-plugin-jsx-a11y` for framework-specific rules.

4. **Custom Rules**: Create custom rules for project-specific patterns. Test them thoroughly and document their purpose.

5. **Performance Matters**: Enable caching, use selective linting for large codebases, and optimize rule configurations.

6. **Overrides for Flexibility**: Use `overrides` to apply different rule sets to different parts of your codebase (tests, config files, etc.).

7. **Auto-fixable Rules**: Prefer rules with auto-fix capabilities. Enable auto-fix in pre-commit hooks for consistent formatting.

8. **Documentation**: Document why specific rules are enabled/disabled. Make configuration maintainable for team members.

9. **CI Integration**: Run ESLint in CI with `--max-warnings=0` to prevent quality regressions. Cache dependencies for faster builds.

10. **Migration Strategy**: When adopting ESLint in legacy codebases, start with critical errors only, then gradually increase strictness. Use overrides to enforce strict rules on new code while being lenient on legacy code.
