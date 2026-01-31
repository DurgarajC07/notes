# Creational Design Patterns in Frontend Development

## Core Concepts

Creational patterns deal with object creation mechanisms, trying to create objects in a manner suitable to the situation. These patterns provide flexibility in what gets created, who creates it, how it gets created, and when. In frontend development, they help manage component instantiation, configuration management, and object pooling for performance optimization.

## Factory Pattern

### Basic Component Factory

```typescript
// Component types
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger';
}

interface InputProps {
  value: string;
  onChange: (value: string) => void;
  type?: 'text' | 'email' | 'password';
}

interface SelectProps {
  value: string;
  options: Array<{ label: string; value: string }>;
  onChange: (value: string) => void;
}

// Factory function
type ComponentType = 'button' | 'input' | 'select';

interface ComponentConfig {
  type: ComponentType;
  props: ButtonProps | InputProps | SelectProps;
}

class ComponentFactory {
  static create(config: ComponentConfig): JSX.Element {
    switch (config.type) {
      case 'button':
        return <Button {...(config.props as ButtonProps)} />;
      case 'input':
        return <Input {...(config.props as InputProps)} />;
      case 'select':
        return <Select {...(config.props as SelectProps)} />;
      default:
        throw new Error(`Unknown component type: ${config.type}`);
    }
  }
}

// Usage
const components = [
  { type: 'button' as const, props: { label: 'Submit', onClick: () => {} } },
  { type: 'input' as const, props: { value: '', onChange: () => {} } },
];

const DynamicForm = () => (
  <form>
    {components.map((config, index) => (
      <div key={index}>{ComponentFactory.create(config)}</div>
    ))}
  </form>
);
```

### Advanced Factory with Type Guards

```typescript
interface ShapeConfig {
  type: "circle" | "rectangle" | "triangle";
  color: string;
}

interface CircleConfig extends ShapeConfig {
  type: "circle";
  radius: number;
}

interface RectangleConfig extends ShapeConfig {
  type: "rectangle";
  width: number;
  height: number;
}

interface TriangleConfig extends ShapeConfig {
  type: "triangle";
  base: number;
  height: number;
}

type AnyShapeConfig = CircleConfig | RectangleConfig | TriangleConfig;

abstract class Shape {
  constructor(protected color: string) {}
  abstract draw(): void;
  abstract getArea(): number;
}

class Circle extends Shape {
  constructor(
    color: string,
    private radius: number,
  ) {
    super(color);
  }

  draw(): void {
    console.log(`Drawing ${this.color} circle with radius ${this.radius}`);
  }

  getArea(): number {
    return Math.PI * this.radius ** 2;
  }
}

class Rectangle extends Shape {
  constructor(
    color: string,
    private width: number,
    private height: number,
  ) {
    super(color);
  }

  draw(): void {
    console.log(`Drawing ${this.color} rectangle ${this.width}x${this.height}`);
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Triangle extends Shape {
  constructor(
    color: string,
    private base: number,
    private height: number,
  ) {
    super(color);
  }

  draw(): void {
    console.log(`Drawing ${this.color} triangle`);
  }

  getArea(): number {
    return (this.base * this.height) / 2;
  }
}

class ShapeFactory {
  static create(config: AnyShapeConfig): Shape {
    switch (config.type) {
      case "circle":
        return new Circle(config.color, config.radius);
      case "rectangle":
        return new Rectangle(config.color, config.width, config.height);
      case "triangle":
        return new Triangle(config.color, config.base, config.height);
    }
  }
}

// Usage
const shapes: AnyShapeConfig[] = [
  { type: "circle", color: "red", radius: 10 },
  { type: "rectangle", color: "blue", width: 20, height: 10 },
  { type: "triangle", color: "green", base: 15, height: 12 },
];

shapes.forEach((config) => {
  const shape = ShapeFactory.create(config);
  shape.draw();
  console.log(`Area: ${shape.getArea()}`);
});
```

### HTTP Client Factory

