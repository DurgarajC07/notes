# React Code Organization

## Core Concept

Effective code organization is essential for maintainable, scalable React applications. This includes structuring files, separating concerns, managing dependencies, and establishing clear boundaries between different parts of the application.

---

## Feature-Based Structure

### Directory Organization

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
│   │   │   └── useAuthRedirect.ts
│   │   ├── services/
│   │   │   └── authService.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   ├── utils/
│   │   │   └── tokenUtils.ts
│   │   └── index.ts
│   ├── products/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── index.ts
│   └── cart/
│       └── ...
├── shared/
│   ├── components/
│   │   ├── Button/
│   │   ├── Input/
│   │   └── Modal/
│   ├── hooks/
│   │   ├── useLocalStorage.ts
│   │   └── useDebounce.ts
│   ├── utils/
│   │   ├── formatters.ts
│   │   └── validators.ts
│   └── types/
│       └── common.types.ts
├── layouts/
│   ├── MainLayout.tsx
│   ├── DashboardLayout.tsx
│   └── AuthLayout.tsx
└── App.tsx
```

### Feature Module Example

```typescript
// features/products/index.ts - Public API
export { ProductList } from './components/ProductList';
export { ProductDetail } from './components/ProductDetail';
export { useProducts } from './hooks/useProducts';
export type { Product, ProductFilter } from './types/product.types';

// features/products/components/ProductList.tsx
import { useProducts } from '../hooks/useProducts';
import { ProductCard } from './ProductCard';
import type { ProductFilter } from '../types/product.types';

export function ProductList({ filter }: { filter: ProductFilter }) {
  const { products, loading, error } = useProducts(filter);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div className="product-grid">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// features/products/hooks/useProducts.ts
import { useQuery } from '@tanstack/react-query';
import { productService } from '../services/productService';
import type { Product, ProductFilter } from '../types/product.types';

export function useProducts(filter: ProductFilter) {
  return useQuery({
    queryKey: ['products', filter],
    queryFn: () => productService.getAll(filter),
    staleTime: 5 * 60 * 1000
  });
}

// features/products/services/productService.ts
import { apiClient } from '@/shared/services/apiClient';
import type { Product, ProductFilter } from '../types/product.types';

export const productService = {
  async getAll(filter: ProductFilter): Promise<Product[]> {
    const response = await apiClient.get('/products', { params: filter });
    return response.data;
  },

  async getById(id: string): Promise<Product> {
    const response = await apiClient.get(`/products/${id}`);
    return response.data;
  }
};
```

---

## Layer Architecture

### Separation of Concerns

```typescript
// ============================================
// 1. PRESENTATION LAYER (Components)
// ============================================

// components/UserProfile.tsx
interface UserProfileProps {
  userId: string;
}

export function UserProfile({ userId }: UserProfileProps) {
  const { user, loading, error, updateUser } = useUserProfile(userId);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!user) return <NotFound />;

  return <UserProfileView user={user} onUpdate={updateUser} />;
}

// ============================================
// 2. BUSINESS LOGIC LAYER (Hooks)
// ============================================

// hooks/useUserProfile.ts
export function useUserProfile(userId: string) {
  const queryClient = useQueryClient();

  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => userService.getById(userId)
  });

  const { mutate: updateUser } = useMutation({
    mutationFn: (updates: Partial<User>) =>
      userService.update(userId, updates),
    onSuccess: () => {
      queryClient.invalidateQueries(['user', userId]);
    }
  });

  return {
    user,
    loading: isLoading,
    error,
    updateUser
  };
}

// ============================================
// 3. DATA ACCESS LAYER (Services)
// ============================================

// services/userService.ts
import { apiClient } from './apiClient';
import type { User } from '../types';

export const userService = {
  async getById(id: string): Promise<User> {
    const response = await apiClient.get(`/users/${id}`);
    return response.data;
  },

  async update(id: string, updates: Partial<User>): Promise<User> {
    const response = await apiClient.patch(`/users/${id}`, updates);
    return response.data;
  }
};

// ============================================
// 4. API CLIENT LAYER
// ============================================

