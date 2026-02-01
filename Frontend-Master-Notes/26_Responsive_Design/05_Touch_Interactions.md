# Touch Interactions: Building Touch-Friendly Web Applications

## Overview

Touch interactions are fundamental to modern web applications, with mobile devices accounting for over 60% of web traffic. Building touch-friendly interfaces requires understanding touch events, gesture handling, accessible touch targets, and the unified pointer events API. This guide covers best practices for creating responsive, accessible touch experiences.

## Core Concepts

### Touch vs Mouse Paradigm Shift

```typescript
interface InteractionParadigm {
  inputType: "mouse" | "touch" | "stylus" | "trackpad";
  characteristics: {
    precision: "pixel" | "finger-sized";
    hoverState: boolean;
    multiTouch: boolean;
    pressure: boolean;
    events: string[];
  };
}

const mouseInteraction: InteractionParadigm = {
  inputType: "mouse",
  characteristics: {
    precision: "pixel",
    hoverState: true,
    multiTouch: false,
    pressure: false,
    events: [
      "mousedown",
      "mousemove",
      "mouseup",
      "click",
      "mouseenter",
      "mouseleave",
    ],
  },
};

const touchInteraction: InteractionParadigm = {
  inputType: "touch",
  characteristics: {
    precision: "finger-sized",
    hoverState: false,
    multiTouch: true,
    pressure: true,
    events: ["touchstart", "touchmove", "touchend", "touchcancel"],
  },
};

// The unified approach: Pointer Events
const pointerInteraction: InteractionParadigm = {
  inputType: "stylus",
  characteristics: {
    precision: "pixel",
    hoverState: true,
    multiTouch: true,
    pressure: true,
    events: [
      "pointerdown",
      "pointermove",
      "pointerup",
      "pointercancel",
      "pointerenter",
      "pointerleave",
    ],
  },
};
```

## 1. Touch Target Sizes (WCAG Guidelines)

### Minimum Touch Target Requirements

```typescript
/**
 * WCAG 2.1 Success Criterion 2.5.5 (Level AAA): 44×44 CSS pixels
 * WCAG 2.2 Success Criterion 2.5.8 (Level AA): 24×24 CSS pixels
 */
interface TouchTargetStandards {
  wcag21_AAA: { minWidth: 44; minHeight: 44 };
  wcag22_AA: { minWidth: 24; minHeight: 24 };
  ios_hig: { minWidth: 44; minHeight: 44 };
  android_material: { minWidth: 48; minHeight: 48 };
  recommended: { minWidth: 48; minHeight: 48 };
  spacing: { minGap: 8 }; // Between touch targets
}

const TOUCH_TARGET: TouchTargetStandards = {
  wcag21_AAA: { minWidth: 44, minHeight: 44 },
  wcag22_AA: { minWidth: 24, minHeight: 24 },
  ios_hig: { minWidth: 44, minHeight: 44 },
  android_material: { minWidth: 48, minHeight: 48 },
  recommended: { minWidth: 48, minHeight: 48 },
  spacing: { minGap: 8 },
};
```

### CSS for Touch-Friendly Buttons

```css
/* Minimum touch target size */
.button {
  min-width: 44px;
  min-height: 44px;
  padding: 12px 24px;

  /* Ensure visual size doesn't shrink below minimum */
  display: inline-flex;
  align-items: center;
  justify-content: center;

  /* Spacing between touch targets */
  margin: 8px;

  /* Touch feedback */
  touch-action: manipulation; /* Disable double-tap zoom */
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0.1);
  user-select: none;
}

/* Icon-only buttons need extra consideration */
.icon-button {
  min-width: 48px;
  min-height: 48px;
  padding: 12px;
  border-radius: 50%;
}

/* Expand touch target without changing visual size */
.small-visual-large-target {
  position: relative;
  width: 24px;
  height: 24px;
}

.small-visual-large-target::before {
  content: "";
  position: absolute;
  top: -12px;
  left: -12px;
  right: -12px;
  bottom: -12px;
  /* Creates a 48x48 touch target around 24x24 visual element */
}

/* Spacing between adjacent touch targets */
.button-group {
  display: flex;
  gap: 8px; /* Minimum spacing */
  flex-wrap: wrap;
}
```

### TypeScript Touch Target Validator

