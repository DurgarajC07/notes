# Frontend Architecture Patterns - Complete Guide

## Core Concepts

Frontend architecture patterns define how to structure applications for maintainability, scalability, and testability. This guide covers all major patterns with production TypeScript implementations.

## 1. Model-View-Controller (MVC)

Traditional pattern separating data (Model), presentation (View), and logic (Controller).

```typescript
// Model: Data and business logic
class UserModel {
  private users: User[] = [];
  private observers: Observer[] = [];

  addUser(user: User): void {
    this.users.push(user);
    this.notifyObservers();
  }

  getUsers(): User[] {
    return [...this.users];
  }

  getUserById(id: string): User | undefined {
    return this.users.find((u) => u.id === id);
  }

  updateUser(id: string, updates: Partial<User>): void {
    const index = this.users.findIndex((u) => u.id === id);
    if (index !== -1) {
      this.users[index] = { ...this.users[index], ...updates };
      this.notifyObservers();
    }
  }

  deleteUser(id: string): void {
    this.users = this.users.filter((u) => u.id !== id);
    this.notifyObservers();
  }

  // Observer pattern for data changes
  subscribe(observer: Observer): void {
    this.observers.push(observer);
  }

  private notifyObservers(): void {
    this.observers.forEach((observer) => observer.update(this.users));
  }
}

// View: UI presentation
class UserView {
  private container: HTMLElement;

  constructor(containerId: string) {
    this.container = document.getElementById(containerId)!;
  }

  render(users: User[]): void {
    this.container.innerHTML = "";

    const table = document.createElement("table");
    table.innerHTML = `
      <thead>
        <tr>
          <th>Name</th>
          <th>Email</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        ${users
          .map(
            (user) => `
          <tr data-id="${user.id}">
            <td>${user.name}</td>
            <td>${user.email}</td>
            <td>
              <button class="edit-btn">Edit</button>
              <button class="delete-btn">Delete</button>
            </td>
          </tr>
        `,
          )
          .join("")}
      </tbody>
    `;

    this.container.appendChild(table);
  }

  bindEditUser(handler: (id: string) => void): void {
    this.container.addEventListener("click", (e) => {
      const target = e.target as HTMLElement;
      if (target.classList.contains("edit-btn")) {
        const row = target.closest("tr");
        const id = row?.getAttribute("data-id");
        if (id) handler(id);
      }
    });
  }

  bindDeleteUser(handler: (id: string) => void): void {
    this.container.addEventListener("click", (e) => {
      const target = e.target as HTMLElement;
      if (target.classList.contains("delete-btn")) {
        const row = target.closest("tr");
        const id = row?.getAttribute("data-id");
        if (id) handler(id);
      }
    });
  }

  showNotification(message: string): void {
    const notification = document.createElement("div");
    notification.className = "notification";
    notification.textContent = message;
    document.body.appendChild(notification);

    setTimeout(() => notification.remove(), 3000);
  }
}

// Controller: Mediates between Model and View
class UserController implements Observer {
  private model: UserModel;
  private view: UserView;

  constructor(model: UserModel, view: UserView) {
    this.model = model;
    this.view = view;

    // Subscribe to model changes
    this.model.subscribe(this);

    // Bind view events
    this.view.bindEditUser(this.handleEditUser.bind(this));
    this.view.bindDeleteUser(this.handleDeleteUser.bind(this));

    // Initial render
    this.view.render(this.model.getUsers());
  }

  // Observer interface implementation
  update(users: User[]): void {
    this.view.render(users);
  }

  handleEditUser(id: string): void {
    const user = this.model.getUserById(id);
    if (user) {
      const name = prompt("Enter new name:", user.name);
      const email = prompt("Enter new email:", user.email);

      if (name && email) {
        this.model.updateUser(id, { name, email });
        this.view.showNotification("User updated successfully");
      }
    }
  }

  handleDeleteUser(id: string): void {
    if (confirm("Are you sure you want to delete this user?")) {
      this.model.deleteUser(id);
      this.view.showNotification("User deleted successfully");
    }
  }

  addUser(user: User): void {
    this.model.addUser(user);
  }
}

interface User {
  id: string;
  name: string;
  email: string;
}

interface Observer {
  update(data: any): void;
}

// Usage
const model = new UserModel();
const view = new UserView("app");
const controller = new UserController(model, view);

controller.addUser({ id: "1", name: "John Doe", email: "john@example.com" });
```

## 2. Model-View-Presenter (MVP)

View is passive, Presenter handles all logic.

```typescript
// Model: Same as MVC
class TaskModel {
  private tasks: Task[] = [];

  async fetchTasks(): Promise<Task[]> {
    const response = await fetch("/api/tasks");
    this.tasks = await response.json();
    return this.tasks;
  }

  async createTask(task: Omit<Task, "id">): Promise<Task> {
    const response = await fetch("/api/tasks", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(task),
    });
    const newTask = await response.json();
    this.tasks.push(newTask);
    return newTask;
  }

  async updateTask(id: string, updates: Partial<Task>): Promise<Task> {
    const response = await fetch(`/api/tasks/${id}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(updates),
    });
    const updatedTask = await response.json();

    const index = this.tasks.findIndex((t) => t.id === id);
    if (index !== -1) {
      this.tasks[index] = updatedTask;
    }

    return updatedTask;
  }

  async deleteTask(id: string): Promise<void> {
    await fetch(`/api/tasks/${id}`, { method: "DELETE" });
    this.tasks = this.tasks.filter((t) => t.id !== id);
  }

  getTasks(): Task[] {
    return [...this.tasks];
  }
}

