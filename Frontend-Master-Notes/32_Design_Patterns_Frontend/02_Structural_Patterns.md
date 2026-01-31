# Structural Design Patterns in Frontend Development

## Core Concepts

Structural patterns deal with object composition and relationships between entities. They help ensure that when one part of a system changes, the entire structure doesn't need to change. In frontend development, these patterns are essential for creating flexible, maintainable component hierarchies and managing complex relationships between UI elements and data layers.

## Adapter Pattern

### API Response Adapter

```typescript
// External API response format
interface ExternalUser {
  user_id: number;
  full_name: string;
  email_address: string;
  created_at: string;
  is_active: boolean;
}

// Internal application format
interface InternalUser {
  id: string;
  name: string;
  email: string;
  createdDate: Date;
  active: boolean;
}

class UserAdapter {
  static toInternal(externalUser: ExternalUser): InternalUser {
    return {
      id: externalUser.user_id.toString(),
      name: externalUser.full_name,
      email: externalUser.email_address,
      createdDate: new Date(externalUser.created_at),
      active: externalUser.is_active,
    };
  }

  static toExternal(internalUser: InternalUser): ExternalUser {
    return {
      user_id: parseInt(internalUser.id),
      full_name: internalUser.name,
      email_address: internalUser.email,
      created_at: internalUser.createdDate.toISOString(),
      is_active: internalUser.active,
    };
  }

  static toInternalList(externalUsers: ExternalUser[]): InternalUser[] {
    return externalUsers.map((user) => this.toInternal(user));
  }
}

// Usage
const apiResponse: ExternalUser = {
  user_id: 123,
  full_name: "John Doe",
  email_address: "john@example.com",
  created_at: "2024-01-01T00:00:00Z",
  is_active: true,
};

const internalUser = UserAdapter.toInternal(apiResponse);
console.log(internalUser);
```

### Storage Adapter

```typescript
interface StorageAdapter {
  get<T>(key: string): T | null;
  set<T>(key: string, value: T): void;
  remove(key: string): void;
  clear(): void;
  has(key: string): boolean;
}

class LocalStorageAdapter implements StorageAdapter {
  get<T>(key: string): T | null {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : null;
    } catch (error) {
      console.error("Error reading from localStorage:", error);
      return null;
    }
  }

  set<T>(key: string, value: T): void {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error("Error writing to localStorage:", error);
    }
  }

  remove(key: string): void {
    localStorage.removeItem(key);
  }

  clear(): void {
    localStorage.clear();
  }

  has(key: string): boolean {
    return localStorage.getItem(key) !== null;
  }
}

class SessionStorageAdapter implements StorageAdapter {
  get<T>(key: string): T | null {
    try {
      const item = sessionStorage.getItem(key);
      return item ? JSON.parse(item) : null;
    } catch (error) {
      console.error("Error reading from sessionStorage:", error);
      return null;
    }
  }

  set<T>(key: string, value: T): void {
    try {
      sessionStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error("Error writing to sessionStorage:", error);
    }
  }

  remove(key: string): void {
    sessionStorage.removeItem(key);
  }

  clear(): void {
    sessionStorage.clear();
  }

  has(key: string): boolean {
    return sessionStorage.getItem(key) !== null;
  }
}

class InMemoryStorageAdapter implements StorageAdapter {
  private storage: Map<string, any> = new Map();

  get<T>(key: string): T | null {
    return this.storage.get(key) ?? null;
  }

  set<T>(key: string, value: T): void {
    this.storage.set(key, value);
  }

  remove(key: string): void {
    this.storage.delete(key);
  }

  clear(): void {
    this.storage.clear();
  }

  has(key: string): boolean {
    return this.storage.has(key);
  }
}

// Factory to get appropriate adapter
class StorageFactory {
  static getAdapter(type: "local" | "session" | "memory"): StorageAdapter {
    switch (type) {
      case "local":
        return new LocalStorageAdapter();
      case "session":
        return new SessionStorageAdapter();
      case "memory":
        return new InMemoryStorageAdapter();
    }
  }
}

// Usage
const storage = StorageFactory.getAdapter("local");
storage.set("user", { id: 1, name: "John" });
const user = storage.get<{ id: number; name: string }>("user");
console.log(user);
```

### Legacy API Adapter