```typescript
interface RequestConfig {
  baseURL: string;
  timeout: number;
  headers: Record<string, string>;
}

interface HttpClient {
  get<T>(url: string, config?: Partial<RequestConfig>): Promise<T>;
  post<T>(url: string, data: any, config?: Partial<RequestConfig>): Promise<T>;
  put<T>(url: string, data: any, config?: Partial<RequestConfig>): Promise<T>;
  delete<T>(url: string, config?: Partial<RequestConfig>): Promise<T>;
}

class FetchClient implements HttpClient {
  constructor(private config: RequestConfig) {}

  private async request<T>(
    method: string,
    url: string,
    data?: any,
    config?: Partial<RequestConfig>,
  ): Promise<T> {
    const mergedConfig = { ...this.config, ...config };
    const fullUrl = `${mergedConfig.baseURL}${url}`;

    const controller = new AbortController();
    const timeoutId = setTimeout(
      () => controller.abort(),
      mergedConfig.timeout,
    );

    try {
      const response = await fetch(fullUrl, {
        method,
        headers: mergedConfig.headers,
        body: data ? JSON.stringify(data) : undefined,
        signal: controller.signal,
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      return await response.json();
    } finally {
      clearTimeout(timeoutId);
    }
  }

  get<T>(url: string, config?: Partial<RequestConfig>): Promise<T> {
    return this.request<T>("GET", url, undefined, config);
  }

  post<T>(url: string, data: any, config?: Partial<RequestConfig>): Promise<T> {
    return this.request<T>("POST", url, data, config);
  }

  put<T>(url: string, data: any, config?: Partial<RequestConfig>): Promise<T> {
    return this.request<T>("PUT", url, data, config);
  }

  delete<T>(url: string, config?: Partial<RequestConfig>): Promise<T> {
    return this.request<T>("DELETE", url, undefined, config);
  }
}

class AxiosClient implements HttpClient {
  constructor(private config: RequestConfig) {}

  // Implementation using axios library
  async get<T>(url: string, config?: Partial<RequestConfig>): Promise<T> {
    // axios implementation
    throw new Error("Not implemented");
  }

  async post<T>(
    url: string,
    data: any,
    config?: Partial<RequestConfig>,
  ): Promise<T> {
    throw new Error("Not implemented");
  }

  async put<T>(
    url: string,
    data: any,
    config?: Partial<RequestConfig>,
  ): Promise<T> {
    throw new Error("Not implemented");
  }

  async delete<T>(url: string, config?: Partial<RequestConfig>): Promise<T> {
    throw new Error("Not implemented");
  }
}

type HttpClientType = "fetch" | "axios";

class HttpClientFactory {
  static create(type: HttpClientType, config: RequestConfig): HttpClient {
    switch (type) {
      case "fetch":
        return new FetchClient(config);
      case "axios":
        return new AxiosClient(config);
      default:
        throw new Error(`Unknown HTTP client type: ${type}`);
    }
  }
}

// Usage
const apiClient = HttpClientFactory.create("fetch", {
  baseURL: "https://api.example.com",
  timeout: 5000,
  headers: {
    "Content-Type": "application/json",
    Authorization: "Bearer token123",
  },
});

interface User {
  id: number;
  name: string;
  email: string;
}

const user = await apiClient.get<User>("/users/1");
```

## Abstract Factory Pattern

### Theme Factory

```typescript
interface Button {
  render(): string;
  onClick(): void;
}

interface Input {
  render(): string;
  getValue(): string;
}

interface Card {
  render(): string;
  getContent(): string;
}

interface UIFactory {
  createButton(): Button;
  createInput(): Input;
  createCard(): Card;
}

// Light Theme Components
class LightButton implements Button {
  render(): string {
    return '<button class="light-button">Click me</button>';
  }

  onClick(): void {
    console.log("Light button clicked");
  }
}

class LightInput implements Input {
  private value = "";

  render(): string {
    return '<input class="light-input" />';
  }

  getValue(): string {
    return this.value;
  }
}

class LightCard implements Card {
  render(): string {
    return '<div class="light-card">Light card content</div>';
  }

  getContent(): string {
    return "Light card content";
  }
}

// Dark Theme Components
class DarkButton implements Button {
  render(): string {
    return '<button class="dark-button">Click me</button>';
  }

  onClick(): void {
    console.log("Dark button clicked");
  }
}

class DarkInput implements Input {
  private value = "";

  render(): string {
    return '<input class="dark-input" />';
  }

  getValue(): string {
    return this.value;
  }
}

class DarkCard implements Card {
  render(): string {
    return '<div class="dark-card">Dark card content</div>';
  }

  getContent(): string {
    return "Dark card content";
  }
}

// Factories
class LightThemeFactory implements UIFactory {
  createButton(): Button {
    return new LightButton();
  }

  createInput(): Input {
    return new LightInput();
  }

  createCard(): Card {
    return new LightCard();
  }
}

class DarkThemeFactory implements UIFactory {
  createButton(): Button {
    return new DarkButton();
  }

  createInput(): Input {
    return new DarkInput();
  }

  createCard(): Card {
    return new DarkCard();
  }
}

// Application
class Application {
  private button: Button;
  private input: Input;
  private card: Card;

  constructor(factory: UIFactory) {
    this.button = factory.createButton();
    this.input = factory.createInput();
    this.card = factory.createCard();
  }

  render(): string {
    return `
      ${this.button.render()}
      ${this.input.render()}
      ${this.card.render()}
    `;
  }
}

// Usage
const isDarkMode = window.matchMedia("(prefers-color-scheme: dark)").matches;
const factory: UIFactory = isDarkMode
  ? new DarkThemeFactory()
  : new LightThemeFactory();

const app = new Application(factory);
console.log(app.render());
```

### Cross-Platform UI Factory