// View Interface: Contract between View and Presenter
interface ITaskView {
  showTasks(tasks: Task[]): void;
  showLoading(): void;
  hideLoading(): void;
  showError(message: string): void;
  getTaskInput(): TaskInput;
  clearTaskInput(): void;
}

// View Implementation: Passive, only renders
class TaskView implements ITaskView {
  private container: HTMLElement;
  private loadingElement: HTMLElement;
  private errorElement: HTMLElement;
  private taskListElement: HTMLElement;
  private taskInputElement: HTMLInputElement;
  private taskDescElement: HTMLTextAreaElement;

  constructor(containerId: string) {
    this.container = document.getElementById(containerId)!;
    this.setupElements();
  }

  private setupElements(): void {
    this.loadingElement = document.createElement("div");
    this.loadingElement.className = "loading hidden";
    this.loadingElement.textContent = "Loading...";

    this.errorElement = document.createElement("div");
    this.errorElement.className = "error hidden";

    this.taskListElement = document.createElement("div");
    this.taskListElement.className = "task-list";

    this.taskInputElement = document.createElement("input");
    this.taskInputElement.type = "text";
    this.taskInputElement.placeholder = "Task title";

    this.taskDescElement = document.createElement("textarea");
    this.taskDescElement.placeholder = "Task description";

    this.container.appendChild(this.loadingElement);
    this.container.appendChild(this.errorElement);
    this.container.appendChild(this.taskInputElement);
    this.container.appendChild(this.taskDescElement);
    this.container.appendChild(this.taskListElement);
  }

  showTasks(tasks: Task[]): void {
    this.taskListElement.innerHTML = tasks
      .map(
        (task) => `
      <div class="task" data-id="${task.id}">
        <h3>${task.title}</h3>
        <p>${task.description}</p>
        <span class="status">${task.completed ? "Completed" : "Pending"}</span>
        <button class="toggle-btn">Toggle</button>
        <button class="delete-btn">Delete</button>
      </div>
    `,
      )
      .join("");
  }

  showLoading(): void {
    this.loadingElement.classList.remove("hidden");
  }

  hideLoading(): void {
    this.loadingElement.classList.add("hidden");
  }

  showError(message: string): void {
    this.errorElement.textContent = message;
    this.errorElement.classList.remove("hidden");
    setTimeout(() => this.errorElement.classList.add("hidden"), 3000);
  }

  getTaskInput(): TaskInput {
    return {
      title: this.taskInputElement.value,
      description: this.taskDescElement.value,
    };
  }

  clearTaskInput(): void {
    this.taskInputElement.value = "";
    this.taskDescElement.value = "";
  }

  bindAddTask(handler: () => void): void {
    const button = document.createElement("button");
    button.textContent = "Add Task";
    button.addEventListener("click", handler);
    this.container.appendChild(button);
  }

  bindToggleTask(handler: (id: string) => void): void {
    this.taskListElement.addEventListener("click", (e) => {
      const target = e.target as HTMLElement;
      if (target.classList.contains("toggle-btn")) {
        const taskEl = target.closest(".task");
        const id = taskEl?.getAttribute("data-id");
        if (id) handler(id);
      }
    });
  }

  bindDeleteTask(handler: (id: string) => void): void {
    this.taskListElement.addEventListener("click", (e) => {
      const target = e.target as HTMLElement;
      if (target.classList.contains("delete-btn")) {
        const taskEl = target.closest(".task");
        const id = taskEl?.getAttribute("data-id");
        if (id) handler(id);
      }
    });
  }
}

// Presenter: Contains all presentation logic
class TaskPresenter {
  private model: TaskModel;
  private view: ITaskView;

  constructor(model: TaskModel, view: ITaskView) {
    this.model = model;
    this.view = view;

    this.initialize();
  }

  private async initialize(): Promise<void> {
    await this.loadTasks();
    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    if (this.view instanceof TaskView) {
      this.view.bindAddTask(this.handleAddTask.bind(this));
      this.view.bindToggleTask(this.handleToggleTask.bind(this));
      this.view.bindDeleteTask(this.handleDeleteTask.bind(this));
    }
  }

  async loadTasks(): Promise<void> {
    try {
      this.view.showLoading();
      const tasks = await this.model.fetchTasks();
      this.view.showTasks(tasks);
    } catch (error) {
      this.view.showError("Failed to load tasks");
    } finally {
      this.view.hideLoading();
    }
  }

  async handleAddTask(): Promise<void> {
    const input = this.view.getTaskInput();

    if (!input.title.trim()) {
      this.view.showError("Task title is required");
      return;
    }

    try {
      this.view.showLoading();
      await this.model.createTask({
        title: input.title,
        description: input.description,
        completed: false,
      });
      this.view.clearTaskInput();
      await this.loadTasks();
    } catch (error) {
      this.view.showError("Failed to create task");
    } finally {
      this.view.hideLoading();
    }
  }

  async handleToggleTask(id: string): Promise<void> {
    try {
      const tasks = this.model.getTasks();
      const task = tasks.find((t) => t.id === id);

      if (task) {
        await this.model.updateTask(id, { completed: !task.completed });
        await this.loadTasks();
      }
    } catch (error) {
      this.view.showError("Failed to update task");
    }
  }

  async handleDeleteTask(id: string): Promise<void> {
    try {
      await this.model.deleteTask(id);
      await this.loadTasks();
    } catch (error) {
      this.view.showError("Failed to delete task");
    }
  }
}

interface Task {
  id: string;
  title: string;
  description: string;
  completed: boolean;
}

interface TaskInput {
  title: string;
  description: string;
}

// Usage
const taskModel = new TaskModel();
const taskView = new TaskView("task-app");
const taskPresenter = new TaskPresenter(taskModel, taskView);
```

## 3. Model-View-ViewModel (MVVM)

View binds to ViewModel, which exposes Model data.

```typescript
// Model
class ProductModel {
  private products: Product[] = [];