```typescript
// Old API interface
interface LegacyPaymentService {
  processPayment(amount: number, cardNumber: string): string;
  refundPayment(transactionId: string): boolean;
}

// New API interface
interface ModernPaymentService {
  charge(request: ChargeRequest): Promise<ChargeResponse>;
  refund(transactionId: string): Promise<RefundResponse>;
}

interface ChargeRequest {
  amount: number;
  currency: string;
  paymentMethod: {
    type: "card";
    cardNumber: string;
    expiryMonth: number;
    expiryYear: number;
    cvv: string;
  };
}

interface ChargeResponse {
  success: boolean;
  transactionId: string;
  message: string;
}

interface RefundResponse {
  success: boolean;
  refundId: string;
  message: string;
}

// Legacy implementation
class LegacyPaymentAPI implements LegacyPaymentService {
  processPayment(amount: number, cardNumber: string): string {
    console.log(`Processing payment of $${amount} with card ${cardNumber}`);
    return `txn_${Date.now()}`;
  }

  refundPayment(transactionId: string): boolean {
    console.log(`Refunding transaction ${transactionId}`);
    return true;
  }
}

// Adapter to use legacy API with modern interface
class PaymentAdapter implements ModernPaymentService {
  constructor(private legacyService: LegacyPaymentService) {}

  async charge(request: ChargeRequest): Promise<ChargeResponse> {
    try {
      const transactionId = this.legacyService.processPayment(
        request.amount,
        request.paymentMethod.cardNumber,
      );

      return {
        success: true,
        transactionId,
        message: "Payment processed successfully",
      };
    } catch (error) {
      return {
        success: false,
        transactionId: "",
        message: "Payment failed",
      };
    }
  }

  async refund(transactionId: string): Promise<RefundResponse> {
    try {
      const success = this.legacyService.refundPayment(transactionId);

      return {
        success,
        refundId: `ref_${Date.now()}`,
        message: success ? "Refund processed" : "Refund failed",
      };
    } catch (error) {
      return {
        success: false,
        refundId: "",
        message: "Refund failed",
      };
    }
  }
}

// Usage
const legacyAPI = new LegacyPaymentAPI();
const modernAPI: ModernPaymentService = new PaymentAdapter(legacyAPI);

const result = await modernAPI.charge({
  amount: 99.99,
  currency: "USD",
  paymentMethod: {
    type: "card",
    cardNumber: "4111111111111111",
    expiryMonth: 12,
    expiryYear: 2025,
    cvv: "123",
  },
});

console.log(result);
```

## Decorator Pattern

### Component Decorator (HOC)

```typescript
interface ComponentProps {
  data: any;
}

// Loading decorator
function withLoading<P extends object>(
  Component: React.ComponentType<P>
): React.FC<P & { isLoading?: boolean }> {
  return (props) => {
    const { isLoading, ...rest } = props as any;

    if (isLoading) {
      return <div className="spinner">Loading...</div>;
    }

    return <Component {...(rest as P)} />;
  };
}

// Error boundary decorator
function withErrorBoundary<P extends object>(
  Component: React.ComponentType<P>
): React.ComponentType<P> {
  return class extends React.Component<P, { hasError: boolean }> {
    constructor(props: P) {
      super(props);
      this.state = { hasError: false };
    }

    static getDerivedStateFromError() {
      return { hasError: true };
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error('Error caught:', error, errorInfo);
    }

    render() {
      if (this.state.hasError) {
        return <div>Something went wrong</div>;
      }

      return <Component {...this.props} />;
    }
  };
}

// Authentication decorator
function withAuth<P extends object>(
  Component: React.ComponentType<P>
): React.FC<P> {
  return (props) => {
    const isAuthenticated = useAuth();

    if (!isAuthenticated) {
      return <Navigate to="/login" />;
    }

    return <Component {...props} />;
  };
}

// Analytics decorator
function withAnalytics<P extends object>(
  Component: React.ComponentType<P>,
  eventName: string
): React.FC<P> {
  return (props) => {
    React.useEffect(() => {
      console.log(`Page viewed: ${eventName}`);
      // analytics.track(eventName);
    }, []);

    return <Component {...props} />;
  };
}

// Usage - compose multiple decorators
const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  return <div>User Profile: {userId}</div>;
};

const EnhancedUserProfile = withAuth(
  withLoading(
    withErrorBoundary(
      withAnalytics(UserProfile, 'user_profile_view')
    )
  )
);

// Usage
<EnhancedUserProfile userId="123" isLoading={false} />;
```

### Method Decorator

```typescript
// Logging decorator
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey} with args:`, args);
    const result = originalMethod.apply(this, args);
    console.log(`${propertyKey} returned:`, result);
    return result;
  };

  return descriptor;
}