```typescript
interface PlatformButton {
  render(): JSX.Element;
  getClassName(): string;
}

interface PlatformModal {
  render(): JSX.Element;
  open(): void;
  close(): void;
}

interface PlatformFactory {
  createButton(props: { label: string; onClick: () => void }): PlatformButton;
  createModal(props: { title: string; content: string }): PlatformModal;
}

// Web Platform
class WebButton implements PlatformButton {
  constructor(private props: { label: string; onClick: () => void }) {}

  render(): JSX.Element {
    return (
      <button
        className={this.getClassName()}
        onClick={this.props.onClick}
      >
        {this.props.label}
      </button>
    );
  }

  getClassName(): string {
    return 'web-button';
  }
}

class WebModal implements PlatformModal {
  constructor(private props: { title: string; content: string }) {}

  render(): JSX.Element {
    return (
      <div className="web-modal">
        <h2>{this.props.title}</h2>
        <p>{this.props.content}</p>
      </div>
    );
  }

  open(): void {
    console.log('Opening web modal');
  }

  close(): void {
    console.log('Closing web modal');
  }
}

// Mobile Platform (React Native style)
class MobileButton implements PlatformButton {
  constructor(private props: { label: string; onClick: () => void }) {}

  render(): JSX.Element {
    return (
      <TouchableOpacity onPress={this.props.onClick}>
        <Text className={this.getClassName()}>{this.props.label}</Text>
      </TouchableOpacity>
    );
  }

  getClassName(): string {
    return 'mobile-button';
  }
}

class MobileModal implements PlatformModal {
  constructor(private props: { title: string; content: string }) {}

  render(): JSX.Element {
    return (
      <View className="mobile-modal">
        <Text>{this.props.title}</Text>
        <Text>{this.props.content}</Text>
      </View>
    );
  }

  open(): void {
    console.log('Opening mobile modal');
  }

  close(): void {
    console.log('Closing mobile modal');
  }
}

// Factories
class WebPlatformFactory implements PlatformFactory {
  createButton(props: { label: string; onClick: () => void }): PlatformButton {
    return new WebButton(props);
  }

  createModal(props: { title: string; content: string }): PlatformModal {
    return new WebModal(props);
  }
}

class MobilePlatformFactory implements PlatformFactory {
  createButton(props: { label: string; onClick: () => void }): PlatformButton {
    return new MobileButton(props);
  }

  createModal(props: { title: string; content: string }): PlatformModal {
    return new MobileModal(props);
  }
}

// Platform detection and usage
const getPlatformFactory = (): PlatformFactory => {
  const isMobile = /Android|webOS|iPhone|iPad|iPod/i.test(navigator.userAgent);
  return isMobile ? new MobilePlatformFactory() : new WebPlatformFactory();
};

const factory = getPlatformFactory();
const button = factory.createButton({
  label: 'Submit',
  onClick: () => console.log('Clicked')
});
```

## Builder Pattern

### Query Builder

```typescript
interface QueryParams {
  select?: string[];
  where?: Record<string, any>;
  orderBy?: Array<{ field: string; direction: "asc" | "desc" }>;
  limit?: number;
  offset?: number;
  join?: Array<{ table: string; on: string }>;
}

class QueryBuilder {
  private params: QueryParams = {};

  select(...fields: string[]): this {
    this.params.select = fields;
    return this;
  }

  where(conditions: Record<string, any>): this {
    this.params.where = { ...this.params.where, ...conditions };
    return this;
  }

  orderBy(field: string, direction: "asc" | "desc" = "asc"): this {
    if (!this.params.orderBy) {
      this.params.orderBy = [];
    }
    this.params.orderBy.push({ field, direction });
    return this;
  }

  limit(count: number): this {
    this.params.limit = count;
    return this;
  }

  offset(count: number): this {
    this.params.offset = count;
    return this;
  }

  join(table: string, on: string): this {
    if (!this.params.join) {
      this.params.join = [];
    }
    this.params.join.push({ table, on });
    return this;
  }

  build(): string {
    let query = "SELECT ";

    // SELECT clause
    if (this.params.select && this.params.select.length > 0) {
      query += this.params.select.join(", ");
    } else {
      query += "*";
    }

    query += " FROM table_name";

    // JOIN clause
    if (this.params.join) {
      this.params.join.forEach(({ table, on }) => {
        query += ` JOIN ${table} ON ${on}`;
      });
    }

    // WHERE clause
    if (this.params.where) {
      const conditions = Object.entries(this.params.where)
        .map(([key, value]) => `${key} = '${value}'`)
        .join(" AND ");
      query += ` WHERE ${conditions}`;
    }

    // ORDER BY clause
    if (this.params.orderBy) {
      const orderClauses = this.params.orderBy
        .map(({ field, direction }) => `${field} ${direction.toUpperCase()}`)
        .join(", ");
      query += ` ORDER BY ${orderClauses}`;
    }

    // LIMIT clause
    if (this.params.limit !== undefined) {
      query += ` LIMIT ${this.params.limit}`;
    }

    // OFFSET clause
    if (this.params.offset !== undefined) {
      query += ` OFFSET ${this.params.offset}`;
    }

    return query;
  }

  async execute<T>(): Promise<T[]> {
    const query = this.build();
    console.log("Executing query:", query);
    // Execute the query against database
    return [];
  }
}

// Usage
const users = await new QueryBuilder()
  .select("id", "name", "email")
  .where({ status: "active", role: "admin" })
  .orderBy("created_at", "desc")
  .limit(10)
  .offset(0)
  .execute<User>();

const complexQuery = new QueryBuilder()
  .select("users.name", "posts.title", "posts.created_at")
  .join("posts", "users.id = posts.user_id")
  .where({ "users.status": "active" })
  .orderBy("posts.created_at", "desc")
  .limit(20)
  .build();
```

### Form Builder

