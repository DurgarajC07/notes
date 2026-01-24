# Browser Process Model

> **Multi-Process Architecture and Site Isolation**

---

## 🎯 Overview

Modern browsers use a **multi-process architecture** where different components run in isolated processes for security, stability, and performance. Understanding this model is crucial for optimizing web applications and debugging complex issues.

**Key Benefits:**

- **Security**: Each site runs in isolated process (site isolation)
- **Stability**: One tab crash doesn't affect others
- **Performance**: Parallel processing across CPU cores
- **Responsiveness**: UI remains responsive even if renderer hangs

---

## 📚 Core Concepts

### **1. Chrome's Multi-Process Architecture**

Chrome pioneered the multi-process model, now adopted by most modern browsers:

```
┌─────────────────────────────────────────────────────┐
│                  Browser Process                    │
│  - UI (tabs, address bar, bookmarks)               │
│  - Network stack                                    │
│  - Storage (cookies, localStorage)                 │
│  - Main browser logic                              │
└──────────┬──────────────────────────────────────────┘
           │
    ┌──────┴───────┬──────────────┬──────────────┐
    │              │              │              │
┌───▼────┐  ┌──────▼─────┐  ┌────▼──────┐  ┌───▼────┐
│Renderer│  │  Renderer  │  │    GPU    │  │ Plugin │
│Process │  │  Process   │  │  Process  │  │Process │
│        │  │            │  │           │  │        │
│ Tab 1  │  │   Tab 2    │  │ Graphics  │  │  PDF   │
│ Site A │  │   Site B   │  │ Composit  │  │ Flash  │
└────────┘  └────────────┘  └───────────┘  └────────┘
```

**Process Types:**

```javascript
// Conceptual representation
class BrowserArchitecture {
  constructor() {
    this.browserProcess = new BrowserProcess();
    this.rendererProcesses = [];
    this.gpuProcess = new GPUProcess();
    this.utilityProcesses = [];
  }

  class BrowserProcess {
    responsibilities = [
      'UI rendering (tabs, address bar)',
      'Network requests',
      'File system access',
      'Cookie/storage management',
      'Process management',
      'Extensions (background pages)'
    ];

    // Browser process is privileged
    hasFileAccess = true;
    hasNetworkAccess = true;
    hasSystemAccess = true;
  }

  class RendererProcess {
    responsibilities = [
      'HTML/CSS parsing',
      'JavaScript execution',
      'DOM manipulation',
      'Layout calculation',
      'Paint',
      'User event handling'
    ];

    // Renderer is sandboxed
    hasFileAccess = false;      // No direct file access
    hasNetworkAccess = false;   // Only via browser process
    hasSystemAccess = false;    // Heavily restricted
  }

  class GPUProcess {
    responsibilities = [
      'Hardware acceleration',
      'WebGL rendering',
      'Canvas 2D acceleration',
      'Video decoding',
      'Compositing'
    ];
  }
}
```

### **2. Site Isolation**

Each site runs in its own isolated process:

```javascript
class SiteIsolation {
  // Determine which process handles which site
  getProcessForSite(url) {
    const site = this.extractSite(url);

    // Check if process exists for this site
    let process = this.siteToProcess.get(site);

    if (!process) {
      // Create new renderer process for this site
      process = this.createRendererProcess(site);
      this.siteToProcess.set(site, process);
    }

    return process;
  }

  extractSite(url) {
    // Site = scheme + eTLD+1
    // https://example.com:443/path → https://example.com
    // https://foo.example.com → https://example.com
    const parsed = new URL(url);
    return `${parsed.protocol}//${this.getETLDPlusOne(parsed.hostname)}`;
  }

  getETLDPlusOne(hostname) {
    // Extract effective TLD + 1
    // foo.bar.example.com → example.com
    // subdomain.github.io → subdomain.github.io
    const parts = hostname.split(".");

    // For .github.io, .co.uk, etc., need 3 parts
    // For .com, .org, etc., need 2 parts
    if (
      this.isPublicSuffix(
        parts[parts.length - 2] + "." + parts[parts.length - 1],
      )
    ) {
      return parts.slice(-3).join(".");
    }
    return parts.slice(-2).join(".");
  }
}

// Example: Site isolation in action
const isolation = new SiteIsolation();

// These run in SAME process (same site)
isolation.getProcessForSite("https://example.com/page1");
isolation.getProcessForSite("https://sub.example.com/page2");
isolation.getProcessForSite("https://example.com:8080/page3");

// This runs in DIFFERENT process (different site)
isolation.getProcessForSite("https://another-site.com/page");
```

**Security Benefits:**

```javascript
// Without site isolation (old Chrome)
class UnsafeOldChrome {
  // ❌ All tabs share one process
  // If attacker exploits Spectre/Meltdown:
  attackerTab() {
    // Can read memory from other tabs in same process!
    const victimPassword = this.readMemory(otherTab.passwordField);
    this.sendToAttacker(victimPassword);
  }
}