// Timing decorator
function time(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor,
) {
  const originalMethod = descriptor.value;

  descriptor.value = async function (...args: any[]) {
    const start = performance.now();
    const result = await originalMethod.apply(this, args);
    const end = performance.now();
    console.log(`${propertyKey} took ${end - start}ms`);
    return result;
  };

  return descriptor;
}

// Memoization decorator
function memoize(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor,
) {
  const originalMethod = descriptor.value;
  const cache = new Map<string, any>();

  descriptor.value = function (...args: any[]) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      console.log("Returning cached result");
      return cache.get(key);
    }

    const result = originalMethod.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

// Retry decorator
function retry(maxAttempts: number = 3, delay: number = 1000) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor,
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      let lastError: Error;

      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;
          console.log(`Attempt ${attempt} failed, retrying...`);

          if (attempt < maxAttempts) {
            await new Promise((resolve) => setTimeout(resolve, delay));
          }
        }
      }

      throw lastError!;
    };

    return descriptor;
  };
}

// Usage
class DataService {
  @log
  @time
  @memoize
  fetchData(id: string): string {
    console.log("Fetching data from API...");
    return `Data for ${id}`;
  }

  @retry(3, 1000)
  async fetchWithRetry(url: string): Promise<any> {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error("Request failed");
    }
    return response.json();
  }
}

const service = new DataService();
service.fetchData("123");
service.fetchData("123"); // Returns cached result
```

### Function Decorator

```typescript
type AsyncFunction<T = any> = (...args: any[]) => Promise<T>;

// Throttle decorator
function throttle<T extends Function>(func: T, delay: number): T {
  let lastCall = 0;
  let timeoutId: NodeJS.Timeout | null = null;

  return ((...args: any[]) => {
    const now = Date.now();

    if (now - lastCall >= delay) {
      lastCall = now;
      return func(...args);
    } else {
      if (timeoutId) {
        clearTimeout(timeoutId);
      }

      timeoutId = setTimeout(
        () => {
          lastCall = Date.now();
          func(...args);
        },
        delay - (now - lastCall),
      );
    }
  }) as any;
}

// Debounce decorator
function debounce<T extends Function>(func: T, delay: number): T {
  let timeoutId: NodeJS.Timeout;

  return ((...args: any[]) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => func(...args), delay);
  }) as any;
}

// Rate limiter decorator
function rateLimit<T extends AsyncFunction>(
  func: T,
  maxCalls: number,
  timeWindow: number,
): T {
  const calls: number[] = [];

  return (async (...args: any[]) => {
    const now = Date.now();

    // Remove old calls outside time window
    while (calls.length > 0 && calls[0] < now - timeWindow) {
      calls.shift();
    }

    if (calls.length >= maxCalls) {
      throw new Error("Rate limit exceeded");
    }

    calls.push(now);
    return func(...args);
  }) as any;
}

// Cache decorator
function cache<T extends Function>(func: T, ttl: number = 60000): T {
  const cache = new Map<string, { value: any; expiry: number }>();

  return ((...args: any[]) => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached && cached.expiry > Date.now()) {
      console.log("Returning cached value");
      return cached.value;
    }

    const result = func(...args);
    cache.set(key, {
      value: result,
      expiry: Date.now() + ttl,
    });

    return result;
  }) as any;
}

// Usage
const handleSearch = debounce((query: string) => {
  console.log("Searching for:", query);
}, 300);

const handleScroll = throttle(() => {
  console.log("Scrolling...");
}, 100);

const apiCall = rateLimit(
  async (endpoint: string) => {
    const response = await fetch(endpoint);
    return response.json();
  },
  10,
  60000,
);

const expensiveCalculation = cache((n: number) => {
  console.log("Calculating...");
  return n * n;
}, 30000);
```

## Facade Pattern

### Complex API Facade

```typescript
// Complex subsystems
class AuthService {
  async login(username: string, password: string): Promise<string> {
    // Complex authentication logic
    return "auth_token";
  }

  async logout(token: string): Promise<void> {
    // Logout logic
  }

  async validateToken(token: string): Promise<boolean> {
    // Token validation
    return true;
  }
}

class UserService {
  async getProfile(userId: string): Promise<any> {
    // Fetch user profile
    return { id: userId, name: "John" };
  }

  async updateProfile(userId: string, data: any): Promise<void> {
    // Update profile
  }
}