```typescript
interface FormField {
  name: string;
  type: 'text' | 'email' | 'password' | 'number' | 'select' | 'checkbox';
  label: string;
  value: any;
  placeholder?: string;
  required?: boolean;
  validation?: (value: any) => string | null;
  options?: Array<{ label: string; value: string }>;
}

interface FormConfig {
  fields: FormField[];
  onSubmit: (data: Record<string, any>) => void;
  submitLabel?: string;
  resetLabel?: string;
}

class FormBuilder {
  private config: Partial<FormConfig> = {
    fields: [],
    submitLabel: 'Submit',
    resetLabel: 'Reset',
  };

  addTextField(
    name: string,
    label: string,
    options: {
      placeholder?: string;
      required?: boolean;
      validation?: (value: string) => string | null;
    } = {}
  ): this {
    this.config.fields!.push({
      name,
      type: 'text',
      label,
      value: '',
      ...options,
    });
    return this;
  }

  addEmailField(
    name: string,
    label: string,
    options: {
      placeholder?: string;
      required?: boolean;
    } = {}
  ): this {
    this.config.fields!.push({
      name,
      type: 'email',
      label,
      value: '',
      validation: (value: string) => {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(value) ? null : 'Invalid email address';
      },
      ...options,
    });
    return this;
  }

  addPasswordField(
    name: string,
    label: string,
    options: {
      placeholder?: string;
      required?: boolean;
      minLength?: number;
    } = {}
  ): this {
    const { minLength = 8, ...rest } = options;
    this.config.fields!.push({
      name,
      type: 'password',
      label,
      value: '',
      validation: (value: string) => {
        return value.length >= minLength
          ? null
          : `Password must be at least ${minLength} characters`;
      },
      ...rest,
    });
    return this;
  }

  addSelectField(
    name: string,
    label: string,
    options: Array<{ label: string; value: string }>,
    config: {
      required?: boolean;
    } = {}
  ): this {
    this.config.fields!.push({
      name,
      type: 'select',
      label,
      value: '',
      options,
      ...config,
    });
    return this;
  }

  addCheckboxField(
    name: string,
    label: string,
    options: {
      required?: boolean;
    } = {}
  ): this {
    this.config.fields!.push({
      name,
      type: 'checkbox',
      label,
      value: false,
      ...options,
    });
    return this;
  }

  onSubmit(handler: (data: Record<string, any>) => void): this {
    this.config.onSubmit = handler;
    return this;
  }

  setSubmitLabel(label: string): this {
    this.config.submitLabel = label;
    return this;
  }

  build(): FormConfig {
    if (!this.config.onSubmit) {
      throw new Error('onSubmit handler is required');
    }
    return this.config as FormConfig;
  }

  buildComponent(): React.FC {
    const config = this.build();

    return () => {
      const [formData, setFormData] = React.useState<Record<string, any>>(
        config.fields.reduce((acc, field) => ({
          ...acc,
          [field.name]: field.value,
        }), {})
      );

      const [errors, setErrors] = React.useState<Record<string, string>>({});

      const handleChange = (name: string, value: any) => {
        setFormData(prev => ({ ...prev, [name]: value }));

        const field = config.fields.find(f => f.name === name);
        if (field?.validation) {
          const error = field.validation(value);
          setErrors(prev => ({
            ...prev,
            [name]: error || '',
          }));
        }
      };

      const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();

        const newErrors: Record<string, string> = {};
        config.fields.forEach(field => {
          if (field.required && !formData[field.name]) {
            newErrors[field.name] = 'This field is required';
          } else if (field.validation) {
            const error = field.validation(formData[field.name]);
            if (error) newErrors[field.name] = error;
          }
        });

        if (Object.keys(newErrors).length > 0) {
          setErrors(newErrors);
          return;
        }

        config.onSubmit(formData);
      };

      return (
        <form onSubmit={handleSubmit}>
          {config.fields.map(field => (
            <div key={field.name}>
              <label>{field.label}</label>
              {field.type === 'select' ? (
                <select
                  value={formData[field.name]}
                  onChange={e => handleChange(field.name, e.target.value)}
                >
                  {field.options?.map(opt => (
                    <option key={opt.value} value={opt.value}>
                      {opt.label}
                    </option>
                  ))}
                </select>
              ) : field.type === 'checkbox' ? (
                <input
                  type="checkbox"
                  checked={formData[field.name]}
                  onChange={e => handleChange(field.name, e.target.checked)}
                />
              ) : (
                <input
                  type={field.type}
                  value={formData[field.name]}
                  placeholder={field.placeholder}
                  onChange={e => handleChange(field.name, e.target.value)}
                />
              )}
              {errors[field.name] && (
                <span className="error">{errors[field.name]}</span>
              )}
            </div>
          ))}
          <button type="submit">{config.submitLabel}</button>
        </form>
      );
    };
  }
}

// Usage
const RegistrationForm = new FormBuilder()
  .addTextField('username', 'Username', {
    placeholder: 'Enter username',
    required: true,
    validation: (value: string) =>
      value.length >= 3 ? null : 'Username must be at least 3 characters',
  })
  .addEmailField('email', 'Email', {
    placeholder: 'Enter email',
    required: true,
  })
  .addPasswordField('password', 'Password', {
    placeholder: 'Enter password',
    required: true,
    minLength: 8,
  })
  .addSelectField('country', 'Country', [
    { label: 'USA', value: 'us' },
    { label: 'UK', value: 'uk' },
    { label: 'Canada', value: 'ca' },
  ], { required: true })
  .addCheckboxField('terms', 'I agree to terms and conditions', {
    required: true,
  })
  .onSubmit(data => {
    console.log('Form submitted:', data);
  })
  .setSubmitLabel('Register')
  .buildComponent();
```