```typescript
interface TouchTargetMetrics {
  element: HTMLElement;
  width: number;
  height: number;
  isAccessible: boolean;
  recommendation?: string;
}

class TouchTargetValidator {
  private readonly MIN_SIZE = 44; // WCAG 2.1 AAA
  private readonly RECOMMENDED_SIZE = 48;
  private readonly MIN_SPACING = 8;

  /**
   * Check if element meets touch target size requirements
   */
  validateElement(element: HTMLElement): TouchTargetMetrics {
    const rect = element.getBoundingClientRect();
    const width = Math.round(rect.width);
    const height = Math.round(rect.height);

    const isAccessible = width >= this.MIN_SIZE && height >= this.MIN_SIZE;

    let recommendation: string | undefined;
    if (!isAccessible) {
      recommendation = `Increase size to at least ${this.MIN_SIZE}×${this.MIN_SIZE}px`;
    } else if (
      width < this.RECOMMENDED_SIZE ||
      height < this.RECOMMENDED_SIZE
    ) {
      recommendation = `Consider increasing to ${this.RECOMMENDED_SIZE}×${this.RECOMMENDED_SIZE}px for better UX`;
    }

    return {
      element,
      width,
      height,
      isAccessible,
      recommendation,
    };
  }

  /**
   * Validate spacing between touch targets
   */
  validateSpacing(elements: HTMLElement[]): {
    element: HTMLElement;
    nextElement: HTMLElement;
    spacing: number;
    isAccessible: boolean;
  }[] {
    const results = [];

    for (let i = 0; i < elements.length - 1; i++) {
      const current = elements[i];
      const next = elements[i + 1];

      const currentRect = current.getBoundingClientRect();
      const nextRect = next.getBoundingClientRect();

      // Calculate spacing (horizontal or vertical)
      const horizontalSpacing = Math.max(0, nextRect.left - currentRect.right);
      const verticalSpacing = Math.max(0, nextRect.top - currentRect.bottom);

      const spacing = Math.min(horizontalSpacing, verticalSpacing);
      const isAccessible = spacing >= this.MIN_SPACING;

      results.push({
        element: current,
        nextElement: next,
        spacing: Math.round(spacing),
        isAccessible,
      });
    }

    return results;
  }

  /**
   * Generate accessibility report for page
   */
  generateReport(selector: string = 'button, a, input, [role="button"]'): void {
    const elements = Array.from(
      document.querySelectorAll<HTMLElement>(selector),
    );

    console.group("Touch Target Accessibility Report");

    // Size validation
    console.log("=== Touch Target Sizes ===");
    elements.forEach((el, index) => {
      const metrics = this.validateElement(el);
      const status = metrics.isAccessible ? "✅" : "❌";
      console.log(
        `${status} Element ${index + 1}: ${metrics.width}×${metrics.height}px`,
        metrics.recommendation || "OK",
      );
    });

    // Spacing validation
    console.log("\n=== Touch Target Spacing ===");
    const spacingResults = this.validateSpacing(elements);
    spacingResults.forEach((result, index) => {
      const status = result.isAccessible ? "✅" : "❌";
      console.log(
        `${status} Gap ${index + 1}: ${result.spacing}px`,
        result.isAccessible
          ? "OK"
          : `Increase to at least ${this.MIN_SPACING}px`,
      );
    });

    console.groupEnd();
  }

  /**
   * Fix touch target size by adding padding
   */
  fixTouchTarget(element: HTMLElement): void {
    const rect = element.getBoundingClientRect();
    const currentWidth = rect.width;
    const currentHeight = rect.height;

    if (
      currentWidth < this.RECOMMENDED_SIZE ||
      currentHeight < this.RECOMMENDED_SIZE
    ) {
      const widthDiff = Math.max(0, this.RECOMMENDED_SIZE - currentWidth);
      const heightDiff = Math.max(0, this.RECOMMENDED_SIZE - currentHeight);

      const currentPadding = parseInt(getComputedStyle(element).padding || "0");
      const newHorizontalPadding = currentPadding + widthDiff / 2;
      const newVerticalPadding = currentPadding + heightDiff / 2;

      element.style.padding = `${newVerticalPadding}px ${newHorizontalPadding}px`;
    }
  }
}

// Usage
const validator = new TouchTargetValidator();

// Validate all interactive elements
validator.generateReport();

// Fix specific element
const smallButton = document.querySelector<HTMLElement>(".small-button")!;
validator.fixTouchTarget(smallButton);
```

## 2. Touch Events API

### Basic Touch Events

```typescript
interface TouchEventHandlers {
  onTouchStart: (event: TouchEvent) => void;
  onTouchMove: (event: TouchEvent) => void;
  onTouchEnd: (event: TouchEvent) => void;
  onTouchCancel: (event: TouchEvent) => void;
}

class TouchEventManager {
  private element: HTMLElement;
  private touchStartPosition: { x: number; y: number } | null = null;

  constructor(element: HTMLElement) {
    this.element = element;
    this.attachListeners();
  }

  /**
   * Attach touch event listeners
   */
  private attachListeners(): void {
    this.element.addEventListener(
      "touchstart",
      this.handleTouchStart.bind(this),
      {
        passive: false, // Allow preventDefault for gesture control
      },
    );

    this.element.addEventListener(
      "touchmove",
      this.handleTouchMove.bind(this),
      {
        passive: false,
      },
    );

    this.element.addEventListener("touchend", this.handleTouchEnd.bind(this));
    this.element.addEventListener(
      "touchcancel",
      this.handleTouchCancel.bind(this),
    );
  }

  /**
   * Handle touch start
   */
  private handleTouchStart(event: TouchEvent): void {
    const touch = event.touches[0];
    this.touchStartPosition = {
      x: touch.clientX,
      y: touch.clientY,
    };

    console.log("Touch start:", {
      touches: event.touches.length,
      position: this.touchStartPosition,
      target: event.target,
    });

    // Add visual feedback
    this.element.classList.add("touching");
  }

  /**
   * Handle touch move
   */
  private handleTouchMove(event: TouchEvent): void {
    if (!this.touchStartPosition) return;

    const touch = event.touches[0];
    const deltaX = touch.clientX - this.touchStartPosition.x;
    const deltaY = touch.clientY - this.touchStartPosition.y;

    console.log("Touch move:", {
      delta: { x: deltaX, y: deltaY },
      current: { x: touch.clientX, y: touch.clientY },
    });

    // Prevent scrolling if handling gesture
    if (Math.abs(deltaX) > 10) {
      event.preventDefault();
    }
  }

  /**
   * Handle touch end
   */
  private handleTouchEnd(event: TouchEvent): void {
    console.log("Touch end:", {
      changedTouches: event.changedTouches.length,
    });

    this.element.classList.remove("touching");
    this.touchStartPosition = null;
  }

  /**
   * Handle touch cancel (interrupted by system)
   */
  private handleTouchCancel(event: TouchEvent): void {
    console.log("Touch cancelled");
    this.element.classList.remove("touching");
    this.touchStartPosition = null;
  }

  /**
   * Get touch information
   */
  getTouchInfo(touch: Touch): {
    identifier: number;
    position: { x: number; y: number };
    radius: { x: number; y: number };
    force: number;
    target: EventTarget | null;
  } {
    return {
      identifier: touch.identifier,
      position: {
        x: touch.clientX,
        y: touch.clientY,
      },
      radius: {
        x: touch.radiusX || 0,
        y: touch.radiusY || 0,
      },
      force: touch.force,
      target: touch.target,
    };
  }

  /**
   * Remove listeners
   */
  destroy(): void {
    this.element.removeEventListener("touchstart", this.handleTouchStart);
    this.element.removeEventListener("touchmove", this.handleTouchMove);
    this.element.removeEventListener("touchend", this.handleTouchEnd);
    this.element.removeEventListener("touchcancel", this.handleTouchCancel);
  }
}

// Usage
const interactiveElement = document.querySelector<HTMLElement>(".swipeable")!;
const touchManager = new TouchEventManager(interactiveElement);
```