class NotificationService {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // Send email
  }

  async sendPush(userId: string, message: string): Promise<void> {
    // Send push notification
  }
}

class AnalyticsService {
  track(event: string, properties: any): void {
    // Track analytics event
  }
}

// Facade to simplify complex operations
class UserFacade {
  private authService = new AuthService();
  private userService = new UserService();
  private notificationService = new NotificationService();
  private analyticsService = new AnalyticsService();

  async registerUser(
    username: string,
    email: string,
    password: string,
  ): Promise<{ success: boolean; userId?: string; error?: string }> {
    try {
      // 1. Create user account
      const userId = await this.createAccount(username, email, password);

      // 2. Send welcome email
      await this.notificationService.sendEmail(
        email,
        "Welcome!",
        "Thanks for registering",
      );

      // 3. Track registration event
      this.analyticsService.track("user_registered", { userId });

      // 4. Auto-login
      const token = await this.authService.login(username, password);

      return { success: true, userId };
    } catch (error) {
      return { success: false, error: (error as Error).message };
    }
  }

  private async createAccount(
    username: string,
    email: string,
    password: string,
  ): Promise<string> {
    // Account creation logic
    return "user_123";
  }

  async loginUser(
    username: string,
    password: string,
  ): Promise<{ success: boolean; token?: string; profile?: any }> {
    try {
      // 1. Authenticate
      const token = await this.authService.login(username, password);

      // 2. Get profile
      const profile = await this.userService.getProfile("user_123");

      // 3. Track login
      this.analyticsService.track("user_logged_in", { userId: profile.id });

      return { success: true, token, profile };
    } catch (error) {
      return { success: false };
    }
  }

  async updateUserProfile(
    userId: string,
    updates: any,
  ): Promise<{ success: boolean }> {
    try {
      // 1. Update profile
      await this.userService.updateProfile(userId, updates);

      // 2. Send notification
      await this.notificationService.sendPush(
        userId,
        "Profile updated successfully",
      );

      // 3. Track event
      this.analyticsService.track("profile_updated", { userId });

      return { success: true };
    } catch (error) {
      return { success: false };
    }
  }
}

// Usage - simple interface for complex operations
const userFacade = new UserFacade();

// Register with one call instead of coordinating multiple services
const result = await userFacade.registerUser(
  "john_doe",
  "john@example.com",
  "password123",
);

console.log(result);
```

### Payment Processing Facade

```typescript
interface PaymentMethod {
  type: "card" | "paypal" | "crypto";
  details: any;
}

interface PaymentResult {
  success: boolean;
  transactionId?: string;
  error?: string;
}

// Complex payment subsystems
class CardProcessor {
  async charge(amount: number, card: any): Promise<string> {
    console.log(`Processing card payment: $${amount}`);
    return `card_txn_${Date.now()}`;
  }

  async refund(transactionId: string, amount: number): Promise<boolean> {
    console.log(`Refunding card transaction: ${transactionId}`);
    return true;
  }
}

class PayPalProcessor {
  async createPayment(amount: number): Promise<string> {
    console.log(`Creating PayPal payment: $${amount}`);
    return `paypal_txn_${Date.now()}`;
  }

  async executePayment(paymentId: string): Promise<boolean> {
    console.log(`Executing PayPal payment: ${paymentId}`);
    return true;
  }
}

class CryptoProcessor {
  async generateAddress(): Promise<string> {
    return "0x1234567890abcdef";
  }

  async waitForConfirmation(address: string, amount: number): Promise<string> {
    console.log(`Waiting for crypto confirmation: ${address}`);
    return `crypto_txn_${Date.now()}`;
  }
}

class FraudDetection {
  async checkTransaction(
    amount: number,
    method: PaymentMethod,
  ): Promise<boolean> {
    console.log("Running fraud detection...");
    return true; // Not fraudulent
  }
}

class TransactionLogger {
  log(type: string, data: any): void {
    console.log(`[${type}]`, data);
  }
}

// Facade
class PaymentFacade {
  private cardProcessor = new CardProcessor();
  private paypalProcessor = new PayPalProcessor();
  private cryptoProcessor = new CryptoProcessor();
  private fraudDetection = new FraudDetection();
  private logger = new TransactionLogger();