## Singleton Pattern

### Configuration Manager

```typescript
interface AppConfig {
  apiUrl: string;
  apiKey: string;
  timeout: number;
  retryAttempts: number;
  environment: "development" | "staging" | "production";
  features: {
    darkMode: boolean;
    analytics: boolean;
    notifications: boolean;
  };
}

class ConfigurationManager {
  private static instance: ConfigurationManager;
  private config: AppConfig;

  private constructor() {
    // Initialize with default configuration
    this.config = {
      apiUrl: process.env.REACT_APP_API_URL || "http://localhost:3000",
      apiKey: process.env.REACT_APP_API_KEY || "",
      timeout: 5000,
      retryAttempts: 3,
      environment: (process.env.NODE_ENV as any) || "development",
      features: {
        darkMode: true,
        analytics: false,
        notifications: true,
      },
    };
  }

  static getInstance(): ConfigurationManager {
    if (!ConfigurationManager.instance) {
      ConfigurationManager.instance = new ConfigurationManager();
    }
    return ConfigurationManager.instance;
  }

  getConfig(): Readonly<AppConfig> {
    return Object.freeze({ ...this.config });
  }

  updateConfig(updates: Partial<AppConfig>): void {
    this.config = { ...this.config, ...updates };
  }

  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    return this.config[key];
  }

  set<K extends keyof AppConfig>(key: K, value: AppConfig[K]): void {
    this.config[key] = value;
  }

  isProduction(): boolean {
    return this.config.environment === "production";
  }

  isDevelopment(): boolean {
    return this.config.environment === "development";
  }

  getApiUrl(): string {
    return this.config.apiUrl;
  }

  getTimeout(): number {
    return this.config.timeout;
  }
}

// Usage
const config = ConfigurationManager.getInstance();
console.log(config.getApiUrl());
console.log(config.get("features").darkMode);

// Attempting to create another instance returns the same instance
const config2 = ConfigurationManager.getInstance();
console.log(config === config2); // true
```

### Global Store Singleton

```typescript
type Listener<T> = (state: T) => void;

class Store<T extends Record<string, any>> {
  private static instances: Map<string, Store<any>> = new Map();
  private state: T;
  private listeners: Set<Listener<T>> = new Set();

  private constructor(
    private name: string,
    initialState: T,
  ) {
    this.state = initialState;
  }

  static getInstance<T extends Record<string, any>>(
    name: string,
    initialState: T,
  ): Store<T> {
    if (!Store.instances.has(name)) {
      Store.instances.set(name, new Store(name, initialState));
    }
    return Store.instances.get(name)!;
  }

  getState(): Readonly<T> {
    return Object.freeze({ ...this.state });
  }

  setState(updates: Partial<T> | ((state: T) => Partial<T>)): void {
    const newState =
      typeof updates === "function" ? updates(this.state) : updates;

    this.state = { ...this.state, ...newState };
    this.notifyListeners();
  }

  subscribe(listener: Listener<T>): () => void {
    this.listeners.add(listener);
    return () => {
      this.listeners.delete(listener);
    };
  }

  private notifyListeners(): void {
    this.listeners.forEach((listener) => listener(this.state));
  }

  reset(newState: T): void {
    this.state = newState;
    this.notifyListeners();
  }
}

// Usage
interface UserState {
  id: number | null;
  name: string;
  email: string;
  isAuthenticated: boolean;
}

const userStore = Store.getInstance<UserState>("user", {
  id: null,
  name: "",
  email: "",
  isAuthenticated: false,
});

// Subscribe to changes
const unsubscribe = userStore.subscribe((state) => {
  console.log("User state changed:", state);
});

// Update state
userStore.setState({
  id: 1,
  name: "John Doe",
  email: "john@example.com",
  isAuthenticated: true,
});

// Get current state
const currentUser = userStore.getState();
console.log(currentUser);

// Unsubscribe
unsubscribe();
```

## Prototype Pattern

### Component Cloning

```typescript
interface Cloneable<T> {
  clone(): T;
}

interface ComponentProps {
  id: string;
  type: string;
  styles: Record<string, string>;
  children: Component[];
}

class Component implements Cloneable<Component> {
  constructor(
    public id: string,
    public type: string,
    public styles: Record<string, string> = {},
    public children: Component[] = [],
  ) {}

  clone(): Component {
    // Deep clone children
    const clonedChildren = this.children.map((child) => child.clone());

    // Create new component with cloned data
    return new Component(
      `${this.id}-copy`,
      this.type,
      { ...this.styles },
      clonedChildren,
    );
  }

  addChild(child: Component): void {
    this.children.push(child);
  }

  setStyle(property: string, value: string): void {
    this.styles[property] = value;
  }

  render(): string {
    const childrenHtml = this.children.map((child) => child.render()).join("");
    const styles = Object.entries(this.styles)
      .map(([key, value]) => `${key}: ${value}`)
      .join("; ");

    return `<${this.type} id="${this.id}" style="${styles}">${childrenHtml}</${this.type}>`;
  }
}

// Usage
const button = new Component("btn-1", "button");
button.setStyle("background-color", "blue");
button.setStyle("color", "white");
button.setStyle("padding", "10px 20px");

const buttonClone = button.clone();
buttonClone.setStyle("background-color", "red");

console.log(button.render());
console.log(buttonClone.render());

// Complex component tree
const card = new Component("card-1", "div");
card.setStyle("border", "1px solid #ccc");
card.setStyle("padding", "20px");

const header = new Component("header-1", "h2");
const content = new Component("content-1", "p");

card.addChild(header);
card.addChild(content);

// Clone entire card with all children
const cardClone = card.clone();
console.log(cardClone.id); // card-1-copy
console.log(cardClone.children[0].id); // header-1-copy
```