### Multi-Touch Handling

```typescript
interface MultiTouchState {
  touches: Map<number, Touch>;
  startTime: number;
  lastUpdateTime: number;
}

class MultiTouchHandler {
  private state: MultiTouchState = {
    touches: new Map(),
    startTime: 0,
    lastUpdateTime: 0,
  };

  /**
   * Handle touchstart with multiple fingers
   */
  handleTouchStart(event: TouchEvent): void {
    this.state.startTime = Date.now();

    // Store all touches
    Array.from(event.touches).forEach((touch) => {
      this.state.touches.set(touch.identifier, touch);
    });

    console.log(`${this.state.touches.size} fingers touching`);

    // Detect gestures
    if (this.state.touches.size === 2) {
      this.handlePinchStart(event);
    } else if (this.state.touches.size === 3) {
      console.log("Three-finger gesture detected");
    }
  }

  /**
   * Handle touchmove with multiple fingers
   */
  handleTouchMove(event: TouchEvent): void {
    this.state.lastUpdateTime = Date.now();

    // Update touch positions
    Array.from(event.touches).forEach((touch) => {
      this.state.touches.set(touch.identifier, touch);
    });

    if (this.state.touches.size === 2) {
      this.handlePinchMove(event);
    }
  }

  /**
   * Handle touchend - remove ended touches
   */
  handleTouchEnd(event: TouchEvent): void {
    // Remove ended touches
    Array.from(event.changedTouches).forEach((touch) => {
      this.state.touches.delete(touch.identifier);
    });

    console.log(`${this.state.touches.size} fingers remaining`);

    // Reset if no touches remaining
    if (this.state.touches.size === 0) {
      this.reset();
    }
  }

  /**
   * Calculate distance between two touches (for pinch)
   */
  private getDistance(touch1: Touch, touch2: Touch): number {
    const dx = touch2.clientX - touch1.clientX;
    const dy = touch2.clientY - touch1.clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }

  /**
   * Calculate center point between two touches
   */
  private getCenter(touch1: Touch, touch2: Touch): { x: number; y: number } {
    return {
      x: (touch1.clientX + touch2.clientX) / 2,
      y: (touch1.clientY + touch2.clientY) / 2,
    };
  }

  /**
   * Handle pinch start
   */
  private initialDistance: number = 0;

  private handlePinchStart(event: TouchEvent): void {
    const touches = Array.from(event.touches);
    if (touches.length === 2) {
      this.initialDistance = this.getDistance(touches[0], touches[1]);
      console.log("Pinch started, initial distance:", this.initialDistance);
    }
  }

  /**
   * Handle pinch move
   */
  private handlePinchMove(event: TouchEvent): void {
    const touches = Array.from(event.touches);
    if (touches.length === 2) {
      const currentDistance = this.getDistance(touches[0], touches[1]);
      const scale = currentDistance / this.initialDistance;
      const center = this.getCenter(touches[0], touches[1]);

      console.log("Pinch:", {
        scale: scale.toFixed(2),
        center,
        gesture: scale > 1 ? "zoom in" : "zoom out",
      });

      // Dispatch custom pinch event
      event.target?.dispatchEvent(
        new CustomEvent("pinch", {
          detail: { scale, center },
        }),
      );
    }
  }

  /**
   * Reset state
   */
  private reset(): void {
    this.state.touches.clear();
    this.initialDistance = 0;
  }
}

// Usage
const multiTouch = new MultiTouchHandler();
const zoomableElement = document.querySelector(".zoomable")!;

zoomableElement.addEventListener("touchstart", (e) =>
  multiTouch.handleTouchStart(e),
);
zoomableElement.addEventListener("touchmove", (e) =>
  multiTouch.handleTouchMove(e),
);
zoomableElement.addEventListener("touchend", (e) =>
  multiTouch.handleTouchEnd(e),
);

// Listen for custom pinch event
zoomableElement.addEventListener("pinch", ((e: CustomEvent) => {
  const { scale, center } = e.detail;
  console.log(`Pinch to scale: ${scale}`);
  // Apply zoom transformation
}) as EventListener);
```

## 3. Pointer Events API (Unified Approach)

### Why Pointer Events?