// With site isolation (modern Chrome)
class SafeModernChrome {
  // ✅ Each site in separate process
  attackerTab() {
    // Cannot read memory from other processes
    // OS-level process isolation
    const victimPassword = this.readMemory(otherTab.passwordField);
    // ❌ Throws: Cannot access memory from different process
  }
}
```

### **3. Process Lifecycle**

How browser creates and manages processes:

```javascript
class ProcessLifecycle {
  constructor() {
    this.processes = new Map();
    this.processLimit = 10; // Max renderer processes
  }

  // Create new renderer process
  createRenderer(site) {
    // Check limit
    if (this.processes.size >= this.processLimit) {
      // Reuse existing process
      return this.findReusableProcess(site);
    }

    // Spawn new process
    const process = {
      id: this.generateId(),
      site: site,
      state: "initializing",
      createdAt: Date.now(),
      memoryUsage: 0,
      isResponsive: true,
    };

    this.processes.set(process.id, process);
    this.initializeProcess(process);

    return process;
  }

  // Initialize process
  async initializeProcess(process) {
    // 1. Start process
    await this.startProcess(process);

    // 2. Setup IPC
    await this.setupIPC(process);

    // 3. Apply sandbox
    await this.applySandbox(process);

    process.state = "ready";
  }

  // Kill process
  killProcess(processId) {
    const process = this.processes.get(processId);

    if (process) {
      // 1. Save state (if needed)
      this.saveProcessState(process);

      // 2. Close all connections
      this.closeIPCConnections(process);

      // 3. Terminate process
      this.terminateProcess(process);

      // 4. Cleanup
      this.processes.delete(processId);
    }
  }

  // Detect unresponsive process
  monitorResponsiveness() {
    setInterval(() => {
      this.processes.forEach((process) => {
        const ping = this.sendPing(process);

        setTimeout(() => {
          if (!ping.received) {
            process.isResponsive = false;
            this.showUnresponsiveDialog(process);
          }
        }, 5000); // 5 second timeout
      });
    }, 10000); // Check every 10 seconds
  }

  showUnresponsiveDialog(process) {
    // Show "Page Unresponsive" dialog
    // Options: Wait | Kill
    const dialog = {
      message: `The page ${process.site} is unresponsive`,
      options: ["Wait", "Kill"],
      onKill: () => this.killProcess(process.id),
    };

    this.showDialog(dialog);
  }
}
```

### **4. Inter-Process Communication (IPC)**

Processes communicate via IPC:

```javascript
class IPCChannel {
  constructor(browserProcess, rendererProcess) {
    this.browser = browserProcess;
    this.renderer = rendererProcess;
    this.messageQueue = [];
  }

  // Renderer → Browser
  rendererToB browser(message) {
    // Example: Network request
    if (message.type === 'network-request') {
      // Renderer asks browser to make request
      // (Renderer has no direct network access)
      this.browser.handleNetworkRequest(message.data);
    }

    // Example: File access
    if (message.type === 'file-read') {
      // Renderer asks browser to read file
      this.browser.handleFileRead(message.data);
    }

    // Example: Cookie access
    if (message.type === 'get-cookies') {
      this.browser.handleGetCookies(message.data);
    }
  }

  // Browser → Renderer
  browserToRenderer(message) {
    // Example: Network response
    if (message.type === 'network-response') {
      this.renderer.handleNetworkResponse(message.data);
    }

    // Example: Navigation
    if (message.type === 'navigate') {
      this.renderer.handleNavigation(message.data);
    }

    // Example: Update
    if (message.type === 'update') {
      this.renderer.handleUpdate(message.data);
    }
  }

  // Renderer → Renderer (via Browser)
  crossOriginMessage(sourceRenderer, targetRenderer, message) {
    // postMessage between different origins
    // Must go through browser process for security

    // 1. Source renderer sends to browser
    this.browserToRenderer({
      type: 'cross-origin-message',
      source: sourceRenderer.id,
      target: targetRenderer.id,
      data: message
    });

    // 2. Browser forwards to target renderer
    this.rendererToB browser({
      type: 'deliver-message',
      target: targetRenderer.id,
      data: message
    });
  }
}

// Example: Network request IPC
class NetworkRequestIPC {
  // In renderer process
  async fetchFromRenderer(url) {
    // Renderer cannot access network directly
    // Must ask browser process
    const requestId = this.generateId();

    return new Promise((resolve, reject) => {
      // Send IPC message to browser
      ipcChannel.send('network-request', {
        id: requestId,
        url: url,
        method: 'GET'
      });

      // Wait for response from browser
      ipcChannel.on('network-response', (data) => {
        if (data.id === requestId) {
          resolve(data.response);
        }
      });
    });
  }

