# React Project Structure

## Core Concept

A well-organized project structure scales with team size and feature complexity. Feature-based organization groups related code together, domain-driven design models business logic, and proper separation of concerns improves maintainability.

---

## Feature-Based Structure

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── PasswordReset.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   └── useSession.ts
│   │   ├── api/
│   │   │   └── authApi.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   ├── utils/
│   │   │   └── validation.ts
│   │   └── index.ts
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductList.tsx
│   │   │   ├── ProductCard.tsx
│   │   │   └── ProductDetail.tsx
│   │   ├── hooks/
│   │   │   └── useProducts.ts
│   │   ├── api/
│   │   │   └── productsApi.ts
│   │   └── index.ts
│   └── cart/
│       ├── components/
│       ├── hooks/
│       └── index.ts
├── shared/
│   ├── components/
│   │   ├── Button/
│   │   ├── Input/
│   │   └── Modal/
│   ├── hooks/
│   │   ├── useDebounce.ts
│   │   └── useLocalStorage.ts
│   ├── utils/
│   │   ├── formatting.ts
│   │   └── validation.ts
│   └── types/
│       └── common.types.ts
├── app/
│   ├── store/
│   │   └── store.ts
│   ├── routes/
│   │   └── routes.tsx
│   └── App.tsx
└── main.tsx
```

---

## Domain-Driven Structure

```
src/
├── domains/
│   ├── user/
│   │   ├── entities/
│   │   │   └── User.ts
│   │   ├── repositories/
│   │   │   └── UserRepository.ts
│   │   ├── services/
│   │   │   └── UserService.ts
│   │   ├── ui/
│   │   │   ├── UserProfile.tsx
│   │   │   └── UserSettings.tsx
│   │   └── index.ts
│   ├── order/
│   │   ├── entities/
│   │   │   ├── Order.ts
│   │   │   └── OrderItem.ts
│   │   ├── repositories/
│   │   │   └── OrderRepository.ts
│   │   ├── services/
│   │   │   └── OrderService.ts
│   │   ├── ui/
│   │   │   ├── OrderList.tsx
│   │   │   └── OrderDetail.tsx
│   │   └── index.ts
│   └── payment/
│       ├── entities/
│       ├── services/
│       └── ui/
├── infrastructure/
│   ├── api/
│   │   ├── client.ts
│   │   └── interceptors.ts
│   ├── storage/
│   │   └── localStorage.ts
│   └── logger/
│       └── logger.ts
└── presentation/
    ├── components/
    ├── layouts/
    └── pages/
```

---

## Layered Architecture

```typescript
// src/layers/

// 1. Presentation Layer
// presentation/components/UserProfile.tsx
import { useUserService } from '@/services';

export function UserProfile() {
  const { user, updateUser } = useUserService();

  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => updateUser({ name: 'New Name' })}>
        Update
      </button>
    </div>
  );
}

// 2. Application/Service Layer
// services/UserService.ts
import { UserRepository } from '@/repositories';
import { User } from '@/entities';

export class UserService {
  constructor(private repo: UserRepository) {}

  async getUser(id: string): Promise<User> {
    return this.repo.findById(id);
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const user = await this.repo.findById(id);
    const updated = { ...user, ...data };
    return this.repo.save(updated);
  }
}

// 3. Domain Layer
// entities/User.ts
export class User {
  constructor(
    public id: string,
    public name: string,
    public email: string
  ) {}

  isValid(): boolean {
    return this.email.includes('@');
  }

  canUpgrade(): boolean {
    return this.email.endsWith('.edu');
  }
}

// 4. Infrastructure Layer
// repositories/UserRepository.ts
import { ApiClient } from '@/infrastructure/api';
import { User } from '@/entities';

export class UserRepository {
  constructor(private api: ApiClient) {}

  async findById(id: string): Promise<User> {
    const data = await this.api.get(`/users/${id}`);
    return new User(data.id, data.name, data.email);
  }

  async save(user: User): Promise<User> {
    const data = await this.api.put(`/users/${user.id}`, user);
    return new User(data.id, data.name, data.email);
  }
}
```

---

## Module Boundaries

```typescript
// features/auth/index.ts
// Explicit exports control public API

export { LoginForm, RegisterForm } from "./components";
export { useAuth, useSession } from "./hooks";
export type { AuthUser, AuthState } from "./types";

// Internal implementation hidden
// ❌ Not exported:
// - ./api/authApi (internal)
// - ./utils/validation (internal)
// - ./components/LoginButton (internal only)

// Other features import from index only
import { useAuth, AuthUser } from "@/features/auth";
```

---

## Barrel Exports

```typescript
// shared/components/index.ts
export { Button } from "./Button";
export { Input } from "./Input";
export { Modal } from "./Modal";
export { Card } from "./Card";

export type { ButtonProps } from "./Button";
export type { InputProps } from "./Input";

// Usage
import { Button, Input, Modal } from "@/shared/components";
```

---

## Absolute Imports

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@/*": ["*"],
      "@features/*": ["features/*"],
      "@shared/*": ["shared/*"],
      "@app/*": ["app/*"]
    }
  }
}

// Usage
import { Button } from '@shared/components';
import { useAuth } from '@features/auth';
import { store } from '@app/store';
```

---

## Co-location Pattern

```typescript
// features/products/components/ProductCard/
ProductCard/
├── ProductCard.tsx
├── ProductCard.test.tsx
├── ProductCard.stories.tsx
├── ProductCard.module.css
├── useProductCard.ts
└── index.ts

// ProductCard/index.ts
export { ProductCard } from './ProductCard';
export type { ProductCardProps } from './ProductCard';
```

