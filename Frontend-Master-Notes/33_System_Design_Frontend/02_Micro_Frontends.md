# Micro-Frontends Architecture - Complete Guide

## Core Concepts

Micro-frontends extend microservices principles to frontend development, allowing teams to build, deploy, and scale frontend applications independently. Each micro-frontend is a self-contained feature owned by a single team.

### Key Benefits

- **Team Autonomy**: Independent development and deployment
- **Technology Flexibility**: Different frameworks per micro-frontend
- **Incremental Upgrades**: Update parts without rewriting everything
- **Faster Development**: Parallel work without conflicts
- **Isolated Failures**: One micro-frontend crash doesn't affect others

## 1. Module Federation (Webpack 5)

Module Federation allows JavaScript applications to dynamically load code from another application at runtime.

### Host Configuration

```typescript
// webpack.config.js (Host Application)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        product: 'product@http://localhost:3001/remoteEntry.js',
        cart: 'cart@http://localhost:3002/remoteEntry.js',
        user: 'user@http://localhost:3003/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};

// TypeScript Module Declaration
declare module 'product/ProductList' {
  const ProductList: React.ComponentType<ProductListProps>;
  export default ProductList;
}

declare module 'cart/Cart' {
  const Cart: React.ComponentType<CartProps>;
  export default Cart;
}

declare module 'user/UserProfile' {
  const UserProfile: React.ComponentType<UserProfileProps>;
  export default UserProfile;
}

// Host App Component
import React, { Suspense, lazy } from 'react';

const ProductList = lazy(() => import('product/ProductList'));
const Cart = lazy(() => import('cart/Cart'));
const UserProfile = lazy(() => import('user/UserProfile'));

function App() {
  return (
    <div className="app">
      <header>
        <Suspense fallback={<div>Loading user...</div>}>
          <UserProfile userId="123" />
        </Suspense>
      </header>

      <main>
        <Suspense fallback={<div>Loading products...</div>}>
          <ProductList />
        </Suspense>

        <Suspense fallback={<div>Loading cart...</div>}>
          <Cart />
        </Suspense>
      </main>
    </div>
  );
}

export default App;

interface ProductListProps {
  onProductSelect?: (productId: string) => void;
}

interface CartProps {
  userId?: string;
}

interface UserProfileProps {
  userId: string;
}
```

### Remote Configuration

```typescript
// webpack.config.js (Remote Product Application)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'product',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/components/ProductList',
        './ProductDetail': './src/components/ProductDetail',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};

// Product Micro-Frontend Component
import React, { useState, useEffect } from 'react';

export interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  imageUrl: string;
}

export interface ProductListProps {
  onProductSelect?: (productId: string) => void;
}

const ProductList: React.FC<ProductListProps> = ({ onProductSelect }) => {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchProducts();
  }, []);

  const fetchProducts = async () => {
    try {
      const response = await fetch('http://localhost:3001/api/products');
      const data = await response.json();
      setProducts(data);
    } catch (error) {
      console.error('Failed to fetch products:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="product-list">
      {products.map(product => (
        <div
          key={product.id}
          className="product-card"
          onClick={() => onProductSelect?.(product.id)}
        >
          <img src={product.imageUrl} alt={product.name} />
          <h3>{product.name}</h3>
          <p>{product.description}</p>
          <span className="price">${product.price.toFixed(2)}</span>
        </div>
      ))}
    </div>
  );
};

export default ProductList;
```

### Dynamic Module Federation

```typescript
// Dynamic Remote Loading
class ModuleFederationLoader {
  private loadedRemotes: Map<string, any> = new Map();

  async loadRemote(
    remoteName: string,
    remoteUrl: string,
    moduleName: string,
  ): Promise<any> {
    const cacheKey = `${remoteName}/${moduleName}`;

    // Check cache
    if (this.loadedRemotes.has(cacheKey)) {
      return this.loadedRemotes.get(cacheKey);
    }

    // Load remote container
    await this.loadScript(remoteName, remoteUrl);

    // Get container
    const container = (window as any)[remoteName];

    // Initialize sharing scope
    await container.init(__webpack_share_scopes__.default);

    // Get module factory
    const factory = await container.get(moduleName);
    const module = factory();

    // Cache module
    this.loadedRemotes.set(cacheKey, module);

    return module;
  }

  private loadScript(remoteName: string, remoteUrl: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const existingScript = document.getElementById(remoteName);

      if (existingScript) {
        resolve();
        return;
      }

      const script = document.createElement("script");
      script.id = remoteName;
      script.src = remoteUrl;
      script.type = "text/javascript";
      script.async = true;

      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load ${remoteUrl}`));

      document.head.appendChild(script);
    });
  }

  clearCache(): void {
    this.loadedRemotes.clear();
  }
}