```typescript
/**
 * Pointer Events unify mouse, touch, and pen inputs
 * - Single event model for all input types
 * - Includes hover support for compatible devices
 * - Provides pressure sensitivity info
 * - Better multi-device support
 */

interface PointerEventInfo {
  pointerId: number;
  pointerType: "mouse" | "pen" | "touch";
  isPrimary: boolean;
  pressure: number; // 0-1
  tiltX: number; // -90 to 90 degrees
  tiltY: number;
  width: number; // Contact geometry
  height: number;
}

class UnifiedPointerHandler {
  private element: HTMLElement;
  private activePointers: Map<number, PointerEventInfo> = new Map();

  constructor(element: HTMLElement) {
    this.element = element;
    this.attachListeners();
  }

  /**
   * Attach pointer event listeners
   */
  private attachListeners(): void {
    this.element.addEventListener(
      "pointerdown",
      this.handlePointerDown.bind(this),
    );
    this.element.addEventListener(
      "pointermove",
      this.handlePointerMove.bind(this),
    );
    this.element.addEventListener("pointerup", this.handlePointerUp.bind(this));
    this.element.addEventListener(
      "pointercancel",
      this.handlePointerCancel.bind(this),
    );

    // Hover events (work for mouse and pen, not touch)
    this.element.addEventListener(
      "pointerenter",
      this.handlePointerEnter.bind(this),
    );
    this.element.addEventListener(
      "pointerleave",
      this.handlePointerLeave.bind(this),
    );
  }

  /**
   * Extract pointer info from event
   */
  private getPointerInfo(event: PointerEvent): PointerEventInfo {
    return {
      pointerId: event.pointerId,
      pointerType: event.pointerType as "mouse" | "pen" | "touch",
      isPrimary: event.isPrimary,
      pressure: event.pressure,
      tiltX: event.tiltX,
      tiltY: event.tiltY,
      width: event.width,
      height: event.height,
    };
  }

  /**
   * Handle pointer down
   */
  private handlePointerDown(event: PointerEvent): void {
    const info = this.getPointerInfo(event);
    this.activePointers.set(info.pointerId, info);

    // Capture pointer for reliable move/up events
    this.element.setPointerCapture(event.pointerId);

    console.log("Pointer down:", {
      type: info.pointerType,
      id: info.pointerId,
      pressure: info.pressure,
      position: { x: event.clientX, y: event.clientY },
    });

    this.element.classList.add("active");
  }

  /**
   * Handle pointer move
   */
  private handlePointerMove(event: PointerEvent): void {
    const info = this.getPointerInfo(event);

    if (this.activePointers.has(info.pointerId)) {
      // Update stored pointer info
      this.activePointers.set(info.pointerId, info);

      console.log("Pointer move:", {
        type: info.pointerType,
        pressure: info.pressure,
        position: { x: event.clientX, y: event.clientY },
      });
    }
  }

  /**
   * Handle pointer up
   */
  private handlePointerUp(event: PointerEvent): void {
    const pointerId = event.pointerId;

    // Release pointer capture
    this.element.releasePointerCapture(pointerId);

    this.activePointers.delete(pointerId);

    console.log("Pointer up:", {
      type: event.pointerType,
      id: pointerId,
    });

    if (this.activePointers.size === 0) {
      this.element.classList.remove("active");
    }
  }

  /**
   * Handle pointer cancel
   */
  private handlePointerCancel(event: PointerEvent): void {
    console.log("Pointer cancelled:", event.pointerId);
    this.activePointers.delete(event.pointerId);
    this.element.classList.remove("active");
  }

  /**
   * Handle pointer enter (hover)
   */
  private handlePointerEnter(event: PointerEvent): void {
    console.log("Pointer enter:", event.pointerType);

    // Only apply hover styles for mouse/pen, not touch
    if (event.pointerType !== "touch") {
      this.element.classList.add("hover");
    }
  }

  /**
   * Handle pointer leave
   */
  private handlePointerLeave(event: PointerEvent): void {
    console.log("Pointer leave:", event.pointerType);
    this.element.classList.remove("hover");
  }

  /**
   * Check if specific pointer type is active
   */
  hasPointerType(type: "mouse" | "pen" | "touch"): boolean {
    return Array.from(this.activePointers.values()).some(
      (pointer) => pointer.pointerType === type,
    );
  }

  /**
   * Get all active pointers
   */
  getActivePointers(): PointerEventInfo[] {
    return Array.from(this.activePointers.values());
  }
}

// Usage
const interactiveEl = document.querySelector<HTMLElement>(".interactive")!;
const pointerHandler = new UnifiedPointerHandler(interactiveEl);
```

### CSS for Pointer Events

```css
/* Enable pointer events handling */
.interactive {
  touch-action: none; /* Disable default touch behaviors */
  cursor: pointer;
}

/* Different styles for different pointer types */
@media (pointer: coarse) {
  /* Touch devices */
  .interactive {
    padding: 16px; /* Larger touch targets */
  }
}

@media (pointer: fine) {
  /* Mouse/stylus devices */
  .interactive {
    padding: 8px;
  }

  .interactive:hover {
    background: #e0e0e0;
  }
}

/* Active state for all pointer types */
.interactive.active {
  background: #2196f3;
  color: white;
}

/* Hover only for non-touch */
.interactive.hover {
  background: #f0f0f0;
}
```

## 4. Gesture Recognition

### Swipe Gesture Detection