  async fetchProducts(): Promise<Product[]> {
    const response = await fetch("/api/products");
    this.products = await response.json();
    return this.products;
  }

  async addProduct(product: Omit<Product, "id">): Promise<Product> {
    const response = await fetch("/api/products", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(product),
    });
    const newProduct = await response.json();
    this.products.push(newProduct);
    return newProduct;
  }

  getProducts(): Product[] {
    return [...this.products];
  }
}

// ViewModel: Exposes data and commands for View
class ProductViewModel {
  private model: ProductModel;

  // Observable properties
  public products = new Observable<Product[]>([]);
  public isLoading = new Observable<boolean>(false);
  public error = new Observable<string | null>(null);
  public searchQuery = new Observable<string>("");
  public filteredProducts = new Observable<Product[]>([]);

  // Commands (actions the view can invoke)
  public commands = {
    loadProducts: this.loadProducts.bind(this),
    addProduct: this.addProduct.bind(this),
    search: this.search.bind(this),
  };

  constructor(model: ProductModel) {
    this.model = model;

    // Set up computed properties
    this.searchQuery.subscribe((query) => {
      this.updateFilteredProducts(query);
    });

    this.products.subscribe((products) => {
      this.updateFilteredProducts(this.searchQuery.getValue());
    });
  }

  private updateFilteredProducts(query: string): void {
    const allProducts = this.products.getValue();

    if (!query) {
      this.filteredProducts.setValue(allProducts);
      return;
    }

    const filtered = allProducts.filter(
      (p) =>
        p.name.toLowerCase().includes(query.toLowerCase()) ||
        p.description.toLowerCase().includes(query.toLowerCase()),
    );

    this.filteredProducts.setValue(filtered);
  }

  async loadProducts(): Promise<void> {
    try {
      this.isLoading.setValue(true);
      this.error.setValue(null);

      const products = await this.model.fetchProducts();
      this.products.setValue(products);
    } catch (err) {
      this.error.setValue("Failed to load products");
    } finally {
      this.isLoading.setValue(false);
    }
  }

  async addProduct(product: Omit<Product, "id">): Promise<void> {
    try {
      this.isLoading.setValue(true);
      this.error.setValue(null);

      const newProduct = await this.model.addProduct(product);
      const current = this.products.getValue();
      this.products.setValue([...current, newProduct]);
    } catch (err) {
      this.error.setValue("Failed to add product");
    } finally {
      this.isLoading.setValue(false);
    }
  }

  search(query: string): void {
    this.searchQuery.setValue(query);
  }
}

// Observable implementation for data binding
class Observable<T> {
  private value: T;
  private listeners: Array<(value: T) => void> = [];

  constructor(initialValue: T) {
    this.value = initialValue;
  }

  getValue(): T {
    return this.value;
  }

  setValue(newValue: T): void {
    if (this.value !== newValue) {
      this.value = newValue;
      this.notify();
    }
  }

  subscribe(listener: (value: T) => void): () => void {
    this.listeners.push(listener);

    // Return unsubscribe function
    return () => {
      this.listeners = this.listeners.filter((l) => l !== listener);
    };
  }

  private notify(): void {
    this.listeners.forEach((listener) => listener(this.value));
  }
}

// View: Binds to ViewModel
class ProductView {
  private viewModel: ProductViewModel;
  private container: HTMLElement;
  private unsubscribers: Array<() => void> = [];

  constructor(viewModel: ProductViewModel, containerId: string) {
    this.viewModel = viewModel;
    this.container = document.getElementById(containerId)!;

    this.setupBindings();
    this.render();
    this.setupEventHandlers();
  }

  private setupBindings(): void {
    // Bind to ViewModel observables
    this.unsubscribers.push(
      this.viewModel.filteredProducts.subscribe((products) => {
        this.renderProducts(products);
      }),
    );

    this.unsubscribers.push(
      this.viewModel.isLoading.subscribe((loading) => {
        this.updateLoadingState(loading);
      }),
    );

    this.unsubscribers.push(
      this.viewModel.error.subscribe((error) => {
        this.showError(error);
      }),
    );
  }

  private render(): void {
    this.container.innerHTML = `
      <div class="product-app">
        <div class="search-bar">
          <input type="text" id="search-input" placeholder="Search products..." />
        </div>
        <div class="loading hidden">Loading...</div>
        <div class="error hidden"></div>
        <div class="product-list"></div>
        <div class="add-product-form">
          <input type="text" id="product-name" placeholder="Product name" />
          <textarea id="product-desc" placeholder="Description"></textarea>
          <input type="number" id="product-price" placeholder="Price" />
          <button id="add-btn">Add Product</button>
        </div>
      </div>
    `;
  }

  private setupEventHandlers(): void {
    // Search input
    const searchInput = this.container.querySelector(
      "#search-input",
    ) as HTMLInputElement;
    searchInput.addEventListener("input", (e) => {
      this.viewModel.commands.search((e.target as HTMLInputElement).value);
    });

    // Add product button
    const addBtn = this.container.querySelector("#add-btn");
    addBtn?.addEventListener("click", () => this.handleAddProduct());

    // Load products on init
    this.viewModel.commands.loadProducts();
  }

  private renderProducts(products: Product[]): void {
    const listEl = this.container.querySelector(".product-list");
    if (!listEl) return;

    listEl.innerHTML = products
      .map(
        (product) => `
      <div class="product-card">
        <h3>${product.name}</h3>
        <p>${product.description}</p>
        <span class="price">$${product.price.toFixed(2)}</span>
      </div>
    `,
      )
      .join("");
  }