// Usage
const loader = new ModuleFederationLoader();

async function loadProductModule() {
  const module = await loader.loadRemote(
    "product",
    "http://localhost:3001/remoteEntry.js",
    "./ProductList",
  );

  return module.default;
}
```

## 2. iframe Isolation

Complete isolation using iframes with postMessage communication.

```typescript
// Parent Application
class IframeMicroFrontend {
  private iframe: HTMLIFrameElement;
  private messageHandlers: Map<string, (data: any) => void> = new Map();

  constructor(private config: IframeMicroFrontendConfig) {
    this.iframe = document.createElement("iframe");
    this.setupIframe();
    this.setupMessageListener();
  }

  private setupIframe(): void {
    this.iframe.src = this.config.url;
    this.iframe.id = this.config.id;

    // Security attributes
    this.iframe.sandbox.add(
      "allow-scripts",
      "allow-same-origin",
      "allow-forms",
      "allow-popups",
    );

    // Styling
    Object.assign(this.iframe.style, {
      border: "none",
      width: "100%",
      height: "100%",
    });
  }

  private setupMessageListener(): void {
    window.addEventListener("message", (event) => {
      // Verify origin
      if (event.origin !== new URL(this.config.url).origin) {
        return;
      }

      const { type, data } = event.data;
      const handler = this.messageHandlers.get(type);

      if (handler) {
        handler(data);
      }
    });
  }

  mount(containerId: string): void {
    const container = document.getElementById(containerId);

    if (!container) {
      throw new Error(`Container ${containerId} not found`);
    }

    container.appendChild(this.iframe);
  }

  unmount(): void {
    this.iframe.remove();
  }

  sendMessage(type: string, data: any): void {
    this.iframe.contentWindow?.postMessage(
      { type, data },
      new URL(this.config.url).origin,
    );
  }

  on(messageType: string, handler: (data: any) => void): void {
    this.messageHandlers.set(messageType, handler);
  }
}

interface IframeMicroFrontendConfig {
  id: string;
  url: string;
}

// Usage in Parent App
const productMicroFrontend = new IframeMicroFrontend({
  id: "product-mfe",
  url: "http://localhost:3001",
});

// Mount
productMicroFrontend.mount("product-container");

// Listen to messages from child
productMicroFrontend.on("product:selected", (data) => {
  console.log("Product selected:", data);
});

// Send message to child
productMicroFrontend.sendMessage("user:login", {
  userId: "123",
  name: "John Doe",
});

// Child Application (inside iframe)
class IframeChild {
  private messageHandlers: Map<string, (data: any) => void> = new Map();

  constructor() {
    this.setupMessageListener();
  }

  private setupMessageListener(): void {
    window.addEventListener("message", (event) => {
      // Verify parent origin
      if (event.origin !== "http://localhost:3000") {
        return;
      }

      const { type, data } = event.data;
      const handler = this.messageHandlers.get(type);

      if (handler) {
        handler(data);
      }
    });
  }

  sendToParent(type: string, data: any): void {
    window.parent.postMessage({ type, data }, "http://localhost:3000");
  }

  on(messageType: string, handler: (data: any) => void): void {
    this.messageHandlers.set(messageType, handler);
  }
}

// Usage in child
const child = new IframeChild();

child.on("user:login", (user) => {
  console.log("User logged in:", user);
});

// Send message to parent
child.sendToParent("product:selected", {
  productId: "123",
  name: "Product Name",
});
```

## 3. Web Components

Standard-based approach using Custom Elements.

```typescript
// Product Card Web Component
class ProductCard extends HTMLElement {
  private shadowRoot: ShadowRoot;
  private product: Product | null = null;

  constructor() {
    super();
    this.shadowRoot = this.attachShadow({ mode: 'open' });
  }

  static get observedAttributes() {
    return ['product-id', 'name', 'price', 'image-url'];
  }

  connectedCallback() {
    this.render();
  }

  attributeChangedCallback(
    name: string,
    oldValue: string,
    newValue: string
  ) {
    if (oldValue !== newValue) {
      this.render();
    }
  }

