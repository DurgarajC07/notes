# 📦 Webpack Deep Dive

> Webpack is the most powerful module bundler for JavaScript applications. Understanding its internals, optimization techniques, and configuration is essential for building production-ready applications.

---

## 📖 1. Concept Explanation

### What is Webpack?

**Webpack** is a static module bundler that:

1. **Resolves dependencies** (import/require)
2. **Transforms code** (loaders: Babel, TypeScript, Sass)
3. **Bundles into static assets** (JS, CSS, images)
4. **Optimizes** (code splitting, tree shaking, minification)

```javascript
// Input (multiple files)
import React from "react";
import "./styles.css";
import logo from "./logo.png";

// Output (single bundle.js)
// All dependencies bundled, optimized, minified
```

---

## 🧠 2. Why It Matters

### Real-World Impact

**Without bundling:**

- 100+ HTTP requests (slow)
- No module system (global scope pollution)
- No tree shaking (large bundle)
- No code splitting (everything upfront)

**With Webpack:**

- 3-5 optimized bundles
- Module system (clean dependencies)
- Tree shaking (dead code removal)
- Code splitting (lazy loading)

**Production example:**

```
Before: 150 files, 5.2MB, 45s load time
After:  3 chunks, 800KB, 2.1s load time (74% faster!)
```

---

## ⚙️ 3. Core Concepts

### 1. Entry

**Starting point for dependency graph:**

```javascript
// webpack.config.js
module.exports = {
  entry: "./src/index.js", // Single entry
};

// Multiple entries (multi-page app)
module.exports = {
  entry: {
    home: "./src/pages/home.js",
    about: "./src/pages/about.js",
    contact: "./src/pages/contact.js",
  },
};
```

---

### 2. Output

**Where to emit bundles:**

```javascript
const path = require("path");

module.exports = {
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].[contenthash].js", // Cache busting
    clean: true, // Clean dist folder before build
  },
};
```

**Filename patterns:**

- `[name]` - Entry name (home, about)
- `[contenthash]` - Content hash for caching
- `[chunkhash]` - Chunk-specific hash
- `[id]` - Chunk ID

---

### 3. Loaders

**Transform files before bundling:**

```javascript
module.exports = {
  module: {
    rules: [
      // Babel (JavaScript)
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: "babel-loader",
          options: {
            presets: ["@babel/preset-env", "@babel/preset-react"],
          },
        },
      },

      // TypeScript
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },

      // CSS
      {
        test: /\.css$/,
        use: ["style-loader", "css-loader"], // Right to left!
      },

      // Sass
      {
        test: /\.scss$/,
        use: [
          "style-loader", // 3. Inject CSS into DOM
          "css-loader", // 2. Convert CSS to JS
          "sass-loader", // 1. Compile Sass to CSS
        ],
      },

      // Images
      {
        test: /\.(png|jpg|gif|svg)$/,
        type: "asset/resource",
        generator: {
          filename: "images/[hash][ext]",
        },
      },

      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: "asset/resource",
        generator: {
          filename: "fonts/[hash][ext]",
        },
      },
    ],
  },
};
```

---

### 4. Plugins

**Extend Webpack functionality:**

```javascript
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
const { BundleAnalyzerPlugin } = require("webpack-bundle-analyzer");
const CompressionPlugin = require("compression-webpack-plugin");

module.exports = {
  plugins: [
    // Generate HTML
    new HtmlWebpackPlugin({
      template: "./src/index.html",
      minify: {
        collapseWhitespace: true,
        removeComments: true,
      },
    }),

    // Extract CSS to separate file
    new MiniCssExtractPlugin({
      filename: "[name].[contenthash].css",
    }),

    // Analyze bundle size
    new BundleAnalyzerPlugin({
      analyzerMode: "static",
      openAnalyzer: false,
    }),

    // Gzip compression
    new CompressionPlugin({
      algorithm: "gzip",
      test: /\.(js|css|html|svg)$/,
      threshold: 10240, // Only assets > 10KB
      minRatio: 0.8,
    }),
  ],
};
```

---

## ✅ 4. Optimization Techniques

### 1. Code Splitting

**Split code into multiple bundles:**

```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        // Vendor (node_modules)
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10,
        },

        // React & React DOM (heavy libraries)
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: "react",
          priority: 20,
        },

        // Common code (used in 2+ chunks)
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },

    // Runtime chunk (Webpack runtime)
    runtimeChunk: "single",
  },
};
```

**Result:**

```
vendors.abc123.js    (React, libraries)
react.def456.js      (React framework)
common.ghi789.js     (Shared code)
runtime.jkl012.js    (Webpack runtime)
home.mno345.js       (Home page code)
about.pqr678.js      (About page code)
```

---

### 2. Tree Shaking

**Remove unused code:**

```javascript
// library.js
export function used() {
  console.log("Used");
}

export function unused() {
  console.log("Never imported");
}

// main.js
import { used } from "./library.js";
used();

// webpack.config.js
module.exports = {
  mode: "production", // Enables tree shaking
  optimization: {
    usedExports: true, // Mark unused exports
    minimize: true, // Remove them
  },
};

// Result: 'unused' function removed from bundle
```

