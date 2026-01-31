# Vite - Modern Frontend Build Tool

## Core Concepts

Vite (French for "fast") is a next-generation frontend build tool that provides instant server start, lightning-fast Hot Module Replacement (HMR), and optimized production builds. Created by Evan You (Vue.js creator), Vite leverages native ES modules and modern JavaScript features to deliver an exceptional developer experience. Unlike traditional bundlers, Vite serves source code over native ESM during development, eliminating the need to bundle code upfront.

## ESM-Based Dev Server

### Native ES Modules

Vite transforms files on-demand as the browser requests them:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    strictPort: true,
    host: true,
  },
  resolve: {
    alias: {
      '@': '/src',
      '@components': '/src/components',
      '@utils': '/src/utils',
    },
  },
})
```

### Dependency Pre-Bundling

Vite pre-bundles dependencies using esbuild:

```typescript
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: ['react', 'react-dom', 'axios'],
    exclude: ['@my-local-package'],
    esbuildOptions: {
      target: 'es2020',
    },
  },
  build: {
    commonjsOptions: {
      include: [/node_modules/],
    },
  },
})
```

## Hot Module Replacement (HMR)

### React Fast Refresh

```typescript
// @vitejs/plugin-react automatically handles HMR
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    react({
      // Enable Fast Refresh
      fastRefresh: true,
      // Customize Fast Refresh options
      babel: {
        parserOpts: {
          plugins: ['decorators-legacy'],
        },
      },
    }),
  ],
})
```

### Custom HMR API

```typescript
// src/utils/api.ts
export const API_URL = 'https://api.example.com'

if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    console.log('API module updated:', newModule)
  })
  
  // Dispose callback
  import.meta.hot.dispose((data) => {
    data.apiUrl = API_URL
  })
  
  // Self-accept
  import.meta.hot.accept()
  
  // Invalidate and reload
  import.meta.hot.invalidate()
}
```

### HMR for CSS

```typescript
// Vite automatically handles CSS HMR
import './styles.css'
import './App.module.css'

// CSS Modules with HMR
import styles from './Button.module.css'

function Button() {
  return <button className={styles.button}>Click me</button>
}
```

## Performance Optimization

### Code Splitting

```typescript
// Automatic code splitting with dynamic imports
import { lazy, Suspense } from 'react'

const Dashboard = lazy(() => import('./pages/Dashboard'))
const Profile = lazy(() => import('./pages/Profile'))

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  )
}

// Manual chunk splitting
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'router': ['react-router-dom'],
          'ui-lib': ['@mui/material', '@emotion/react'],
        },
      },
    },
  },
})
```

### Asset Optimization

```typescript
export default defineConfig({
  build: {
    // Asset inlining threshold
    assetsInlineLimit: 4096, // 4kb
    
    // CSS code splitting
    cssCodeSplit: true,
    
    // Source maps
    sourcemap: true,
    
    // Minification
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    
    // Chunk size warnings
    chunkSizeWarningLimit: 500,
  },
  
  // Image optimization
  assetsInclude: ['**/*.png', '**/*.jpg', '**/*.svg'],
})
```

## Plugin Ecosystem

### Official Plugins

```typescript
import react from '@vitejs/plugin-react'
import vue from '@vitejs/plugin-vue'
import legacy from '@vitejs/plugin-legacy'

export default defineConfig({
  plugins: [
    react(),
    
    // Legacy browser support
    legacy({
      targets: ['defaults', 'not IE 11'],
      additionalLegacyPolyfills: ['regenerator-runtime/runtime'],
    }),
  ],
})
```

### Popular Community Plugins

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { VitePWA } from 'vite-plugin-pwa'
import compression from 'vite-plugin-compression'
import checker from 'vite-plugin-checker'
import svgr from 'vite-plugin-svgr'

export default defineConfig({
  plugins: [
    react(),
    
    // PWA support
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#ffffff',
      },
    }),
    
    // Gzip compression
    compression({
      algorithm: 'gzip',
      ext: '.gz',
    }),
    
    // TypeScript checker
    checker({
      typescript: true,
      eslint: {
        lintCommand: 'eslint \"./src/**/*.{ts,tsx}\"',
      },
    }),
    
    // SVG as React components
    svgr(),
  ],
})
```

### Custom Plugin Development