---

## API Layer Organization

```typescript
// api/client.ts
export class ApiClient {
  constructor(private baseURL: string) {}

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`);
    return response.json();
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });
    return response.json();
  }
}

// api/endpoints.ts
export const ENDPOINTS = {
  users: {
    list: () => "/users",
    detail: (id: string) => `/users/${id}`,
    create: () => "/users",
    update: (id: string) => `/users/${id}`,
  },
  products: {
    list: () => "/products",
    detail: (id: string) => `/products/${id}`,
  },
} as const;

// Usage
import { api } from "@/api/client";
import { ENDPOINTS } from "@/api/endpoints";

const users = await api.get(ENDPOINTS.users.list());
const user = await api.get(ENDPOINTS.users.detail("123"));
```

---

## Routing Structure

```typescript
// app/routes/routes.tsx
import { lazy } from 'react';
import { RouteObject } from 'react-router-dom';

const HomePage = lazy(() => import('@/features/home/pages/HomePage'));
const ProductsPage = lazy(() => import('@/features/products/pages/ProductsPage'));
const ProductDetailPage = lazy(() => import('@/features/products/pages/ProductDetailPage'));

export const routes: RouteObject[] = [
  {
    path: '/',
    element: <HomePage />
  },
  {
    path: '/products',
    children: [
      {
        index: true,
        element: <ProductsPage />
      },
      {
        path: ':id',
        element: <ProductDetailPage />
      }
    ]
  }
];
```

---

## Environment Configuration

```typescript
// config/env.ts
interface Config {
  apiUrl: string;
  environment: "development" | "staging" | "production";
  features: {
    analytics: boolean;
    betaFeatures: boolean;
  };
}

const config: Config = {
  apiUrl: import.meta.env.VITE_API_URL || "http://localhost:3000",
  environment: (import.meta.env.MODE as any) || "development",
  features: {
    analytics: import.meta.env.VITE_ENABLE_ANALYTICS === "true",
    betaFeatures: import.meta.env.VITE_BETA_FEATURES === "true",
  },
};

export default config;

// Usage
import config from "@/config/env";
fetch(`${config.apiUrl}/users`);
```

---

## Real-World: E-Commerce Structure

```
src/
├── features/
│   ├── catalog/
│   │   ├── components/
│   │   │   ├── ProductGrid.tsx
│   │   │   ├── ProductCard.tsx
│   │   │   ├── ProductFilters.tsx
│   │   │   └── SearchBar.tsx
│   │   ├── hooks/
│   │   │   ├── useProducts.ts
│   │   │   ├── useSearch.ts
│   │   │   └── useFilters.ts
│   │   ├── pages/
│   │   │   ├── CatalogPage.tsx
│   │   │   └── ProductDetailPage.tsx
│   │   ├── api/
│   │   │   └── catalogApi.ts
│   │   └── types/
│   │       └── product.types.ts
│   ├── cart/
│   │   ├── components/
│   │   │   ├── CartDrawer.tsx
│   │   │   ├── CartItem.tsx
│   │   │   └── CartSummary.tsx
│   │   ├── hooks/
│   │   │   └── useCart.ts
│   │   ├── store/
│   │   │   └── cartSlice.ts
│   │   └── types/
│   │       └── cart.types.ts
│   ├── checkout/
│   │   ├── components/
│   │   │   ├── CheckoutForm.tsx
│   │   │   ├── PaymentMethod.tsx
│   │   │   └── ShippingAddress.tsx
│   │   ├── hooks/
│   │   │   └── useCheckout.ts
│   │   ├── pages/
│   │   │   └── CheckoutPage.tsx
│   │   └── api/
│   │       └── checkoutApi.ts
│   ├── orders/
│   │   ├── components/
│   │   ├── pages/
│   │   └── api/
│   └── auth/
│       ├── components/
│       ├── hooks/
│       └── api/
├── shared/
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   ├── Modal/
│   │   │   └── Card/
│   │   └── layout/
│   │       ├── Header/
│   │       ├── Footer/
│   │       └── Sidebar/
│   ├── hooks/
│   │   ├── useDebounce.ts
│   │   ├── useLocalStorage.ts
│   │   └── useMediaQuery.ts
│   ├── utils/
│   │   ├── currency.ts
│   │   ├── date.ts
│   │   └── validation.ts
│   └── types/
│       └── common.types.ts
├── app/
│   ├── store/
│   │   └── store.ts
│   ├── routes/
│   │   └── routes.tsx
│   ├── providers/
│   │   └── AppProviders.tsx
│   └── App.tsx
├── api/
│   ├── client.ts
│   ├── endpoints.ts
│   └── interceptors.ts
├── config/
│   ├── env.ts
│   └── constants.ts
└── main.tsx
```

---

## Best Practices

✅ **Group by feature**, not by type  
✅ **Co-locate related code**  
✅ **Use absolute imports**  
✅ **Control exports** via index files  
✅ **Separate business logic** from UI  
✅ **Layer architecture** for complexity  
❌ **Don't create deep nesting** (max 3-4 levels)  
❌ **Don't share implementation details** between features  
❌ **Don't couple features** directly

---

## Key Takeaways

1. **Feature-based structure scales** better than type-based
2. **Domain-driven design** models business logic
3. **Layered architecture** separates concerns
4. **Explicit exports** control public API
5. **Absolute imports** improve readability
6. **Co-location** keeps related code together
7. **Structure evolves** with project needs