  async processPayment(
    amount: number,
    method: PaymentMethod,
  ): Promise<PaymentResult> {
    try {
      // 1. Check for fraud
      this.logger.log("FRAUD_CHECK", { amount, method });
      const isFraudulent = await this.fraudDetection.checkTransaction(
        amount,
        method,
      );

      if (isFraudulent) {
        return { success: false, error: "Transaction flagged as fraudulent" };
      }

      // 2. Process based on payment method
      let transactionId: string;

      switch (method.type) {
        case "card":
          transactionId = await this.cardProcessor.charge(
            amount,
            method.details,
          );
          break;
        case "paypal":
          const paymentId = await this.paypalProcessor.createPayment(amount);
          await this.paypalProcessor.executePayment(paymentId);
          transactionId = paymentId;
          break;
        case "crypto":
          const address = await this.cryptoProcessor.generateAddress();
          transactionId = await this.cryptoProcessor.waitForConfirmation(
            address,
            amount,
          );
          break;
      }

      // 3. Log success
      this.logger.log("PAYMENT_SUCCESS", { amount, transactionId });

      return { success: true, transactionId };
    } catch (error) {
      this.logger.log("PAYMENT_ERROR", { error });
      return { success: false, error: (error as Error).message };
    }
  }

  async refundPayment(
    transactionId: string,
    amount: number,
    method: PaymentMethod,
  ): Promise<PaymentResult> {
    try {
      switch (method.type) {
        case "card":
          await this.cardProcessor.refund(transactionId, amount);
          break;
        case "paypal":
          // PayPal refund logic
          break;
        case "crypto":
          // Crypto refund logic (if applicable)
          break;
      }

      this.logger.log("REFUND_SUCCESS", { transactionId, amount });
      return { success: true };
    } catch (error) {
      return { success: false, error: (error as Error).message };
    }
  }
}

// Usage
const paymentFacade = new PaymentFacade();

const result = await paymentFacade.processPayment(99.99, {
  type: "card",
  details: {
    number: "4111111111111111",
    expiry: "12/25",
    cvv: "123",
  },
});

console.log(result);
```

## Proxy Pattern

### Lazy Loading Proxy

```typescript
interface ImageComponent {
  render(): string;
  getSize(): number;
}

class RealImage implements ImageComponent {
  private data: string;

  constructor(private filename: string) {
    this.loadFromDisk();
  }

  private loadFromDisk(): void {
    console.log(`Loading image from disk: ${this.filename}`);
    // Simulate expensive operation
    this.data = `[IMAGE DATA: ${this.filename}]`;
  }

  render(): string {
    return `<img src="${this.filename}" />`;
  }

  getSize(): number {
    return this.data.length;
  }
}

class ImageProxy implements ImageComponent {
  private realImage: RealImage | null = null;

  constructor(private filename: string) {}

  private getRealImage(): RealImage {
    if (!this.realImage) {
      this.realImage = new RealImage(this.filename);
    }
    return this.realImage;
  }

  render(): string {
    // Load image only when rendering
    return this.getRealImage().render();
  }

  getSize(): number {
    if (!this.realImage) {
      return 0; // Not loaded yet
    }
    return this.realImage.getSize();
  }

  isLoaded(): boolean {
    return this.realImage !== null;
  }
}

// Usage
const image1 = new ImageProxy("photo1.jpg");
console.log("Image created, but not loaded yet");
console.log(`Is loaded: ${image1.isLoaded()}`);

// Image loads when first accessed
console.log(image1.render());
console.log(`Is loaded: ${image1.isLoaded()}`);
```

### Caching Proxy

```typescript
interface DataService {
  fetchData(key: string): Promise<any>;
}

class RealDataService implements DataService {
  async fetchData(key: string): Promise<any> {
    console.log(`Fetching ${key} from database...`);
    // Simulate database call
    await new Promise((resolve) => setTimeout(resolve, 1000));
    return { key, data: `Data for ${key}`, timestamp: Date.now() };
  }
}

class CachingProxy implements DataService {
  private cache: Map<string, { data: any; expiry: number }> = new Map();
  private realService: RealDataService;

  constructor(private ttl: number = 60000) {
    this.realService = new RealDataService();
  }

  async fetchData(key: string): Promise<any> {
    const cached = this.cache.get(key);

    // Return cached data if valid
    if (cached && cached.expiry > Date.now()) {
      console.log(`Returning cached data for ${key}`);
      return cached.data;
    }

    // Fetch from real service
    const data = await this.realService.fetchData(key);

    // Cache the result
    this.cache.set(key, {
      data,
      expiry: Date.now() + this.ttl,
    });

    return data;
  }

  clearCache(): void {
    this.cache.clear();
  }