```typescript
import type { Plugin } from 'vite'

// Custom markdown plugin
function markdownPlugin(): Plugin {
  return {
    name: 'vite-plugin-markdown',
    
    transform(code, id) {
      if (id.endsWith('.md')) {
        // Transform markdown to JS
        const html = \<div>\</div>\
        return {
          code: \export default \\,
          map: null,
        }
      }
    },
    
    handleHotUpdate({ file, server }) {
      if (file.endsWith('.md')) {
        server.ws.send({
          type: 'full-reload',
          path: '*',
        })
      }
    },
  }
}

export default defineConfig({
  plugins: [markdownPlugin()],
})
```

## SSR Support

### SSR Configuration

```typescript
// vite.config.ts
export default defineConfig({
  ssr: {
    noExternal: ['react', 'react-dom'],
    external: ['@prisma/client'],
  },
  build: {
    ssr: true,
    rollupOptions: {
      input: {
        server: './src/entry-server.tsx',
      },
    },
  },
})
```

### Server Entry Point

```typescript
// src/entry-server.tsx
import ReactDOMServer from 'react-dom/server'
import { StaticRouter } from 'react-router-dom/server'
import App from './App'

export function render(url: string) {
  const html = ReactDOMServer.renderToString(
    <StaticRouter location={url}>
      <App />
    </StaticRouter>
  )
  
  return { html }
}
```

### Client Entry Point

```typescript
// src/entry-client.tsx
import ReactDOM from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App'

ReactDOM.hydrateRoot(
  document.getElementById('root')!,
  <BrowserRouter>
    <App />
  </BrowserRouter>
)
```

## Production Build with Rollup

### Build Configuration

```typescript
export default defineConfig({
  build: {
    target: 'es2020',
    outDir: 'dist',
    assetsDir: 'assets',
    
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            return 'vendor'
          }
        },
        chunkFileNames: 'js/[name]-[hash].js',
        entryFileNames: 'js/[name]-[hash].js',
        assetFileNames: '[ext]/[name]-[hash].[ext]',
      },
    },
    
    reportCompressedSize: true,
    chunkSizeWarningLimit: 1000,
  },
})
```

## Migration from Webpack/CRA

### Webpack to Vite Migration

```typescript
// Before (webpack.config.js)
module.exports = {
  entry: './src/index.js',
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  module: {
    rules: [
      {
        test: /\\.jsx?\$/,
        use: 'babel-loader',
      },
    ],
  },
}

// After (vite.config.ts)
export default defineConfig({
  resolve: {
    alias: {
      '@': '/src',
    },
  },
  plugins: [react()],
})
```

### Environment Variables

```typescript
// Before (.env files with REACT_APP_ prefix)
REACT_APP_API_URL=https://api.example.com

// After (.env files with VITE_ prefix)
VITE_API_URL=https://api.example.com

// Usage
const apiUrl = import.meta.env.VITE_API_URL
const isDev = import.meta.env.DEV
const isProd = import.meta.env.PROD
```

## Real-World Production Config

```typescript
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  
  return {
    plugins: [react()],
    
    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
      },
    },
    
    server: {
      port: 3000,
      proxy: {
        '/api': {
          target: env.VITE_API_URL,
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\\/api/, ''),
        },
      },
    },
    
    build: {
      sourcemap: command === 'serve',
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ['react', 'react-dom'],
            router: ['react-router-dom'],
          },
        },
      },
    },
    
    optimizeDeps: {
      include: ['react', 'react-dom'],
    },
  }
})
```

## Best Practices

1. **Use native ESM** during development for instant server start
2. **Enable Fast Refresh** for React/Vue for better DX
3. **Configure dependency pre-bundling** for faster cold starts
4. **Leverage code splitting** with dynamic imports
5. **Optimize assets** with proper inlining thresholds
6. **Use plugins** for common tasks (PWA, compression, etc.)
7. **Configure SSR** properly for SEO-critical apps
8. **Monitor bundle size** with build reports
9. **Proxy API calls** in development to avoid CORS
10. **Type-safe config** with TypeScript

## 10 Key Takeaways

1. **Vite uses native ESM** in development for instant server start
2. **esbuild pre-bundles** dependencies for fast cold starts
3. **HMR is instant** with module-level updates, no full reload
4. **Rollup for production** provides optimal bundle sizes
5. **Plugin ecosystem** is rich with official and community plugins
6. **SSR support** is built-in and straightforward
7. **Migration from Webpack/CRA** is relatively simple
8. **Bundle sizes** are typically 40-50% smaller than Webpack
9. **Dev server starts** in <1 second vs 10-30s for Webpack
10. **Production builds** are fast with parallel processing via esbuild