```typescript
interface SwipeConfig {
  threshold: number; // Minimum distance (px)
  timeLimit: number; // Maximum time (ms)
  velocityThreshold: number; // Minimum velocity (px/ms)
}

interface SwipeResult {
  direction: "left" | "right" | "up" | "down";
  distance: number;
  velocity: number;
  duration: number;
}

class SwipeGestureDetector {
  private element: HTMLElement;
  private config: SwipeConfig;
  private startPoint: { x: number; y: number; time: number } | null = null;

  constructor(element: HTMLElement, config: Partial<SwipeConfig> = {}) {
    this.element = element;
    this.config = {
      threshold: config.threshold || 50,
      timeLimit: config.timeLimit || 300,
      velocityThreshold: config.velocityThreshold || 0.3,
    };

    this.attachListeners();
  }

  /**
   * Attach event listeners
   */
  private attachListeners(): void {
    // Use pointer events for unified handling
    this.element.addEventListener("pointerdown", this.handleStart.bind(this));
    this.element.addEventListener("pointermove", this.handleMove.bind(this));
    this.element.addEventListener("pointerup", this.handleEnd.bind(this));
    this.element.addEventListener(
      "pointercancel",
      this.handleCancel.bind(this),
    );

    // Prevent default touch behaviors
    this.element.style.touchAction = "pan-y"; // Allow vertical scroll
  }

  /**
   * Handle gesture start
   */
  private handleStart(event: PointerEvent): void {
    this.startPoint = {
      x: event.clientX,
      y: event.clientY,
      time: Date.now(),
    };

    this.element.setPointerCapture(event.pointerId);
  }

  /**
   * Handle gesture move
   */
  private handleMove(event: PointerEvent): void {
    if (!this.startPoint) return;

    const deltaX = event.clientX - this.startPoint.x;
    const deltaY = event.clientY - this.startPoint.y;

    // Prevent scroll if horizontal swipe detected
    if (Math.abs(deltaX) > Math.abs(deltaY) && Math.abs(deltaX) > 10) {
      event.preventDefault();
    }
  }

  /**
   * Handle gesture end
   */
  private handleEnd(event: PointerEvent): void {
    if (!this.startPoint) return;

    const endPoint = {
      x: event.clientX,
      y: event.clientY,
      time: Date.now(),
    };

    const swipe = this.detectSwipe(this.startPoint, endPoint);

    if (swipe) {
      this.dispatchSwipeEvent(swipe);
    }

    this.element.releasePointerCapture(event.pointerId);
    this.startPoint = null;
  }

  /**
   * Handle gesture cancel
   */
  private handleCancel(event: PointerEvent): void {
    this.startPoint = null;
  }

  /**
   * Detect swipe from start and end points
   */
  private detectSwipe(
    start: { x: number; y: number; time: number },
    end: { x: number; y: number; time: number },
  ): SwipeResult | null {
    const deltaX = end.x - start.x;
    const deltaY = end.y - start.y;
    const duration = end.time - start.time;

    // Check time limit
    if (duration > this.config.timeLimit) {
      return null;
    }

    const absX = Math.abs(deltaX);
    const absY = Math.abs(deltaY);
    const distance = Math.sqrt(deltaX ** 2 + deltaY ** 2);

    // Check threshold
    if (distance < this.config.threshold) {
      return null;
    }

    // Calculate velocity
    const velocity = distance / duration;
    if (velocity < this.config.velocityThreshold) {
      return null;
    }

    // Determine direction (horizontal vs vertical)
    const direction: SwipeResult["direction"] =
      absX > absY
        ? deltaX > 0
          ? "right"
          : "left"
        : deltaY > 0
          ? "down"
          : "up";

    return {
      direction,
      distance,
      velocity,
      duration,
    };
  }

  /**
   * Dispatch custom swipe event
   */
  private dispatchSwipeEvent(swipe: SwipeResult): void {
    const event = new CustomEvent("swipe", {
      detail: swipe,
      bubbles: true,
    });

    this.element.dispatchEvent(event);

    // Also dispatch direction-specific event
    const directionEvent = new CustomEvent(`swipe${swipe.direction}`, {
      detail: swipe,
      bubbles: true,
    });

    this.element.dispatchEvent(directionEvent);
  }
}

// Usage
const swipeableCard = document.querySelector<HTMLElement>(".card")!;
const swipeDetector = new SwipeGestureDetector(swipeableCard, {
  threshold: 75,
  timeLimit: 250,
  velocityThreshold: 0.5,
});

// Listen for swipe events
swipeableCard.addEventListener("swipe", ((e: CustomEvent<SwipeResult>) => {
  console.log("Swipe detected:", e.detail);
}) as EventListener);

swipeableCard.addEventListener("swipeleft", () => {
  console.log("Swiped left - next item");
  // Navigate to next
});

swipeableCard.addEventListener("swiperight", () => {
  console.log("Swiped right - previous item");
  // Navigate to previous
});
```

### Pinch-to-Zoom Gesture