  private render() {
    const productId = this.getAttribute('product-id') || '';
    const name = this.getAttribute('name') || '';
    const price = this.getAttribute('price') || '0';
    const imageUrl = this.getAttribute('image-url') || '';

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          border: 1px solid #ddd;
          border-radius: 8px;
          padding: 16px;
          margin: 8px;
          cursor: pointer;
          transition: transform 0.2s;
        }

        :host(:hover) {
          transform: translateY(-4px);
          box-shadow: 0 4px 8px rgba(0,0,0,0.1);
        }

        .image {
          width: 100%;
          height: 200px;
          object-fit: cover;
          border-radius: 4px;
        }

        .name {
          font-size: 18px;
          font-weight: bold;
          margin: 12px 0;
        }

        .price {
          font-size: 20px;
          color: #2196F3;
        }

        button {
          width: 100%;
          padding: 12px;
          background: #2196F3;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          margin-top: 12px;
        }

        button:hover {
          background: #1976D2;
        }
      </style>

      <div class="product-card">
        <img class="image" src="${imageUrl}" alt="${name}" />
        <div class="name">${name}</div>
        <div class="price">$${parseFloat(price).toFixed(2)}</div>
        <button id="add-to-cart">Add to Cart</button>
      </div>
    `;

    // Add event listener
    const button = this.shadowRoot.querySelector('#add-to-cart');
    button?.addEventListener('click', () => this.handleAddToCart());
  }

  private handleAddToCart() {
    const event = new CustomEvent('add-to-cart', {
      detail: {
        productId: this.getAttribute('product-id'),
        name: this.getAttribute('name'),
        price: this.getAttribute('price'),
      },
      bubbles: true,
      composed: true, // Cross shadow DOM boundary
    });

    this.dispatchEvent(event);
  }
}

// Register Custom Element
customElements.define('product-card', ProductCard);

// Usage in any framework or vanilla JS
function App() {
  useEffect(() => {
    const handleAddToCart = (event: Event) => {
      const customEvent = event as CustomEvent;
      console.log('Add to cart:', customEvent.detail);
    };

    document.addEventListener('add-to-cart', handleAddToCart);

    return () => {
      document.removeEventListener('add-to-cart', handleAddToCart);
    };
  }, []);

  return (
    <div>
      <product-card
        product-id="1"
        name="Premium Laptop"
        price="1299.99"
        image-url="https://example.com/laptop.jpg"
      />
    </div>
  );
}

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}
```

### Advanced Web Component with Properties

```typescript
// Product List Web Component
class ProductListElement extends HTMLElement {
  private shadowRoot: ShadowRoot;
  private _products: Product[] = [];

  constructor() {
    super();
    this.shadowRoot = this.attachShadow({ mode: 'open' });
  }

  // Property getter/setter
  get products(): Product[] {
    return this._products;
  }

  set products(value: Product[]) {
    this._products = value;
    this.render();
  }

  connectedCallback() {
    this.render();
  }

  private render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
        }

        .product-list {
          display: grid;
          grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
          gap: 16px;
          padding: 16px;
        }

        .loading {
          text-align: center;
          padding: 40px;
          font-size: 18px;
          color: #666;
        }

        .empty {
          text-align: center;
          padding: 40px;
          color: #999;
        }
      </style>

      <div class="product-list">
        ${this._products.length === 0
          ? '<div class="empty">No products available</div>'
          : this._products.map(product => `
              <product-card
                product-id="${product.id}"
                name="${product.name}"
                price="${product.price}"
                image-url="${product.imageUrl}"
              ></product-card>
            `).join('')}
      </div>
    `;
  }
}

customElements.define('product-list', ProductListElement);

// Usage with TypeScript
declare global {
  interface HTMLElementTagNameMap {
    'product-card': ProductCard;
    'product-list': ProductListElement;
  }
}

// React Wrapper for Web Component
import React, { useRef, useEffect } from 'react';

interface ProductListProps {
  products: Product[];
  onAddToCart?: (product: Product) => void;
}

const ProductListComponent: React.FC<ProductListProps> = ({
  products,
  onAddToCart,
}) => {
  const ref = useRef<ProductListElement>(null);

  useEffect(() => {
    if (ref.current) {
      ref.current.products = products;
    }
  }, [products]);

  useEffect(() => {
    const handleAddToCart = (event: Event) => {
      const customEvent = event as CustomEvent<Product>;
      onAddToCart?.(customEvent.detail);
    };

    const element = ref.current;
    element?.addEventListener('add-to-cart', handleAddToCart);

    return () => {
      element?.removeEventListener('add-to-cart', handleAddToCart);
    };
  }, [onAddToCart]);

  return <product-list ref={ref} />;
};
```

## 4. Single-SPA Framework

Meta-framework for micro-frontends.

```typescript
// Root Config (Shell)
import { registerApplication, start } from 'single-spa';