  // In browser process
  handleNetworkRequest(request) {
    // Browser makes actual network request
    fetch(request.url)
      .then(response => response.text())
      .then(data => {
        // Send response back to renderer
        ipcChannel.send('network-response', {
          id: request.id,
          response: data
        });
      });
  }
}
```

### **5. Sandboxing**

Renderer processes run in a restricted sandbox:

```javascript
class Sandbox {
  // Renderer process restrictions
  getRestrictions() {
    return {
      fileSystem: {
        read: false, // Cannot read files
        write: false, // Cannot write files
        execute: false, // Cannot execute binaries
      },

      network: {
        direct: false, // No direct sockets
        httpOnly: true, // Only via browser process
      },

      systemCalls: {
        createProcess: false,
        killProcess: false,
        accessMemory: false,
        accessRegistry: false,
      },

      devices: {
        camera: "permission-required",
        microphone: "permission-required",
        location: "permission-required",
        usb: false,
        bluetooth: "permission-required",
      },
    };
  }

  // Permission requests
  async requestPermission(renderer, permission) {
    // Renderer must ask for sensitive permissions
    const dialog = this.showPermissionDialog({
      site: renderer.site,
      permission: permission,
      message: `${renderer.site} wants to access your ${permission}`,
    });

    const granted = await dialog.waitForUserChoice();

    if (granted) {
      this.grantPermission(renderer, permission);
    }

    return granted;
  }

  // Escape sandbox (security vulnerability)
  detectSandboxEscape() {
    // Monitor for suspicious activity
    setInterval(() => {
      this.processes.forEach((renderer) => {
        // Check for unauthorized system calls
        if (renderer.attemptedSystemCalls.length > 0) {
          this.handleSecurityViolation(renderer);
        }

        // Check for unusual memory access
        if (renderer.memoryAccessViolations > 0) {
          this.handleSecurityViolation(renderer);
        }
      });
    }, 1000);
  }

  handleSecurityViolation(renderer) {
    // Kill process immediately
    this.killProcess(renderer.id);

    // Log security event
    this.logSecurityEvent({
      type: "sandbox-escape-attempt",
      process: renderer.id,
      site: renderer.site,
      timestamp: Date.now(),
    });

    // Show warning to user
    this.showSecurityWarning(renderer.site);
  }
}
```

### **6. Process-per-Site vs Process-per-Site-Instance**

Different strategies for process allocation:

```javascript
class ProcessAllocationStrategy {
  // Strategy 1: Process-per-Site
  processPerSite() {
    // All pages from same site share one process

    const example1 = "https://example.com/page1";
    const example2 = "https://example.com/page2";
    // → Both use SAME process

    // Memory efficient but less secure
  }

  // Strategy 2: Process-per-Site-Instance
  processPerSiteInstance() {
    // Each tab gets its own process

    const tab1 = openTab("https://example.com/page1");
    const tab2 = openTab("https://example.com/page2");
    // → Each uses DIFFERENT process

    // More secure but uses more memory
  }

  // Strategy 3: Strict Site Isolation (Chrome default)
  strictSiteIsolation() {
    // Each site in separate process
    // BUT: same-site pages can share if same-origin

    const page1 = "https://example.com/page1";
    const page2 = "https://example.com/page2";
    // → SAME process (same origin)

    const page3 = "https://sub.example.com/page";
    // → SAME process (same site, different origin)

    const page4 = "https://other-site.com/page";
    // → DIFFERENT process (different site)
  }

  // Strategy 4: Out-of-Process iframes (OOPIF)
  outOfProcessIframes() {
    // Each cross-site iframe in separate process

    const html = `
      <html>
        <body>
          <!-- Main page: example.com process -->
          <h1>Main Page (example.com)</h1>
          
          <!-- Cross-site iframe: different process -->
          <iframe src="https://other-site.com/widget"></iframe>
          
          <!-- Same-site iframe: same process -->
          <iframe src="https://example.com/sidebar"></iframe>
        </body>
      </html>
    `;

    // Main page: process 1
    // other-site.com iframe: process 2
    // example.com iframe: process 1
  }
}
```

### **7. Process Memory Management**

Managing memory across processes:

```javascript
class ProcessMemoryManager {
  constructor() {
    this.processes = new Map();
    this.memoryLimit = 2 * 1024 * 1024 * 1024; // 2GB per process
  }

  monitorMemory() {
    setInterval(() => {
      this.processes.forEach((process, id) => {
        const memory = this.getProcessMemory(process);

        process.memoryUsage = memory;

        // Check if over limit
        if (memory > this.memoryLimit) {
          this.handleMemoryExceeded(process);
        }

        // Check for memory leaks
        if (this.detectMemoryLeak(process)) {
          this.handleMemoryLeak(process);
        }
      });
    }, 5000);
  }

