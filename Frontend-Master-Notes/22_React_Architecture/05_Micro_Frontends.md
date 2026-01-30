# Micro Frontends

## Core Concept

Micro Frontends extend microservices architecture to frontend development, allowing independent teams to build, test, and deploy features autonomously. This pattern enables large-scale applications to be split into smaller, more manageable pieces.

---

## Approaches to Micro Frontends

### 1. Build-Time Integration

```typescript
// Package.json in shell app
{
  "dependencies": {
    "@myorg/header-mfe": "^1.2.0",
    "@myorg/products-mfe": "^2.1.0",
    "@myorg/cart-mfe": "^1.5.0"
  }
}

// Shell App.tsx
import { Header } from '@myorg/header-mfe';
import { ProductList } from '@myorg/products-mfe';
import { Cart } from '@myorg/cart-mfe';

function App() {
  return (
    <div>
      <Header />
      <ProductList />
      <Cart />
    </div>
  );
}

// Pros: Simple, type-safe, optimized bundles
// Cons: Requires redeployment of shell for updates
```

### 2. Runtime Integration with Module Federation

```javascript
// Webpack 5 Module Federation
// products-mfe/webpack.config.js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "products",
      filename: "remoteEntry.js",
      exposes: {
        "./ProductList": "./src/components/ProductList",
        "./ProductDetail": "./src/components/ProductDetail",
      },
      shared: {
        react: { singleton: true, requiredVersion: "^18.0.0" },
        "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
      },
    }),
  ],
};

// shell-app/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "shell",
      remotes: {
        products: "products@http://localhost:3001/remoteEntry.js",
        cart: "cart@http://localhost:3002/remoteEntry.js",
      },
      shared: {
        react: { singleton: true },
        "react-dom": { singleton: true },
      },
    }),
  ],
};

// shell-app/src/App.tsx
import React, { lazy, Suspense } from "react";

const ProductList = lazy(() => import("products/ProductList"));
const Cart = lazy(() => import("cart/Cart"));

function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading products...</div>}>
        <ProductList />
      </Suspense>

      <Suspense fallback={<div>Loading cart...</div>}>
        <Cart />
      </Suspense>
    </div>
  );
}
```

---

## Module Federation Deep Dive

### Advanced Configuration

```typescript
// shared-config.js - Reusable federation config
export const sharedDependencies = {
  react: {
    singleton: true,
    requiredVersion: "^18.0.0",
    strictVersion: true,
  },
  "react-dom": {
    singleton: true,
    requiredVersion: "^18.0.0",
  },
  "react-router-dom": {
    singleton: true,
    requiredVersion: "^6.0.0",
  },
  "@tanstack/react-query": {
    singleton: true,
    requiredVersion: "^4.0.0",
  },
};

// products-mfe/webpack.config.js
const { sharedDependencies } = require("./shared-config");

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "products",
      filename: "remoteEntry.js",
      exposes: {
        "./ProductList": "./src/ProductList",
        "./ProductDetail": "./src/ProductDetail",
        "./ProductProvider": "./src/providers/ProductProvider",
      },
      shared: sharedDependencies,
    }),
  ],
};

// Dynamic remote loading
// shell-app/src/utils/loadRemote.ts
export async function loadRemote<T>(
  remoteName: string,
  modulePath: string,
): Promise<T> {
  try {
    // @ts-ignore
    const container = await import(remoteName);
    await container.init(__webpack_share_scopes__.default);
    const factory = await container.get(modulePath);
    return factory();
  } catch (error) {
    console.error(`Failed to load ${remoteName}/${modulePath}`, error);
    throw error;
  }
}

// Usage
const ProductList = await loadRemote<React.ComponentType>(
  "products",
  "./ProductList",
);
```

---

## Communication Between Micro Frontends

### Event Bus Pattern