// Register micro-frontends
registerApplication({
  name: '@org/header',
  app: () => System.import('@org/header'),
  activeWhen: ['/'],
  customProps: {
    apiUrl: 'https://api.example.com',
  },
});

registerApplication({
  name: '@org/products',
  app: () => System.import('@org/products'),
  activeWhen: ['/products'],
});

registerApplication({
  name: '@org/cart',
  app: () => System.import('@org/cart'),
  activeWhen: ['/cart'],
});

registerApplication({
  name: '@org/checkout',
  app: () => System.import('@org/checkout'),
  activeWhen: ['/checkout'],
});

// Start single-spa
start({
  urlRerouteOnly: true,
});

// Micro-Frontend Application (React)
import React from 'react';
import ReactDOM from 'react-dom';
import singleSpaReact from 'single-spa-react';
import ProductApp from './ProductApp';

const lifecycles = singleSpaReact({
  React,
  ReactDOM,
  rootComponent: ProductApp,
  errorBoundary(err, info, props) {
    return (
      <div>
        <h1>Error in Products Micro-Frontend</h1>
        <p>{err.message}</p>
      </div>
    );
  },
});

export const { bootstrap, mount, unmount } = lifecycles;

// Product App Component
const ProductApp: React.FC = () => {
  const [products, setProducts] = useState<Product[]>([]);

  useEffect(() => {
    fetchProducts();
  }, []);

  const fetchProducts = async () => {
    const response = await fetch('/api/products');
    const data = await response.json();
    setProducts(data);
  };

  return (
    <div className="product-app">
      <h1>Products</h1>
      <ProductList products={products} />
    </div>
  );
};

export default ProductApp;
```

### Single-SPA Parcel (Reusable Component)

```typescript
// Parcel Definition
import Parcel from 'single-spa-react/parcel';

const shoppingCartParcel = singleSpaReact({
  React,
  ReactDOM,
  rootComponent: ShoppingCart,
});

// Usage in any micro-frontend
function CheckoutPage() {
  return (
    <div>
      <h1>Checkout</h1>

      {/* Embed shopping cart parcel */}
      <Parcel
        config={shoppingCartParcel}
        wrapWith="div"
        customProps={{
          userId: '123',
          onCheckout: handleCheckout,
        }}
      />
    </div>
  );
}
```

## 5. Import Maps

Native browser support for module resolution.

```typescript
// index.html
<!DOCTYPE html>
<html>
<head>
  <script type="importmap">
  {
    "imports": {
      "react": "https://esm.sh/react@18",
      "react-dom": "https://esm.sh/react-dom@18",
      "@org/shared": "/shared/index.js",
      "@org/header": "/micro-frontends/header/index.js",
      "@org/products": "/micro-frontends/products/index.js",
      "@org/cart": "/micro-frontends/cart/index.js"
    }
  }
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module">
    import { renderHeader } from '@org/header';
    import { renderProducts } from '@org/products';
    import { renderCart } from '@org/cart';

    renderHeader(document.getElementById('header'));
    renderProducts(document.getElementById('products'));
    renderCart(document.getElementById('cart'));
  </script>
</body>
</html>

// Micro-Frontend Module
// micro-frontends/products/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import { EventBus } from '@org/shared';

export function renderProducts(container: HTMLElement) {
  const root = ReactDOM.createRoot(container);

  root.render(
    <ProductsApp eventBus={EventBus.getInstance()} />
  );

  // Return unmount function
  return () => {
    root.unmount();
  };
}

const ProductsApp: React.FC<{ eventBus: EventBus }> = ({ eventBus }) => {
  const [products, setProducts] = useState<Product[]>([]);

  useEffect(() => {
    // Subscribe to global events
    const unsubscribe = eventBus.subscribe('user:login', (user) => {
      console.log('User logged in:', user);
      fetchUserProducts(user.id);
    });

    return unsubscribe;
  }, [eventBus]);

  return (
    <div className="products-app">
      {/* Product list UI */}
    </div>
  );
};
```

## 6. Runtime vs Build-Time Integration

### Runtime Integration

```typescript
// Runtime Module Loader
class RuntimeModuleLoader {
  private cache: Map<string, any> = new Map();