  getProcessMemory(process) {
    // Get memory usage from OS
    return {
      heapUsed: process.heapUsed,
      heapTotal: process.heapTotal,
      external: process.external,
      rss: process.rss, // Resident Set Size
      total: process.heapUsed + process.external,
    };
  }

  handleMemoryExceeded(process) {
    console.warn(`Process ${process.id} exceeded memory limit`);

    // Try to free memory
    this.requestGarbageCollection(process);

    // If still over limit, kill process
    setTimeout(() => {
      if (process.memoryUsage > this.memoryLimit) {
        this.killProcess(process.id);
        this.showOutOfMemoryError(process.site);
      }
    }, 1000);
  }

  detectMemoryLeak(process) {
    // Check for continuous growth
    const history = process.memoryHistory || [];
    history.push(process.memoryUsage.total);

    if (history.length > 10) {
      history.shift(); // Keep last 10 samples
    }

    process.memoryHistory = history;

    // Detect leak: continuous growth
    if (history.length >= 10) {
      const isGrowing = history.every(
        (val, i) => i === 0 || val > history[i - 1],
      );

      const growthRate = (history[9] - history[0]) / history[0];

      return isGrowing && growthRate > 0.5; // 50% growth
    }

    return false;
  }

  handleMemoryLeak(process) {
    console.warn(`Memory leak detected in process ${process.id}`);

    // Log for debugging
    this.logMemoryLeak({
      process: process.id,
      site: process.site,
      memoryHistory: process.memoryHistory,
    });

    // Show warning to user
    this.showMemoryLeakWarning(process.site);
  }
}
```

### **8. Crash Handling**

How browser handles process crashes:

```javascript
class CrashHandler {
  handleProcessCrash(process) {
    console.error(`Process ${process.id} crashed`);

    // 1. Log crash
    this.logCrash({
      processId: process.id,
      site: process.site,
      timestamp: Date.now(),
      reason: process.crashReason,
      stackTrace: process.stackTrace,
    });

    // 2. Show sad tab
    this.showSadTab(process);

    // 3. Cleanup
    this.cleanupCrashedProcess(process);

    // 4. Offer reload
    this.offerReload(process);
  }

  showSadTab(process) {
    // Display "Aw, Snap!" page
    return {
      title: "Aw, Snap!",
      message: `Something went wrong while displaying ${process.site}`,
      options: [
        {
          label: "Reload",
          action: () => this.reloadProcess(process),
        },
        {
          label: "Send Report",
          action: () => this.sendCrashReport(process),
        },
      ],
    };
  }

  reloadProcess(process) {
    // Create new process for same site
    const newProcess = this.createRenderer(process.site);

    // Restore state
    this.restoreProcessState(newProcess, process.savedState);

    return newProcess;
  }

  sendCrashReport(process) {
    // Send crash report to server
    const report = {
      processId: process.id,
      site: process.site,
      reason: process.crashReason,
      stackTrace: process.stackTrace,
      memoryUsage: process.memoryUsage,
      userAgent: navigator.userAgent,
      timestamp: Date.now(),
    };

    fetch("https://crash-reports.browser.com/submit", {
      method: "POST",
      body: JSON.stringify(report),
    });
  }