  private updateLoadingState(loading: boolean): void {
    const loadingEl = this.container.querySelector(".loading");
    if (loading) {
      loadingEl?.classList.remove("hidden");
    } else {
      loadingEl?.classList.add("hidden");
    }
  }

  private showError(error: string | null): void {
    const errorEl = this.container.querySelector(".error") as HTMLElement;
    if (error) {
      errorEl.textContent = error;
      errorEl.classList.remove("hidden");
      setTimeout(() => errorEl.classList.add("hidden"), 3000);
    }
  }

  private handleAddProduct(): void {
    const nameInput = this.container.querySelector(
      "#product-name",
    ) as HTMLInputElement;
    const descInput = this.container.querySelector(
      "#product-desc",
    ) as HTMLTextAreaElement;
    const priceInput = this.container.querySelector(
      "#product-price",
    ) as HTMLInputElement;

    const product = {
      name: nameInput.value,
      description: descInput.value,
      price: parseFloat(priceInput.value),
    };

    this.viewModel.commands.addProduct(product);

    // Clear inputs
    nameInput.value = "";
    descInput.value = "";
    priceInput.value = "";
  }

  destroy(): void {
    this.unsubscribers.forEach((unsub) => unsub());
  }
}

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
}

// Usage
const productModel = new ProductModel();
const productViewModel = new ProductViewModel(productModel);
const productView = new ProductView(productViewModel, "product-app");
```

## 4. Flux Architecture

Unidirectional data flow: Action → Dispatcher → Store → View.

```typescript
// Action Types
enum ActionType {
  ADD_TODO = "ADD_TODO",
  TOGGLE_TODO = "TOGGLE_TODO",
  DELETE_TODO = "DELETE_TODO",
  LOAD_TODOS = "LOAD_TODOS",
  SET_LOADING = "SET_LOADING",
  SET_ERROR = "SET_ERROR",
}

// Actions
interface Action {
  type: ActionType;
  payload?: any;
}

class TodoActions {
  static addTodo(text: string): Action {
    return {
      type: ActionType.ADD_TODO,
      payload: { text },
    };
  }

  static toggleTodo(id: string): Action {
    return {
      type: ActionType.TOGGLE_TODO,
      payload: { id },
    };
  }

  static deleteTodo(id: string): Action {
    return {
      type: ActionType.DELETE_TODO,
      payload: { id },
    };
  }

  static loadTodos(todos: Todo[]): Action {
    return {
      type: ActionType.LOAD_TODOS,
      payload: { todos },
    };
  }

  static setLoading(loading: boolean): Action {
    return {
      type: ActionType.SET_LOADING,
      payload: { loading },
    };
  }

  static setError(error: string): Action {
    return {
      type: ActionType.SET_ERROR,
      payload: { error },
    };
  }
}

// Dispatcher: Central hub for actions
class Dispatcher {
  private callbacks: Map<string, (action: Action) => void> = new Map();

  register(id: string, callback: (action: Action) => void): void {
    this.callbacks.set(id, callback);
  }

  unregister(id: string): void {
    this.callbacks.delete(id);
  }

  dispatch(action: Action): void {
    this.callbacks.forEach((callback) => callback(action));
  }
}

// Store: Holds application state
class TodoStore {
  private todos: Todo[] = [];
  private loading: boolean = false;
  private error: string | null = null;
  private listeners: Array<() => void> = [];
  private dispatcher: Dispatcher;

  constructor(dispatcher: Dispatcher) {
    this.dispatcher = dispatcher;

    // Register with dispatcher
    this.dispatcher.register("TodoStore", this.handleAction.bind(this));
  }

  private handleAction(action: Action): void {
    switch (action.type) {
      case ActionType.ADD_TODO:
        this.todos.push({
          id: crypto.randomUUID(),
          text: action.payload.text,
          completed: false,
        });
        this.emitChange();
        break;

      case ActionType.TOGGLE_TODO:
        this.todos = this.todos.map((todo) =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo,
        );
        this.emitChange();
        break;

      case ActionType.DELETE_TODO:
        this.todos = this.todos.filter((todo) => todo.id !== action.payload.id);
        this.emitChange();
        break;

      case ActionType.LOAD_TODOS:
        this.todos = action.payload.todos;
        this.emitChange();
        break;

      case ActionType.SET_LOADING:
        this.loading = action.payload.loading;
        this.emitChange();
        break;

      case ActionType.SET_ERROR:
        this.error = action.payload.error;
        this.emitChange();
        break;
    }
  }

  getTodos(): Todo[] {
    return [...this.todos];
  }

  isLoading(): boolean {
    return this.loading;
  }

  getError(): string | null {
    return this.error;
  }

  addChangeListener(listener: () => void): void {
    this.listeners.push(listener);
  }

  removeChangeListener(listener: () => void): void {
    this.listeners = this.listeners.filter((l) => l !== listener);
  }

  private emitChange(): void {
    this.listeners.forEach((listener) => listener());
  }
}

// View Component
class TodoView {
  private store: TodoStore;
  private dispatcher: Dispatcher;
  private container: HTMLElement;

  constructor(store: TodoStore, dispatcher: Dispatcher, containerId: string) {
    this.store = store;
    this.dispatcher = dispatcher;
    this.container = document.getElementById(containerId)!;

    // Listen to store changes
    this.store.addChangeListener(this.render.bind(this));

    this.setupUI();
    this.render();
  }