  async loadModule(url: string): Promise<any> {
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }

    const response = await fetch(url);
    const code = await response.text();

    // Create module
    const module = { exports: {} };
    const moduleFunction = new Function("module", "exports", code);
    moduleFunction(module, module.exports);

    this.cache.set(url, module.exports);
    return module.exports;
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Usage
const loader = new RuntimeModuleLoader();

async function loadMicroFrontend(url: string, container: string) {
  const module = await loader.loadModule(url);

  if (module.mount) {
    await module.mount(container);
  }
}
```

### Build-Time Integration

```typescript
// Build-time imports (static)
import ProductList from '../products/ProductList';
import ShoppingCart from '../cart/ShoppingCart';
import UserProfile from '../user/UserProfile';

function App() {
  return (
    <div>
      <UserProfile />
      <ProductList />
      <ShoppingCart />
    </div>
  );
}

// Advantages: Type safety, tree shaking, faster load times
// Disadvantages: Coupled deployments, monolithic builds
```

## 7. Deployment Strategies

### Independent Deployments

```typescript
// Deployment Configuration
interface DeploymentConfig {
  microFrontends: MicroFrontendDeployment[];
  registry: string;
}

interface MicroFrontendDeployment {
  name: string;
  version: string;
  url: string;
  dependencies: Record<string, string>;
}

class DeploymentManager {
  private config: DeploymentConfig;

  constructor(config: DeploymentConfig) {
    this.config = config;
  }

  async deployMicroFrontend(name: string, version: string): Promise<void> {
    // Build micro-frontend
    await this.build(name);

    // Upload to CDN
    const url = await this.uploadToCDN(name, version);

    // Update registry
    await this.updateRegistry(name, version, url);

    // Notify other micro-frontends
    await this.notifyUpdate(name, version);
  }

  private async build(name: string): Promise<void> {
    // Run build process
    console.log(`Building ${name}...`);
  }

  private async uploadToCDN(name: string, version: string): Promise<string> {
    // Upload to CDN and return URL
    return `https://cdn.example.com/${name}/${version}/bundle.js`;
  }