```typescript
// shared/eventBus.ts
type EventCallback<T = any> = (data: T) => void;

class EventBus {
  private events: Map<string, Set<EventCallback>> = new Map();

  on<T>(event: string, callback: EventCallback<T>) {
    if (!this.events.has(event)) {
      this.events.set(event, new Set());
    }
    this.events.get(event)!.add(callback);

    // Return unsubscribe function
    return () => {
      this.events.get(event)?.delete(callback);
    };
  }

  emit<T>(event: string, data: T) {
    this.events.get(event)?.forEach(callback => callback(data));
  }

  off(event: string, callback?: EventCallback) {
    if (callback) {
      this.events.get(event)?.delete(callback);
    } else {
      this.events.delete(event);
    }
  }
}

export const eventBus = new EventBus();

// products-mfe - Emit event
import { eventBus } from '@shared/eventBus';

function ProductCard({ product }: { product: Product }) {
  const addToCart = () => {
    eventBus.emit('cart:add', {
      productId: product.id,
      quantity: 1
    });
  };

  return <button onClick={addToCart}>Add to Cart</button>;
}

// cart-mfe - Listen for event
import { eventBus } from '@shared/eventBus';

function Cart() {
  const [items, setItems] = useState<CartItem[]>([]);

  useEffect(() => {
    const unsubscribe = eventBus.on('cart:add', (data) => {
      setItems(prev => [...prev, data]);
    });

    return unsubscribe;
  }, []);

  return <div>Cart items: {items.length}</div>;
}
```

### Shared State with Zustand

```typescript
// shared/store/cartStore.ts
import create from 'zustand';

interface CartState {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartState>((set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item]
  })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter(item => item.id !== id)
  })),
  clearCart: () => set({ items: [] })
}));

// products-mfe
import { useCartStore } from '@shared/store/cartStore';

function ProductCard({ product }: { product: Product }) {
  const addItem = useCartStore(state => state.addItem);

  return (
    <button onClick={() => addItem({ id: product.id, quantity: 1 })}>
      Add to Cart
    </button>
  );
}

// cart-mfe
import { useCartStore } from '@shared/store/cartStore';

function CartSummary() {
  const items = useCartStore(state => state.items);
  return <div>Total items: {items.length}</div>;
}
```

---

## Routing in Micro Frontends

### Shell Routing

```typescript
// shell-app/src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { lazy, Suspense } from 'react';

const ProductsApp = lazy(() => import('products/App'));
const CartApp = lazy(() => import('cart/App'));
const CheckoutApp = lazy(() => import('checkout/App'));

function App() {
  return (
    <BrowserRouter>
      <Header />
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/products/*" element={<ProductsApp />} />
          <Route path="/cart/*" element={<CartApp />} />
          <Route path="/checkout/*" element={<CheckoutApp />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// products-mfe/src/App.tsx
import { Routes, Route } from 'react-router-dom';

function ProductsApp() {
  return (
    <Routes>
      <Route index element={<ProductList />} />
      <Route path=":id" element={<ProductDetail />} />
      <Route path="category/:category" element={<CategoryPage />} />
    </Routes>
  );
}
```

---

## Monorepo Structure

### Nx Monorepo

```
my-app/
├── apps/
│   ├── shell/
│   │   ├── src/
│   │   ├── webpack.config.js
│   │   └── project.json
│   ├── products-mfe/
│   │   ├── src/
│   │   ├── webpack.config.js
│   │   └── project.json
│   └── cart-mfe/
│       ├── src/
│       ├── webpack.config.js
│       └── project.json
├── libs/
│   ├── shared-ui/
│   │   ├── src/
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   └── index.ts
│   │   └── project.json
│   ├── shared-types/
│   │   └── src/
│   │       ├── product.types.ts
│   │       ├── user.types.ts
│   │       └── index.ts
│   └── shared-utils/
│       └── src/
│           ├── formatters.ts
│           ├── validators.ts
│           └── index.ts
├── nx.json
└── package.json
```

```json
// apps/products-mfe/project.json
{
  "name": "products-mfe",
  "targets": {
    "build": {
      "executor": "@nrwl/webpack:webpack",
      "options": {
        "webpackConfig": "apps/products-mfe/webpack.config.js"
      }
    },
    "serve": {
      "executor": "@nrwl/webpack:dev-server",
      "options": {
        "port": 3001
      }
    }
  }
}
```

---

## Deployment Strategies

### Independent Deployments

```yaml
# GitHub Actions - products-mfe
name: Deploy Products MFE

on:
  push:
    branches: [main]
    paths:
      - "apps/products-mfe/**"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build
        run: |
          npm ci
          npm run build:products-mfe

      - name: Deploy to S3
        run: |
          aws s3 sync dist/products-mfe s3://mfe-products-${{ env.ENVIRONMENT }}

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_DIST_ID }}
```

### Version Management