```typescript
interface PinchState {
  initialDistance: number;
  initialScale: number;
  currentScale: number;
  center: { x: number; y: number };
}

class PinchZoomHandler {
  private element: HTMLElement;
  private state: PinchState | null = null;
  private minScale: number = 0.5;
  private maxScale: number = 3;
  private currentTransform = {
    scale: 1,
    translateX: 0,
    translateY: 0,
  };

  constructor(element: HTMLElement, minScale = 0.5, maxScale = 3) {
    this.element = element;
    this.minScale = minScale;
    this.maxScale = maxScale;
    this.attachListeners();
  }

  /**
   * Attach event listeners
   */
  private attachListeners(): void {
    this.element.addEventListener(
      "pointerdown",
      this.handlePointerDown.bind(this),
    );
    this.element.addEventListener(
      "pointermove",
      this.handlePointerMove.bind(this),
    );
    this.element.addEventListener("pointerup", this.handlePointerUp.bind(this));
    this.element.addEventListener(
      "pointercancel",
      this.handlePointerCancel.bind(this),
    );

    // Disable default touch actions
    this.element.style.touchAction = "none";
  }

  private pointers: Map<number, PointerEvent> = new Map();

  /**
   * Handle pointer down
   */
  private handlePointerDown(event: PointerEvent): void {
    this.pointers.set(event.pointerId, event);
    this.element.setPointerCapture(event.pointerId);

    // Start pinch if two pointers
    if (this.pointers.size === 2) {
      const [p1, p2] = Array.from(this.pointers.values());
      this.startPinch(p1, p2);
    }
  }

  /**
   * Handle pointer move
   */
  private handlePointerMove(event: PointerEvent): void {
    if (!this.pointers.has(event.pointerId)) return;

    this.pointers.set(event.pointerId, event);

    if (this.pointers.size === 2) {
      const [p1, p2] = Array.from(this.pointers.values());
      this.updatePinch(p1, p2);
    }
  }

  /**
   * Handle pointer up
   */
  private handlePointerUp(event: PointerEvent): void {
    this.pointers.delete(event.pointerId);
    this.element.releasePointerCapture(event.pointerId);

    if (this.pointers.size < 2) {
      this.endPinch();
    }
  }

  /**
   * Handle pointer cancel
   */
  private handlePointerCancel(event: PointerEvent): void {
    this.pointers.delete(event.pointerId);
    this.endPinch();
  }

  /**
   * Calculate distance between two pointers
   */
  private getDistance(p1: PointerEvent, p2: PointerEvent): number {
    const dx = p2.clientX - p1.clientX;
    const dy = p2.clientY - p1.clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }

  /**
   * Calculate center between two pointers
   */
  private getCenter(
    p1: PointerEvent,
    p2: PointerEvent,
  ): { x: number; y: number } {
    return {
      x: (p1.clientX + p2.clientX) / 2,
      y: (p1.clientY + p2.clientY) / 2,
    };
  }

  /**
   * Start pinch gesture
   */
  private startPinch(p1: PointerEvent, p2: PointerEvent): void {
    this.state = {
      initialDistance: this.getDistance(p1, p2),
      initialScale: this.currentTransform.scale,
      currentScale: this.currentTransform.scale,
      center: this.getCenter(p1, p2),
    };
  }

  /**
   * Update pinch gesture
   */
  private updatePinch(p1: PointerEvent, p2: PointerEvent): void {
    if (!this.state) return;

    const currentDistance = this.getDistance(p1, p2);
    const scale =
      (currentDistance / this.state.initialDistance) * this.state.initialScale;

    // Clamp scale
    const clampedScale = Math.min(
      Math.max(scale, this.minScale),
      this.maxScale,
    );

    this.state.currentScale = clampedScale;
    this.currentTransform.scale = clampedScale;

    this.applyTransform();

    // Dispatch custom event
    this.element.dispatchEvent(
      new CustomEvent("pinchzoom", {
        detail: {
          scale: clampedScale,
          center: this.state.center,
        },
      }),
    );
  }

  /**
   * End pinch gesture
   */
  private endPinch(): void {
    this.state = null;
  }

  /**
   * Apply transform to element
   */
  private applyTransform(): void {
    const { scale, translateX, translateY } = this.currentTransform;
    this.element.style.transform = `translate(${translateX}px, ${translateY}px) scale(${scale})`;
  }

  /**
   * Reset zoom
   */
  reset(): void {
    this.currentTransform = { scale: 1, translateX: 0, translateY: 0 };
    this.applyTransform();
  }
}

// Usage
const zoomableImage = document.querySelector<HTMLElement>(".zoomable-image")!;
const pinchZoom = new PinchZoomHandler(zoomableImage, 0.5, 4);

// Listen for pinch events
zoomableImage.addEventListener("pinchzoom", ((e: CustomEvent) => {
  console.log("Current zoom scale:", e.detail.scale);
}) as EventListener);

// Double-tap to reset
let lastTap = 0;
zoomableImage.addEventListener("pointerdown", (e) => {
  const now = Date.now();
  if (now - lastTap < 300) {
    pinchZoom.reset();
  }
  lastTap = now;
});
```

## 5. Hover Alternatives for Touch Devices

### Touch-Friendly Navigation

```html
<!-- Desktop: hover dropdown -->
<!-- Mobile: tap to expand -->
<nav class="responsive-nav">
  <ul class="nav-menu">
    <li class="nav-item has-submenu">
      <a href="#" class="nav-link">Products</a>
      <ul class="submenu">
        <li><a href="#">Product 1</a></li>
        <li><a href="#">Product 2</a></li>
        <li><a href="#">Product 3</a></li>
      </ul>
    </li>
  </ul>
</nav>

<style>
  .submenu {
    display: none;
    position: absolute;
    top: 100%;
    left: 0;
    background: white;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    min-width: 200px;
  }

  /* Desktop: hover to show */
  @media (pointer: fine) {
    .has-submenu:hover .submenu {
      display: block;
    }
  }

  /* Touch: tap to toggle */
  @media (pointer: coarse) {
    .has-submenu.active .submenu {
      display: block;
    }

    .nav-link::after {
      content: " ▾";
    }
  }
</style>
```

### TypeScript Touch/Hover Handler

```typescript
class ResponsiveInteractionHandler {
  private element: HTMLElement;
  private isTouch: boolean = false;

  constructor(element: HTMLElement) {
    this.element = element;
    this.detectInputType();
    this.attachListeners();
  }

  /**
   * Detect primary input type
   */
  private detectInputType(): void {
    // Check if touch is available
    this.isTouch = window.matchMedia("(pointer: coarse)").matches;

    // Or use feature detection
    const hasTouch = "ontouchstart" in window || navigator.maxTouchPoints > 0;

    console.log("Input type:", this.isTouch ? "touch" : "mouse");
  }

  /**
   * Attach appropriate listeners
   */
  private attachListeners(): void {
    if (this.isTouch) {
      this.setupTouchBehavior();
    } else {
      this.setupMouseBehavior();
    }
  }

  /**
   * Setup touch behavior (tap to toggle)
   */
  private setupTouchBehavior(): void {
    this.element.addEventListener("click", (event) => {
      event.preventDefault();
      this.element.classList.toggle("active");
    });
  }

  /**
   * Setup mouse behavior (hover)
   */
  private setupMouseBehavior(): void {
    this.element.addEventListener("mouseenter", () => {
      this.element.classList.add("active");
    });

    this.element.addEventListener("mouseleave", () => {
      this.element.classList.remove("active");
    });
  }
}

// Usage
document.querySelectorAll<HTMLElement>(".has-submenu").forEach((item) => {
  new ResponsiveInteractionHandler(item);
});
```

