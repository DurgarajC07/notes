# Tree Shaking and Dead Code Elimination

## Core Concepts

Tree shaking is the process of eliminating dead code (unused exports) from JavaScript bundles. It's called "tree shaking" because it shakes off the dead leaves (unused code) from the dependency tree. Modern bundlers use static analysis of ES modules to determine which exports are used and which can be safely removed, dramatically reducing bundle sizes.

## ESM vs CommonJS

### Why ESM is Required

```javascript
// ESM - Static analysis possible
import { used } from './utils'
export { used }

// CommonJS - Dynamic, can't tree shake
const utils = require('./utils')
module.exports = utils.used
```

### Module Format Comparison

```typescript
// utils.ts
export function used() {
  return 'I am used'
}

export function unused() {
  return 'I am NOT used'
}

// app.ts with ESM
import { used } from './utils'
used() // Only 'used' is bundled

// app.ts with CommonJS
const { used } = require('./utils')
used() // Both functions might be bundled
```

## sideEffects Flag

### package.json Configuration

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "sideEffects": false
}
```

### Selective sideEffects

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```

### Component Example

```typescript
// Button.tsx - No side effects
export function Button({ onClick, children }: ButtonProps) {
  return <button onClick={onClick}>{children}</button>
}

// Button.css - Has side effects
.button {
  padding: 10px 20px;
}

// Import only component, not CSS
import { Button } from './Button' // CSS won't be included if not imported
```

## Webpack Tree Shaking Config

### Production Mode

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
    minimize: true,
    sideEffects: true,
    
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            dead_code: true,
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ['console.log'],
          },
          mangle: true,
        },
      }),
    ],
  },
}
```

### Module Concatenation (Scope Hoisting)

```javascript
module.exports = {
  optimization: {
    concatenateModules: true, // Enables scope hoisting
  },
}
```

## Rollup Configuration

### Basic Tree Shaking

```javascript
// rollup.config.js
import { terser } from 'rollup-plugin-terser'

export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'esm',
  },
  plugins: [
    terser({
      compress: {
        dead_code: true,
        drop_console: true,
      },
      mangle: true,
    }),
  ],
  treeshake: {
    moduleSideEffects: false,
    propertyReadSideEffects: false,
    tryCatchDeoptimization: false,
  },
}
```

## Debugging Tree Shaking

### Webpack Bundle Analyzer

```javascript
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
    }),
  ],
}
```

### Checking Used Exports

```bash
# Webpack verbose output
webpack --mode production --display-used-exports --display-optimization-bailout

# Check what's included
npx webpack-bundle-analyzer dist/stats.json
```

## Best Practices

1. **Use ESM** exclusively for better tree shaking
2. **Mark side effects** in package.json accurately
3. **Import selectively** - named imports over namespace
4. **Avoid default exports** for better tree shaking
5. **Pure functions** - avoid global state mutations
6. **Test bundles** with bundle analyzer regularly
7. **Lodash optimization** - use lodash-es or babel-plugin-lodash
8. **CSS modules** - mark CSS imports with sideEffects
9. **Library authors** - provide ESM builds
10. **Monitor sizes** - use size-limit or bundlesize

## 10 Key Takeaways

1. **ESM required** - CommonJS can't be tree shaken
2. **Static analysis** - bundlers analyze import/export
3. **sideEffects flag** critical for aggressive tree shaking
4. **Production mode** enables tree shaking automatically
5. **Named imports** better than namespace imports
6. **Terser/UglifyJS** removes dead code after tree shaking
7. **Scope hoisting** further reduces bundle size
8. **Debug carefully** - some code is kept for side effects
9. **Library builds** must provide ESM format
10. **30-50% reduction** typical with proper tree shaking