// services/apiClient.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

---

## Component Organization

### Single Responsibility Principle

```typescript
// ❌ Bad - Component doing too much
function UserDashboard() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [filter, setFilter] = useState('');

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });
  }, []);

  const filteredUsers = users.filter(u =>
    u.name.toLowerCase().includes(filter.toLowerCase())
  );

  const exportToCSV = () => {
    const csv = filteredUsers.map(u =>
      `${u.name},${u.email},${u.role}`
    ).join('\n');

    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = 'users.csv';
    link.click();
  };

  return (
    <div>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <button onClick={exportToCSV}>Export</button>
      {loading ? <div>Loading...</div> : (
        <table>
          {filteredUsers.map(user => (
            <tr key={user.id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
            </tr>
          ))}
        </table>
      )}
    </div>
  );
}

// ✅ Good - Separated concerns
function UserDashboard() {
  return (
    <div>
      <UserFilters />
      <UserTable />
      <UserExportButton />
    </div>
  );
}

function UserFilters() {
  const { filter, setFilter } = useUserFilter();
  return <SearchInput value={filter} onChange={setFilter} />;
}

function UserTable() {
  const { users, loading } = useFilteredUsers();

  if (loading) return <LoadingSpinner />;

  return (
    <Table>
      {users.map(user => (
        <UserRow key={user.id} user={user} />
      ))}
    </Table>
  );
}

function UserExportButton() {
  const { exportUsers } = useUserExport();
  return <Button onClick={exportUsers}>Export</Button>;
}
```

---

## Dependency Management

### Dependency Injection

```typescript
// ============================================
// Services with DI
// ============================================

// services/types.ts
export interface IUserService {
  getById(id: string): Promise<User>;
  update(id: string, data: Partial<User>): Promise<User>;
}

export interface IAuthService {
  login(credentials: Credentials): Promise<AuthToken>;
  logout(): Promise<void>;
}

// services/userService.ts
export class UserService implements IUserService {
  constructor(private apiClient: AxiosInstance) {}

  async getById(id: string): Promise<User> {
    const response = await this.apiClient.get(`/users/${id}`);
    return response.data;
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    const response = await this.apiClient.patch(`/users/${id}`, data);
    return response.data;
  }
}

// context/ServicesContext.tsx
interface Services {
  userService: IUserService;
  authService: IAuthService;
}

const ServicesContext = createContext<Services | undefined>(undefined);

export function ServicesProvider({ children }: { children: ReactNode }) {
  const apiClient = createApiClient();

  const services: Services = {
    userService: new UserService(apiClient),
    authService: new AuthService(apiClient)
  };

  return (
    <ServicesContext.Provider value={services}>
      {children}
    </ServicesContext.Provider>
  );
}

export function useServices() {
  const context = useContext(ServicesContext);
  if (!context) throw new Error('useServices must be within provider');
  return context;
}

// Usage in components
function UserProfile({ userId }: { userId: string }) {
  const { userService } = useServices();
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    userService.getById(userId).then(setUser);
  }, [userId, userService]);

  return <div>{user?.name}</div>;
}
```

---

## File Naming Conventions

### Consistent Naming

```
// Components - PascalCase
Button.tsx
UserProfile.tsx
ProductCard.tsx

// Hooks - camelCase with 'use' prefix
useAuth.ts
useProducts.ts
useLocalStorage.ts

// Utils/Helpers - camelCase
formatDate.ts
validateEmail.ts
apiHelpers.ts

// Types - PascalCase with .types suffix
user.types.ts
product.types.ts
api.types.ts

// Constants - UPPER_SNAKE_CASE
API_ENDPOINTS.ts
CONFIG.ts

// Services - camelCase with 'Service' suffix
authService.ts
userService.ts
storageService.ts
```

---

## Module Boundaries

### Public API Exports

```typescript
// features/auth/index.ts - Public API only
export { LoginForm } from "./components/LoginForm";
export { useAuth } from "./hooks/useAuth";
export type { User, AuthToken } from "./types/auth.types";

// Don't export internal utilities
// ❌ export { validatePassword } from './utils/validation';

// features/auth/components/LoginForm.tsx
// Internal to feature - not exported
import { validatePassword } from "../utils/validation";

// Other features import from public API
import { useAuth, type User } from "@/features/auth";
```