### Tooltip Alternatives

```css
/* Desktop: hover tooltip */
@media (pointer: fine) {
  .tooltip-trigger:hover .tooltip {
    opacity: 1;
    visibility: visible;
  }
}

/* Touch: tap to show, tap outside to hide */
@media (pointer: coarse) {
  .tooltip-trigger.show-tooltip .tooltip {
    opacity: 1;
    visibility: visible;
  }

  /* Make trigger area larger */
  .tooltip-trigger {
    min-width: 44px;
    min-height: 44px;
  }
}
```

```typescript
class TouchFriendlyTooltip {
  private trigger: HTMLElement;
  private tooltip: HTMLElement;

  constructor(trigger: HTMLElement, tooltip: HTMLElement) {
    this.trigger = trigger;
    this.tooltip = tooltip;
    this.attachListeners();
  }

  private attachListeners(): void {
    // Touch devices: tap to toggle
    this.trigger.addEventListener("click", (event) => {
      event.stopPropagation();
      this.toggle();
    });

    // Hide when tapping outside
    document.addEventListener("click", () => {
      this.hide();
    });

    // Prevent tooltip clicks from closing
    this.tooltip.addEventListener("click", (event) => {
      event.stopPropagation();
    });
  }

  private toggle(): void {
    this.trigger.classList.toggle("show-tooltip");
  }

  private hide(): void {
    this.trigger.classList.remove("show-tooltip");
  }
}
```

## 6. Passive Event Listeners

### Understanding Passive Listeners

```typescript
/**
 * Passive listeners improve scroll performance
 * by telling browser that preventDefault() won't be called
 */

// ❌ BAD: Blocks scrolling until handler completes
element.addEventListener("touchstart", (event) => {
  // Heavy processing
  doExpensiveWork();
  // Browser waits to see if preventDefault() is called
});

// ✅ GOOD: Passive listener allows scroll immediately
element.addEventListener(
  "touchstart",
  (event) => {
    doExpensiveWork();
    // Browser knows preventDefault() won't be called
  },
  { passive: true },
);

// ⚠️ IMPORTANT: Can't call preventDefault() in passive listener
element.addEventListener(
  "touchmove",
  (event) => {
    // This will be ignored with a warning
    event.preventDefault(); // ❌ Won't work
  },
  { passive: true },
);
```

### Passive Listener Best Practices

```typescript
class PassiveEventHandler {
  /**
   * Attach passive listeners for better performance
   */
  static attachPassiveListeners(element: HTMLElement): void {
    // Passive touch events for scrollable content
    element.addEventListener("touchstart", this.handleTouchStart, {
      passive: true,
    });

    element.addEventListener("touchmove", this.handleTouchMove, {
      passive: true,
    });

    // Wheel events should usually be passive
    element.addEventListener("wheel", this.handleWheel, {
      passive: true,
    });
  }

  /**
   * Non-passive when you need preventDefault
   */
  static attachInteractiveListeners(element: HTMLElement): void {
    // Need to prevent default for custom gesture handling
    element.addEventListener("touchstart", this.handleInteractiveTouchStart, {
      passive: false,
    });

    element.addEventListener(
      "touchmove",
      (event) => {
        // Can call preventDefault here
        if (this.shouldPreventScroll(event)) {
          event.preventDefault();
        }
      },
      { passive: false },
    );
  }

  private static handleTouchStart(event: TouchEvent): void {
    // Just observing, not preventing
    console.log("Touch started");
  }

  private static handleTouchMove(event: TouchEvent): void {
    // Just observing
    console.log("Touch moving");
  }

  private static handleWheel(event: WheelEvent): void {
    // Just observing
    console.log("Wheel event");
  }

  private static handleInteractiveTouchStart(event: TouchEvent): void {
    // May need to prevent default
  }

  private static shouldPreventScroll(event: TouchEvent): boolean {
    // Custom logic to determine if scroll should be prevented
    return false;
  }

  /**
   * Feature detection for passive listener support
   */
  static supportsPassive(): boolean {
    let passive = false;

    try {
      const options = Object.defineProperty({}, "passive", {
        get: () => {
          passive = true;
          return true;
        },
      });

      window.addEventListener("test", null as any, options);
      window.removeEventListener("test", null as any, options);
    } catch (err) {
      passive = false;
    }

    return passive;
  }

  /**
   * Get options object for addEventListener
   */
  static getListenerOptions(passive: boolean = true): AddEventListenerOptions {
    const supportsPassive = this.supportsPassive();

    return supportsPassive ? { passive } : (false as any); // Legacy browsers need boolean
  }
}

// Usage
const scrollableElement = document.querySelector(".scrollable")!;

// Passive listeners for observing
PassiveEventHandler.attachPassiveListeners(scrollableElement as HTMLElement);

// Non-passive for interactive gestures
const gestureElement = document.querySelector(".gesture-zone")!;
PassiveEventHandler.attachInteractiveListeners(gestureElement as HTMLElement);
```

## 7. Touch-Friendly Form Inputs

### Input Field Optimization