### Object Registry with Prototypes

```typescript
interface Prototype {
  clone(): Prototype;
  getType(): string;
}

class Shape implements Prototype {
  constructor(
    protected type: string,
    protected color: string,
    protected x: number,
    protected y: number,
  ) {}

  clone(): Shape {
    return new Shape(this.type, this.color, this.x, this.y);
  }

  getType(): string {
    return this.type;
  }

  setPosition(x: number, y: number): void {
    this.x = x;
    this.y = y;
  }

  setColor(color: string): void {
    this.color = color;
  }

  render(): string {
    return `${this.type} at (${this.x}, ${this.y}) with color ${this.color}`;
  }
}

class CircleShape extends Shape {
  constructor(
    color: string,
    x: number,
    y: number,
    private radius: number,
  ) {
    super("circle", color, x, y);
  }

  clone(): CircleShape {
    return new CircleShape(this.color, this.x, this.y, this.radius);
  }

  setRadius(radius: number): void {
    this.radius = radius;
  }

  render(): string {
    return `${super.render()} with radius ${this.radius}`;
  }
}

class RectangleShape extends Shape {
  constructor(
    color: string,
    x: number,
    y: number,
    private width: number,
    private height: number,
  ) {
    super("rectangle", color, x, y);
  }

  clone(): RectangleShape {
    return new RectangleShape(
      this.color,
      this.x,
      this.y,
      this.width,
      this.height,
    );
  }

  setDimensions(width: number, height: number): void {
    this.width = width;
    this.height = height;
  }

  render(): string {
    return `${super.render()} with dimensions ${this.width}x${this.height}`;
  }
}

class PrototypeRegistry {
  private prototypes: Map<string, Prototype> = new Map();

  register(key: string, prototype: Prototype): void {
    this.prototypes.set(key, prototype);
  }

  unregister(key: string): void {
    this.prototypes.delete(key);
  }

  create(key: string): Prototype | null {
    const prototype = this.prototypes.get(key);
    return prototype ? prototype.clone() : null;
  }

  listPrototypes(): string[] {
    return Array.from(this.prototypes.keys());
  }
}

// Usage
const registry = new PrototypeRegistry();

// Register prototypes
registry.register("blue-circle", new CircleShape("blue", 0, 0, 50));
registry.register("red-rectangle", new RectangleShape("red", 0, 0, 100, 50));

// Create instances from prototypes
const circle1 = registry.create("blue-circle") as CircleShape;
circle1.setPosition(100, 100);

const circle2 = registry.create("blue-circle") as CircleShape;
circle2.setPosition(200, 200);
circle2.setColor("green");

const rectangle = registry.create("red-rectangle") as RectangleShape;
rectangle.setPosition(50, 50);

console.log(circle1.render());
console.log(circle2.render());
console.log(rectangle.render());
```

## Object Pool Pattern

### Connection Pool