  private setupUI(): void {
    this.container.innerHTML = `
      <div class="todo-app">
        <input type="text" id="todo-input" placeholder="Add new todo" />
        <button id="add-btn">Add</button>
        <div class="loading hidden">Loading...</div>
        <div class="error hidden"></div>
        <ul id="todo-list"></ul>
      </div>
    `;

    // Event handlers
    const addBtn = this.container.querySelector("#add-btn");
    addBtn?.addEventListener("click", () => this.handleAddTodo());

    const input = this.container.querySelector(
      "#todo-input",
    ) as HTMLInputElement;
    input?.addEventListener("keypress", (e) => {
      if (e.key === "Enter") this.handleAddTodo();
    });
  }

  private render(): void {
    const todos = this.store.getTodos();
    const loading = this.store.isLoading();
    const error = this.store.getError();

    // Update loading state
    const loadingEl = this.container.querySelector(".loading");
    if (loading) {
      loadingEl?.classList.remove("hidden");
    } else {
      loadingEl?.classList.add("hidden");
    }

    // Update error state
    const errorEl = this.container.querySelector(".error") as HTMLElement;
    if (error) {
      errorEl.textContent = error;
      errorEl.classList.remove("hidden");
    } else {
      errorEl.classList.add("hidden");
    }

    // Render todos
    const listEl = this.container.querySelector("#todo-list");
    if (listEl) {
      listEl.innerHTML = todos
        .map(
          (todo) => `
        <li class="${todo.completed ? "completed" : ""}" data-id="${todo.id}">
          <input type="checkbox" ${todo.completed ? "checked" : ""} class="toggle-checkbox" />
          <span>${todo.text}</span>
          <button class="delete-btn">Delete</button>
        </li>
      `,
        )
        .join("");

      // Attach event listeners
      listEl.querySelectorAll(".toggle-checkbox").forEach((checkbox) => {
        checkbox.addEventListener("change", (e) => {
          const li = (e.target as HTMLElement).closest("li");
          const id = li?.getAttribute("data-id");
          if (id) this.handleToggleTodo(id);
        });
      });

      listEl.querySelectorAll(".delete-btn").forEach((btn) => {
        btn.addEventListener("click", (e) => {
          const li = (e.target as HTMLElement).closest("li");
          const id = li?.getAttribute("data-id");
          if (id) this.handleDeleteTodo(id);
        });
      });
    }
  }

  private handleAddTodo(): void {
    const input = this.container.querySelector(
      "#todo-input",
    ) as HTMLInputElement;
    const text = input.value.trim();

    if (text) {
      this.dispatcher.dispatch(TodoActions.addTodo(text));
      input.value = "";
    }
  }

  private handleToggleTodo(id: string): void {
    this.dispatcher.dispatch(TodoActions.toggleTodo(id));
  }

  private handleDeleteTodo(id: string): void {
    this.dispatcher.dispatch(TodoActions.deleteTodo(id));
  }
}

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Usage
const dispatcher = new Dispatcher();
const todoStore = new TodoStore(dispatcher);
const todoView = new TodoView(todoStore, dispatcher, "todo-app");
```

## 5. Redux Architecture

Predictable state container with single store and pure reducers.

```typescript
// State
interface AppState {
  cart: CartState;
  user: UserState;
  products: ProductState;
}

interface CartState {
  items: CartItem[];
  total: number;
}

interface UserState {
  id: string | null;
  name: string | null;
  isAuthenticated: boolean;
}

interface ProductState {
  items: Product[];
  loading: boolean;
  error: string | null;
}

// Action Types
enum CartActionType {
  ADD_TO_CART = "cart/add",
  REMOVE_FROM_CART = "cart/remove",
  UPDATE_QUANTITY = "cart/updateQuantity",
  CLEAR_CART = "cart/clear",
}

enum UserActionType {
  LOGIN = "user/login",
  LOGOUT = "user/logout",
}

enum ProductActionType {
  FETCH_START = "products/fetchStart",
  FETCH_SUCCESS = "products/fetchSuccess",
  FETCH_FAILURE = "products/fetchFailure",
}

// Actions
type CartAction =
  | { type: CartActionType.ADD_TO_CART; payload: CartItem }
  | { type: CartActionType.REMOVE_FROM_CART; payload: string }
  | {
      type: CartActionType.UPDATE_QUANTITY;
      payload: { id: string; quantity: number };
    }
  | { type: CartActionType.CLEAR_CART };

type UserAction =
  | { type: UserActionType.LOGIN; payload: { id: string; name: string } }
  | { type: UserActionType.LOGOUT };

type ProductAction =
  | { type: ProductActionType.FETCH_START }
  | { type: ProductActionType.FETCH_SUCCESS; payload: Product[] }
  | { type: ProductActionType.FETCH_FAILURE; payload: string };

type AppAction = CartAction | UserAction | ProductAction;

// Action Creators
const cartActions = {
  addToCart: (item: CartItem): CartAction => ({
    type: CartActionType.ADD_TO_CART,
    payload: item,
  }),

  removeFromCart: (id: string): CartAction => ({
    type: CartActionType.REMOVE_FROM_CART,
    payload: id,
  }),

  updateQuantity: (id: string, quantity: number): CartAction => ({
    type: CartActionType.UPDATE_QUANTITY,
    payload: { id, quantity },
  }),

  clearCart: (): CartAction => ({
    type: CartActionType.CLEAR_CART,
  }),
};