```html
<!-- Optimized mobile inputs -->
<form class="mobile-friendly-form">
  <!-- Email with appropriate keyboard -->
  <input
    type="email"
    inputmode="email"
    autocomplete="email"
    placeholder="Email address"
    aria-label="Email address"
  />

  <!-- Telephone with numeric keyboard -->
  <input
    type="tel"
    inputmode="tel"
    autocomplete="tel"
    placeholder="Phone number"
    pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}"
    aria-label="Phone number"
  />

  <!-- Numeric input -->
  <input
    type="number"
    inputmode="numeric"
    min="0"
    max="100"
    step="1"
    placeholder="Age"
    aria-label="Age"
  />

  <!-- URL input -->
  <input
    type="url"
    inputmode="url"
    autocomplete="url"
    placeholder="Website"
    aria-label="Website URL"
  />

  <!-- Large textarea for touch typing -->
  <textarea
    rows="4"
    placeholder="Your message"
    aria-label="Message"
    style="min-height: 100px"
  ></textarea>

  <!-- Large touch-friendly button -->
  <button type="submit" style="min-height: 48px; padding: 12px 32px">
    Submit
  </button>
</form>

<style>
  .mobile-friendly-form input,
  .mobile-friendly-form textarea {
    min-height: 44px;
    padding: 12px 16px;
    font-size: 16px; /* Prevents zoom on focus in iOS */
    width: 100%;
    margin-bottom: 16px;
    border: 2px solid #ccc;
    border-radius: 8px;
    touch-action: manipulation;
  }

  .mobile-friendly-form input:focus,
  .mobile-friendly-form textarea:focus {
    outline: none;
    border-color: #2196f3;
    box-shadow: 0 0 0 3px rgba(33, 150, 243, 0.1);
  }
</style>
```

## 8. Complete Touch Interaction System

```typescript
/**
 * Comprehensive touch interaction system
 */
class TouchInteractionSystem {
  private element: HTMLElement;
  private config: {
    minTouchTarget: number;
    swipeThreshold: number;
    doubleTapDelay: number;
    longPressDelay: number;
  };

  constructor(element: HTMLElement) {
    this.element = element;
    this.config = {
      minTouchTarget: 44,
      swipeThreshold: 50,
      doubleTapDelay: 300,
      longPressDelay: 500,
    };

    this.init();
  }

  private init(): void {
    this.validateTouchTargets();
    this.setupGestureDetection();
    this.setupAccessibility();
  }

  /**
   * Validate all touch targets meet minimum size
   */
  private validateTouchTargets(): void {
    const validator = new TouchTargetValidator();
    const interactiveElements = this.element.querySelectorAll<HTMLElement>(
      'button, a, input, [role="button"]',
    );

    interactiveElements.forEach((el) => {
      const metrics = validator.validateElement(el);
      if (!metrics.isAccessible) {
        console.warn("Touch target too small:", el, metrics.recommendation);
        validator.fixTouchTarget(el);
      }
    });
  }

  /**
   * Setup comprehensive gesture detection
   */
  private setupGestureDetection(): void {
    // Swipe detection
    const swipeDetector = new SwipeGestureDetector(this.element);

    // Pinch-to-zoom
    if (this.element.classList.contains("zoomable")) {
      new PinchZoomHandler(this.element);
    }

    // Unified pointer events
    new UnifiedPointerHandler(this.element);
  }

  /**
   * Setup accessibility features
   */
  private setupAccessibility(): void {
    // Ensure focus visible
    this.element
      .querySelectorAll<HTMLElement>("button, a, input")
      .forEach((el) => {
        el.style.outlineOffset = "2px";
      });

    // Add touch feedback
    this.addTouchFeedback();
  }

  /**
   * Add visual feedback for touch interactions
   */
  private addTouchFeedback(): void {
    const style = document.createElement("style");
    style.textContent = `
      .touch-feedback {
        position: relative;
        overflow: hidden;
      }
      
      .touch-feedback::after {
        content: '';
        position: absolute;
        top: 50%;
        left: 50%;
        width: 0;
        height: 0;
        border-radius: 50%;
        background: rgba(255, 255, 255, 0.5);
        transform: translate(-50%, -50%);
        transition: width 0.3s, height 0.3s;
      }
      
      .touch-feedback.active::after {
        width: 100%;
        height: 100%;
      }
    `;
    document.head.appendChild(style);
  }
}

// Initialize for entire application
window.addEventListener("DOMContentLoaded", () => {
  const app = document.getElementById("app")!;
  new TouchInteractionSystem(app);
});
```

## 10 Key Takeaways

1. **44×44px Minimum**: WCAG 2.1 AAA requires 44×44 CSS pixels for touch targets. Use 48×48px for better UX. Apply min-width/min-height and adequate spacing (8px minimum).

2. **Pointer Events Over Touch Events**: Use Pointer Events API for unified handling of mouse, touch, and pen. Single event model (`pointerdown`, `pointermove`, `pointerup`) works across all input types.

3. **Touch Event Sequence**: `touchstart` → `touchmove` → `touchend` or `touchcancel`. Unlike mouse, touch has no hover state and supports multi-touch via `touches` array.

4. **Passive Listeners**: Use `{ passive: true }` for scroll-related listeners (`touchstart`, `touchmove`, `wheel`) to prevent scroll jank. Only use `passive: false` when you need `preventDefault()`.

5. **No Hover on Touch**: Touch devices lack hover state. Use tap-to-toggle for menus/tooltips, not hover-only interactions. Test with `@media (pointer: coarse)` and provide visible affordances.

6. **Gesture Recognition**: Implement swipe (threshold: 50px, time limit: 300ms), pinch-to-zoom (two-finger distance tracking), and long-press (500ms delay). Prevent default behaviors selectively.

7. **touch-action CSS**: Use `touch-action: manipulation` to disable double-tap zoom, `touch-action: pan-y` to allow only vertical scroll, or `touch-action: none` for custom gesture handling.

8. **Pointer Capture**: Call `setPointerCapture()` on `pointerdown` to receive `pointermove`/`pointerup` even if pointer leaves element. Release with `releasePointerCapture()`.

9. **Multi-Touch Support**: Track multiple touches via `identifier` property. Use `touches` (active), `targetTouches` (on element), and `changedTouches` (just changed) arrays.

10. **Form Input Optimization**: Use `inputmode` attribute (email, tel, numeric) for appropriate keyboards. Font-size ≥16px prevents iOS zoom. Min-height 44px for text inputs.