```typescript
// shell-app/remoteConfig.ts
export const remoteConfig = {
  products: {
    url: process.env.PRODUCTS_MFE_URL || "http://localhost:3001/remoteEntry.js",
    version: "1.2.0",
  },
  cart: {
    url: process.env.CART_MFE_URL || "http://localhost:3002/remoteEntry.js",
    version: "2.3.1",
  },
};

// Dynamic remote loading with version check
async function loadRemoteWithVersion(config: RemoteConfig) {
  const remote = await loadRemote(config.url);

  if (remote.version !== config.version) {
    console.warn(
      `Version mismatch: expected ${config.version}, got ${remote.version}`,
    );
  }

  return remote;
}
```

---

## Styling Isolation

### CSS Modules

```typescript
// products-mfe/ProductCard.module.css
.card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
}

// products-mfe/ProductCard.tsx
import styles from './ProductCard.module.css';

export function ProductCard({ product }: { product: Product }) {
  return (
    <div className={styles.card}>
      {product.name}
    </div>
  );
}
```

### Shadow DOM

```typescript
// Web Component with Shadow DOM
class ProductCard extends HTMLElement {
  connectedCallback() {
    const shadow = this.attachShadow({ mode: "open" });

    const style = document.createElement("style");
    style.textContent = `
      .card {
        border: 1px solid #ddd;
        padding: 1rem;
      }
    `;

    const div = document.createElement("div");
    div.className = "card";
    div.textContent = this.getAttribute("product-name") || "";

    shadow.appendChild(style);
    shadow.appendChild(div);
  }
}

customElements.define("product-card", ProductCard);
```

---

## Error Handling and Fallbacks

### Graceful Degradation

```typescript
// shell-app/RemoteFallback.tsx
interface RemoteFallbackProps {
  remoteName: string;
  error: Error;
}

function RemoteFallback({ remoteName, error }: RemoteFallbackProps) {
  return (
    <div className="remote-error">
      <h3>Unable to load {remoteName}</h3>
      <p>Please try again later</p>
      <button onClick={() => window.location.reload()}>
        Reload Page
      </button>
    </div>
  );
}

// Safe remote component loader
function SafeRemoteComponent({
  remoteName,
  moduleName,
  fallback
}: {
  remoteName: string;
  moduleName: string;
  fallback: React.ReactNode;
}) {
  const [Component, setComponent] = useState<React.ComponentType | null>(null);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    loadRemote(remoteName, moduleName)
      .then(setComponent)
      .catch(setError);
  }, [remoteName, moduleName]);

  if (error) {
    return <RemoteFallback remoteName={remoteName} error={error} />;
  }

  if (!Component) {
    return <>{fallback}</>;
  }

  return <Component />;
}
```

---

## Testing Micro Frontends

### Integration Testing

```typescript
// test/integration/micro-frontends.test.tsx
import { render, screen, waitFor } from '@testing-library/react';

// Mock remote module
jest.mock('products/ProductList', () => ({
  default: () => <div>Mocked Product List</div>
}));

test('shell app loads products micro frontend', async () => {
  render(<App />);

  await waitFor(() => {
    expect(screen.getByText('Mocked Product List')).toBeInTheDocument();
  });
});

// E2E testing across micro frontends
describe('Cart integration', () => {
  it('adds product from products MFE to cart MFE', async () => {
    await page.goto('http://localhost:3000/products');
    await page.click('[data-testid="add-to-cart-btn"]');

    await page.goto('http://localhost:3000/cart');
    const cartItems = await page.$$('[data-testid="cart-item"]');

    expect(cartItems).toHaveLength(1);
  });
});
```

---

## Performance Optimization

### Code Splitting and Lazy Loading

```typescript
// Lazy load remote modules
const ProductList = lazy(() =>
  import("products/ProductList").catch(
    () => import("./fallbacks/ProductListFallback"),
  ),
);

// Preload critical remotes
function preloadRemotes() {
  const link = document.createElement("link");
  link.rel = "preload";
  link.as = "script";
  link.href = "http://products-mfe.com/remoteEntry.js";
  document.head.appendChild(link);
}

// Preload on mount
useEffect(() => {
  preloadRemotes();
}, []);
```

---

## Key Takeaways

1. **Module Federation** enables runtime integration of micro frontends
2. **Independent deployment** allows teams to ship features autonomously
3. **Shared dependencies** must be singletons (React, Router, etc.)
4. **Communication** via event bus or shared state management
5. **Monorepo structure** with Nx or Turborepo for code sharing
6. **Version management** crucial for remote module compatibility
7. **Error boundaries** and fallbacks for graceful degradation
8. **CSS isolation** with CSS Modules or Shadow DOM
9. **Testing** requires mocking remotes and E2E cross-MFE tests
10. **Performance** considerations: lazy loading, preloading, code splitting