  getCacheSize(): number {
    return this.cache.size;
  }
}

// Usage
const dataService = new CachingProxy(30000);

// First call - fetches from database
const data1 = await dataService.fetchData("user:123");

// Second call - returns from cache
const data2 = await dataService.fetchData("user:123");

console.log(data1 === data2); // true
```

### Access Control Proxy

```typescript
interface SecureResource {
  read(): string;
  write(data: string): void;
  delete(): void;
}

class Resource implements SecureResource {
  private data: string = "";

  read(): string {
    return this.data;
  }

  write(data: string): void {
    this.data = data;
  }

  delete(): void {
    this.data = "";
  }
}

type Permission = "read" | "write" | "delete";

class AccessControlProxy implements SecureResource {
  private resource: Resource;

  constructor(
    private user: { id: string; role: string },
    private permissions: Permission[],
  ) {
    this.resource = new Resource();
  }

  private hasPermission(permission: Permission): boolean {
    return this.permissions.includes(permission);
  }

  private checkAccess(permission: Permission): void {
    if (!this.hasPermission(permission)) {
      throw new Error(
        `User ${this.user.id} does not have ${permission} permission`,
      );
    }
  }

  read(): string {
    this.checkAccess("read");
    console.log(`User ${this.user.id} reading resource`);
    return this.resource.read();
  }

  write(data: string): void {
    this.checkAccess("write");
    console.log(`User ${this.user.id} writing to resource`);
    this.resource.write(data);
  }

  delete(): void {
    this.checkAccess("delete");
    console.log(`User ${this.user.id} deleting resource`);
    this.resource.delete();
  }
}

// Usage
const admin = new AccessControlProxy({ id: "admin1", role: "admin" }, [
  "read",
  "write",
  "delete",
]);

const reader = new AccessControlProxy({ id: "user1", role: "user" }, ["read"]);

admin.write("Secret data");
console.log(admin.read());

console.log(reader.read()); // OK
// reader.write('data'); // Throws error
```

## Composite Pattern

### UI Component Tree

```typescript
interface UIComponent {
  render(): string;
  add?(component: UIComponent): void;
  remove?(component: UIComponent): void;
  getChild?(index: number): UIComponent | null;
}

// Leaf components
class Button implements UIComponent {
  constructor(
    private label: string,
    private onClick: () => void,
  ) {}

  render(): string {
    return `<button onclick="${this.onClick}">${this.label}</button>`;
  }
}

class Input implements UIComponent {
  constructor(
    private type: string,
    private placeholder: string,
  ) {}

  render(): string {
    return `<input type="${this.type}" placeholder="${this.placeholder}" />`;
  }
}

class Text implements UIComponent {
  constructor(private content: string) {}

  render(): string {
    return `<span>${this.content}</span>`;
  }
}

// Composite components
class Container implements UIComponent {
  private children: UIComponent[] = [];

  constructor(private className: string = "") {}

  add(component: UIComponent): void {
    this.children.push(component);
  }

  remove(component: UIComponent): void {
    const index = this.children.indexOf(component);
    if (index > -1) {
      this.children.splice(index, 1);
    }
  }

  getChild(index: number): UIComponent | null {
    return this.children[index] ?? null;
  }

  render(): string {
    const childrenHtml = this.children.map((child) => child.render()).join("");
    return `<div class="${this.className}">${childrenHtml}</div>`;
  }
}

class Form implements UIComponent {
  private children: UIComponent[] = [];

  constructor(private onSubmit: () => void) {}

  add(component: UIComponent): void {
    this.children.push(component);
  }

  remove(component: UIComponent): void {
    const index = this.children.indexOf(component);
    if (index > -1) {
      this.children.splice(index, 1);
    }
  }

  render(): string {
    const childrenHtml = this.children.map((child) => child.render()).join("");
    return `<form onsubmit="${this.onSubmit}">${childrenHtml}</form>`;
  }
}

// Usage - build complex UI tree
const loginForm = new Form(() => console.log("Login submitted"));

const headerContainer = new Container("form-header");
headerContainer.add(new Text("Login"));

const inputContainer = new Container("form-inputs");
inputContainer.add(new Input("email", "Enter email"));
inputContainer.add(new Input("password", "Enter password"));

const buttonContainer = new Container("form-actions");
buttonContainer.add(new Button("Login", () => console.log("Login clicked")));
buttonContainer.add(new Button("Cancel", () => console.log("Cancel clicked")));