**Requirements:**

- ES6 modules (import/export)
- Production mode
- Side-effect-free code

**Package.json sideEffects:**

```json
{
  "sideEffects": false
}

// Or specify files with side effects
{
  "sideEffects": ["*.css", "*.scss"]
}
```

---

### 3. Minification

```javascript
const TerserPlugin = require("terser-webpack-plugin");
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      // JavaScript
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true, // Remove console.log
            drop_debugger: true,
          },
          format: {
            comments: false, // Remove comments
          },
        },
        extractComments: false,
      }),

      // CSS
      new CssMinimizerPlugin(),
    ],
  },
};
```

---

### 4. Caching

**Long-term caching with contenthash:**

```javascript
module.exports = {
  output: {
    filename: "[name].[contenthash].js",
    chunkFilename: "[name].[contenthash].js",
  },

  optimization: {
    moduleIds: "deterministic", // Stable module IDs
    runtimeChunk: "single", // Separate runtime chunk

    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          chunks: "all",
        },
      },
    },
  },
};
```

**Result:**

- Content changes → Hash changes → Browser fetches new file
- Content unchanged → Hash same → Browser uses cached file

---

## 🚀 5. Performance Optimization

### 1. Build Performance

```javascript
module.exports = {
  // Cache builds
  cache: {
    type: "filesystem",
    cacheDirectory: path.resolve(__dirname, ".webpack_cache"),
  },

  // Parallel builds
  module: {
    rules: [
      {
        test: /\.js$/,
        use: [
          {
            loader: "thread-loader",
            options: {
              workers: 4, // CPU cores
            },
          },
          "babel-loader",
        ],
      },
    ],
  },

  // Faster source maps
  devtool: "eval-cheap-module-source-map", // Development
  // devtool: 'source-map', // Production

  // Exclude from transpilation
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: "babel-loader",
      },
    ],
  },
};
```

---

### 2. Runtime Performance

```javascript
// Lazy loading (dynamic import)
import React, { lazy, Suspense } from "react";

const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyComponent />
    </Suspense>
  );
}

// Prefetching
import(/* webpackPrefetch: true */ "./future-module.js");

// Preloading
import(/* webpackPreload: true */ "./critical-module.js");
```

---

## 🧪 6. Complete Production Config

```javascript
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
const TerserPlugin = require("terser-webpack-plugin");
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
const { BundleAnalyzerPlugin } = require("webpack-bundle-analyzer");
const CompressionPlugin = require("compression-webpack-plugin");

module.exports = {
  mode: "production",

  entry: "./src/index.tsx",

  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "js/[name].[contenthash:8].js",
    chunkFilename: "js/[name].[contenthash:8].chunk.js",
    assetModuleFilename: "assets/[hash][ext]",
    clean: true,
    publicPath: "/",
  },

  resolve: {
    extensions: [".tsx", ".ts", ".js", ".jsx"],
    alias: {
      "@": path.resolve(__dirname, "src"),
      "@components": path.resolve(__dirname, "src/components"),
      "@utils": path.resolve(__dirname, "src/utils"),
    },
  },

  module: {
    rules: [
      // TypeScript
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },

      // JavaScript
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: "babel-loader",
          options: {
            presets: [
              "@babel/preset-env",
              "@babel/preset-react",
              "@babel/preset-typescript",
            ],
            plugins: ["@babel/plugin-transform-runtime"],
          },
        },
      },

      // CSS
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          "css-loader",
          {
            loader: "postcss-loader",
            options: {
              postcssOptions: {
                plugins: ["autoprefixer", "cssnano"],
              },
            },
          },
        ],
      },

      // Sass
      {
        test: /\.scss$/,
        use: [
          MiniCssExtractPlugin.loader,
          "css-loader",
          "postcss-loader",
          "sass-loader",
        ],
      },

      // Images
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/,
        type: "asset",
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // Inline images < 10KB
          },
        },
        generator: {
          filename: "images/[hash][ext]",
        },
      },

      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: "asset/resource",
        generator: {
          filename: "fonts/[hash][ext]",
        },
      },
    ],
  },

  plugins: [
    new HtmlWebpackPlugin({
      template: "./public/index.html",
      minify: {
        collapseWhitespace: true,
        removeComments: true,
        removeRedundantAttributes: true,
        useShortDoctype: true,
        removeEmptyAttributes: true,
        removeStyleLinkTypeAttributes: true,
        keepClosingSlash: true,
        minifyJS: true,
        minifyCSS: true,
        minifyURLs: true,
      },
    }),

    new MiniCssExtractPlugin({
      filename: "css/[name].[contenthash:8].css",
      chunkFilename: "css/[name].[contenthash:8].chunk.css",
    }),

    new CompressionPlugin({
      algorithm: "gzip",
      test: /\.(js|css|html|svg)$/,
      threshold: 10240,
      minRatio: 0.8,
    }),

    new BundleAnalyzerPlugin({
      analyzerMode: "static",
      openAnalyzer: false,
      reportFilename: "bundle-report.html",
    }),
  ],

  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ["console.log"],
          },
          format: {
            comments: false,
          },
        },
        extractComments: false,
      }),
      new CssMinimizerPlugin(),
    ],

    splitChunks: {
      chunks: "all",
      maxInitialRequests: 25,
      minSize: 20000,
      cacheGroups: {
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: "react",
          priority: 20,
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },

    runtimeChunk: {
      name: "runtime",
    },

    moduleIds: "deterministic",
  },

  cache: {
    type: "filesystem",
    cacheDirectory: path.resolve(__dirname, ".webpack_cache"),
  },

  performance: {
    hints: "warning",
    maxEntrypointSize: 512000,
    maxAssetSize: 512000,
  },
};
```

