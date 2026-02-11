# 📱💻 Desktop & Mobile Application Development

> **Building Native-Quality Web Applications for Desktop and Mobile Platforms**

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [Progressive Web Apps (PWA)](#progressive-web-apps-pwa)
- [Mobile-First Development](#mobile-first-development)
- [Responsive vs Adaptive Design](#responsive-vs-adaptive-design)
- [Touch & Gesture Interactions](#touch--gesture-interactions)
- [Desktop Application Patterns](#desktop-application-patterns)
- [Mobile Application Patterns](#mobile-application-patterns)
- [Offline Functionality](#offline-functionality)
- [Device APIs](#device-apis)
- [Performance Optimization](#performance-optimization)
- [App Distribution](#app-distribution)
- [Platform-Specific Considerations](#platform-specific-considerations)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)

---

## Introduction

Modern web technologies enable building applications that blur the line between web and native apps. With PWAs, responsive design, and device APIs, developers can create experiences that work seamlessly across desktop and mobile platforms.

### Application Types

```typescript
interface ApplicationType {
  type: string;
  technologies: string[];
  platforms: string[];
  capabilities: string[];
  limitations: string[];
}

const appTypes: ApplicationType[] = [
  {
    type: "Progressive Web App (PWA)",
    technologies: ["Service Workers", "Web App Manifest", "HTTPS"],
    platforms: ["Web", "Mobile", "Desktop"],
    capabilities: [
      "Offline support",
      "Push notifications",
      "Add to home screen",
      "Background sync",
      "Installation",
    ],
    limitations: [
      "Limited iOS support",
      "No app store distribution (iOS)",
      "Restricted device APIs",
    ],
  },
  {
    type: "Hybrid App",
    technologies: ["Capacitor", "Ionic", "Cordova"],
    platforms: ["iOS", "Android", "Web"],
    capabilities: [
      "Native device APIs",
      "App store distribution",
      "Single codebase",
      "Native UI components",
    ],
    limitations: [
      "Performance overhead",
      "Larger bundle size",
      "Plugin dependencies",
    ],
  },
  {
    type: "Electron Desktop App",
    technologies: ["Electron", "Node.js", "Chromium"],
    platforms: ["Windows", "macOS", "Linux"],
    capabilities: [
      "Full system access",
      "Native menus",
      "System tray",
      "Auto-updates",
    ],
    limitations: ["Large bundle size", "Memory intensive", "Security concerns"],
  },
  {
    type: "Responsive Web App",
    technologies: ["HTML", "CSS", "JavaScript"],
    platforms: ["All browsers"],
    capabilities: [
      "Universal access",
      "No installation",
      "Automatic updates",
      "SEO friendly",
    ],
    limitations: [
      "No offline by default",
      "Limited device access",
      "Network dependent",
    ],
  },
];
```

---

## Progressive Web Apps (PWA)

### PWA Requirements

```typescript
/**
 * Core PWA implementation
 * Three pillars: Reliable, Fast, Engaging
 */

// 1. Web App Manifest
interface WebAppManifest {
  name: string;
  short_name: string;
  description: string;
  start_url: string;
  display: "standalone" | "fullscreen" | "minimal-ui" | "browser";
  background_color: string;
  theme_color: string;
  orientation?: "portrait" | "landscape" | "any";
  icons: Array<{
    src: string;
    sizes: string;
    type: string;
    purpose?: "any" | "maskable" | "monochrome";
  }>;
  categories?: string[];
  shortcuts?: Array<{
    name: string;
    url: string;
    description?: string;
    icons?: Array<{ src: string; sizes: string }>;
  }>;
}

// manifest.json
const manifest: WebAppManifest = {
  name: "My Progressive Web App",
  short_name: "MyPWA",
  description: "A fully-featured PWA",
  start_url: "/",
  display: "standalone",
  background_color: "#ffffff",
  theme_color: "#4285f4",
  orientation: "portrait",
  icons: [
    {
      src: "/icons/icon-72x72.png",
      sizes: "72x72",
      type: "image/png",
    },
    {
      src: "/icons/icon-96x96.png",
      sizes: "96x96",
      type: "image/png",
    },
    {
      src: "/icons/icon-128x128.png",
      sizes: "128x128",
      type: "image/png",
    },
    {
      src: "/icons/icon-144x144.png",
      sizes: "144x144",
      type: "image/png",
    },
    {
      src: "/icons/icon-152x152.png",
      sizes: "152x152",
      type: "image/png",
    },
    {
      src: "/icons/icon-192x192.png",
      sizes: "192x192",
      type: "image/png",
      purpose: "any",
    },
    {
      src: "/icons/icon-384x384.png",
      sizes: "384x384",
      type: "image/png",
    },
    {
      src: "/icons/icon-512x512.png",
      sizes: "512x512",
      type: "image/png",
      purpose: "maskable",
    },
  ],
  categories: ["productivity", "utilities"],
  shortcuts: [
    {
      name: "New Document",
      url: "/new",
      description: "Create a new document",
    },
    {
      name: "Search",
      url: "/search",
      description: "Search documents",
    },
  ],
};
```

### Service Worker Implementation

```typescript
// service-worker.ts
const CACHE_NAME = "my-pwa-v1";
const urlsToCache = [
  "/",
  "/index.html",
  "/styles/main.css",
  "/scripts/app.js",
  "/images/logo.png",
];

// Install event - cache assets
self.addEventListener("install", (event: ExtendableEvent) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      console.log("Opened cache");
      return cache.addAll(urlsToCache);
    }),
  );
});

// Fetch event - serve from cache, fallback to network
self.addEventListener("fetch", (event: FetchEvent) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      // Cache hit - return response
      if (response) {
        return response;
      }

      // Clone request for network
      const fetchRequest = event.request.clone();

      return fetch(fetchRequest).then((response) => {
        // Check if valid response
        if (!response || response.status !== 200 || response.type !== "basic") {
          return response;
        }

        // Clone response for cache
        const responseToCache = response.clone();

        caches.open(CACHE_NAME).then((cache) => {
          cache.put(event.request, responseToCache);
        });

        return response;
      });
    }),
  );
});

// Activate event - clean up old caches
self.addEventListener("activate", (event: ExtendableEvent) => {
  const cacheWhitelist = [CACHE_NAME];

  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        }),
      );
    }),
  );
});
```

### PWA Installation

```typescript
/**
 * Handle PWA installation
 */
class PWAInstaller {
  private deferredPrompt: any = null;
  private isInstalled = false;

  constructor() {
    this.init();
  }

  private init(): void {
    // Detect if app is installed
    if (window.matchMedia("(display-mode: standalone)").matches) {
      this.isInstalled = true;
      console.log("PWA is installed");
    }

    // Capture install prompt
    window.addEventListener("beforeinstallprompt", (e) => {
      e.preventDefault();
      this.deferredPrompt = e;
      this.showInstallButton();
    });

    // Detect successful installation
    window.addEventListener("appinstalled", () => {
      console.log("PWA was installed");
      this.isInstalled = true;
      this.hideInstallButton();
    });
  }

  async promptInstall(): Promise<boolean> {
    if (!this.deferredPrompt) {
      console.log("Install prompt not available");
      return false;
    }

    // Show install prompt
    this.deferredPrompt.prompt();

    // Wait for user choice
    const { outcome } = await this.deferredPrompt.userChoice;
    console.log(`User response: ${outcome}`);

    // Clear prompt
    this.deferredPrompt = null;

    return outcome === "accepted";
  }

  private showInstallButton(): void {
    const button = document.getElementById("install-button");
    if (button) {
      button.style.display = "block";
      button.addEventListener("click", () => this.promptInstall());
    }
  }

  private hideInstallButton(): void {
    const button = document.getElementById("install-button");
    if (button) {
      button.style.display = "none";
    }
  }

  isAppInstalled(): boolean {
    return this.isInstalled;
  }
}

// Usage
const installer = new PWAInstaller();
```

---

## Mobile-First Development

### Mobile-First CSS Strategy

```css
/* ✅ Start with mobile styles (default) */
.container {
  padding: 1rem;
  display: block;
}

.card {
  width: 100%;
  margin-bottom: 1rem;
}

/* ✅ Enhance for tablets */
@media (min-width: 768px) {
  .container {
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
  }

  .card {
    width: calc(50% - 1rem);
    display: inline-block;
  }
}

/* ✅ Enhance for desktop */
@media (min-width: 1024px) {
  .container {
    padding: 3rem;
  }

  .card {
    width: calc(33.333% - 1rem);
  }
}

/* ✅ Large screens */
@media (min-width: 1440px) {
  .container {
    max-width: 1400px;
  }
}
```

### Mobile-First React Component

```typescript
import { useState, useEffect } from "react";

interface Breakpoint {
  isMobile: boolean;
  isTablet: boolean;
  isDesktop: boolean;
  isLargeDesktop: boolean;
}

/**
 * Hook for responsive breakpoints
 */
function useBreakpoint(): Breakpoint {
  const [breakpoint, setBreakpoint] = useState<Breakpoint>({
    isMobile: true,
    isTablet: false,
    isDesktop: false,
    isLargeDesktop: false,
  });

  useEffect(() => {
    const updateBreakpoint = () => {
      const width = window.innerWidth;

      setBreakpoint({
        isMobile: width < 768,
        isTablet: width >= 768 && width < 1024,
        isDesktop: width >= 1024 && width < 1440,
        isLargeDesktop: width >= 1440,
      });
    };

    updateBreakpoint();
    window.addEventListener("resize", updateBreakpoint);

    return () => window.removeEventListener("resize", updateBreakpoint);
  }, []);

  return breakpoint;
}

// Usage
function ResponsiveComponent() {
  const { isMobile, isTablet, isDesktop } = useBreakpoint();

  return (
    <div>
      {isMobile && <MobileNav />}
      {(isTablet || isDesktop) && <DesktopNav />}

      {isMobile ? (
        <MobileLayout />
      ) : (
        <DesktopLayout />
      )}
    </div>
  );
}
```

---

## Responsive vs Adaptive Design

### Responsive Design

```typescript
/**
 * Responsive: Flexible layouts that adapt fluidly
 */

// CSS approach
const responsiveCSS = `
.container {
  width: 100%;
  max-width: 1200px;
  padding: clamp(1rem, 5vw, 3rem);
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: clamp(1rem, 3vw, 2rem);
}

.text {
  font-size: clamp(1rem, 2.5vw, 1.5rem);
}
`;

// TypeScript implementation
interface ResponsiveConfig {
  minWidth: number;
  maxWidth: number;
  fluidScale: boolean;
  breakpoints: number[];
}

class ResponsiveLayout {
  private config: ResponsiveConfig;

  constructor(config: ResponsiveConfig) {
    this.config = config;
  }

  // Calculate fluid typography
  getFluidSize(min: number, max: number): string {
    const { minWidth, maxWidth } = this.config;
    return `clamp(${min}rem, ${((max - min) / (maxWidth - minWidth)) * 100}vw, ${max}rem)`;
  }

  // Calculate responsive columns
  getColumns(): number {
    const width = window.innerWidth;
    if (width < 768) return 1;
    if (width < 1024) return 2;
    if (width < 1440) return 3;
    return 4;
  }
}
```

### Adaptive Design

```typescript
/**
 * Adaptive: Specific layouts for specific breakpoints
 */
interface AdaptiveBreakpoint {
  name: string;
  minWidth: number;
  layout: "mobile" | "tablet" | "desktop";
  columns: number;
}

class AdaptiveLayout {
  private breakpoints: AdaptiveBreakpoint[] = [
    {
      name: "mobile",
      minWidth: 0,
      layout: "mobile",
      columns: 1,
    },
    {
      name: "tablet",
      minWidth: 768,
      layout: "tablet",
      columns: 2,
    },
    {
      name: "desktop",
      minWidth: 1024,
      layout: "desktop",
      columns: 3,
    },
  ];

  getCurrentLayout(): "mobile" | "tablet" | "desktop" {
    const width = window.innerWidth;

    // Find matching breakpoint
    const breakpoint = [...this.breakpoints]
      .reverse()
      .find((bp) => width >= bp.minWidth);

    return breakpoint?.layout || "mobile";
  }

  // Load layout-specific components
  async loadLayoutComponents(): Promise<any> {
    const layout = this.getCurrentLayout();

    switch (layout) {
      case "mobile":
        return import("./layouts/MobileLayout");
      case "tablet":
        return import("./layouts/TabletLayout");
      case "desktop":
        return import("./layouts/DesktopLayout");
    }
  }
}

// React implementation
function AdaptiveApp() {
  const [Layout, setLayout] = useState<any>(null);
  const adaptive = new AdaptiveLayout();

  useEffect(() => {
    const loadLayout = async () => {
      const layout = await adaptive.loadLayoutComponents();
      setLayout(() => layout.default);
    };

    loadLayout();

    window.addEventListener("resize", loadLayout);
    return () => window.removeEventListener("resize", loadLayout);
  }, []);

  if (!Layout) return <div>Loading...</div>;

  return <Layout />;
}
```

---

## Touch & Gesture Interactions

### Touch Event Handling

```typescript
/**
 * Touch events for mobile devices
 */
class TouchHandler {
  private element: HTMLElement;
  private touchStartX = 0;
  private touchStartY = 0;
  private touchEndX = 0;
  private touchEndY = 0;

  constructor(element: HTMLElement) {
    this.element = element;
    this.init();
  }

  private init(): void {
    // Prevent default touch behavior
    this.element.style.touchAction = "none";

    this.element.addEventListener("touchstart", this.handleTouchStart);
    this.element.addEventListener("touchmove", this.handleTouchMove);
    this.element.addEventListener("touchend", this.handleTouchEnd);
  }

  private handleTouchStart = (e: TouchEvent): void => {
    this.touchStartX = e.touches[0].clientX;
    this.touchStartY = e.touches[0].clientY;
  };

  private handleTouchMove = (e: TouchEvent): void => {
    // Prevent scrolling while touching
    e.preventDefault();

    this.touchEndX = e.touches[0].clientX;
    this.touchEndY = e.touches[0].clientY;

    // Calculate movement
    const deltaX = this.touchEndX - this.touchStartX;
    const deltaY = this.touchEndY - this.touchStartY;

    // Apply transformation
    this.element.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
  };

  private handleTouchEnd = (): void => {
    const deltaX = this.touchEndX - this.touchStartX;
    const deltaY = this.touchEndY - this.touchStartY;

    // Detect swipe direction
    if (Math.abs(deltaX) > Math.abs(deltaY)) {
      if (deltaX > 50) {
        this.onSwipeRight();
      } else if (deltaX < -50) {
        this.onSwipeLeft();
      }
    } else {
      if (deltaY > 50) {
        this.onSwipeDown();
      } else if (deltaY < -50) {
        this.onSwipeUp();
      }
    }

    // Reset position
    this.element.style.transform = "";
  };

  // Override these methods
  protected onSwipeLeft(): void {
    console.log("Swiped left");
  }

  protected onSwipeRight(): void {
    console.log("Swiped right");
  }

  protected onSwipeUp(): void {
    console.log("Swiped up");
  }

  protected onSwipeDown(): void {
    console.log("Swiped down");
  }

  destroy(): void {
    this.element.removeEventListener("touchstart", this.handleTouchStart);
    this.element.removeEventListener("touchmove", this.handleTouchMove);
    this.element.removeEventListener("touchend", this.handleTouchEnd);
  }
}

// Usage
const carousel = document.querySelector(".carousel") as HTMLElement;
const touchHandler = new TouchHandler(carousel);
```

### Gesture Recognition

```typescript
/**
 * Advanced gesture recognition
 */
interface GestureEvent {
  type: "tap" | "doubletap" | "pinch" | "rotate" | "swipe";
  startX: number;
  startY: number;
  endX: number;
  endY: number;
  distance?: number;
  angle?: number;
}

class GestureRecognizer {
  private element: HTMLElement;
  private touches: Touch[] = [];
  private startTime = 0;
  private lastTap = 0;

  constructor(element: HTMLElement) {
    this.element = element;
    this.init();
  }

  private init(): void {
    this.element.addEventListener("touchstart", this.onTouchStart);
    this.element.addEventListener("touchmove", this.onTouchMove);
    this.element.addEventListener("touchend", this.onTouchEnd);
  }

  private onTouchStart = (e: TouchEvent): void => {
    this.touches = Array.from(e.touches);
    this.startTime = Date.now();

    // Multi-touch gestures
    if (this.touches.length === 2) {
      this.onPinchStart();
    }
  };

  private onTouchMove = (e: TouchEvent): void => {
    if (e.touches.length === 2) {
      this.onPinchMove(Array.from(e.touches));
    }
  };

  private onTouchEnd = (e: TouchEvent): void => {
    const duration = Date.now() - this.startTime;
    const touch = this.touches[0];

    // Detect tap (quick touch)
    if (duration < 200 && this.touches.length === 1) {
      // Check for double tap
      if (Date.now() - this.lastTap < 300) {
        this.onDoubleTap(touch);
      } else {
        this.onTap(touch);
      }
      this.lastTap = Date.now();
    }

    this.touches = [];
  };

  private onPinchStart(): void {
    console.log("Pinch started");
  }

  private onPinchMove(touches: Touch[]): void {
    const [touch1, touch2] = touches;

    // Calculate distance between touches
    const distance = Math.hypot(
      touch2.clientX - touch1.clientX,
      touch2.clientY - touch1.clientY,
    );

    // Calculate angle
    const angle = Math.atan2(
      touch2.clientY - touch1.clientY,
      touch2.clientX - touch1.clientX,
    );

    console.log("Pinch/rotate", { distance, angle });
  }

  protected onTap(touch: Touch): void {
    console.log("Tap at", touch.clientX, touch.clientY);
  }

  protected onDoubleTap(touch: Touch): void {
    console.log("Double tap at", touch.clientX, touch.clientY);
  }
}
```

### Touch-Friendly UI

```css
/* ✅ Touch-friendly button sizes */
.touch-button {
  /* Minimum 44x44px for touch targets */
  min-width: 44px;
  min-height: 44px;
  padding: 12px 24px;

  /* Prevent text selection on tap */
  user-select: none;
  -webkit-user-select: none;

  /* Remove tap highlight */
  -webkit-tap-highlight-color: transparent;

  /* Smooth touch feedback */
  transition: transform 0.1s ease;
}

.touch-button:active {
  transform: scale(0.95);
}

/* ✅ Spacing for fat fingers */
.touch-menu li {
  padding: 16px;
  min-height: 56px;
}

/* ✅ Prevent zoom on input focus (iOS) */
input,
select,
textarea {
  font-size: 16px; /* Prevents zoom on iOS */
}
```

---

## Desktop Application Patterns

### Electron App Structure

```typescript
// main.ts - Electron main process
import { app, BrowserWindow, Menu, Tray, ipcMain } from "electron";
import path from "path";

class DesktopApp {
  private mainWindow: BrowserWindow | null = null;
  private tray: Tray | null = null;

  constructor() {
    app.whenReady().then(() => {
      this.createWindow();
      this.createMenu();
      this.createTray();
      this.setupIPC();
    });

    app.on("window-all-closed", () => {
      if (process.platform !== "darwin") {
        app.quit();
      }
    });

    app.on("activate", () => {
      if (BrowserWindow.getAllWindows().length === 0) {
        this.createWindow();
      }
    });
  }

  private createWindow(): void {
    this.mainWindow = new BrowserWindow({
      width: 1200,
      height: 800,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: path.join(__dirname, "preload.js"),
      },
      titleBarStyle: "hidden", // Custom title bar
      frame: false, // Frameless window
    });

    // Load app
    if (process.env.NODE_ENV === "development") {
      this.mainWindow.loadURL("http://localhost:3000");
      this.mainWindow.webContents.openDevTools();
    } else {
      this.mainWindow.loadFile("dist/index.html");
    }
  }

  private createMenu(): void {
    const template: Electron.MenuItemConstructorOptions[] = [
      {
        label: "File",
        submenu: [
          { label: "New", accelerator: "CmdOrCtrl+N", click: () => {} },
          { label: "Open", accelerator: "CmdOrCtrl+O", click: () => {} },
          { type: "separator" },
          { label: "Quit", accelerator: "CmdOrCtrl+Q", role: "quit" },
        ],
      },
      {
        label: "Edit",
        submenu: [
          { role: "undo" },
          { role: "redo" },
          { type: "separator" },
          { role: "cut" },
          { role: "copy" },
          { role: "paste" },
        ],
      },
      {
        label: "View",
        submenu: [
          { role: "reload" },
          { role: "forceReload" },
          { role: "toggleDevTools" },
          { type: "separator" },
          { role: "resetZoom" },
          { role: "zoomIn" },
          { role: "zoomOut" },
        ],
      },
    ];

    const menu = Menu.buildFromTemplate(template);
    Menu.setApplicationMenu(menu);
  }

  private createTray(): void {
    this.tray = new Tray(path.join(__dirname, "icon.png"));

    const contextMenu = Menu.buildFromTemplate([
      { label: "Show App", click: () => this.mainWindow?.show() },
      { label: "Quit", click: () => app.quit() },
    ]);

    this.tray.setContextMenu(contextMenu);
    this.tray.setToolTip("My Desktop App");
  }

  private setupIPC(): void {
    // Handle IPC messages from renderer
    ipcMain.handle("get-app-version", () => {
      return app.getVersion();
    });

    ipcMain.handle("minimize-window", () => {
      this.mainWindow?.minimize();
    });

    ipcMain.handle("maximize-window", () => {
      if (this.mainWindow?.isMaximized()) {
        this.mainWindow.unmaximize();
      } else {
        this.mainWindow?.maximize();
      }
    });

    ipcMain.handle("close-window", () => {
      this.mainWindow?.close();
    });
  }
}

new DesktopApp();
```

### Desktop UI Components

```typescript
// Custom title bar for frameless window
function CustomTitleBar() {
  const minimize = () => {
    // @ts-ignore
    window.electron.minimizeWindow();
  };

  const maximize = () => {
    // @ts-ignore
    window.electron.maximizeWindow();
  };

  const close = () => {
    // @ts-ignore
    window.electron.closeWindow();
  };

  return (
    <div className="title-bar">
      <div className="title-bar-drag-region">
        <div className="app-name">My Desktop App</div>
      </div>
      <div className="window-controls">
        <button onClick={minimize} className="window-control">
          −
        </button>
        <button onClick={maximize} className="window-control">
          □
        </button>
        <button onClick={close} className="window-control close">
          ×
        </button>
      </div>
    </div>
  );
}
```

```css
/* Title bar styles */
.title-bar {
  display: flex;
  justify-content: space-between;
  height: 32px;
  background: #2c2c2c;
  color: white;
  -webkit-app-region: drag; /* Makes entire bar draggable */
}

.window-controls {
  display: flex;
  -webkit-app-region: no-drag; /* Buttons not draggable */
}

.window-control {
  width: 46px;
  height: 32px;
  background: transparent;
  border: none;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

.window-control:hover {
  background: rgba(255, 255, 255, 0.1);
}

.window-control.close:hover {
  background: #e81123;
}
```

---

## Mobile Application Patterns

### Bottom Navigation

```typescript
// Mobile-first bottom navigation
function BottomNavigation() {
  const [active, setActive] = useState("home");

  const navItems = [
    { id: "home", icon: "🏠", label: "Home" },
    { id: "search", icon: "🔍", label: "Search" },
    { id: "favorites", icon: "❤️", label: "Favorites" },
    { id: "profile", icon: "👤", label: "Profile" },
  ];

  return (
    <nav className="bottom-nav">
      {navItems.map((item) => (
        <button
          key={item.id}
          className={`nav-item ${active === item.id ? "active" : ""}`}
          onClick={() => setActive(item.id)}
        >
          <span className="nav-icon">{item.icon}</span>
          <span className="nav-label">{item.label}</span>
        </button>
      ))}
    </nav>
  );
}
```

```css
.bottom-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  background: white;
  border-top: 1px solid #e0e0e0;
  padding: 8px 0;
  z-index: 1000;

  /* Safe area for iOS notch */
  padding-bottom: calc(8px + env(safe-area-inset-bottom));
}

.nav-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 8px;
  background: none;
  border: none;
  color: #757575;
  min-height: 56px; /* Touch-friendly */
}

.nav-item.active {
  color: #1976d2;
}

.nav-icon {
  font-size: 24px;
}

.nav-label {
  font-size: 12px;
}
```

### Pull-to-Refresh

```typescript
/**
 * Pull-to-refresh for mobile
 */
class PullToRefresh {
  private element: HTMLElement;
  private refreshCallback: () => Promise<void>;
  private startY = 0;
  private currentY = 0;
  private isPulling = false;
  private threshold = 80;

  constructor(element: HTMLElement, onRefresh: () => Promise<void>) {
    this.element = element;
    this.refreshCallback = onRefresh;
    this.init();
  }

  private init(): void {
    this.element.addEventListener("touchstart", this.onTouchStart);
    this.element.addEventListener("touchmove", this.onTouchMove);
    this.element.addEventListener("touchend", this.onTouchEnd);
  }

  private onTouchStart = (e: TouchEvent): void => {
    // Only trigger if scrolled to top
    if (this.element.scrollTop === 0) {
      this.startY = e.touches[0].clientY;
      this.isPulling = true;
    }
  };

  private onTouchMove = (e: TouchEvent): void => {
    if (!this.isPulling) return;

    this.currentY = e.touches[0].clientY;
    const pullDistance = this.currentY - this.startY;

    if (pullDistance > 0) {
      // Apply resistance curve
      const resistance = Math.min(pullDistance / 2, this.threshold);
      this.element.style.transform = `translateY(${resistance}px)`;

      // Show loading indicator if past threshold
      if (resistance >= this.threshold) {
        this.showRefreshIndicator();
      }
    }
  };

  private onTouchEnd = async (): Promise<void> => {
    if (!this.isPulling) return;

    const pullDistance = this.currentY - this.startY;

    if (pullDistance >= this.threshold) {
      // Trigger refresh
      await this.refreshCallback();
    }

    // Reset
    this.element.style.transform = "";
    this.hideRefreshIndicator();
    this.isPulling = false;
    this.startY = 0;
    this.currentY = 0;
  };

  private showRefreshIndicator(): void {
    const indicator = document.createElement("div");
    indicator.className = "refresh-indicator";
    indicator.textContent = "⟳";
    this.element.prepend(indicator);
  }

  private hideRefreshIndicator(): void {
    const indicator = this.element.querySelector(".refresh-indicator");
    indicator?.remove();
  }
}

// Usage
const container = document.querySelector(".content") as HTMLElement;
new PullToRefresh(container, async () => {
  await fetchFreshData();
});
```

### Mobile Drawer/Sheet

```typescript
// Bottom sheet for mobile
function BottomSheet({
  isOpen,
  onClose,
  children,
}: {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}) {
  const sheetRef = useRef<HTMLDivElement>(null);
  const [startY, setStartY] = useState(0);
  const [currentY, setCurrentY] = useState(0);

  const handleTouchStart = (e: React.TouchEvent) => {
    setStartY(e.touches[0].clientY);
  };

  const handleTouchMove = (e: React.TouchEvent) => {
    const touch = e.touches[0].clientY;
    setCurrentY(touch);

    const diff = touch - startY;
    if (diff > 0 && sheetRef.current) {
      sheetRef.current.style.transform = `translateY(${diff}px)`;
    }
  };

  const handleTouchEnd = () => {
    const diff = currentY - startY;

    // Close if swiped down more than 100px
    if (diff > 100) {
      onClose();
    } else if (sheetRef.current) {
      sheetRef.current.style.transform = "";
    }
  };

  if (!isOpen) return null;

  return (
    <>
      <div className="sheet-overlay" onClick={onClose} />
      <div
        ref={sheetRef}
        className="bottom-sheet"
        onTouchStart={handleTouchStart}
        onTouchMove={handleTouchMove}
        onTouchEnd={handleTouchEnd}
      >
        <div className="sheet-handle" />
        <div className="sheet-content">{children}</div>
      </div>
    </>
  );
}
```

```css
.sheet-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 999;
  animation: fadeIn 0.3s;
}

.bottom-sheet {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  background: white;
  border-radius: 16px 16px 0 0;
  z-index: 1000;
  animation: slideUp 0.3s;
  transition: transform 0.3s;

  /* Safe area for iOS */
  padding-bottom: env(safe-area-inset-bottom);
}

.sheet-handle {
  width: 40px;
  height: 4px;
  background: #ccc;
  border-radius: 2px;
  margin: 12px auto;
}

.sheet-content {
  padding: 16px;
  max-height: 80vh;
  overflow-y: auto;
}

@keyframes slideUp {
  from {
    transform: translateY(100%);
  }
  to {
    transform: translateY(0);
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```

---

## Offline Functionality

### Service Worker Caching Strategies

```typescript
/**
 * Different caching strategies for PWA
 */

// 1. Cache First (good for static assets)
async function cacheFirst(request: Request): Promise<Response> {
  const cached = await caches.match(request);
  if (cached) {
    return cached;
  }

  const response = await fetch(request);
  const cache = await caches.open("static-v1");
  cache.put(request, response.clone());

  return response;
}

// 2. Network First (good for dynamic content)
async function networkFirst(request: Request): Promise<Response> {
  try {
    const response = await fetch(request);
    const cache = await caches.open("dynamic-v1");
    cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await caches.match(request);
    if (cached) {
      return cached;
    }
    throw new Error("No network and no cache");
  }
}

// 3. Stale While Revalidate (good for frequently updated content)
async function staleWhileRevalidate(request: Request): Promise<Response> {
  const cached = await caches.match(request);

  const fetchPromise = fetch(request).then((response) => {
    const cache = caches.open("dynamic-v1");
    cache.then((c) => c.put(request, response.clone()));
    return response;
  });

  return cached || fetchPromise;
}

// 4. Cache Only (for offline-first apps)
async function cacheOnly(request: Request): Promise<Response> {
  const cached = await caches.match(request);
  if (!cached) {
    throw new Error("Not in cache");
  }
  return cached;
}

// 5. Network Only (for always-fresh content)
async function networkOnly(request: Request): Promise<Response> {
  return fetch(request);
}
```

### Offline State Management

```typescript
/**
 * Handle online/offline state
 */
class OfflineManager {
  private isOnline = navigator.onLine;
  private listeners: Array<(online: boolean) => void> = [];
  private pendingRequests: Array<() => Promise<any>> = [];

  constructor() {
    window.addEventListener("online", this.handleOnline);
    window.addEventListener("offline", this.handleOffline);
  }

  private handleOnline = () => {
    this.isOnline = true;
    this.notify();
    this.processPendingRequests();
  };

  private handleOffline = () => {
    this.isOnline = false;
    this.notify();
  };

  private notify(): void {
    this.listeners.forEach((listener) => listener(this.isOnline));
  }

  subscribe(listener: (online: boolean) => void): () => void {
    this.listeners.push(listener);

    // Return unsubscribe function
    return () => {
      this.listeners = this.listeners.filter((l) => l !== listener);
    };
  }

  // Queue requests when offline
  async queueRequest(request: () => Promise<any>): Promise<void> {
    if (this.isOnline) {
      await request();
    } else {
      this.pendingRequests.push(request);
      // Save to IndexedDB for persistence
      await this.savePendingRequests();
    }
  }

  private async processPendingRequests(): Promise<void> {
    while (this.pendingRequests.length > 0 && this.isOnline) {
      const request = this.pendingRequests.shift();
      try {
        await request?.();
      } catch (error) {
        console.error("Failed to process pending request:", error);
        // Re-queue if failed
        if (request) {
          this.pendingRequests.unshift(request);
        }
        break;
      }
    }

    await this.savePendingRequests();
  }

  private async savePendingRequests(): Promise<void> {
    // Save to IndexedDB
    const db = await this.openDB();
    const tx = db.transaction("pending-requests", "readwrite");
    const store = tx.objectStore("pending-requests");

    await store.clear();

    for (const request of this.pendingRequests) {
      await store.add({ request: request.toString() });
    }
  }

  private async openDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open("offline-manager", 1);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);

      request.onupgradeneeded = () => {
        const db = request.result;
        if (!db.objectStoreNames.contains("pending-requests")) {
          db.createObjectStore("pending-requests", { autoIncrement: true });
        }
      };
    });
  }

  getStatus(): boolean {
    return this.isOnline;
  }
}

// Usage
const offlineManager = new OfflineManager();

offlineManager.subscribe((online) => {
  if (online) {
    showNotification("Back online");
  } else {
    showNotification("You are offline");
  }
});

// Queue request when offline
await offlineManager.queueRequest(async () => {
  await saveData(formData);
});
```

---

## Device APIs

### Accessing Device Features

```typescript
/**
 * Wrapper for device APIs
 */
class DeviceAPIs {
  // Camera access
  static async getCamera(): Promise<MediaStream> {
    try {
      return await navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: "user", // or "environment" for back camera
          width: { ideal: 1280 },
          height: { ideal: 720 },
        },
      });
    } catch (error) {
      throw new Error("Camera access denied");
    }
  }

  // Geolocation
  static async getLocation(): Promise<GeolocationPosition> {
    return new Promise((resolve, reject) => {
      if (!navigator.geolocation) {
        reject(new Error("Geolocation not supported"));
        return;
      }

      navigator.geolocation.getCurrentPosition(resolve, reject, {
        enableHighAccuracy: true,
        timeout: 5000,
        maximumAge: 0,
      });
    });
  }

  // Device orientation
  static requestOrientation(): Promise<void> {
    return new Promise((resolve, reject) => {
      if (typeof DeviceOrientationEvent === "undefined") {
        reject(new Error("Device orientation not supported"));
        return;
      }

      // iOS 13+ requires permission
      if (
        typeof (DeviceOrientationEvent as any).requestPermission === "function"
      ) {
        (DeviceOrientationEvent as any)
          .requestPermission()
          .then((response: string) => {
            if (response === "granted") {
              resolve();
            } else {
              reject(new Error("Permission denied"));
            }
          });
      } else {
        resolve();
      }
    });
  }

  // Vibration
  static vibrate(pattern: number | number[]): boolean {
    if ("vibrate" in navigator) {
      return navigator.vibrate(pattern);
    }
    return false;
  }

  // Device battery
  static async getBattery(): Promise<any> {
    if ("getBattery" in navigator) {
      return await (navigator as any).getBattery();
    }
    throw new Error("Battery API not supported");
  }

  // Share API
  static async share(data: ShareData): Promise<void> {
    if (navigator.share) {
      await navigator.share(data);
    } else {
      throw new Error("Web Share API not supported");
    }
  }

  // Clipboard
  static async copyToClipboard(text: string): Promise<void> {
    if (navigator.clipboard) {
      await navigator.clipboard.writeText(text);
    } else {
      // Fallback
      const textarea = document.createElement("textarea");
      textarea.value = text;
      document.body.appendChild(textarea);
      textarea.select();
      document.execCommand("copy");
      document.body.removeChild(textarea);
    }
  }

  // File picker
  static async pickFile(accept?: string): Promise<File | null> {
    return new Promise((resolve) => {
      const input = document.createElement("input");
      input.type = "file";
      if (accept) input.accept = accept;

      input.onchange = () => {
        resolve(input.files?.[0] || null);
      };

      input.click();
    });
  }

  // Notifications
  static async requestNotificationPermission(): Promise<NotificationPermission> {
    if ("Notification" in window) {
      return await Notification.requestPermission();
    }
    throw new Error("Notifications not supported");
  }

  static async showNotification(
    title: string,
    options?: NotificationOptions,
  ): Promise<void> {
    const permission = await this.requestNotificationPermission();

    if (permission === "granted") {
      new Notification(title, options);
    }
  }
}

// Usage examples
async function examples() {
  // Camera
  const stream = await DeviceAPIs.getCamera();
  const video = document.querySelector("video")!;
  video.srcObject = stream;

  // Location
  const position = await DeviceAPIs.getLocation();
  console.log(position.coords.latitude, position.coords.longitude);

  // Vibrate
  DeviceAPIs.vibrate([200, 100, 200]);

  // Share
  await DeviceAPIs.share({
    title: "Check this out",
    text: "Amazing content",
    url: window.location.href,
  });

  // Notification
  await DeviceAPIs.showNotification("Hello", {
    body: "This is a notification",
    icon: "/icon.png",
  });
}
```

---

## Performance Optimization

### Mobile Performance

```typescript
/**
 * Performance optimizations for mobile
 */
class MobilePerformance {
  // Lazy load images
  static lazyLoadImages(): void {
    if ("IntersectionObserver" in window) {
      const images = document.querySelectorAll("img[data-src]");

      const observer = new IntersectionObserver(
        (entries) => {
          entries.forEach((entry) => {
            if (entry.isIntersecting) {
              const img = entry.target as HTMLImageElement;
              img.src = img.dataset.src!;
              img.removeAttribute("data-src");
              observer.unobserve(img);
            }
          });
        },
        { rootMargin: "50px" },
      );

      images.forEach((img) => observer.observe(img));
    }
  }

  // Reduce reflows/repaints
  static batchDOMUpdates(updates: Array<() => void>): void {
    requestAnimationFrame(() => {
      updates.forEach((update) => update());
    });
  }

  // Throttle scroll events
  static throttleScroll(callback: () => void, delay = 100): () => void {
    let lastCall = 0;

    return () => {
      const now = Date.now();
      if (now - lastCall >= delay) {
        lastCall = now;
        callback();
      }
    };
  }

  // Debounce resize events
  static debounceResize(callback: () => void, delay = 250): () => void {
    let timeout: NodeJS.Timeout;

    return () => {
      clearTimeout(timeout);
      timeout = setTimeout(callback, delay);
    };
  }

  // Optimize animations
  static optimizeAnimation(callback: () => void): void {
    if ("requestIdleCallback" in window) {
      requestIdleCallback(callback, { timeout: 1000 });
    } else {
      setTimeout(callback, 1);
    }
  }

  // Reduce bundle size
  static async lazyLoadComponent(path: string): Promise<any> {
    return await import(/* webpackChunkName: "[request]" */ `${path}`);
  }
}
```

### Code Splitting

```typescript
// React lazy loading
import { lazy, Suspense } from "react";

// Lazy load routes
const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Profile = lazy(() => import("./pages/Profile"));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}

// Component-level code splitting
function HeavyComponent() {
  const [module, setModule] = useState<any>(null);

  useEffect(() => {
    import("./heavy-module").then((mod) => {
      setModule(() => mod.default);
    });
  }, []);

  if (!module) return <div>Loading...</div>;

  const Component = module;
  return <Component />;
}
```

---

## App Distribution

### PWA Installation

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- PWA Meta Tags -->
    <meta name="theme-color" content="#4285f4" />
    <meta name="description" content="My awesome PWA" />

    <!-- iOS Safari -->
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta
      name="apple-mobile-web-app-status-bar-style"
      content="black-translucent"
    />
    <meta name="apple-mobile-web-app-title" content="MyPWA" />

    <!-- iOS Icons -->
    <link rel="apple-touch-icon" href="/icons/icon-152x152.png" />
    <link
      rel="apple-touch-icon"
      sizes="180x180"
      href="/icons/icon-180x180.png"
    />

    <!-- Manifest -->
    <link rel="manifest" href="/manifest.json" />

    <title>My PWA</title>
  </head>
  <body>
    <div id="root"></div>

    <!-- Register service worker -->
    <script>
      if ("serviceWorker" in navigator) {
        window.addEventListener("load", () => {
          navigator.serviceWorker
            .register("/service-worker.js")
            .then((registration) => {
              console.log("SW registered:", registration);
            })
            .catch((error) => {
              console.log("SW registration failed:", error);
            });
        });
      }
    </script>
  </body>
</html>
```

### Electron Packaging

```javascript
// electron-builder.json
{
  "appId": "com.myapp.desktop",
  "productName": "My Desktop App",
  "directories": {
    "output": "dist",
    "buildResources": "build"
  },
  "files": [
    "build/**/*",
    "node_modules/**/*",
    "package.json"
  ],
  "mac": {
    "category": "public.app-category.productivity",
    "icon": "build/icon.icns",
    "target": ["dmg", "zip"]
  },
  "win": {
    "icon": "build/icon.ico",
    "target": ["nsis", "portable"]
  },
  "linux": {
    "icon": "build/icon.png",
    "target": ["AppImage", "deb"]
  }
}
```

---

## Platform-Specific Considerations

### iOS/Safari Quirks

```typescript
/**
 * iOS-specific fixes and considerations
 */
class iOSFixes {
  static isIOS(): boolean {
    return (
      /iPad|iPhone|iPod/.test(navigator.userAgent) && !(window as any).MSStream
    );
  }

  static applyFixes(): void {
    if (!this.isIOS()) return;

    this.fixViewportHeight();
    this.fixInputZoom();
    this.fixScrolling();
    this.fixVideoPlayback();
  }

  // Fix 100vh on iOS
  private static fixViewportHeight(): void {
    const setHeight = () => {
      document.documentElement.style.setProperty(
        "--vh",
        `${window.innerHeight * 0.01}px`,
      );
    };

    setHeight();
    window.addEventListener("resize", setHeight);
  }

  //   CSS: height: calc(var(--vh, 1vh) * 100);

  // Prevent zoom on input focus
  private static fixInputZoom(): void {
    const inputs = document.querySelectorAll("input, select, textarea");
    inputs.forEach((input) => {
      const element = input as HTMLElement;
      const fontSize = window.getComputedStyle(element).fontSize;

      if (parseFloat(fontSize) < 16) {
        element.style.fontSize = "16px";
      }
    });
  }

  // Enable momentum scrolling
  private static fixScrolling(): void {
    document.body.style.webkitOverflowScrolling = "touch";
  }

  // Fix video autoplay
  private static fixVideoPlayback(): void {
    const videos = document.querySelectorAll("video");
    videos.forEach((video) => {
      video.playsInline = true;
      video.muted = true; // Required for autoplay
    });
  }
}

// Apply on load
iOSFixes.applyFixes();
```

### Android Considerations

```typescript
/**
 * Android-specific optimizations
 */
class AndroidOptimizations {
  static isAndroid(): boolean {
    return /Android/i.test(navigator.userAgent);
  }

  static applyOptimizations(): void {
    if (!this.isAndroid()) return;

    this.optimizeTouchDelay();
    this.add304ButtonFix();
  }

  // Remove 300ms tap delay
  private static optimizeTouchDelay(): void {
    // Add to CSS: touch-action: manipulation;
    document.body.style.touchAction = "manipulation";
  }

  // 300px tap delay workaround
  private static add304ButtonFix(): void {
    const meta = document.createElement("meta");
    meta.name = "viewport";
    meta.content = "width=device-width, user-scalable=no";
    document.head.appendChild(meta);
  }
}
```

---

## Best Practices

### 1. Mobile-First Approach

```typescript
// ✅ Start with mobile, enhance for desktop
const styles = {
  mobile: {
    fontSize: "14px",
    padding: "8px",
  },
  desktop: {
    fontSize: "16px",
    padding: "16px",
  },
};
```

### 2. Touch-Friendly UI

- Minimum 44x44px touch targets
- Space interactive elements
- Provide visual feedback on touch
- Support gestures (swipe, pinch)

### 3. Performance First

- Optimize for slow networks
- Minimize bundle size
- Lazy load resources
- Use code splitting
- Implement caching strategies

### 4. Offline Support

- Cache critical resources
- Queue failed requests
- Show offline indicator
- Sync when back online

### 5. Platform Guidelines

- Follow iOS Human Interface Guidelines
- Follow Material Design for Android
- Use native patterns where appropriate
- Test on real devices

### 6. Accessibility

- Support keyboard navigation
- Provide ARIA labels
- Ensure color contrast
- Test with screen readers

---

## Interview Questions

### Conceptual Questions

1. **What is a Progressive Web App (PWA)?**
   - Web app that works like native app
   - Uses service workers, manifest
   - Installable, offline-capable
   - Push notifications

2. **Explain the difference between responsive and adaptive design.**
   - Responsive: Fluid layouts, one codebase
   - Adaptive: Specific breakpoints, distinct layouts
   - Responsive scales, adaptive switches

3. **What are service workers and what are they used for?**
   - JavaScript running in background
   - Intercepts network requests
   - Enables offline functionality
   - Push notifications, background sync

4. **How do you make a web app work offline?**
   - Service worker caching
   - Cache First strategy
   - IndexedDB for data
   - Queue failed requests

### Technical Questions

5. **How do you handle touch events in JavaScript?**

```typescript
element.addEventListener("touchstart", (e) => {
  const touch = e.touches[0];
  // Handle touch
});
element.addEventListener("touchmove", (e) => {});
element.addEventListener("touchend", (e) => {});
```

6. **What are different service worker caching strategies?**
   - Cache First: Static assets
   - Network First: Dynamic content
   - Stale While Revalidate: Frequent updates
   - Cache Only: Offline-first
   - Network Only: Always fresh

7. ** How do you optimize performance for mobile devices?**
   - Reduce bundle size
   - Lazy load images/components
   - Minimize reflows
   - Use hardware acceleration
   - Code splitting

8. **What is the difference between Electron and PWA?**
   - Electron: Desktop apps, full system access
   - PWA: Web apps, limited APIs
   - Electron: Larger bundle, native menus
   - PWA: Lightweight, browser-based

9. **How do you access device features like camera and location?**

```typescript
// Camera
const stream = await navigator.mediaDevices.getUserMedia({ video: true });

// Location
navigator.geolocation.getCurrentPosition((position) => {
  console.log(position.coords);
});
```

10. **What are the requirements for a PWA?**
    - HTTPS
    - Service worker
    - Web app manifest
    - Icons (multiple sizes)
    - Valid start_url

### Scenario-Based Questions

11. **A user reports your PWA isn't installable on iOS. What could be wrong?**
    - Check manifest.json format
    - Ensure proper icons
    - iOS requires apple-touch-icon
    - Check HTTPS
    - Safari has limited PWA support

12. **How would you implement pull-to-refresh?**
    - Detect touch at scroll top
    - Track touch distance
    - Apply resistance curve
    - Trigger refresh at threshold
    - Reset UI after complete

13. **Your desktop app needs auto-updates. How do you implement this?**
    - Use electron-updater
    - Check for updates on launch
    - Download in background
    - Prompt user to restart
    - Handle update errors

14. **How do you handle the iOS 100vh issue?**

```typescript
// Set CSS variable
const setHeight = () => {
  document.documentElement.style.setProperty(
    "--vh",
    `${window.innerHeight * 0.01}px`,
  );
};

// CSS: height: calc(var(--vh, 1vh) * 100);
```

---

## Summary

### Key Takeaways

1. **PWAs**: Combine web and native app capabilities
2. **Mobile-First**: Start with mobile, enhance for desktop
3. **Touch Events**: Handle gestures, provide feedback
4. **Offline Support**: Cache strategically, queue requests
5. **Performance**: Optimize for mobile networks
6. **Platform-Specific**: Handle iOS/Android quirks
7. **Device APIs**: Access system features with permissions
8. **Distribution**: Multiple channels (web, stores, desktop)

### Development Checklist

- [ ] Implement service worker
- [ ] Create web app manifest
- [ ] Optimize for mobile performance
- [ ] Add touch gesture support
- [ ] Test offline functionality
- [ ] Handle platform-specific quirks
- [ ] Implement installation prompt
- [ ] Add push notifications
- [ ] Test on real devices
- [ ] Follow platform guidelines
- [ ] Ensure accessibility
- [ ] Optimize bundle size

### Resources

- [PWA Builder](https://www.pwabuilder.com)
- [Workbox](https://developers.google.com/web/tools/workbox) - Service Worker Library
- [Electron](https://www.electronjs.org) - Desktop Apps
- [Capacitor](https://capacitorjs.com) - Hybrid Apps
- [Web.dev](https://web.dev/progressive-web-apps/) - PWA Guide

---

**Next Steps**: Build a full PWA, implement offline functionality, experiment with device APIs, and test on multiple platforms.