loginForm.add(headerContainer);
loginForm.add(inputContainer);
loginForm.add(buttonContainer);

console.log(loginForm.render());
```

### File System Composite

```typescript
interface FileSystemComponent {
  getName(): string;
  getSize(): number;
  print(indent?: string): void;
}

class File implements FileSystemComponent {
  constructor(
    private name: string,
    private size: number,
  ) {}

  getName(): string {
    return this.name;
  }

  getSize(): number {
    return this.size;
  }

  print(indent: string = ""): void {
    console.log(`${indent}📄 ${this.name} (${this.size} bytes)`);
  }
}

class Directory implements FileSystemComponent {
  private children: FileSystemComponent[] = [];

  constructor(private name: string) {}

  getName(): string {
    return this.name;
  }

  getSize(): number {
    return this.children.reduce((total, child) => total + child.getSize(), 0);
  }

  add(component: FileSystemComponent): void {
    this.children.push(component);
  }

  remove(component: FileSystemComponent): void {
    const index = this.children.indexOf(component);
    if (index > -1) {
      this.children.splice(index, 1);
    }
  }

  getChildren(): FileSystemComponent[] {
    return [...this.children];
  }

  print(indent: string = ""): void {
    console.log(`${indent}📁 ${this.name} (${this.getSize()} bytes)`);
    this.children.forEach((child) => child.print(indent + "  "));
  }
}

// Usage
const root = new Directory("root");

const documents = new Directory("documents");
documents.add(new File("resume.pdf", 50000));
documents.add(new File("cover-letter.pdf", 25000));

const photos = new Directory("photos");
photos.add(new File("vacation1.jpg", 200000));
photos.add(new File("vacation2.jpg", 180000));

const videos = new Directory("videos");
videos.add(new File("tutorial.mp4", 5000000));

root.add(documents);
root.add(photos);
root.add(videos);
root.add(new File("readme.txt", 1000));

root.print();
console.log(`Total size: ${root.getSize()} bytes`);
```

## Bridge Pattern

### UI Theme Bridge

```typescript
// Implementation interface
interface Theme {
  getBackgroundColor(): string;
  getTextColor(): string;
  getBorderColor(): string;
  getFontFamily(): string;
}

// Concrete implementations
class LightTheme implements Theme {
  getBackgroundColor(): string {
    return "#ffffff";
  }

  getTextColor(): string {
    return "#000000";
  }

  getBorderColor(): string {
    return "#cccccc";
  }

  getFontFamily(): string {
    return "Arial, sans-serif";
  }
}

class DarkTheme implements Theme {
  getBackgroundColor(): string {
    return "#1a1a1a";
  }

  getTextColor(): string {
    return "#ffffff";
  }

  getBorderColor(): string {
    return "#444444";
  }

  getFontFamily(): string {
    return "Arial, sans-serif";
  }
}

class HighContrastTheme implements Theme {
  getBackgroundColor(): string {
    return "#000000";
  }

  getTextColor(): string {
    return "#ffff00";
  }

  getBorderColor(): string {
    return "#ffffff";
  }

  getFontFamily(): string {
    return "Arial, sans-serif";
  }
}

// Abstraction
abstract class UIElement {
  constructor(protected theme: Theme) {}

  abstract render(): string;

  setTheme(theme: Theme): void {
    this.theme = theme;
  }
}

// Refined abstractions
class ThemedButton extends UIElement {
  constructor(
    theme: Theme,
    private label: string,
  ) {
    super(theme);
  }

  render(): string {
    return `
      <button style="
        background-color: ${this.theme.getBackgroundColor()};
        color: ${this.theme.getTextColor()};
        border: 1px solid ${this.theme.getBorderColor()};
        font-family: ${this.theme.getFontFamily()};
      ">
        ${this.label}
      </button>
    `;
  }
}

class ThemedPanel extends UIElement {
  constructor(
    theme: Theme,
    private content: string,
  ) {
    super(theme);
  }

  render(): string {
    return `
      <div style="
        background-color: ${this.theme.getBackgroundColor()};
        color: ${this.theme.getTextColor()};
        border: 2px solid ${this.theme.getBorderColor()};
        font-family: ${this.theme.getFontFamily()};
        padding: 20px;
      ">
        ${this.content}
      </div>
    `;
  }
}

// Usage
const lightTheme = new LightTheme();
const darkTheme = new DarkTheme();

const button = new ThemedButton(lightTheme, "Click me");
console.log(button.render());

// Switch theme at runtime
button.setTheme(darkTheme);
console.log(button.render());

