# Webpack Module Federation

## Core Concepts

Module Federation is a revolutionary Webpack feature enabling JavaScript applications to dynamically load code from other applications at runtime. It allows multiple separate builds to form a single application, perfect for micro-frontends and distributed teams. This pattern is crucial for large-scale enterprise applications.

## Basic Setup

### Host Configuration

```typescript
// webpack.config.js (Host App)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin')

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app1: 'app1@http://localhost:3001/remoteEntry.js',
        app2: 'app2@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
      },
    }),
  ],
}
```

### Remote Configuration

```typescript
// webpack.config.js (Remote App)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin')

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/components/Button',
        './Header': './src/components/Header',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
      },
    }),
  ],
}
```

## Sharing React Across Apps

### Singleton Pattern

```typescript
// Ensure single React instance
shared: {
  react: {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^18.0.0',
    eager: false,
  },
  'react-dom': {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^18.0.0',
    eager: false,
  },
}
```

### Version Strategies

```typescript
shared: {
  // Strict version matching
  lodash: {
    singleton: true,
    strictVersion: true,
    requiredVersion: '4.17.21',
  },
  
  // Fallback to local version
  axios: {
    singleton: false,
    requiredVersion: '^1.0.0',
  },
  
  // Load immediately
  'react-router-dom': {
    singleton: true,
    eager: true,
  },
}
```

## Runtime vs Build-Time Sharing

### Runtime Loading

```typescript
// Host app dynamically loads remote
const RemoteButton = React.lazy(() => import('app1/Button'))

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <RemoteButton onClick={() => console.log('Clicked!')} />
    </Suspense>
  )
}
```

### Build-Time Configuration

```typescript
// Type definitions for remote modules
declare module 'app1/Button' {
  const Button: React.FC<{ onClick: () => void }>
  export default Button
}

// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "app1/*": ["./src/@types/app1/*"]
    }
  }
}
```

## Versioning Strategies

### Semantic Versioning

```typescript
shared: {
  react: {
    singleton: true,
    requiredVersion: '^18.2.0',
    version: '18.2.0',
  },
}
```

### Version Resolution

```typescript
// Advanced version handling
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin')

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
          
          // Fallback behavior
          import: 'react',
          shareKey: 'react',
          shareScope: 'default',
          
          // Version prioritization
          version: false, // Use package.json version
          
          // Load eagerly
          eager: false,
        },
      },
    }),
  ],
}
```

## Host/Remote Configuration

### Bidirectional Sharing

```typescript
// App can be both host and remote
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      
      // Expose modules
      exposes: {
        './Button': './src/components/Button',
      },
      
      // Consume modules
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js',
      },
      
      shared: ['react', 'react-dom'],
    }),
  ],
}
```

### Dynamic Remote Loading

```typescript
// Load remotes dynamically
function loadComponent(scope: string, module: string) {
  return async () => {
    await __webpack_init_sharing__('default')
    
    const container = window[scope]
    await container.init(__webpack_share_scopes__.default)
    
    const factory = await container.get(module)
    const Module = factory()
    return Module
  }
}

// Usage
const RemoteButton = React.lazy(loadComponent('app1', './Button'))
```

## Deployment Strategies

### CDN Deployment

```typescript
// Production config
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app1: \pp1@\/remoteEntry.js\,
        app2: \pp2@\/remoteEntry.js\,
      },
    }),
  ],
}
```

### Version Management

```typescript
// Remote entry with versioning
const remoteUrl = process.env.NODE_ENV === 'production'
  ? 'https://cdn.example.com/app1/v1.2.3/remoteEntry.js'
  : 'http://localhost:3001/remoteEntry.js'

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      remotes: {
        app1: \pp1@\\,
      },
    }),
  ],
}
```

## Best Practices

1. **Use singleton** for React/Vue to avoid duplicate instances
2. **Version carefully** with semantic versioning
3. **Type safety** with TypeScript declarations
4. **Error boundaries** around remote components
5. **Lazy loading** for on-demand module loading
6. **Versioned URLs** for production deployments
7. **Monitor bundle** sizes with shared dependencies
8. **Test locally** before deploying to production
9. **Document APIs** for exposed modules
10. **Cache strategy** for remote entry files

## 10 Key Takeaways

1. **Module Federation** enables runtime code sharing between apps
2. **Singleton pattern** prevents duplicate React instances
3. **Version resolution** handles dependency conflicts
4. **Bidirectional** - apps can be both host and remote
5. **Dynamic loading** reduces initial bundle size
6. **Micro-frontends** become practical with Module Federation
7. **Production ready** - used by major enterprises
8. **TypeScript support** requires manual type declarations
9. **Deployment** requires proper URL management
10. **Performance** - shared deps loaded once across all apps