// Reducers
function cartReducer(
  state: CartState = { items: [], total: 0 },
  action: AppAction,
): CartState {
  switch (action.type) {
    case CartActionType.ADD_TO_CART:
      const existingItem = state.items.find(
        (item) => item.id === action.payload.id,
      );

      if (existingItem) {
        return {
          ...state,
          items: state.items.map((item) =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + action.payload.quantity }
              : item,
          ),
          total: calculateTotal([
            ...state.items.map((item) =>
              item.id === action.payload.id
                ? { ...item, quantity: item.quantity + action.payload.quantity }
                : item,
            ),
          ]),
        };
      }

      return {
        ...state,
        items: [...state.items, action.payload],
        total: calculateTotal([...state.items, action.payload]),
      };

    case CartActionType.REMOVE_FROM_CART:
      return {
        ...state,
        items: state.items.filter((item) => item.id !== action.payload),
        total: calculateTotal(
          state.items.filter((item) => item.id !== action.payload),
        ),
      };

    case CartActionType.UPDATE_QUANTITY:
      return {
        ...state,
        items: state.items.map((item) =>
          item.id === action.payload.id
            ? { ...item, quantity: action.payload.quantity }
            : item,
        ),
        total: calculateTotal(
          state.items.map((item) =>
            item.id === action.payload.id
              ? { ...item, quantity: action.payload.quantity }
              : item,
          ),
        ),
      };

    case CartActionType.CLEAR_CART:
      return { items: [], total: 0 };

    default:
      return state;
  }
}

function userReducer(
  state: UserState = { id: null, name: null, isAuthenticated: false },
  action: AppAction,
): UserState {
  switch (action.type) {
    case UserActionType.LOGIN:
      return {
        id: action.payload.id,
        name: action.payload.name,
        isAuthenticated: true,
      };

    case UserActionType.LOGOUT:
      return {
        id: null,
        name: null,
        isAuthenticated: false,
      };

    default:
      return state;
  }
}

function productReducer(
  state: ProductState = { items: [], loading: false, error: null },
  action: AppAction,
): ProductState {
  switch (action.type) {
    case ProductActionType.FETCH_START:
      return { ...state, loading: true, error: null };

    case ProductActionType.FETCH_SUCCESS:
      return { ...state, loading: false, items: action.payload };

    case ProductActionType.FETCH_FAILURE:
      return { ...state, loading: false, error: action.payload };

    default:
      return state;
  }
}

// Root Reducer
function rootReducer(state: AppState | undefined, action: AppAction): AppState {
  return {
    cart: cartReducer(state?.cart, action),
    user: userReducer(state?.user, action),
    products: productReducer(state?.products, action),
  };
}

// Store
class Store<S, A> {
  private state: S;
  private reducer: (state: S | undefined, action: A) => S;
  private listeners: Array<() => void> = [];

  constructor(reducer: (state: S | undefined, action: A) => S) {
    this.reducer = reducer;
    this.state = reducer(undefined, {} as A);
  }

  getState(): S {
    return this.state;
  }

  dispatch(action: A): void {
    this.state = this.reducer(this.state, action);
    this.listeners.forEach((listener) => listener());
  }

  subscribe(listener: () => void): () => void {
    this.listeners.push(listener);

    return () => {
      this.listeners = this.listeners.filter((l) => l !== listener);
    };
  }
}

// Middleware
type Middleware<S, A> = (
  store: Store<S, A>,
) => (next: (action: A) => void) => (action: A) => void;

// Logger Middleware
const loggerMiddleware: Middleware<AppState, AppAction> =
  (store) => (next) => (action) => {
    console.log("Dispatching:", action);
    const prevState = store.getState();
    next(action);
    const nextState = store.getState();
    console.log("Previous State:", prevState);
    console.log("Next State:", nextState);
  };

// Async Action Middleware (Thunk)
type ThunkAction<S, A> = (
  dispatch: (action: A) => void,
  getState: () => S,
) => Promise<void>;

function thunkMiddleware<S, A>(
  store: Store<S, A>,
): (
  next: (action: A | ThunkAction<S, A>) => void,
) => (action: A | ThunkAction<S, A>) => void {
  return (next) => (action) => {
    if (typeof action === "function") {
      return action(store.dispatch.bind(store), store.getState.bind(store));
    }
    return next(action);
  };
}

// Async Action Creator
function fetchProducts(): ThunkAction<AppState, AppAction> {
  return async (dispatch, getState) => {
    dispatch({ type: ProductActionType.FETCH_START });

    try {
      const response = await fetch("/api/products");
      const products = await response.json();
      dispatch({ type: ProductActionType.FETCH_SUCCESS, payload: products });
    } catch (error) {
      dispatch({
        type: ProductActionType.FETCH_FAILURE,
        payload: "Failed to fetch products",
      });
    }
  };
}

// Selectors
const selectors = {
  getCartItems: (state: AppState) => state.cart.items,
  getCartTotal: (state: AppState) => state.cart.total,
  getCartItemCount: (state: AppState) =>
    state.cart.items.reduce((sum, item) => sum + item.quantity, 0),
  isUserAuthenticated: (state: AppState) => state.user.isAuthenticated,
  getProducts: (state: AppState) => state.products.items,
  isLoadingProducts: (state: AppState) => state.products.loading,
};

// Helper Functions
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
}

// Usage
const store = new Store<AppState, AppAction>(rootReducer);

// Subscribe to changes
store.subscribe(() => {
  console.log("State changed:", store.getState());
});

// Dispatch actions
store.dispatch(
  cartActions.addToCart({
    id: "1",
    name: "Product 1",
    price: 29.99,
    quantity: 1,
  }),
);
```

## 6. Clean Architecture (Frontend)

Separation of concerns with dependency inversion.

```typescript
// Domain Layer: Entities
class User {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string
  ) {}

  static create(email: string, name: string): User {
    return new User(crypto.randomUUID(), email, name);
  }
}

// Domain Layer: Use Cases (Application Business Rules)
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

class GetUserUseCase {
  constructor(private userRepository: UserRepository) {}

  async execute(userId: string): Promise<User> {
    const user = await this.userRepository.findById(userId);

    if (!user) {
      throw new Error('User not found');
    }

    return user;
  }
}

class CreateUserUseCase {
  constructor(private userRepository: UserRepository) {}