const panel = new ThemedPanel(darkTheme, "Panel content");
console.log(panel.render());
```

## Real-World Usage

### Complete E-commerce Checkout Flow

```typescript
// Uses multiple structural patterns together
class CheckoutFacade {
  private cartProxy: ShoppingCartProxy;
  private paymentAdapter: PaymentAdapter;
  private orderDecorator: OrderDecorator;

  constructor(
    userId: string,
    private theme: Theme,
  ) {
    this.cartProxy = new ShoppingCartProxy(userId);
    this.paymentAdapter = new PaymentAdapter(new LegacyPaymentAPI());
    this.orderDecorator = new OrderDecorator();
  }

  async checkout(
    paymentMethod: PaymentMethod,
    shippingAddress: Address,
  ): Promise<CheckoutResult> {
    try {
      // 1. Get cart items (via proxy with caching)
      const items = await this.cartProxy.getItems();

      // 2. Calculate total
      const total = items.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0,
      );

      // 3. Process payment (via adapter)
      const paymentResult = await this.paymentAdapter.charge({
        amount: total,
        currency: "USD",
        paymentMethod,
      });

      if (!paymentResult.success) {
        throw new Error("Payment failed");
      }

      // 4. Create order with decorators (logging, notifications, etc.)
      const order = await this.orderDecorator.createOrder({
        items,
        total,
        transactionId: paymentResult.transactionId!,
        shippingAddress,
      });

      // 5. Clear cart
      await this.cartProxy.clear();

      return {
        success: true,
        orderId: order.id,
        transactionId: paymentResult.transactionId!,
      };
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message,
      };
    }
  }

  renderCheckoutUI(): string {
    const button = new ThemedButton(this.theme, "Complete Purchase");
    const panel = new ThemedPanel(this.theme, "Checkout Summary");
    return `${panel.render()}${button.render()}`;
  }
}
```

## Production Patterns

### Memoized Component Adapter

```typescript
interface Props {
  data: any;
  onUpdate: (data: any) => void;
}

function withMemoization<P extends object>(
  Component: React.ComponentType<P>
): React.FC<P> {
  return React.memo((props) => {
    return <Component {...props} />;
  }, (prevProps, nextProps) => {
    // Custom comparison logic
    return JSON.stringify(prevProps) === JSON.stringify(nextProps);
  });
}

// Combining adapter with memoization
const MemoizedUserList = withMemoization(
  ({ users }: { users: InternalUser[] }) => {
    return (
      <div>
        {users.map(user => (
          <div key={user.id}>{user.name}</div>
        ))}
      </div>
    );
  }
);

// Using adapter to transform API data
const UserListContainer = ({ externalUsers }: { externalUsers: ExternalUser[] }) => {
  const internalUsers = React.useMemo(
    () => UserAdapter.toInternalList(externalUsers),
    [externalUsers]
  );

  return <MemoizedUserList users={internalUsers} />;
};
```

## Best Practices

1. **Use Adapter for incompatible interfaces** - Don't modify existing code, adapt it
2. **Decorator for adding responsibilities** - Add behavior without changing original code
3. **Facade for complex subsystems** - Provide simple interface to complex systems
4. **Proxy for controlled access** - Add lazy loading, caching, or access control
5. **Composite for tree structures** - Treat individual and composite objects uniformly
6. **Bridge for separating abstraction from implementation** - Allow independent variation
7. **Keep adapters thin** - Only transform data, don't add business logic
8. **Chain decorators carefully** - Order matters, especially with error boundaries
9. **Cache invalidation in proxies** - Implement proper cache expiry strategies
10. **Type safety** - Use TypeScript interfaces to ensure pattern implementations are correct

## 10 Key Takeaways

1. **Adapter Pattern** converts incompatible interfaces, essential for integrating third-party APIs
2. **Decorator Pattern** adds behavior dynamically, perfect for HOCs and middleware
3. **Facade Pattern** simplifies complex subsystems with unified interface
4. **Proxy Pattern** controls access to objects, enables lazy loading and caching
5. **Composite Pattern** builds tree structures, treats leaves and branches uniformly
6. **Bridge Pattern** separates abstraction from implementation for flexibility
7. **Structural patterns** focus on relationships and composition between objects
8. **Combine patterns** for complex scenarios (Facade + Adapter + Proxy)
9. **React HOCs** are decorators, composition is composite pattern
10. **Choose wisely** - use structural patterns to manage complexity, not create it