```typescript
interface PooledObject {
  id: string;
  isAvailable: boolean;
  lastUsed: number;
}

class Connection implements PooledObject {
  id: string;
  isAvailable: boolean = true;
  lastUsed: number = Date.now();

  constructor(id: string) {
    this.id = id;
  }

  connect(): void {
    console.log(`Connection ${this.id} established`);
  }

  disconnect(): void {
    console.log(`Connection ${this.id} closed`);
  }

  query(sql: string): Promise<any[]> {
    console.log(`Executing query on connection ${this.id}:`, sql);
    return Promise.resolve([]);
  }

  reset(): void {
    console.log(`Connection ${this.id} reset`);
  }
}

class ConnectionPool {
  private pool: Connection[] = [];
  private inUse: Set<string> = new Set();

  constructor(
    private minSize: number = 5,
    private maxSize: number = 20,
    private idleTimeout: number = 60000, // 1 minute
  ) {
    this.initialize();
    this.startIdleChecker();
  }

  private initialize(): void {
    for (let i = 0; i < this.minSize; i++) {
      const connection = new Connection(`conn-${i}`);
      connection.connect();
      this.pool.push(connection);
    }
  }

  async acquire(): Promise<Connection> {
    // Find available connection
    let connection = this.pool.find((conn) => conn.isAvailable);

    // If no available connection and pool not at max, create new one
    if (!connection && this.pool.length < this.maxSize) {
      connection = new Connection(`conn-${this.pool.length}`);
      connection.connect();
      this.pool.push(connection);
    }

    // If still no connection, wait for one to become available
    if (!connection) {
      connection = await this.waitForConnection();
    }

    connection.isAvailable = false;
    connection.lastUsed = Date.now();
    this.inUse.add(connection.id);

    return connection;
  }

  release(connection: Connection): void {
    connection.isAvailable = true;
    connection.lastUsed = Date.now();
    connection.reset();
    this.inUse.delete(connection.id);
  }

  private async waitForConnection(): Promise<Connection> {
    return new Promise((resolve) => {
      const interval = setInterval(() => {
        const connection = this.pool.find((conn) => conn.isAvailable);
        if (connection) {
          clearInterval(interval);
          resolve(connection);
        }
      }, 100);
    });
  }

  private startIdleChecker(): void {
    setInterval(() => {
      const now = Date.now();
      const connections = this.pool.filter(
        (conn) =>
          conn.isAvailable &&
          now - conn.lastUsed > this.idleTimeout &&
          this.pool.length > this.minSize,
      );

      connections.forEach((conn) => {
        conn.disconnect();
        const index = this.pool.indexOf(conn);
        if (index > -1) {
          this.pool.splice(index, 1);
        }
      });
    }, this.idleTimeout);
  }

  getStats(): {
    total: number;
    available: number;
    inUse: number;
  } {
    return {
      total: this.pool.length,
      available: this.pool.filter((c) => c.isAvailable).length,
      inUse: this.inUse.size,
    };
  }

  async shutdown(): Promise<void> {
    this.pool.forEach((conn) => conn.disconnect());
    this.pool = [];
    this.inUse.clear();
  }
}

// Usage
const pool = new ConnectionPool(5, 20, 60000);

async function executeQuery(sql: string): Promise<any[]> {
  const connection = await pool.acquire();
  try {
    return await connection.query(sql);
  } finally {
    pool.release(connection);
  }
}

// Execute multiple queries concurrently
const queries = [
  "SELECT * FROM users",
  "SELECT * FROM posts",
  "SELECT * FROM comments",
];

Promise.all(queries.map((q) => executeQuery(q))).then(() => {
  console.log("Pool stats:", pool.getStats());
});
```

### React Component Pool

```typescript
interface PooledComponent<P = any> {
  id: string;
  component: React.ReactElement<P>;
  isAvailable: boolean;
  lastUsed: number;
}

class ComponentPool<P = any> {
  private pool: PooledComponent<P>[] = [];
  private componentFactory: (id: string) => React.ReactElement<P>;

  constructor(
    componentFactory: (id: string) => React.ReactElement<P>,
    initialSize: number = 10
  ) {
    this.componentFactory = componentFactory;
    this.initialize(initialSize);
  }

  private initialize(size: number): void {
    for (let i = 0; i < size; i++) {
      this.pool.push({
        id: `component-${i}`,
        component: this.componentFactory(`component-${i}`),
        isAvailable: true,
        lastUsed: Date.now(),
      });
    }
  }

  acquire(): PooledComponent<P> | null {
    const item = this.pool.find(item => item.isAvailable);

    if (!item) {
      // Create new component if pool is empty
      const newItem: PooledComponent<P> = {
        id: `component-${this.pool.length}`,
        component: this.componentFactory(`component-${this.pool.length}`),
        isAvailable: false,
        lastUsed: Date.now(),
      };
      this.pool.push(newItem);
      return newItem;
    }

    item.isAvailable = false;
    item.lastUsed = Date.now();
    return item;
  }

  release(id: string): void {
    const item = this.pool.find(item => item.id === id);
    if (item) {
      item.isAvailable = true;
      item.lastUsed = Date.now();
    }
  }

  getAvailableCount(): number {
    return this.pool.filter(item => item.isAvailable).length;
  }
}

// Usage with virtual scrolling
const ListItemPool = new ComponentPool<{ data: any }>(
  (id) => <ListItem key={id} data={null} />,
  50
);

const VirtualList: React.FC<{ items: any[] }> = ({ items }) => {
  const [visibleItems, setVisibleItems] = React.useState<any[]>([]);

  React.useEffect(() => {
    // Get only visible items based on scroll position
    const visible = items.slice(0, 20);
    setVisibleItems(visible);
  }, [items]);

  return (
    <div className="virtual-list">
      {visibleItems.map((item, index) => {
        const pooled = ListItemPool.acquire();
        return pooled ? (
          React.cloneElement(pooled.component, { data: item, key: index })
        ) : null;
      })}
    </div>
  );
};
```

## Real-World Usage

### E-commerce Product Factory

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

interface PhysicalProduct extends Product {
  weight: number;
  dimensions: { length: number; width: number; height: number };
  shippingClass: string;
}

interface DigitalProduct extends Product {
  downloadUrl: string;
  fileSize: number;
  format: string;
}

interface SubscriptionProduct extends Product {
  billingCycle: "monthly" | "yearly";
  features: string[];
  trialDays: number;
}

class ProductFactory {
  static createProduct(
    type: "physical" | "digital" | "subscription",
    baseData: Product,
    specificData: any,
  ): PhysicalProduct | DigitalProduct | SubscriptionProduct {
    switch (type) {
      case "physical":
        return { ...baseData, ...specificData } as PhysicalProduct;
      case "digital":
        return { ...baseData, ...specificData } as DigitalProduct;
      case "subscription":
        return { ...baseData, ...specificData } as SubscriptionProduct;
    }
  }