  async execute(email: string, name: string): Promise<User> {
    // Validate
    if (!this.isValidEmail(email)) {
      throw new Error('Invalid email format');
    }

    if (name.length < 2) {
      throw new Error('Name must be at least 2 characters');
    }

    // Check if user exists
    const existingUser = await this.userRepository.findByEmail(email);
    if (existingUser) {
      throw new Error('User with this email already exists');
    }

    // Create and save
    const user = User.create(email, name);
    await this.userRepository.save(user);

    return user;
  }

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}

// Infrastructure Layer: Repository Implementation
class HttpUserRepository implements UserRepository {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async findById(id: string): Promise<User | null> {
    try {
      const response = await fetch(`${this.baseUrl}/users/${id}`);

      if (response.status === 404) {
        return null;
      }

      if (!response.ok) {
        throw new Error('Failed to fetch user');
      }

      const data = await response.json();
      return new User(data.id, data.email, data.name);
    } catch (error) {
      console.error('Error fetching user:', error);
      return null;
    }
  }

  async findByEmail(email: string): Promise<User | null> {
    try {
      const response = await fetch(`${this.baseUrl}/users?email=${encodeURIComponent(email)}`);

      if (!response.ok) {
        throw new Error('Failed to fetch user');
      }

      const data = await response.json();

      if (data.length === 0) {
        return null;
      }

      const userData = data[0];
      return new User(userData.id, userData.email, userData.name);
    } catch (error) {
      console.error('Error fetching user by email:', error);
      return null;
    }
  }