### Barrel Exports

```typescript
// shared/components/index.ts
export { Button } from "./Button/Button";
export { Input } from "./Input/Input";
export { Modal } from "./Modal/Modal";
export { Card } from "./Card/Card";

// Usage - clean imports
import { Button, Input, Modal } from "@/shared/components";

// Instead of
// import { Button } from '@/shared/components/Button/Button';
// import { Input } from '@/shared/components/Input/Input';
```

---

## Configuration Management

### Environment Configuration

```typescript
// config/env.ts
export const env = {
  apiUrl: process.env.REACT_APP_API_URL!,
  environment: process.env.NODE_ENV,
  isDevelopment: process.env.NODE_ENV === "development",
  isProduction: process.env.NODE_ENV === "production",
  features: {
    analytics: process.env.REACT_APP_ENABLE_ANALYTICS === "true",
    darkMode: process.env.REACT_APP_ENABLE_DARK_MODE === "true",
  },
} as const;

// Validate required env vars at startup
function validateEnv() {
  const required = ["REACT_APP_API_URL"];
  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(`Missing required env vars: ${missing.join(", ")}`);
  }
}

validateEnv();

// config/app.config.ts
export const appConfig = {
  pagination: {
    defaultPageSize: 20,
    maxPageSize: 100,
  },
  cache: {
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
  },
  validation: {
    passwordMinLength: 8,
    emailRegex: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
  },
} as const;
```

---

## Testing Organization

### Co-located Tests

```
features/
├── products/
│   ├── components/
│   │   ├── ProductCard.tsx
│   │   ├── ProductCard.test.tsx
│   │   ├── ProductList.tsx
│   │   └── ProductList.test.tsx
│   ├── hooks/
│   │   ├── useProducts.ts
│   │   └── useProducts.test.ts
│   └── services/
│       ├── productService.ts
│       └── productService.test.ts
```

### Test Utilities

```typescript
// test/utils/renderWithProviders.tsx
export function renderWithProviders(
  ui: ReactElement,
  options?: {
    initialState?: Partial<AppState>;
    services?: Partial<Services>;
  }
) {
  const mockServices = {
    userService: createMockUserService(),
    ...options?.services
  };

  return render(
    <ServicesProvider value={mockServices}>
      <QueryClientProvider client={createTestQueryClient()}>
        <BrowserRouter>
          {ui}
        </BrowserRouter>
      </QueryClientProvider>
    </ServicesProvider>
  );
}
```

---

## Import Organization

### Import Order

```typescript
// 1. External libraries
import React, { useState, useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { useNavigate } from "react-router-dom";

// 2. Internal modules (absolute imports)
import { Button, Input } from "@/shared/components";
import { useAuth } from "@/features/auth";
import { formatDate } from "@/shared/utils";

// 3. Relative imports
import { UserCard } from "./UserCard";
import { validateForm } from "../utils/validation";

// 4. Types
import type { User } from "@/features/auth/types";
import type { FormData } from "./types";

// 5. Styles
import "./UserProfile.css";
```

### Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@/*": ["*"],
      "@/features/*": ["features/*"],
      "@/shared/*": ["shared/*"],
      "@/components/*": ["shared/components/*"],
      "@/hooks/*": ["shared/hooks/*"],
      "@/utils/*": ["shared/utils/*"]
    }
  }
}
```

---

## Key Takeaways

1. **Feature-based structure** over layer-based for scalability
2. **Single Responsibility** - each file/component has one clear purpose
3. **Public API exports** - control what's exposed from modules
4. **Dependency injection** for testable, flexible code
5. **Consistent naming** - PascalCase components, camelCase hooks/utils
6. **Co-located tests** near the code they test
7. **Layer architecture** - presentation, business logic, data access
8. **Configuration management** - centralized env and app config
9. **Import organization** - consistent ordering with path aliases
10. **Module boundaries** - clear separation between features
