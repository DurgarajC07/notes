# E-Commerce Platform Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Patterns](#architecture-patterns)
3. [Product Catalog System](#product-catalog-system)
4. [Shopping Cart Implementation](#shopping-cart-implementation)
5. [Checkout Flow](#checkout-flow)
6. [Payment Processing](#payment-processing)
7. [Inventory Management](#inventory-management)
8. [State Management](#state-management)
9. [Performance Optimization](#performance-optimization)
10. [Security Considerations](#security-considerations)
11. [Scaling Strategies](#scaling-strategies)
12. [Key Takeaways](#key-takeaways)

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CDN Layer                               │
│         (Static Assets, Images, Product Media)               │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Load Balancer                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │               │
┌───────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
│   Frontend   │ │  Backend │ │   Search    │
│   React App  │ │   API    │ │  Service    │
│              │ │          │ │ (Elastic)   │
└──────┬───────┘ └────┬─────┘ └──────┬──────┘
       │              │                │
       │         ┌────▼────────────────▼────┐
       │         │                           │
       │    ┌────▼─────┐            ┌───────▼──────┐
       │    │ Product  │            │   Inventory  │
       │    │ Database │            │   Database   │
       │    │ (Postgres)            │  (MongoDB)   │
       │    └──────────┘            └──────────────┘
       │
┌──────▼───────────────────────────────────────┐
│           External Services                   │
│  - Payment Gateway (Stripe/PayPal)           │
│  - Email Service (SendGrid)                  │
│  - Analytics (Google Analytics)              │
│  - Monitoring (Datadog/Sentry)               │
└──────────────────────────────────────────────┘
```

## Architecture Patterns

### Micro-Frontend Architecture

```typescript
// Main application shell
interface MicroFrontend {
  name: string;
  host: string;
  entryPoint: string;
}

interface AppShellConfig {
  microFrontends: MicroFrontend[];
  sharedDependencies: string[];
  authProvider: AuthProvider;
}

class AppShell {
  private config: AppShellConfig;
  private loadedApps: Map<string, any> = new Map();

  constructor(config: AppShellConfig) {
    this.config = config;
  }

  async loadMicroFrontend(name: string): Promise<void> {
    const mfe = this.config.microFrontends.find((m) => m.name === name);
    if (!mfe || this.loadedApps.has(name)) return;

    const script = document.createElement("script");
    script.src = `${mfe.host}${mfe.entryPoint}`;
    script.onload = () => {
      const app = (window as any)[`${name}App`];
      this.loadedApps.set(name, app);
    };
    document.body.appendChild(script);
  }

  unmountMicroFrontend(name: string): void {
    const app = this.loadedApps.get(name);
    if (app && app.unmount) {
      app.unmount();
      this.loadedApps.delete(name);
    }
  }
}
```

### Module Federation Setup

```typescript
// webpack.config.js (TypeScript interface)
interface ModuleFederationConfig {
  name: string;
  filename: string;
  exposes: Record<string, string>;
  shared: Record<string, SharedConfig>;
}

interface SharedConfig {
  singleton: boolean;
  requiredVersion: string;
  eager?: boolean;
}

const config: ModuleFederationConfig = {
  name: "productCatalog",
  filename: "remoteEntry.js",
  exposes: {
    "./ProductList": "./src/components/ProductList",
    "./ProductDetail": "./src/components/ProductDetail",
    "./ProductSearch": "./src/components/ProductSearch",
  },
  shared: {
    react: { singleton: true, requiredVersion: "^18.0.0" },
    "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
    "react-router-dom": { singleton: true, requiredVersion: "^6.0.0" },
  },
};
```

## Product Catalog System

### Type Definitions

```typescript
// Product types
interface Product {
  id: string;
  sku: string;
  name: string;
  description: string;
  shortDescription?: string;
  category: Category;
  subcategories: Subcategory[];
  brand: Brand;
  price: Price;
  inventory: InventoryStatus;
  images: ProductImage[];
  variants: ProductVariant[];
  attributes: ProductAttribute[];
  ratings: RatingsSummary;
  seo: SEOMetadata;
  createdAt: Date;
  updatedAt: Date;
}

interface Price {
  amount: number;
  currency: string;
  compareAtPrice?: number;
  discountPercentage?: number;
  taxRate: number;
  finalPrice: number;
}

interface ProductVariant {
  id: string;
  sku: string;
  options: VariantOption[];
  price: Price;
  inventory: InventoryStatus;
  images: ProductImage[];
  enabled: boolean;
}

interface VariantOption {
  name: string; // e.g., "Size", "Color"
  value: string; // e.g., "Large", "Red"
}

interface InventoryStatus {
  available: number;
  reserved: number;
  incoming: number;
  warehouseLocation?: string;
  lowStockThreshold: number;
  isInStock: boolean;
}

interface ProductImage {
  id: string;
  url: string;
  alt: string;
  order: number;
  type: "thumbnail" | "main" | "gallery";
  variants: ImageVariant[];
}

interface ImageVariant {
  size: "small" | "medium" | "large" | "original";
  url: string;
  width: number;
  height: number;
}

interface Category {
  id: string;
  name: string;
  slug: string;
  parentId?: string;
  level: number;
  imageUrl?: string;
}

interface Brand {
  id: string;
  name: string;
  slug: string;
  logoUrl?: string;
}

interface RatingsSummary {
  average: number;
  count: number;
  distribution: Record<number, number>;
}

interface SEOMetadata {
  title: string;
  description: string;
  keywords: string[];
  canonicalUrl: string;
  ogImage?: string;
}
```

### Product Catalog Service

```typescript
class ProductCatalogService {
  private cache: Map<string, Product> = new Map();
  private cacheExpiry: Map<string, number> = new Map();
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 minutes

  constructor(
    private apiClient: ApiClient,
    private searchService: SearchService,
  ) {}

  async getProduct(id: string): Promise<Product> {
    const cached = this.getCached(id);
    if (cached) return cached;

    const product = await this.apiClient.get<Product>(`/products/${id}`);
    this.setCache(id, product);
    return product;
  }

  async getProducts(
    params: ProductQueryParams,
  ): Promise<PaginatedResult<Product>> {
    const {
      page = 1,
      limit = 20,
      category,
      brand,
      priceRange,
      sortBy,
    } = params;

    const queryString = new URLSearchParams({
      page: page.toString(),
      limit: limit.toString(),
      ...(category && { category }),
      ...(brand && { brand }),
      ...(priceRange && {
        minPrice: priceRange.min.toString(),
        maxPrice: priceRange.max.toString(),
      }),
      ...(sortBy && { sortBy }),
    }).toString();

    return await this.apiClient.get<PaginatedResult<Product>>(
      `/products?${queryString}`,
    );
  }

  async searchProducts(
    query: string,
    filters?: SearchFilters,
  ): Promise<SearchResult<Product>> {
    return await this.searchService.search<Product>({
      index: "products",
      query,
      filters,
      boost: {
        name: 3,
        description: 2,
        brand: 1.5,
        category: 1,
      },
    });
  }

  async getRelatedProducts(
    productId: string,
    limit: number = 6,
  ): Promise<Product[]> {
    return await this.apiClient.get<Product[]>(
      `/products/${productId}/related?limit=${limit}`,
    );
  }

  async getProductsByCategory(
    categoryId: string,
    params: QueryParams,
  ): Promise<Product[]> {
    return await this.apiClient.get<Product[]>(
      `/categories/${categoryId}/products`,
      { params },
    );
  }

  private getCached(id: string): Product | null {
    const expiry = this.cacheExpiry.get(id);
    if (expiry && Date.now() < expiry) {
      return this.cache.get(id) || null;
    }
    this.cache.delete(id);
    this.cacheExpiry.delete(id);
    return null;
  }

  private setCache(id: string, product: Product): void {
    this.cache.set(id, product);
    this.cacheExpiry.set(id, Date.now() + this.CACHE_TTL);
  }
}

interface ProductQueryParams {
  page?: number;
  limit?: number;
  category?: string;
  brand?: string;
  priceRange?: { min: number; max: number };
  sortBy?: "price_asc" | "price_desc" | "rating" | "newest";
}

interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  limit: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

### Product List Component

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { useInfiniteQuery } from '@tanstack/react-query';
import { useIntersectionObserver } from '@/hooks/useIntersectionObserver';

interface ProductListProps {
  categoryId?: string;
  searchQuery?: string;
  filters?: ProductFilters;
}

interface ProductFilters {
  brands?: string[];
  priceRange?: { min: number; max: number };
  rating?: number;
  inStock?: boolean;
}

export const ProductList: React.FC<ProductListProps> = ({
  categoryId,
  searchQuery,
  filters,
}) => {
  const [sortBy, setSortBy] = useState<SortOption>('relevance');

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    error,
  } = useInfiniteQuery({
    queryKey: ['products', categoryId, searchQuery, filters, sortBy],
    queryFn: ({ pageParam = 1 }) =>
      fetchProducts({
        page: pageParam,
        categoryId,
        searchQuery,
        filters,
        sortBy,
      }),
    getNextPageParam: (lastPage) =>
      lastPage.hasNext ? lastPage.page + 1 : undefined,
  });

  const { ref: loadMoreRef } = useIntersectionObserver({
    onIntersect: () => {
      if (hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    },
    threshold: 0.1,
  });

  if (isLoading) return <ProductListSkeleton />;
  if (isError) return <ErrorDisplay error={error} />;

  const products = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <div className="product-list-container">
      <ProductListHeader
        totalCount={data?.pages[0].total ?? 0}
        sortBy={sortBy}
        onSortChange={setSortBy}
      />

      <div className="product-grid">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>

      {hasNextPage && (
        <div ref={loadMoreRef} className="load-more-trigger">
          {isFetchingNextPage && <Spinner />}
        </div>
      )}
    </div>
  );
};

// Product Card Component
interface ProductCardProps {
  product: Product;
}

const ProductCard: React.FC<ProductCardProps> = ({ product }) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  const [isWishlisted, setIsWishlisted] = useState(false);
  const { addToCart } = useCart();

  const handleAddToCart = useCallback(() => {
    addToCart({
      productId: product.id,
      variantId: product.variants[0]?.id,
      quantity: 1,
    });
  }, [product.id, addToCart]);

  const handleWishlistToggle = useCallback(() => {
    setIsWishlisted((prev) => !prev);
    // API call to update wishlist
  }, []);

  return (
    <article className="product-card">
      <Link to={`/products/${product.id}`} className="product-card__link">
        <div className="product-card__image-wrapper">
          {!imageLoaded && <ImageSkeleton />}
          <img
            src={product.images[0]?.variants.find(v => v.size === 'medium')?.url}
            alt={product.images[0]?.alt}
            loading="lazy"
            onLoad={() => setImageLoaded(true)}
            className={imageLoaded ? 'loaded' : 'loading'}
          />
          {product.price.discountPercentage && (
            <span className="product-card__badge">
              -{product.price.discountPercentage}%
            </span>
          )}
        </div>

        <div className="product-card__content">
          <h3 className="product-card__title">{product.name}</h3>
          <div className="product-card__brand">{product.brand.name}</div>

          <div className="product-card__rating">
            <StarRating value={product.ratings.average} />
            <span>({product.ratings.count})</span>
          </div>

          <div className="product-card__price">
            {product.price.compareAtPrice && (
              <span className="price--compare">
                ${product.price.compareAtPrice.toFixed(2)}
              </span>
            )}
            <span className="price--final">
              ${product.price.finalPrice.toFixed(2)}
            </span>
          </div>
        </div>
      </Link>

      <div className="product-card__actions">
        <button
          onClick={handleWishlistToggle}
          className={`btn-icon ${isWishlisted ? 'active' : ''}`}
          aria-label="Add to wishlist"
        >
          <HeartIcon filled={isWishlisted} />
        </button>
        <button
          onClick={handleAddToCart}
          className="btn-primary"
          disabled={!product.inventory.isInStock}
        >
          {product.inventory.isInStock ? 'Add to Cart' : 'Out of Stock'}
        </button>
      </div>
    </article>
  );
};
```

## Shopping Cart Implementation

### Cart State Management

```typescript
// Cart types
interface CartItem {
  id: string;
  productId: string;
  variantId?: string;
  quantity: number;
  price: number;
  product: Product;
  variant?: ProductVariant;
  addedAt: Date;
}

interface Cart {
  id: string;
  userId?: string;
  items: CartItem[];
  subtotal: number;
  tax: number;
  shipping: number;
  discount: number;
  total: number;
  couponCode?: string;
  updatedAt: Date;
}

interface AddToCartPayload {
  productId: string;
  variantId?: string;
  quantity: number;
}

// Cart store using Zustand
import create from "zustand";
import { persist } from "zustand/middleware";

interface CartStore {
  cart: Cart | null;
  isLoading: boolean;
  error: string | null;

  // Actions
  addItem: (payload: AddToCartPayload) => Promise<void>;
  removeItem: (itemId: string) => Promise<void>;
  updateQuantity: (itemId: string, quantity: number) => Promise<void>;
  clearCart: () => Promise<void>;
  applyCoupon: (code: string) => Promise<void>;
  removeCoupon: () => Promise<void>;
  syncCart: () => Promise<void>;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      cart: null,
      isLoading: false,
      error: null,

      addItem: async (payload) => {
        set({ isLoading: true, error: null });
        try {
          const currentCart = get().cart;
          const existingItem = currentCart?.items.find(
            (item) =>
              item.productId === payload.productId &&
              item.variantId === payload.variantId,
          );

          let updatedCart: Cart;
          if (existingItem) {
            updatedCart = await cartApi.updateQuantity(
              existingItem.id,
              existingItem.quantity + payload.quantity,
            );
          } else {
            updatedCart = await cartApi.addItem(payload);
          }

          set({ cart: updatedCart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      removeItem: async (itemId) => {
        set({ isLoading: true, error: null });
        try {
          const updatedCart = await cartApi.removeItem(itemId);
          set({ cart: updatedCart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      updateQuantity: async (itemId, quantity) => {
        if (quantity < 1) {
          return get().removeItem(itemId);
        }

        set({ isLoading: true, error: null });
        try {
          const updatedCart = await cartApi.updateQuantity(itemId, quantity);
          set({ cart: updatedCart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      clearCart: async () => {
        set({ isLoading: true, error: null });
        try {
          await cartApi.clear();
          set({ cart: null, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      applyCoupon: async (code) => {
        set({ isLoading: true, error: null });
        try {
          const updatedCart = await cartApi.applyCoupon(code);
          set({ cart: updatedCart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      removeCoupon: async () => {
        set({ isLoading: true, error: null });
        try {
          const updatedCart = await cartApi.removeCoupon();
          set({ cart: updatedCart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },

      syncCart: async () => {
        set({ isLoading: true, error: null });
        try {
          const cart = await cartApi.getCart();
          set({ cart, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
          throw error;
        }
      },
    }),
    {
      name: "cart-storage",
      partialize: (state) => ({ cart: state.cart }),
    },
  ),
);
```

### Cart Component

```typescript
import React, { useEffect } from 'react';
import { useCartStore } from '@/stores/cartStore';
import { formatCurrency } from '@/utils/format';

export const ShoppingCart: React.FC = () => {
  const { cart, isLoading, addItem, removeItem, updateQuantity, applyCoupon } =
    useCartStore();
  const [couponCode, setCouponCode] = React.useState('');
  const [couponError, setCouponError] = React.useState('');

  const handleQuantityChange = async (itemId: string, newQuantity: number) => {
    try {
      await updateQuantity(itemId, newQuantity);
    } catch (error) {
      console.error('Failed to update quantity:', error);
    }
  };

  const handleApplyCoupon = async () => {
    if (!couponCode.trim()) return;

    try {
      setCouponError('');
      await applyCoupon(couponCode);
      setCouponCode('');
    } catch (error) {
      setCouponError('Invalid coupon code');
    }
  };

  if (!cart || cart.items.length === 0) {
    return <EmptyCart />;
  }

  return (
    <div className="shopping-cart">
      <div className="cart-items">
        <h2>Shopping Cart ({cart.items.length} items)</h2>

        {cart.items.map((item) => (
          <CartItem
            key={item.id}
            item={item}
            onQuantityChange={(qty) => handleQuantityChange(item.id, qty)}
            onRemove={() => removeItem(item.id)}
          />
        ))}
      </div>

      <div className="cart-summary">
        <h3>Order Summary</h3>

        <div className="summary-row">
          <span>Subtotal:</span>
          <span>{formatCurrency(cart.subtotal)}</span>
        </div>

        {cart.discount > 0 && (
          <div className="summary-row discount">
            <span>Discount:</span>
            <span>-{formatCurrency(cart.discount)}</span>
          </div>
        )}

        <div className="summary-row">
          <span>Shipping:</span>
          <span>{cart.shipping > 0 ? formatCurrency(cart.shipping) : 'FREE'}</span>
        </div>

        <div className="summary-row">
          <span>Tax:</span>
          <span>{formatCurrency(cart.tax)}</span>
        </div>

        <div className="summary-row total">
          <span>Total:</span>
          <span>{formatCurrency(cart.total)}</span>
        </div>

        <div className="coupon-section">
          <input
            type="text"
            placeholder="Coupon code"
            value={couponCode}
            onChange={(e) => setCouponCode(e.target.value)}
          />
          <button onClick={handleApplyCoupon}>Apply</button>
          {couponError && <span className="error">{couponError}</span>}
        </div>

        <button className="checkout-btn" onClick={() => window.location.href = '/checkout'}>
          Proceed to Checkout
        </button>
      </div>
    </div>
  );
};

interface CartItemProps {
  item: CartItem;
  onQuantityChange: (quantity: number) => void;
  onRemove: () => void;
}

const CartItem: React.FC<CartItemProps> = ({ item, onQuantityChange, onRemove }) => {
  return (
    <div className="cart-item">
      <img
        src={item.product.images[0]?.variants[0]?.url}
        alt={item.product.name}
        className="cart-item__image"
      />

      <div className="cart-item__details">
        <h4>{item.product.name}</h4>
        {item.variant && (
          <div className="cart-item__variant">
            {item.variant.options.map((opt) => (
              <span key={opt.name}>
                {opt.name}: {opt.value}
              </span>
            ))}
          </div>
        )}
        <div className="cart-item__price">
          {formatCurrency(item.price)}
        </div>
      </div>

      <div className="cart-item__quantity">
        <button
          onClick={() => onQuantityChange(item.quantity - 1)}
          disabled={item.quantity <= 1}
        >
          -
        </button>
        <span>{item.quantity}</span>
        <button onClick={() => onQuantityChange(item.quantity + 1)}>+</button>
      </div>

      <div className="cart-item__total">
        {formatCurrency(item.price * item.quantity)}
      </div>

      <button className="cart-item__remove" onClick={onRemove}>
        Remove
      </button>
    </div>
  );
};
```

## Checkout Flow

### Checkout Types

```typescript
interface CheckoutState {
  step: CheckoutStep;
  shippingAddress: Address | null;
  billingAddress: Address | null;
  shippingMethod: ShippingMethod | null;
  paymentMethod: PaymentMethod | null;
  isProcessing: boolean;
  error: string | null;
}

type CheckoutStep = "shipping" | "payment" | "review" | "confirmation";

interface Address {
  id?: string;
  firstName: string;
  lastName: string;
  company?: string;
  address1: string;
  address2?: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
  phone: string;
  isDefault?: boolean;
}

interface ShippingMethod {
  id: string;
  name: string;
  description: string;
  estimatedDays: number;
  price: number;
  carrier: string;
}

interface PaymentMethod {
  type: "credit_card" | "paypal" | "apple_pay" | "google_pay";
  data: CreditCardData | PayPalData | WalletData;
}

interface CreditCardData {
  cardNumber: string;
  expiryMonth: string;
  expiryYear: string;
  cvv: string;
  cardholderName: string;
}

interface Order {
  id: string;
  orderNumber: string;
  userId: string;
  items: OrderItem[];
  shippingAddress: Address;
  billingAddress: Address;
  shippingMethod: ShippingMethod;
  paymentMethod: string;
  subtotal: number;
  shipping: number;
  tax: number;
  discount: number;
  total: number;
  status: OrderStatus;
  createdAt: Date;
  updatedAt: Date;
}

type OrderStatus =
  | "pending"
  | "processing"
  | "shipped"
  | "delivered"
  | "cancelled"
  | "refunded";

interface OrderItem {
  id: string;
  productId: string;
  variantId?: string;
  productName: string;
  quantity: number;
  price: number;
  total: number;
  imageUrl: string;
}
```

### Checkout Service

```typescript
class CheckoutService {
  constructor(
    private apiClient: ApiClient,
    private paymentGateway: PaymentGateway,
  ) {}

  async validateAddress(address: Address): Promise<AddressValidationResult> {
    return await this.apiClient.post<AddressValidationResult>(
      "/checkout/validate-address",
      address,
    );
  }

  async getShippingMethods(address: Address): Promise<ShippingMethod[]> {
    return await this.apiClient.post<ShippingMethod[]>(
      "/checkout/shipping-methods",
      address,
    );
  }

  async calculateTax(
    items: CartItem[],
    shippingAddress: Address,
  ): Promise<TaxCalculation> {
    return await this.apiClient.post<TaxCalculation>(
      "/checkout/calculate-tax",
      {
        items,
        shippingAddress,
      },
    );
  }

  async createPaymentIntent(amount: number): Promise<PaymentIntent> {
    return await this.paymentGateway.createIntent({
      amount,
      currency: "usd",
    });
  }

  async processPayment(
    paymentMethod: PaymentMethod,
    amount: number,
  ): Promise<PaymentResult> {
    switch (paymentMethod.type) {
      case "credit_card":
        return await this.processCreditCard(
          paymentMethod.data as CreditCardData,
          amount,
        );
      case "paypal":
        return await this.processPayPal(
          paymentMethod.data as PayPalData,
          amount,
        );
      default:
        throw new Error("Unsupported payment method");
    }
  }

  async createOrder(orderData: CreateOrderRequest): Promise<Order> {
    return await this.apiClient.post<Order>("/orders", orderData);
  }

  async getOrder(orderId: string): Promise<Order> {
    return await this.apiClient.get<Order>(`/orders/${orderId}`);
  }

  private async processCreditCard(
    cardData: CreditCardData,
    amount: number,
  ): Promise<PaymentResult> {
    // Tokenize card data
    const token = await this.paymentGateway.tokenizeCard(cardData);

    // Process payment
    return await this.paymentGateway.charge({
      token,
      amount,
      currency: "usd",
    });
  }

  private async processPayPal(
    paypalData: PayPalData,
    amount: number,
  ): Promise<PaymentResult> {
    return await this.paymentGateway.processPayPal({
      paypalData,
      amount,
    });
  }
}
```

### Checkout Component

```typescript
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useCartStore } from '@/stores/cartStore';

export const Checkout: React.FC = () => {
  const navigate = useNavigate();
  const { cart } = useCartStore();
  const [state, setState] = useState<CheckoutState>({
    step: 'shipping',
    shippingAddress: null,
    billingAddress: null,
    shippingMethod: null,
    paymentMethod: null,
    isProcessing: false,
    error: null,
  });

  const handleShippingSubmit = async (address: Address) => {
    setState((prev) => ({ ...prev, shippingAddress: address, step: 'payment' }));
  };

  const handlePaymentSubmit = async (paymentMethod: PaymentMethod) => {
    setState((prev) => ({ ...prev, paymentMethod, step: 'review' }));
  };

  const handlePlaceOrder = async () => {
    setState((prev) => ({ ...prev, isProcessing: true, error: null }));

    try {
      const checkoutService = new CheckoutService(apiClient, paymentGateway);

      // Process payment
      const paymentResult = await checkoutService.processPayment(
        state.paymentMethod!,
        cart!.total
      );

      if (!paymentResult.success) {
        throw new Error('Payment failed');
      }

      // Create order
      const order = await checkoutService.createOrder({
        cartId: cart!.id,
        shippingAddress: state.shippingAddress!,
        billingAddress: state.billingAddress!,
        shippingMethod: state.shippingMethod!,
        paymentMethodId: paymentResult.paymentMethodId,
      });

      // Clear cart and navigate to confirmation
      await useCartStore.getState().clearCart();
      navigate(`/checkout/confirmation/${order.id}`);
    } catch (error) {
      setState((prev) => ({
        ...prev,
        isProcessing: false,
        error: (error as Error).message,
      }));
    }
  };

  return (
    <div className="checkout">
      <CheckoutProgress step={state.step} />

      {state.step === 'shipping' && (
        <ShippingStep onSubmit={handleShippingSubmit} />
      )}

      {state.step === 'payment' && (
        <PaymentStep
          shippingAddress={state.shippingAddress!}
          onSubmit={handlePaymentSubmit}
        />
      )}

      {state.step === 'review' && (
        <ReviewStep
          shippingAddress={state.shippingAddress!}
          billingAddress={state.billingAddress!}
          paymentMethod={state.paymentMethod!}
          cart={cart!}
          onPlaceOrder={handlePlaceOrder}
          isProcessing={state.isProcessing}
        />
      )}

      {state.error && (
        <div className="checkout-error">{state.error}</div>
      )}
    </div>
  );
};
```

## Payment Processing

### Payment Gateway Integration

```typescript
interface PaymentGatewayConfig {
  apiKey: string;
  environment: "test" | "production";
  webhookSecret: string;
}

class StripePaymentGateway implements PaymentGateway {
  private stripe: Stripe;

  constructor(config: PaymentGatewayConfig) {
    this.stripe = new Stripe(config.apiKey, {
      apiVersion: "2023-10-16",
    });
  }

  async createIntent(params: CreateIntentParams): Promise<PaymentIntent> {
    const intent = await this.stripe.paymentIntents.create({
      amount: Math.round(params.amount * 100), // Convert to cents
      currency: params.currency,
      automatic_payment_methods: {
        enabled: true,
      },
      metadata: params.metadata,
    });

    return {
      id: intent.id,
      clientSecret: intent.client_secret!,
      amount: intent.amount / 100,
      status: intent.status,
    };
  }

  async tokenizeCard(cardData: CreditCardData): Promise<string> {
    const token = await this.stripe.tokens.create({
      card: {
        number: cardData.cardNumber,
        exp_month: parseInt(cardData.expiryMonth),
        exp_year: parseInt(cardData.expiryYear),
        cvc: cardData.cvv,
        name: cardData.cardholderName,
      },
    });

    return token.id;
  }

  async charge(params: ChargeParams): Promise<PaymentResult> {
    try {
      const charge = await this.stripe.charges.create({
        amount: Math.round(params.amount * 100),
        currency: params.currency,
        source: params.token,
        description: params.description,
      });

      return {
        success: charge.status === "succeeded",
        transactionId: charge.id,
        paymentMethodId: charge.payment_method as string,
        amount: charge.amount / 100,
      };
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message,
      };
    }
  }

  async refund(transactionId: string, amount?: number): Promise<RefundResult> {
    const refund = await this.stripe.refunds.create({
      charge: transactionId,
      amount: amount ? Math.round(amount * 100) : undefined,
    });

    return {
      success: refund.status === "succeeded",
      refundId: refund.id,
      amount: refund.amount / 100,
    };
  }

  async verifyWebhook(payload: string, signature: string): Promise<boolean> {
    try {
      this.stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!,
      );
      return true;
    } catch (error) {
      return false;
    }
  }
}
```

### Payment Form Component

```typescript
import React, { useState } from 'react';
import { CardElement, useStripe, useElements } from '@stripe/react-stripe-js';

interface PaymentFormProps {
  amount: number;
  onSuccess: (paymentMethodId: string) => void;
  onError: (error: string) => void;
}

export const PaymentForm: React.FC<PaymentFormProps> = ({
  amount,
  onSuccess,
  onError,
}) => {
  const stripe = useStripe();
  const elements = useElements();
  const [isProcessing, setIsProcessing] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!stripe || !elements) return;

    setIsProcessing(true);

    try {
      const { error, paymentMethod } = await stripe.createPaymentMethod({
        type: 'card',
        card: elements.getElement(CardElement)!,
      });

      if (error) {
        throw new Error(error.message);
      }

      onSuccess(paymentMethod!.id);
    } catch (error) {
      onError((error as Error).message);
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="payment-form">
      <div className="form-group">
        <label>Card Details</label>
        <CardElement
          options={{
            style: {
              base: {
                fontSize: '16px',
                color: '#424770',
                '::placeholder': {
                  color: '#aab7c4',
                },
              },
              invalid: {
                color: '#9e2146',
              },
            },
          }}
        />
      </div>

      <button
        type="submit"
        disabled={!stripe || isProcessing}
        className="submit-payment-btn"
      >
        {isProcessing ? 'Processing...' : `Pay ${formatCurrency(amount)}`}
      </button>
    </form>
  );
};
```

## Inventory Management

### Inventory Service

```typescript
class InventoryService {
  constructor(
    private apiClient: ApiClient,
    private redis: RedisClient,
  ) {}

  async checkAvailability(
    productId: string,
    variantId?: string,
    quantity: number = 1,
  ): Promise<InventoryCheck> {
    const cacheKey = `inventory:${productId}:${variantId || "default"}`;

    // Try cache first
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      const inventory: InventoryStatus = JSON.parse(cached);
      return {
        available: inventory.available >= quantity,
        quantity: inventory.available,
      };
    }

    // Fetch from API
    const inventory = await this.apiClient.get<InventoryStatus>(
      `/inventory/${productId}`,
      { params: { variantId } },
    );

    // Cache for 1 minute
    await this.redis.setex(cacheKey, 60, JSON.stringify(inventory));

    return {
      available: inventory.available >= quantity,
      quantity: inventory.available,
    };
  }

  async reserveInventory(
    items: ReserveInventoryItem[],
  ): Promise<ReservationResult> {
    return await this.apiClient.post<ReservationResult>("/inventory/reserve", {
      items,
    });
  }

  async releaseInventory(reservationId: string): Promise<void> {
    await this.apiClient.post(`/inventory/release/${reservationId}`);
  }

  async updateInventory(
    productId: string,
    variantId: string | undefined,
    quantity: number,
  ): Promise<InventoryStatus> {
    const inventory = await this.apiClient.put<InventoryStatus>(
      `/inventory/${productId}`,
      { variantId, quantity },
    );

    // Invalidate cache
    const cacheKey = `inventory:${productId}:${variantId || "default"}`;
    await this.redis.del(cacheKey);

    return inventory;
  }

  async getLowStockProducts(threshold: number = 10): Promise<Product[]> {
    return await this.apiClient.get<Product[]>(
      `/inventory/low-stock?threshold=${threshold}`,
    );
  }
}

interface ReserveInventoryItem {
  productId: string;
  variantId?: string;
  quantity: number;
}

interface ReservationResult {
  reservationId: string;
  expiresAt: Date;
  items: ReservedItem[];
}

interface ReservedItem {
  productId: string;
  variantId?: string;
  quantityReserved: number;
}
```

## Performance Optimization

### Image Optimization

```typescript
// Responsive image component with lazy loading
interface OptimizedImageProps {
  src: string;
  alt: string;
  sizes?: string;
  priority?: boolean;
  onLoad?: () => void;
}

export const OptimizedImage: React.FC<OptimizedImageProps> = ({
  src,
  alt,
  sizes = '100vw',
  priority = false,
  onLoad,
}) => {
  const [loaded, setLoaded] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    if (priority && imgRef.current) {
      const img = new Image();
      img.src = src;
      img.onload = () => {
        setLoaded(true);
        onLoad?.();
      };
    }
  }, [src, priority, onLoad]);

  const srcSet = generateSrcSet(src);

  return (
    <picture>
      <source
        type="image/webp"
        srcSet={srcSet.webp}
        sizes={sizes}
      />
      <img
        ref={imgRef}
        src={src}
        srcSet={srcSet.default}
        sizes={sizes}
        alt={alt}
        loading={priority ? 'eager' : 'lazy'}
        decoding="async"
        onLoad={() => {
          setLoaded(true);
          onLoad?.();
        }}
        className={loaded ? 'loaded' : 'loading'}
      />
    </picture>
  );
};

function generateSrcSet(src: string) {
  const sizes = [320, 640, 768, 1024, 1280, 1920];
  return {
    webp: sizes.map(size => `${src}?w=${size}&fm=webp ${size}w`).join(', '),
    default: sizes.map(size => `${src}?w=${size} ${size}w`).join(', '),
  };
}
```

### Code Splitting

```typescript
// Route-based code splitting
import { lazy, Suspense } from 'react';

const ProductList = lazy(() => import('./pages/ProductList'));
const ProductDetail = lazy(() => import('./pages/ProductDetail'));
const Cart = lazy(() => import('./pages/Cart'));
const Checkout = lazy(() => import('./pages/Checkout'));

export const AppRoutes = () => (
  <Suspense fallback={<PageLoader />}>
    <Routes>
      <Route path="/products" element={<ProductList />} />
      <Route path="/products/:id" element={<ProductDetail />} />
      <Route path="/cart" element={<Cart />} />
      <Route path="/checkout" element={<Checkout />} />
    </Routes>
  </Suspense>
);
```

## Security Considerations

### Input Validation

```typescript
import { z } from "zod";

const AddressSchema = z.object({
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  address1: z.string().min(5).max(100),
  address2: z.string().max(100).optional(),
  city: z.string().min(2).max(50),
  state: z.string().length(2),
  postalCode: z.string().regex(/^\d{5}(-\d{4})?$/),
  country: z.string().length(2),
  phone: z.string().regex(/^\+?1?\d{10,14}$/),
});

const PaymentSchema = z.object({
  cardNumber: z.string().regex(/^\d{13,19}$/),
  expiryMonth: z.string().regex(/^(0[1-9]|1[0-2])$/),
  expiryYear: z.string().regex(/^\d{4}$/),
  cvv: z.string().regex(/^\d{3,4}$/),
  cardholderName: z.string().min(3).max(100),
});

export function validateAddress(data: unknown): Address {
  return AddressSchema.parse(data);
}

export function validatePayment(data: unknown): CreditCardData {
  return PaymentSchema.parse(data);
}
```

### XSS Protection

```typescript
import DOMPurify from 'dompurify';

export function sanitizeHTML(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href', 'target'],
  });
}

// Usage in components
const ProductDescription: React.FC<{ html: string }> = ({ html }) => {
  return (
    <div
      dangerouslySetInnerHTML={{ __html: sanitizeHTML(html) }}
    />
  );
};
```

## Scaling Strategies

### Caching Strategy

```typescript
class CacheManager {
  private redis: RedisClient;
  private memcache: MemcacheClient;

  constructor(redis: RedisClient, memcache: MemcacheClient) {
    this.redis = redis;
    this.memcache = memcache;
  }

  async get<T>(key: string): Promise<T | null> {
    // Try memcache first (L1 cache)
    let data = await this.memcache.get(key);
    if (data) return JSON.parse(data) as T;

    // Try Redis (L2 cache)
    data = await this.redis.get(key);
    if (data) {
      // Populate memcache
      await this.memcache.set(key, data, 300); // 5 min
      return JSON.parse(data) as T;
    }

    return null;
  }

  async set<T>(key: string, value: T, ttl: number): Promise<void> {
    const serialized = JSON.stringify(value);

    // Set in both caches
    await Promise.all([
      this.redis.setex(key, ttl, serialized),
      this.memcache.set(key, serialized, Math.min(ttl, 300)),
    ]);
  }

  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length === 0) return;

    await Promise.all([
      this.redis.del(...keys),
      ...keys.map((key) => this.memcache.del(key)),
    ]);
  }
}
```

### Database Optimization

```typescript
// Read replica routing
class DatabaseRouter {
  private primary: DatabaseConnection;
  private replicas: DatabaseConnection[];
  private currentReplicaIndex = 0;

  constructor(primary: DatabaseConnection, replicas: DatabaseConnection[]) {
    this.primary = primary;
    this.replicas = replicas;
  }

  getPrimary(): DatabaseConnection {
    return this.primary;
  }

  getReplica(): DatabaseConnection {
    if (this.replicas.length === 0) return this.primary;

    // Round-robin selection
    const replica = this.replicas[this.currentReplicaIndex];
    this.currentReplicaIndex =
      (this.currentReplicaIndex + 1) % this.replicas.length;
    return replica;
  }

  async query<T>(
    sql: string,
    params: any[],
    write: boolean = false,
  ): Promise<T> {
    const connection = write ? this.getPrimary() : this.getReplica();
    return await connection.query<T>(sql, params);
  }
}
```

## Key Takeaways

### 1. **Micro-Frontend Architecture**

Use module federation for independent deployment of product catalog, cart, and checkout modules. This enables team autonomy and reduces deployment risks.

### 2. **Optimistic UI Updates**

Implement optimistic updates for cart operations to provide instant feedback. Always have rollback mechanisms for failed operations.

### 3. **Multi-Layer Caching**

Use CDN for static assets, Redis for session data, and in-memory cache for frequently accessed product data. Implement cache invalidation strategies.

### 4. **Inventory Reservation**

Reserve inventory during checkout with time-based expiration. This prevents overselling while maintaining good user experience.

### 5. **Payment Security**

Never store sensitive payment data. Use tokenization and PCI-compliant payment gateways. Implement 3D Secure for card transactions.

### 6. **Progressive Image Loading**

Use responsive images with multiple formats (WebP, AVIF). Implement lazy loading for below-the-fold images and prioritize hero images.

### 7. **Search Optimization**

Use Elasticsearch for product search with proper relevance scoring. Implement faceted search and autocomplete with debouncing.

### 8. **Database Scaling**

Use read replicas for product queries. Implement connection pooling and query optimization. Consider sharding for large product catalogs.

### 9. **Error Handling**

Implement comprehensive error handling with user-friendly messages. Log errors to monitoring services and have alerting in place.

### 10. **A/B Testing Infrastructure**

Build feature flags and A/B testing into checkout flow. Test different layouts, CTAs, and payment options to optimize conversion rates.

---

**Further Reading:**

- Stripe Payment Integration Guide
- AWS E-Commerce Reference Architecture
- Shopify Engineering Blog
- Microservices Patterns by Chris Richardson