  async save(user: User): Promise<void> {
    const response = await fetch(`${this.baseUrl}/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        id: user.id,
        email: user.email,
        name: user.name,
      }),
    });

    if (!response.ok) {
      throw new Error('Failed to save user');
    }
  }

  async delete(id: string): Promise<void> {
    const response = await fetch(`${this.baseUrl}/users/${id}`, {
      method: 'DELETE',
    });

    if (!response.ok) {
      throw new Error('Failed to delete user');
    }
  }
}

// Presentation Layer: Controller/Presenter
class UserController {
  constructor(
    private getUserUseCase: GetUserUseCase,
    private createUserUseCase: CreateUserUseCase
  ) {}

  async getUser(id: string): Promise<UserViewModel> {
    try {
      const user = await this.getUserUseCase.execute(id);
      return this.mapToViewModel(user);
    } catch (error) {
      throw new Error(`Failed to get user: ${(error as Error).message}`);
    }
  }

  async createUser(email: string, name: string): Promise<UserViewModel> {
    try {
      const user = await this.createUserUseCase.execute(email, name);
      return this.mapToViewModel(user);
    } catch (error) {
      throw new Error(`Failed to create user: ${(error as Error).message}`);
    }
  }

  private mapToViewModel(user: User): UserViewModel {
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      initials: this.getInitials(user.name),
    };
  }

  private getInitials(name: string): string {
    return name
      .split(' ')
      .map(part => part[0])
      .join('')
      .toUpperCase()
      .substring(0, 2);
  }
}

interface UserViewModel {
  id: string;
  email: string;
  name: string;
  initials: string;
}

// Presentation Layer: View (React Component)
function UserProfileView({ userId }: { userId: string }) {
  const [user, setUser] = useState<UserViewModel | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadUser = async () => {
      try {
        setLoading(true);
        const userData = await controller.getUser(userId);
        setUser(userData);
      } catch (err) {
        setError((err as Error).message);
      } finally {
        setLoading(false);
      }
    };

    loadUser();
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div className="user-profile">
      <div className="avatar">{user.initials}</div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// Dependency Injection Container
class DependencyContainer {
  private static instance: DependencyContainer;
  private repository: UserRepository;
  private getUserUseCase: GetUserUseCase;
  private createUserUseCase: CreateUserUseCase;
  private controller: UserController;

  private constructor() {
    // Initialize dependencies
    this.repository = new HttpUserRepository('https://api.example.com');
    this.getUserUseCase = new GetUserUseCase(this.repository);
    this.createUserUseCase = new CreateUserUseCase(this.repository);
    this.controller = new UserController(this.getUserUseCase, this.createUserUseCase);
  }

  static getInstance(): DependencyContainer {
    if (!DependencyContainer.instance) {
      DependencyContainer.instance = new DependencyContainer();
    }
    return DependencyContainer.instance;
  }

  getUserController(): UserController {
    return this.controller;
  }
}

// Usage
const container = DependencyContainer.getInstance();
const controller = container.getUserController();
```

## 7. Micro-Frontends Architecture

Independent, loosely coupled frontend applications.

```typescript
// Micro-Frontend Container (Shell)
class MicroFrontendContainer {
  private mountedApps: Map<string, MicroFrontend> = new Map();

  async loadMicroFrontend(config: MicroFrontendConfig): Promise<void> {
    const { name, url, container } = config;

    // Fetch micro-frontend bundle
    const response = await fetch(url);
    const code = await response.text();

    // Create isolated scope
    const microFrontend = this.createMicroFrontend(name, code, container);

    // Mount micro-frontend
    await microFrontend.mount();

    // Store reference
    this.mountedApps.set(name, microFrontend);
  }

  async unloadMicroFrontend(name: string): Promise<void> {
    const app = this.mountedApps.get(name);

    if (app) {
      await app.unmount();
      this.mountedApps.delete(name);
    }
  }

  private createMicroFrontend(
    name: string,
    code: string,
    container: string,
  ): MicroFrontend {
    return new MicroFrontend(name, code, container);
  }
}

class MicroFrontend {
  private name: string;
  private code: string;
  private container: string;
  private exports: any = null;

  constructor(name: string, code: string, container: string) {
    this.name = name;
    this.code = code;
    this.container = container;
  }

  async mount(): Promise<void> {
    // Execute micro-frontend code in isolated scope
    const moduleFunction = new Function("exports", this.code);
    this.exports = {};
    moduleFunction(this.exports);

    // Call mount lifecycle
    if (this.exports.mount) {
      await this.exports.mount(this.container);
    }
  }

  async unmount(): Promise<void> {
    // Call unmount lifecycle
    if (this.exports.unmount) {
      await this.exports.unmount();
    }

    // Clear container
    const containerEl = document.getElementById(this.container);
    if (containerEl) {
      containerEl.innerHTML = "";
    }
  }
}

interface MicroFrontendConfig {
  name: string;
  url: string;
  container: string;
}

// Shared Event Bus for micro-frontend communication
class EventBus {
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

    // Return unsubscribe function
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

// Usage in Shell Application
async function initializeShell() {
  const container = new MicroFrontendContainer();
  const eventBus = EventBus.getInstance();

  // Load micro-frontends
  await container.loadMicroFrontend({
    name: "header",
    url: "/micro-frontends/header.js",
    container: "header-container",
  });

  await container.loadMicroFrontend({
    name: "products",
    url: "/micro-frontends/products.js",
    container: "products-container",
  });

  // Setup inter-app communication
  eventBus.subscribe("user:login", (user) => {
    console.log("User logged in:", user);
    // Notify all micro-frontends
  });
}
```

## Architecture Patterns Comparison Matrix

```typescript
interface ArchitecturePattern {
  name: string;
  complexity: "Low" | "Medium" | "High";
  testability: "Low" | "Medium" | "High";
  scalability: "Low" | "Medium" | "High";
  learningCurve: "Easy" | "Medium" | "Steep";
  bestFor: string[];
  drawbacks: string[];
}

const architectureComparison: ArchitecturePattern[] = [
  {
    name: "MVC",
    complexity: "Medium",
    testability: "Medium",
    scalability: "Medium",
    learningCurve: "Easy",
    bestFor: ["Traditional web apps", "Server-rendered apps", "Simple SPAs"],
    drawbacks: ["Tight coupling", "Complex data flow", "Harder to test"],
  },
  {
    name: "MVP",
    complexity: "Medium",
    testability: "High",
    scalability: "Medium",
    learningCurve: "Medium",
    bestFor: ["Desktop apps", "Mobile apps", "Testable UIs"],
    drawbacks: ["More boilerplate", "Passive view limitations"],
  },
  {
    name: "MVVM",
    complexity: "Medium",
    testability: "High",
    scalability: "High",
    learningCurve: "Medium",
    bestFor: ["Data-binding frameworks", "Reactive UIs", "Complex forms"],
    drawbacks: ["Learning curve", "Debugging complexity", "Memory leaks risk"],
  },
  {
    name: "Flux",
    complexity: "Medium",
    testability: "High",
    scalability: "High",
    learningCurve: "Medium",
    bestFor: ["React apps", "Predictable state", "Complex data flows"],
    drawbacks: ["Boilerplate code", "Overkill for simple apps"],
  },
  {
    name: "Redux",
    complexity: "High",
    testability: "High",
    scalability: "High",
    learningCurve: "Steep",
    bestFor: ["Large React apps", "Time-travel debugging", "Complex state"],
    drawbacks: [
      "Lots of boilerplate",
      "Steep learning curve",
      "Over-engineering risk",
    ],
  },
  {
    name: "Clean Architecture",
    complexity: "High",
    testability: "High",
    scalability: "High",
    learningCurve: "Steep",
    bestFor: ["Enterprise apps", "Long-term projects", "Team collaboration"],
    drawbacks: [
      "High initial cost",
      "Over-engineering risk",
      "Lots of abstractions",
    ],
  },
  {
    name: "Micro-Frontends",
    complexity: "High",
    testability: "High",
    scalability: "High",
    learningCurve: "Steep",
    bestFor: ["Large teams", "Independent deployments", "Polyglot frontends"],
    drawbacks: ["Complexity", "Performance overhead", "Shared dependencies"],
  },
];
```

## Best Practices

1. **Choose based on requirements** - Don't over-engineer small apps
2. **Separation of concerns** - Keep business logic separate from UI
3. **Testability first** - Architecture should facilitate testing
4. **Unidirectional data flow** - Easier to reason about and debug
5. **Dependency injection** - Loosely coupled, testable code
6. **Single responsibility** - Each component/module does one thing
7. **Immutable state** - Predictable state changes
8. **Progressive enhancement** - Start simple, add complexity as needed
9. **Documentation** - Architecture decisions should be documented
10. **Team alignment** - Architecture should match team structure

## Key Takeaways

1. **MVC separates concerns** but can lead to tight coupling - best for traditional web apps with server-side rendering

2. **MVP makes testing easier** with passive views and testable presenters - ideal for mobile and desktop applications

3. **MVVM enables data binding** with observable ViewModels - perfect for reactive UIs and frameworks like Angular/Vue

4. **Flux enforces unidirectional flow** preventing complex data mutations - foundational pattern for React applications

5. **Redux centralizes state** with pure reducers and middleware - industry standard for large React apps with complex state

6. **Clean Architecture maximizes testability** with dependency inversion - enterprise-grade pattern for long-term maintainability

7. **Micro-Frontends enable team autonomy** with independent deployments - scales to large organizations with multiple teams

8. **Pattern selection matters** - complexity should match application needs, avoid over-engineering small projects

9. **All patterns trade complexity for benefits** - MVC is simple but less testable, Clean Architecture is testable but complex

10. **Modern frontends favor composition** - Redux + Clean Architecture, Micro-Frontends + Event-Driven, mix patterns strategically