  // Protect browser from renderer crashes
  isolateFailures() {
    // ✅ One tab crashes → other tabs unaffected
    tab1.crash();
    // tab2 still works
    // tab3 still works

    // ✅ Renderer crash → browser process unaffected
    rendererProcess.crash();
    // Browser UI still works
    // Other tabs still work

    // ❌ Browser process crash → everything crashes
    browserProcess.crash();
    // All tabs crash
    // Entire browser crashes
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Detecting Process Model from JavaScript**

```javascript
class ProcessModelDetector {
  // Detect if site isolation is enabled
  async detectSiteIsolation() {
    try {
      // Try to access cross-origin iframe
      const iframe = document.createElement("iframe");
      iframe.src = "https://example.com";
      document.body.appendChild(iframe);

      await new Promise((resolve) => {
        iframe.onload = resolve;
      });

      // Try to access iframe's document
      try {
        const doc = iframe.contentDocument;
        console.log(
          "Site isolation: DISABLED (can access cross-origin iframe)",
        );
        return false;
      } catch (e) {
        console.log(
          "Site isolation: ENABLED (cannot access cross-origin iframe)",
        );
        return true;
      }
    } catch (e) {
      console.log("Site isolation: LIKELY ENABLED");
      return true;
    }
  }

  // Check if running in isolated process
  isIsolatedProcess() {
    // Check for COOP/COEP headers
    const hasCOOP = this.hasHeader("Cross-Origin-Opener-Policy");
    const hasCOEP = this.hasHeader("Cross-Origin-Embedder-Policy");

    return hasCOOP && hasCOEP;
  }

  hasHeader(headerName) {
    // Check if response had specific header
    // (Only available via Service Worker or server logs)
    return navigator.serviceWorker?.controller
      ? this.checkViaServiceWorker(headerName)
      : false;
  }

  // Measure process overhead
  async measureProcessOverhead() {
    const results = {
      memoryUsage: 0,
      processCount: 0,
    };

    // Open multiple tabs to same site
    const windows = [];
    for (let i = 0; i < 5; i++) {
      windows.push(window.open(location.href));
    }

    // Measure memory
    if (performance.memory) {
      results.memoryUsage = performance.memory.usedJSHeapSize;
    }

    // Clean up
    windows.forEach((w) => w.close());

    return results;
  }
}

// Usage
const detector = new ProcessModelDetector();
detector.detectSiteIsolation().then((enabled) => {
  console.log("Site isolation:", enabled ? "ON" : "OFF");
});
```

### **Example 2: Cross-Origin Communication**

```javascript
class CrossOriginComm {
  // Setup communication between different processes
  setupCrossOriginMessaging() {
    // Main page (process 1)
    const iframe = document.createElement("iframe");
    iframe.src = "https://other-domain.com/widget.html";
    document.body.appendChild(iframe);

    // Send message to iframe (different process)
    iframe.onload = () => {
      // Goes through browser process IPC
      iframe.contentWindow.postMessage(
        {
          type: "init",
          data: { userId: 123 },
        },
        "https://other-domain.com",
      );
    };

    // Receive messages from iframe
    window.addEventListener("message", (event) => {
      // Verify origin (security!)
      if (event.origin !== "https://other-domain.com") {
        return;
      }

      console.log("Message from iframe:", event.data);
    });
  }

  // SharedArrayBuffer (requires cross-origin isolation)
  async setupSharedMemory() {
    // Requires COOP and COEP headers for security
    if (crossOriginIsolated) {
      const sharedBuffer = new SharedArrayBuffer(1024);
      const sharedArray = new Int32Array(sharedBuffer);

      // Share with worker (same process group)
      const worker = new Worker("worker.js");
      worker.postMessage({ buffer: sharedBuffer });

      console.log("SharedArrayBuffer enabled");
    } else {
      console.log(
        "SharedArrayBuffer disabled (requires cross-origin isolation)",
      );
    }
  }

  // Check cross-origin isolation
  checkCrossOriginIsolation() {
    return {
      isolated: crossOriginIsolated,
      requirements: {
        "Cross-Origin-Opener-Policy": "same-origin",
        "Cross-Origin-Embedder-Policy": "require-corp",
      },
      currentHeaders: {
        coop: this.getResponseHeader("Cross-Origin-Opener-Policy"),
        coep: this.getResponseHeader("Cross-Origin-Embedder-Policy"),
      },
    };
  }
}
```

### **Example 3: Memory-Efficient Tab Management**

```javascript
class TabMemoryManager {
  constructor() {
    this.tabs = new Map();
    this.memoryLimit = 500 * 1024 * 1024; // 500MB per tab
  }

  // Discard inactive tabs to save memory
  async discardInactiveTabs() {
    const inactiveTabs = Array.from(this.tabs.values())
      .filter(tab => this.isInactive(tab))
      .sort((a, b) => a.lastActive - b.lastActive);

    for (const tab of inactiveTabs) {
      if (this.shouldDiscard(tab)) {
        await this.discardTab(tab);
      }
    }
  }

  isInactive(tab) {
    const inactiveTime = Date.now() - tab.lastActive;
    return inactiveTime > 10 * 60 * 1000; // 10 minutes
  }

  shouldDiscard(tab) {
    return (
      !tab.isPlaying Audio &&          // Not playing media
      !tab.hasFormInput &&            // No unsaved form data
      !tab.isCapturingMedia &&        // Not using camera/mic
      tab.memoryUsage > 100 * 1024 * 1024 // Using > 100MB
    );
  }

  async discardTab(tab) {
    // Save tab state
    tab.savedState = {
      url: tab.url,
      scrollPosition: tab.scrollY,
      formData: this.extractFormData(tab)
    };

    // Kill renderer process
    this.killProcess(tab.processId);

    tab.discarded = true;
    tab.processId = null;

    console.log(`Discarded tab: ${tab.url} (freed ${tab.memoryUsage / 1024 / 1024}MB)`);
  }

  async restoreTab(tab) {
    if (!tab.discarded) return;

    // Create new process
    const process = this.createRenderer(tab.site);

    // Load page
    await process.navigate(tab.savedState.url);

    // Restore state
    await process.setScrollPosition(tab.savedState.scrollPosition);
    await process.restoreFormData(tab.savedState.formData);

    tab.discarded = false;
    tab.processId = process.id;

    console.log(`Restored tab: ${tab.url}`);
  }

  // Automatic discarding based on memory pressure
  enableAutomaticDiscarding() {
    setInterval(() => {
      const totalMemory = this.getTotalMemoryUsage();
      const threshold = 0.8; // 80% of system memory

      if (totalMemory > threshold * this.getSystemMemory()) {
        console.warn('Memory pressure detected, discarding tabs...');
        this.discardInactiveTabs();
      }
    }, 30000); // Check every 30 seconds
  }
}
```

---

## 🏗️ Best Practices

### **1. Enable Cross-Origin Isolation for SharedArrayBuffer**

```html
<!-- Requires these headers -->
Cross-Origin-Opener-Policy: same-origin Cross-Origin-Embedder-Policy:
require-corp

<!-- Then in JavaScript -->
<script>
  if (crossOriginIsolated) {
    // Can use SharedArrayBuffer, high-precision timers
    const buffer = new SharedArrayBuffer(1024);
  } else {
    console.warn("Cross-origin isolation not enabled");
  }
</script>
```

### **2. Use postMessage for Cross-Origin Communication**

```javascript
// ✅ Good: Safe cross-origin communication
iframe.contentWindow.postMessage(data, "https://trusted-origin.com");

window.addEventListener("message", (e) => {
  // Always verify origin!
  if (e.origin !== "https://trusted-origin.com") return;

  handleMessage(e.data);
});

// ❌ Bad: Don't use wildcards
iframe.contentWindow.postMessage(data, "*"); // Insecure!
```

### **3. Monitor Memory Usage**

```javascript
if (performance.memory) {
  console.log({
    usedJSHeapSize: performance.memory.usedJSHeapSize / 1024 / 1024 + " MB",
    totalJSHeapSize: performance.memory.totalJSHeapSize / 1024 / 1024 + " MB",
    jsHeapSizeLimit: performance.memory.jsHeapSizeLimit / 1024 / 1024 + " MB",
  });
}
```

### **4. Handle Process Crashes Gracefully**

```javascript
// Detect if running in crashed state
if (performance.navigation.type === performance.navigation.TYPE_RELOAD) {
  // Possible crash reload
  console.log("Page was reloaded (possible crash)");

  // Restore state from localStorage
  restoreApplicationState();
}

// Save state periodically
setInterval(() => {
  localStorage.setItem("appState", JSON.stringify(getCurrentState()));
}, 5000);
```

### **5. Optimize for Multi-Process Architecture**

```javascript
// ✅ Good: Minimize cross-process communication
// Keep related components in same origin

// ❌ Bad: Excessive cross-origin iframes
// Each iframe = new process = more memory

// ✅ Good: Use Service Worker for caching
// Shared across all pages of same origin
navigator.serviceWorker.register("/sw.js");

// ✅ Good: Use BroadcastChannel for same-origin tabs
const channel = new BroadcastChannel("my-channel");
channel.postMessage("Hello other tabs!"); // No IPC overhead
```

---

## 🧪 Interview Questions

### **Q1: Explain Chrome's multi-process architecture and its benefits.**

**Answer:**

Chrome uses a **multi-process architecture** where different components run in separate OS processes:

**Process Types:**

1. **Browser Process**: Main process, handles UI, network, storage
2. **Renderer Processes**: One per site/tab, runs web content
3. **GPU Process**: Hardware acceleration, compositing
4. **Plugin Processes**: Flash, PDF viewer (legacy)
5. **Utility Processes**: Audio, network service, etc.

**Benefits:**

1. **Security (Site Isolation)**:
   - Each site in separate process
   - One site cannot access memory of another site
   - Protects against Spectre/Meltdown attacks

2. **Stability**:
   - One tab crash doesn't affect others
   - Renderer crash → show "Aw, Snap!" page
   - Browser process stays alive

3. **Performance**:
   - Parallel processing across CPU cores
   - GPU acceleration in separate process
   - Better resource management

4. **Responsiveness**:
   - UI thread separate from render threads
   - Browser stays responsive even if page hangs

**Example:**

```javascript
// Tab 1: Runs in process A
// https://example.com

// Tab 2: Runs in process B
// https://another-site.com

// If Tab 1 crashes:
tab1.crash();
// → Only Tab 1 shows "Aw, Snap!"
// → Tab 2 continues working
// → Browser UI continues working
```

### **Q2: What is site isolation and why is it important for security?**

**Answer:**

**Site Isolation** ensures each site runs in a separate renderer process, providing strong security boundaries.

**How it works:**

```
https://example.com      → Process 1
https://sub.example.com  → Process 1 (same site)
https://other-site.com   → Process 2 (different site)

<iframe src="https://ads.com"> → Process 3 (cross-site iframe)
```

**Security benefits:**

1. **Prevents Cross-Site Data Theft**:

```javascript
// Without site isolation:
// Attacker could exploit Spectre to read memory
const victimData = specter.readMemory(sameProcess); // ❌ Possible

// With site isolation:
const victimData = specter.readMemory(differentProcess); // ✅ Blocked by OS
```

2. **Protects Against Compromised Renderers**:

```javascript
// If attacker exploits renderer vulnerability:
// Without isolation: Can access all tabs
// With isolation: Can only access own site's data
```

3. **Enforces Same-Origin Policy**:

```javascript
// Cross-site iframes cannot access parent
iframe.contentWindow.document; // ❌ SecurityError

// Must use postMessage
iframe.contentWindow.postMessage(data, origin); // ✅ Safe
```

**Trade-offs:**

- **Pro**: Strong security boundaries
- **Con**: More memory usage (~10% overhead)
- **Con**: Slightly slower cross-site navigation

**Check if enabled:**

```javascript
// Try accessing cross-origin iframe
try {
  iframe.contentDocument; // Throws if isolated
  console.log("Site isolation: OFF");
} catch (e) {
  console.log("Site isolation: ON");
}
```

### **Q3: How does the browser handle renderer process crashes?**

**Answer:**

When a renderer process crashes, the browser:

**1. Detects Crash:**

```javascript
// Browser process monitors renderer health
rendererProcess.on("crash", (info) => {
  console.error("Renderer crashed:", info);
});
```

**2. Shows "Aw, Snap!" Page:**

```
┌─────────────────────────────────────┐
│  Aw, Snap!                         │
│  Something went wrong while        │
│  displaying this webpage.          │
│                                    │
│  [Reload]  [Send feedback]        │
└─────────────────────────────────────┘
```

**3. Isolates Failure:**

```javascript
// Other tabs unaffected
tab1.crash(); // Shows "Aw, Snap!"
// tab2 still works
// tab3 still works
// Browser UI still works
```

**4. Cleanup:**

```javascript
class CrashHandler {
  handleCrash(renderer) {
    // 1. Log crash data
    this.logCrash(renderer);

    // 2. Free resources
    this.freeMemory(renderer);
    this.closeConnections(renderer);

    // 3. Remove from process list
    this.processes.delete(renderer.id);

    // 4. Offer reload
    this.showReloadOption(renderer);
  }
}
```

**5. Optional Reload:**

```javascript
// User clicks "Reload"
function reloadTab(tab) {
  // Create new renderer process
  const newRenderer = createRenderer(tab.site);

  // Navigate to same URL
  newRenderer.navigate(tab.url);

  // Restore scroll position (if available)
  if (tab.savedState) {
    newRenderer.restoreState(tab.savedState);
  }
}
```

**Automatic Crash Recovery:**

```javascript
class AutoRecovery {
  constructor() {
    this.crashCount = new Map();
    this.maxRetries = 3;
  }

  handleCrash(tab) {
    const crashes = this.crashCount.get(tab.id) || 0;

    if (crashes < this.maxRetries) {
      // Auto-reload
      setTimeout(() => {
        this.reloadTab(tab);
        this.crashCount.set(tab.id, crashes + 1);
      }, 1000);
    } else {
      // Too many crashes, give up
      this.showPermanentError(tab);
    }
  }
}
```

### **Q4: What are the memory implications of the multi-process model?**

**Answer:**

Multi-process architecture uses more memory than single-process:

**Memory Overhead:**

```javascript
// Single process (old browsers):
1 tab = 50MB

// Multi-process (Chrome):
1 tab = 50MB (renderer) + 10-20MB (process overhead)

// Example with 10 tabs:
Single process: 500MB
Multi-process: 700MB (+40% overhead)
```

**Where memory goes:**

1. **Process Base Cost** (~10-20MB each):

```javascript
{
  code: '5MB',           // Binary code
  stack: '2MB',          // Call stack
  heap: '5MB',           // V8 heap
  sharedLibraries: '8MB' // Loaded libraries
}
```

2. **Per-Site Overhead**:

```javascript
// Duplicate resources across processes:
- JavaScript engine instance
- Style engine
- Layout engine
- Paint/composite infrastructure
```

3. **IPC Buffers**:

```javascript
// Communication between processes
{
  messages: '1-5MB per process pair',
  sharedMemory: '5-10MB for compositor'
}
```

**Memory Optimizations:**

```javascript
class MemoryOptimizer {
  // 1. Process sharing
  shareSameSiteProcesses() {
    // Reuse process for same site
    window.open("https://example.com/page1"); // Process A
    window.open("https://example.com/page2"); // Process A (reused!)
  }

  // 2. Discard inactive tabs
  async discardTab(tab) {
    // Kill process, save state
    tab.processId = null;
    tab.savedState = this.saveState(tab);
    // Saves ~70MB per tab
  }

  // 3. Limit process count
  enforceProcessLimit() {
    const MAX_PROCESSES = 10;

    if (this.processes.size > MAX_PROCESSES) {
      // Reuse least recently used process
      const lru = this.findLRU();
      return lru;
    }
  }
}
```

**Trade-off Analysis:**

```
Single Process:
  ✅ Less memory
  ❌ No security isolation
  ❌ One crash kills all
  ❌ No parallelism

Multi-Process:
  ✅ Security isolation
  ✅ Crash isolation
  ✅ Parallel processing
  ❌ More memory
```

**Mobile Considerations:**

```javascript
// Mobile browsers use fewer processes
if (isMobile) {
  maxProcesses = 4; // vs 10+ on desktop
  aggressiveDiscarding = true;
  shareProcesses = true;
}
```

### **Q5: How does Inter-Process Communication (IPC) work in Chrome?**

**Answer:**

IPC allows processes to communicate while maintaining isolation:

**IPC Methods:**

1. **Message Passing** (Most Common):

```javascript
// Renderer → Browser
ipcRenderer.send("channel", { data: "hello" });

// Browser receives
ipcMain.on("channel", (event, data) => {
  console.log(data); // { data: 'hello' }
});

// Browser → Renderer (reply)
event.reply("channel-reply", { response: "world" });
```

2. **Shared Memory** (Fast, but limited):

```javascript
// Only with cross-origin isolation
if (crossOriginIsolated) {
  const sharedBuffer = new SharedArrayBuffer(1024);
  worker.postMessage({ buffer: sharedBuffer });
  // Both can read/write same memory
}
```

3. **Unix Domain Sockets** (Chrome internal):

```javascript
// Low-level IPC mechanism
// Not exposed to JavaScript
// Used by Chrome internally
```

**Common IPC Scenarios:**

```javascript
// 1. Network request
// Renderer asks browser to fetch
rendererIPC.send("fetch", { url: "https://api.com/data" });

// Browser makes request
browserIPC.on("fetch", async (event, { url }) => {
  const response = await fetch(url);
  event.reply("fetch-response", await response.text());
});

// 2. File access
// Renderer asks browser to read file
rendererIPC.send("read-file", { path: "/path/to/file" });

// Browser reads file (renderer can't access filesystem)
browserIPC.on("read-file", (event, { path }) => {
  const contents = fs.readFileSync(path);
  event.reply("file-contents", contents);
});

// 3. Cookie access
// Renderer asks for cookies
rendererIPC.send("get-cookies", { domain: "example.com" });

// Browser returns cookies
browserIPC.on("get-cookies", (event, { domain }) => {
  const cookies = cookieStore.get(domain);
  event.reply("cookies", cookies);
});
```

**Performance Considerations:**

```javascript
// IPC has overhead (~0.1-1ms per message)

// ❌ Bad: Excessive IPC
for (let i = 0; i < 1000; i++) {
  ipc.send("message", { index: i }); // 1000 IPCs!
}

// ✅ Good: Batch messages
ipc.send("batch-messages", {
  items: Array.from({ length: 1000 }, (_, i) => ({ index: i })),
}); // 1 IPC!
```

**Security:**

```javascript
// Browser validates all IPC messages
browserIPC.on("read-file", (event, { path }) => {
  // ✅ Validate path
  if (!isPathAllowed(path)) {
    throw new Error("Unauthorized path access");
  }

  // ✅ Check origin
  if (event.senderOrigin !== "https://trusted-site.com") {
    throw new Error("Untrusted origin");
  }

  // Then proceed
  const contents = fs.readFileSync(path);
  event.reply("file-contents", contents);
});
```

---

## 📚 Key Takeaways

1. **Multi-process architecture**: Browser process + multiple renderer processes + GPU process
2. **Site isolation**: Each site in separate process for security (Spectre/Meltdown protection)
3. **Crash isolation**: One tab crash doesn't affect others, browser stays alive
4. **Sandboxing**: Renderer processes heavily restricted (no file/network/system access)
5. **IPC overhead**: Communication between processes has latency (~0.1-1ms per message)
6. **Memory trade-off**: Multi-process uses ~40% more memory but provides security
7. **Out-of-process iframes**: Cross-site iframes run in separate processes
8. **Process limits**: Browsers limit max processes (typically 10-20) to conserve memory
9. **Cross-origin isolation**: Required for SharedArrayBuffer and high-precision timers
10. **Process recovery**: Browsers auto-recover from crashes, discarded tabs save memory

---

**Master the browser process model for security, stability, and performance!**