  static createProductFromAPI(data: any): Product {
    const base: Product = {
      id: data.id,
      name: data.name,
      price: data.price,
      category: data.category,
    };

    if (data.type === "physical") {
      return this.createProduct("physical", base, {
        weight: data.weight,
        dimensions: data.dimensions,
        shippingClass: data.shipping_class,
      });
    } else if (data.type === "digital") {
      return this.createProduct("digital", base, {
        downloadUrl: data.download_url,
        fileSize: data.file_size,
        format: data.format,
      });
    } else {
      return this.createProduct("subscription", base, {
        billingCycle: data.billing_cycle,
        features: data.features,
        trialDays: data.trial_days,
      });
    }
  }
}

// Usage
const products = [
  {
    id: "1",
    name: "Laptop",
    price: 999,
    category: "Electronics",
    type: "physical",
    weight: 2.5,
    dimensions: { length: 35, width: 25, height: 2 },
    shipping_class: "standard",
  },
  {
    id: "2",
    name: "E-book",
    price: 9.99,
    category: "Books",
    type: "digital",
    download_url: "https://example.com/ebook.pdf",
    file_size: 5242880,
    format: "PDF",
  },
];

const productInstances = products.map((data) =>
  ProductFactory.createProductFromAPI(data),
);
```

## Production Patterns

### Lazy Initialization Singleton

```typescript
class LazyService {
  private static instance: LazyService | null = null;
  private initialized: boolean = false;
  private data: any = null;

  private constructor() {
    // Private constructor prevents direct instantiation
  }

  static getInstance(): LazyService {
    if (!LazyService.instance) {
      LazyService.instance = new LazyService();
    }
    return LazyService.instance;
  }

  async initialize(): Promise<void> {
    if (this.initialized) return;

    console.log("Initializing service...");
    // Expensive initialization
    this.data = await this.fetchData();
    this.initialized = true;
  }

  private async fetchData(): Promise<any> {
    return new Promise((resolve) => {
      setTimeout(() => resolve({ message: "Data loaded" }), 1000);
    });
  }

  getData(): any {
    if (!this.initialized) {
      throw new Error("Service not initialized");
    }
    return this.data;
  }

  isInitialized(): boolean {
    return this.initialized;
  }
}

// Usage
const service = LazyService.getInstance();
await service.initialize();
console.log(service.getData());
```

### Factory with Dependency Injection

```typescript
interface Logger {
  log(message: string): void;
  error(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }

  error(message: string): void {
    console.error(`[ERROR] ${message}`);
  }
}

class FileLogger implements Logger {
  constructor(private filename: string) {}

  log(message: string): void {
    // Write to file
    console.log(`[FILE LOG] ${this.filename}: ${message}`);
  }

  error(message: string): void {
    // Write error to file
    console.log(`[FILE ERROR] ${this.filename}: ${message}`);
  }
}

class ServiceFactory {
  constructor(private logger: Logger) {}

  createUserService(): UserService {
    return new UserService(this.logger);
  }

  createProductService(): ProductService {
    return new ProductService(this.logger);
  }
}

class UserService {
  constructor(private logger: Logger) {}

  async getUser(id: string): Promise<any> {
    this.logger.log(`Fetching user ${id}`);
    // Fetch user logic
    return { id, name: "John Doe" };
  }
}

class ProductService {
  constructor(private logger: Logger) {}

  async getProduct(id: string): Promise<any> {
    this.logger.log(`Fetching product ${id}`);
    // Fetch product logic
    return { id, name: "Product" };
  }
}

// Usage
const logger = new ConsoleLogger();
const factory = new ServiceFactory(logger);

const userService = factory.createUserService();
const productService = factory.createProductService();

await userService.getUser("123");
await productService.getProduct("456");
```

## Best Practices

1. **Use Factory Pattern for complex object creation** - When object creation involves multiple steps or conditional logic
2. **Singleton for truly global state** - Use sparingly, prefer dependency injection for testability
3. **Builder for objects with many optional parameters** - Better than constructors with many parameters
4. **Prototype for expensive object creation** - Clone existing objects instead of creating from scratch
5. **Abstract Factory for families of related objects** - Maintain consistency across related objects
6. **Object Pool for expensive resource management** - Reuse objects like database connections, DOM nodes
7. **Make singletons thread-safe** - Use proper locking mechanisms in multi-threaded environments
8. **Implement proper cleanup** - Release resources in object pools, close connections in singletons
9. **Type safety with TypeScript** - Use generics and discriminated unions for type-safe factories
10. **Document factory methods** - Clear documentation helps other developers understand object creation patterns

## 10 Key Takeaways

1. **Factory Pattern** encapsulates object creation logic and provides flexibility in instantiation
2. **Abstract Factory** creates families of related objects without specifying concrete classes
3. **Builder Pattern** constructs complex objects step by step with fluent interface
4. **Singleton** ensures single instance with global access point (use cautiously)
5. **Prototype Pattern** clones existing objects for better performance with expensive creation
6. **Object Pool** reuses expensive objects to improve performance and resource management
7. **Creational patterns** separate object creation from usage for better maintainability
8. **Type safety** is crucial - leverage TypeScript's type system for compile-time safety
9. **Dependency injection** with factories improves testability and flexibility
10. **Choose the right pattern** based on problem context - no pattern fits all scenarios