---

## 🏗️ 7. Real-World Use Cases

### Large E-commerce Site

**Before optimization:**

- Bundle: 3.2MB
- Initial load: 8.5s
- Lighthouse: 42

**After optimization:**

```javascript
// Code splitting by route
const Home = lazy(() => import('./pages/Home'));
const Product = lazy(() => import('./pages/Product'));
const Cart = lazy(() => import('./pages/Cart'));
const Checkout = lazy(() => import('./pages/Checkout'));

// Vendor splitting
splitChunks: {
  cacheGroups: {
    react: { /* React + React DOM */ },
    ui: { /* UI library (MUI) */ },
    vendor: { /* Other node_modules */ }
  }
}

// Image optimization
{
  test: /\.(png|jpg)$/,
  use: [
    {
      loader: 'image-webpack-loader',
      options: {
        mozjpeg: { quality: 80 },
        pngquant: { quality: [0.65, 0.90] }
      }
    }
  ]
}
```

**Results:**

- Bundle: 850KB (73% smaller)
- Initial load: 2.1s (75% faster)
- Lighthouse: 94

---

## ❓ 8. Interview Questions

### Q1: Explain Webpack's bundling process

**Answer:**

**1. Entry:** Start from entry point(s)

```javascript
entry: "./src/index.js";
```

**2. Dependency Graph:** Resolve all imports/requires

```
index.js
  ├─ App.js
  │   ├─ Header.js
  │   └─ Footer.js
  ├─ utils.js
  └─ styles.css
```

**3. Loaders:** Transform files

```javascript
.tsx → ts-loader → .js
.scss → sass-loader → css-loader → style-loader
.png → file-loader → URL
```

**4. Plugins:** Perform optimizations

- Minify (TerserPlugin)
- Extract CSS (MiniCssExtractPlugin)
- Generate HTML (HtmlWebpackPlugin)

**5. Output:** Emit bundled files

```
dist/
  main.abc123.js
  vendors.def456.js
  styles.ghi789.css
  index.html
```

---

### Q2: How does code splitting work?

**Answer:**

**Dynamic imports create separate chunks:**

```javascript
// Before (single bundle)
import HeavyComponent from "./HeavyComponent";

// After (code splitting)
const HeavyComponent = lazy(() => import("./HeavyComponent"));

// Webpack creates separate chunk:
// main.js (200KB)
// HeavyComponent.chunk.js (500KB) - loaded on demand
```

**Configuration:**

```javascript
optimization: {
  splitChunks: {
    chunks: 'all', // Split all chunks (sync + async)
    cacheGroups: {
      vendor: {
        test: /node_modules/,
        name: 'vendors' // All node_modules → vendors.js
      }
    }
  }
}
```

**Benefits:**

- Smaller initial bundle
- Faster initial load
- Lazy load on demand
- Better caching (vendor code rarely changes)

---

### Q3: What's tree shaking and how does it work?

**Answer:**

**Tree shaking removes unused code:**

```javascript
// library.js
export function used() {
  /* ... */
}
export function unused() {
  /* ... */
} // Never imported!

// main.js
import { used } from "./library";
used();

// Bundle includes only 'used', not 'unused'
```

**Requirements:**

1. **ES6 modules** (not CommonJS):

```javascript
// ✅ Tree-shakeable
import { func } from "./module";

// ❌ Not tree-shakeable
const { func } = require("./module");
```

2. **Production mode**:

```javascript
mode: "production"; // Enables UglifyJS/Terser
```

3. **Side-effect-free**:

```json
// package.json
{
  "sideEffects": false
}
```

**How it works:**

1. Mark unused exports (dead code)
2. Minifier removes them

---

## 🎯 Summary

**Webpack core:**

- Entry → Loaders → Plugins → Output
- Dependency graph resolution
- Code transformation

**Optimization:**

- ✅ Code splitting (splitChunks)
- ✅ Tree shaking (remove unused)
- ✅ Minification (Terser)
- ✅ Caching (contenthash)
- ✅ Lazy loading (dynamic imports)

**Production checklist:**

- ✅ Extract CSS to separate file
- ✅ Compress assets (gzip/brotli)
- ✅ Analyze bundle size
- ✅ Set performance budgets
- ✅ Enable source maps

Master Webpack for production-ready builds! 📦