  private async updateRegistry(
    name: string,
    version: string,
    url: string,
  ): Promise<void> {
    await fetch(`${this.config.registry}/micro-frontends/${name}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ version, url }),
    });
  }

  private async notifyUpdate(name: string, version: string): Promise<void> {
    // Send webhook or publish event
    await fetch(`${this.config.registry}/webhooks/update`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name, version }),
    });
  }
}
```

## 8. Shared Dependencies

### Shared Library Management

```typescript
// Shared Library Package
// packages/shared/src/index.ts

export class EventBus {
  private static instance: EventBus;
  private events: Map<string, Array<(data: any) => void>> = new Map();

  private constructor() {}

  static getInstance(): EventBus {
    if (!EventBus.instance) {
      EventBus.instance = new EventBus();
    }
    return EventBus.instance;
  }

  subscribe(event: string, callback: (data: any) => void): () => void {
    if (!this.events.has(event)) {
      this.events.set(event, []);
    }

    this.events.get(event)!.push(callback);

    return () => {
      const callbacks = this.events.get(event);
      if (callbacks) {
        this.events.set(
          event,
          callbacks.filter((cb) => cb !== callback),
        );
      }
    };
  }

  publish(event: string, data: any): void {
    const callbacks = this.events.get(event);
    if (callbacks) {
      callbacks.forEach((callback) => callback(data));
    }
  }
}

export class StateManager {
  private state: Record<string, any> = {};
  private listeners: Map<string, Array<(value: any) => void>> = new Map();

  setState(key: string, value: any): void {
    this.state[key] = value;
    this.notify(key, value);
  }

  getState(key: string): any {
    return this.state[key];
  }

  subscribe(key: string, listener: (value: any) => void): () => void {
    if (!this.listeners.has(key)) {
      this.listeners.set(key, []);
    }

    this.listeners.get(key)!.push(listener);

    // Call immediately with current value
    listener(this.state[key]);

    return () => {
      const callbacks = this.listeners.get(key);
      if (callbacks) {
        this.listeners.set(
          key,
          callbacks.filter((cb) => cb !== listener),
        );
      }
    };
  }

  private notify(key: string, value: any): void {
    const callbacks = this.listeners.get(key);
    if (callbacks) {
      callbacks.forEach((callback) => callback(value));
    }
  }
}

export const sharedState = new StateManager();
export const eventBus = EventBus.getInstance();
```

### Webpack Configuration for Shared Dependencies

```typescript
// webpack.config.js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const packageJson = require("./package.json");

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "product",
      filename: "remoteEntry.js",
      exposes: {
        "./ProductList": "./src/components/ProductList",
      },
      shared: {
        // Share React with specific version
        react: {
          singleton: true,
          requiredVersion: packageJson.dependencies.react,
          eager: false,
        },
        "react-dom": {
          singleton: true,
          requiredVersion: packageJson.dependencies["react-dom"],
          eager: false,
        },
        // Share custom library
        "@org/shared": {
          singleton: true,
          requiredVersion: packageJson.dependencies["@org/shared"],
        },
      },
    }),
  ],
};
```

## Real-World Example: E-Commerce Platform

```typescript
// Shell Application
import { registerApplication, start } from 'single-spa';
import { EventBus, sharedState } from '@org/shared';

const eventBus = EventBus.getInstance();

// Register micro-frontends
registerApplication({
  name: '@ecommerce/header',
  app: () => System.import('@ecommerce/header'),
  activeWhen: ['/'],
  customProps: { eventBus, sharedState },
});

registerApplication({
  name: '@ecommerce/products',
  app: () => System.import('@ecommerce/products'),
  activeWhen: ['/products'],
  customProps: { eventBus, sharedState },
});

registerApplication({
  name: '@ecommerce/cart',
  app: () => System.import('@ecommerce/cart'),
  activeWhen: ['/cart'],
  customProps: { eventBus, sharedState },
});

registerApplication({
  name: '@ecommerce/checkout',
  app: () => System.import('@ecommerce/checkout'),
  activeWhen: ['/checkout'],
  customProps: { eventBus, sharedState },
});

// Global event handlers
eventBus.subscribe('user:login', (user) => {
  sharedState.setState('currentUser', user);
  console.log('User logged in:', user);
});

eventBus.subscribe('cart:add', (item) => {
  const cart = sharedState.getState('cart') || [];
  cart.push(item);
  sharedState.setState('cart', cart);
  console.log('Item added to cart:', item);
});

start();

// Products Micro-Frontend
const ProductsMicroFrontend = ({ eventBus, sharedState }: CustomProps) => {
  const [products, setProducts] = useState<Product[]>([]);
  const [currentUser, setCurrentUser] = useState(null);

  useEffect(() => {
    // Subscribe to shared state
    const unsubscribe = sharedState.subscribe('currentUser', setCurrentUser);
    return unsubscribe;
  }, []);

  const handleAddToCart = (product: Product) => {
    eventBus.publish('cart:add', product);
  };

  return (
    <div className="products">
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={handleAddToCart}
        />
      ))}
    </div>
  );
};
```

## Best Practices

1. **Define clear boundaries** - Each micro-frontend should own a complete feature
2. **Minimize shared dependencies** - Reduces coupling and deployment complexity
3. **Use semantic versioning** - Clear versioning strategy for micro-frontends
4. **Implement fallbacks** - Graceful degradation when micro-frontend fails
5. **Monitor performance** - Track load times and bundle sizes
6. **Establish communication patterns** - Event bus or shared state
7. **Version shared libraries carefully** - Breaking changes affect all micro-frontends
8. **Implement error boundaries** - Isolate failures to single micro-frontend
9. **Use CDN for hosting** - Fast, globally distributed micro-frontends
10. **Automate deployments** - CI/CD pipeline for independent deployments

## Key Takeaways

1. **Module Federation is production-ready** - Webpack 5's killer feature enables true runtime composition with shared dependencies

2. **iframes provide complete isolation** - Perfect for third-party integrations, but communication overhead and styling challenges

3. **Web Components are standards-based** - Framework-agnostic, true encapsulation, but limited browser support for older versions

4. **Single-SPA is battle-tested** - Meta-framework handles routing, lifecycle, handles multiple frameworks simultaneously

5. **Import Maps are the future** - Native browser support for module resolution, but current browser support is limited

6. **Runtime integration is flexible** - Deploy independently, update without rebuilding, but slower initial loads

7. **Shared dependencies need strategy** - Balance between duplication (larger bundles) and shared (version conflicts)

8. **Independent deployments unlock velocity** - Teams ship without coordination, but require robust monitoring and error handling

9. **Micro-frontends add complexity** - Only adopt when team size and application scale justify the architectural overhead

10. **Communication is critical** - Event bus, shared state, or custom protocols must be well-defined and documented
