# Responsive Images: Modern Techniques for Performance and Quality

## Overview

Responsive images are critical for modern web performance, ensuring users receive appropriately sized and formatted images based on their device capabilities, screen resolution, and viewport size. This guide covers modern techniques including srcset, sizes, picture element, next-gen formats, and lazy loading strategies.

## Core Concepts

### The Problem Space

```typescript
interface ImageDeliveryChallenge {
  deviceDiversity: {
    screenSizes: string[]; // 320px to 5K displays
    pixelDensities: number[]; // 1x, 2x, 3x, 4x
    networkConditions: string[]; // slow 3G to fiber
    formatSupport: string[]; // JPEG, WebP, AVIF, etc.
  };
  performanceGoals: {
    initialLoadTime: number; // < 2.5s LCP
    bandwidthUsage: string; // minimal data transfer
    visualQuality: string; // crisp on all screens
  };
}

// Without responsive images: waste of bandwidth
const unnecessaryDataTransfer = {
  scenario: "Mobile user receives desktop-sized image",
  imageSize: "2.5MB (4000x3000)",
  actualNeed: "150KB (800x600)",
  wastedData: "2.35MB (94% waste)",
  loadTimeImpact: "+3-5 seconds on 3G",
};
```

## 1. srcset and sizes Attributes

### Basic srcset Syntax

```html
<!-- Density descriptors (x): For fixed-width images -->
<img
  src="logo-1x.png"
  srcset="logo-1x.png 1x, logo-2x.png 2x, logo-3x.png 3x"
  alt="Company Logo"
  width="200"
  height="50"
/>

<!-- Width descriptors (w): For flexible images -->
<img
  src="photo-800.jpg"
  srcset="
    photo-400.jpg   400w,
    photo-800.jpg   800w,
    photo-1200.jpg 1200w,
    photo-1600.jpg 1600w,
    photo-2000.jpg 2000w
  "
  sizes="(max-width: 600px) 100vw,
         (max-width: 1200px) 50vw,
         800px"
  alt="Product photo"
/>
```

### Advanced sizes Attribute

```html
<!-- Complex layout calculations -->
<img
  srcset="
    hero-400.jpg   400w,
    hero-800.jpg   800w,
    hero-1200.jpg 1200w,
    hero-1600.jpg 1600w,
    hero-2400.jpg 2400w
  "
  sizes="(max-width: 600px) calc(100vw - 32px),
         (max-width: 1200px) calc(50vw - 24px),
         (max-width: 1800px) calc(33.333vw - 20px),
         600px"
  src="hero-800.jpg"
  alt="Hero image"
/>

<!-- Container query style sizing -->
<img
  srcset="card-300.jpg 300w, card-600.jpg 600w, card-900.jpg 900w"
  sizes="(min-width: 1200px) 300px,
         (min-width: 768px) calc((100vw - 64px) / 3),
         (min-width: 480px) calc((100vw - 48px) / 2),
         calc(100vw - 32px)"
  src="card-600.jpg"
  alt="Card image"
/>
```

### TypeScript Helper for srcset Generation

```typescript
interface ImageVariant {
  width: number;
  path: string;
  density?: number;
}

interface SrcSetConfig {
  variants: ImageVariant[];
  baseUrl?: string;
  format?: "width" | "density";
}

class ResponsiveImageGenerator {
  /**
   * Generate srcset string from image variants
   */
  static generateSrcSet(config: SrcSetConfig): string {
    const { variants, baseUrl = "", format = "width" } = config;

    return variants
      .map((variant) => {
        const url = `${baseUrl}${variant.path}`;
        const descriptor =
          format === "density" ? `${variant.density}x` : `${variant.width}w`;
        return `${url} ${descriptor}`;
      })
      .join(",\n          ");
  }

  /**
   * Generate sizes attribute based on breakpoints
   */
  static generateSizes(
    breakpoints: {
      maxWidth: number;
      size: string;
    }[],
  ): string {
    const mediaQueries = breakpoints
      .slice(0, -1)
      .map((bp) => `(max-width: ${bp.maxWidth}px) ${bp.size}`)
      .join(",\n         ");

    const defaultSize = breakpoints[breakpoints.length - 1].size;
    return `${mediaQueries},\n         ${defaultSize}`;
  }

  /**
   * Calculate optimal image width based on DPR
   */
  static calculateOptimalWidth(
    displayWidth: number,
    devicePixelRatio: number = window.devicePixelRatio,
  ): number {
    return Math.ceil(displayWidth * devicePixelRatio);
  }

  /**
   * Select appropriate image from srcset
   */
  static selectImageVariant(
    variants: ImageVariant[],
    targetWidth: number,
  ): ImageVariant {
    // Sort by width
    const sorted = [...variants].sort((a, b) => a.width - b.width);

    // Find smallest image that's >= target width
    const match = sorted.find((v) => v.width >= targetWidth);

    // If no match, return largest available
    return match || sorted[sorted.length - 1];
  }
}

// Usage example
const productImageConfig: SrcSetConfig = {
  baseUrl: "/images/products/",
  format: "width",
  variants: [
    { width: 400, path: "product-400.jpg" },
    { width: 800, path: "product-800.jpg" },
    { width: 1200, path: "product-1200.jpg" },
    { width: 1600, path: "product-1600.jpg" },
  ],
};

const srcset = ResponsiveImageGenerator.generateSrcSet(productImageConfig);
// Returns: "/images/products/product-400.jpg 400w,
//           /images/products/product-800.jpg 800w, ..."

const sizes = ResponsiveImageGenerator.generateSizes([
  { maxWidth: 600, size: "100vw" },
  { maxWidth: 1200, size: "50vw" },
  { maxWidth: Infinity, size: "600px" },
]);
// Returns: "(max-width: 600px) 100vw,
//          (max-width: 1200px) 50vw,
//          600px"
```

## 2. Picture Element for Art Direction

### Basic Picture Usage

```html
<!-- Art direction: different crops for different viewports -->
<picture>
  <!-- Mobile: portrait crop -->
  <source
    media="(max-width: 767px)"
    srcset="hero-mobile-400.jpg 400w, hero-mobile-800.jpg 800w"
    sizes="100vw"
  />

  <!-- Tablet: square crop -->
  <source
    media="(max-width: 1023px)"
    srcset="hero-tablet-600.jpg 600w, hero-tablet-1200.jpg 1200w"
    sizes="100vw"
  />

  <!-- Desktop: landscape crop -->
  <source
    media="(min-width: 1024px)"
    srcset="
      hero-desktop-1200.jpg 1200w,
      hero-desktop-1800.jpg 1800w,
      hero-desktop-2400.jpg 2400w
    "
    sizes="100vw"
  />

  <!-- Fallback -->
  <img src="hero-desktop-1200.jpg" alt="Hero banner" loading="lazy" />
</picture>
```

### Format Selection with Picture

```html
<!-- Modern format support with fallbacks -->
<picture>
  <!-- AVIF: Best compression, newest format -->
  <source
    type="image/avif"
    srcset="photo-400.avif 400w, photo-800.avif 800w, photo-1200.avif 1200w"
    sizes="(max-width: 768px) 100vw, 800px"
  />

  <!-- WebP: Good compression, wide support -->
  <source
    type="image/webp"
    srcset="photo-400.webp 400w, photo-800.webp 800w, photo-1200.webp 1200w"
    sizes="(max-width: 768px) 100vw, 800px"
  />

  <!-- JPEG: Universal fallback -->
  <img
    src="photo-800.jpg"
    srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1200.jpg 1200w"
    sizes="(max-width: 768px) 100vw, 800px"
    alt="Product showcase"
    loading="lazy"
  />
</picture>
```

### TypeScript Picture Element Builder

```typescript
interface PictureSource {
  media?: string;
  type?: string;
  srcset: string;
  sizes?: string;
}

interface PictureConfig {
  sources: PictureSource[];
  fallback: {
    src: string;
    alt: string;
    loading?: "lazy" | "eager";
    decoding?: "async" | "sync" | "auto";
    fetchpriority?: "high" | "low" | "auto";
  };
}

class PictureElementBuilder {
  private config: PictureConfig;

  constructor(config: PictureConfig) {
    this.config = config;
  }

  /**
   * Generate HTML string for picture element
   */
  toHTML(): string {
    const sources = this.config.sources
      .map((source) => {
        const attrs = [
          source.media && `media="${source.media}"`,
          source.type && `type="${source.type}"`,
          `srcset="${source.srcset}"`,
          source.sizes && `sizes="${source.sizes}"`,
        ]
          .filter(Boolean)
          .join("\n    ");

        return `  <source\n    ${attrs}\n  />`;
      })
      .join("\n");

    const { src, alt, loading, decoding, fetchpriority } = this.config.fallback;
    const imgAttrs = [
      `src="${src}"`,
      `alt="${alt}"`,
      loading && `loading="${loading}"`,
      decoding && `decoding="${decoding}"`,
      fetchpriority && `fetchpriority="${fetchpriority}"`,
    ]
      .filter(Boolean)
      .join("\n    ");

    return `<picture>\n${sources}\n  <img\n    ${imgAttrs}\n  />\n</picture>`;
  }

  /**
   * Create picture element in DOM
   */
  createElement(): HTMLPictureElement {
    const picture = document.createElement("picture");

    // Add source elements
    this.config.sources.forEach((source) => {
      const sourceEl = document.createElement("source");
      if (source.media) sourceEl.media = source.media;
      if (source.type) sourceEl.type = source.type;
      sourceEl.srcset = source.srcset;
      if (source.sizes) sourceEl.sizes = source.sizes;
      picture.appendChild(sourceEl);
    });

    // Add img element
    const img = document.createElement("img");
    Object.assign(img, this.config.fallback);
    picture.appendChild(img);

    return picture;
  }
}

// Usage: Format-based picture element
const formatPicture = new PictureElementBuilder({
  sources: [
    {
      type: "image/avif",
      srcset: "hero-800.avif 800w, hero-1600.avif 1600w",
      sizes: "100vw",
    },
    {
      type: "image/webp",
      srcset: "hero-800.webp 800w, hero-1600.webp 1600w",
      sizes: "100vw",
    },
  ],
  fallback: {
    src: "hero-800.jpg",
    alt: "Hero image",
    loading: "lazy",
    decoding: "async",
  },
});

// Usage: Art direction picture element
const artDirectionPicture = new PictureElementBuilder({
  sources: [
    {
      media: "(max-width: 767px)",
      srcset: "banner-mobile-600.jpg 600w, banner-mobile-1200.jpg 1200w",
      sizes: "100vw",
    },
    {
      media: "(min-width: 768px)",
      srcset: "banner-desktop-1200.jpg 1200w, banner-desktop-2400.jpg 2400w",
      sizes: "100vw",
    },
  ],
  fallback: {
    src: "banner-desktop-1200.jpg",
    alt: "Marketing banner",
    loading: "eager",
    fetchpriority: "high",
  },
});
```

## 3. Modern Image Formats: WebP and AVIF

### Format Comparison

```typescript
interface ImageFormatComparison {
  format: string;
  compression: number; // 0-100, higher = better
  quality: number; // 0-100, higher = better
  browserSupport: number; // % of browsers
  fileSize: number; // KB for sample image
  useCase: string;
}

const formatComparison: ImageFormatComparison[] = [
  {
    format: "JPEG",
    compression: 60,
    quality: 70,
    browserSupport: 100,
    fileSize: 100,
    useCase: "Universal fallback, photos",
  },
  {
    format: "WebP",
    compression: 85,
    quality: 85,
    browserSupport: 97,
    fileSize: 40,
    useCase: "Modern browsers, 60% smaller",
  },
  {
    format: "AVIF",
    compression: 95,
    quality: 90,
    browserSupport: 89,
    fileSize: 25,
    useCase: "Cutting edge, 75% smaller",
  },
];
```

### Progressive Enhancement Strategy

```html
<!-- Best practice: AVIF → WebP → JPEG -->
<picture>
  <!-- AVIF: Newest, best compression -->
  <source
    type="image/avif"
    srcset="
      product-400.avif   400w,
      product-800.avif   800w,
      product-1200.avif 1200w,
      product-1600.avif 1600w
    "
    sizes="(max-width: 768px) 100vw,
           (max-width: 1200px) 50vw,
           600px"
  />

  <!-- WebP: Good compression, wide support -->
  <source
    type="image/webp"
    srcset="
      product-400.webp   400w,
      product-800.webp   800w,
      product-1200.webp 1200w,
      product-1600.webp 1600w
    "
    sizes="(max-width: 768px) 100vw,
           (max-width: 1200px) 50vw,
           600px"
  />

  <!-- JPEG: Universal fallback -->
  <img
    src="product-800.jpg"
    srcset="
      product-400.jpg   400w,
      product-800.jpg   800w,
      product-1200.jpg 1200w,
      product-1600.jpg 1600w
    "
    sizes="(max-width: 768px) 100vw,
           (max-width: 1200px) 50vw,
           600px"
    alt="Product image"
    loading="lazy"
    decoding="async"
  />
</picture>
```

### Format Detection and Serving

```typescript
class ImageFormatDetector {
  private supportCache: Map<string, boolean> = new Map();

  /**
   * Check if browser supports specific image format
   */
  async supportsFormat(format: "webp" | "avif" | "jpeg"): Promise<boolean> {
    // Check cache first
    if (this.supportCache.has(format)) {
      return this.supportCache.get(format)!;
    }

    // Test images (1x1 pixel)
    const testImages: Record<string, string> = {
      webp: "data:image/webp;base64,UklGRiIAAABXRUJQVlA4IBYAAAAwAQCdASoBAAEADsD+JaQAA3AAAAAA",
      avif: "data:image/avif;base64,AAAAIGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZk1BMUIAAADybWV0YQAAAAAAAAAoaGRscgAAAAAAAAAAcGljdAAAAAAAAAAAAAAAAGxpYmF2aWYAAAAADnBpdG0AAAAAAAEAAAAeaWxvYwAAAABEAAABAAEAAAABAAABGgAAAB0AAAAoaWluZgAAAAAAAQAAABppbmZlAgAAAAABAABhdjAxQ29sb3IAAAAAamlwcnAAAABLaXBjbwAAABRpc3BlAAAAAAAAAAIAAAACAAAAEHBpeGkAAAAAAwgICAAAAAxhdjFDgQ0MAAAAABNjb2xybmNseAACAAIAAYAAAAAXaXBtYQAAAAAAAAABAAEEAQKDBAAAACVtZGF0EgAKCBgANogQEAwgMg8f8D///8WfhwB8+ErK42A=",
      jpeg: "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQH/2wBDAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQH/wAARCAABAAEDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAv/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/8QAFQEBAQAAAAAAAAAAAAAAAAAAAAX/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oADAMBAAIRAxEAPwA/wA==",
    };

    try {
      const supported = await this.testImageFormat(testImages[format]);
      this.supportCache.set(format, supported);
      return supported;
    } catch {
      this.supportCache.set(format, false);
      return false;
    }
  }

  /**
   * Test if image format loads successfully
   */
  private testImageFormat(base64: string): Promise<boolean> {
    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => resolve(img.width === 1 && img.height === 1);
      img.onerror = () => resolve(false);
      img.src = base64;
    });
  }

  /**
   * Get best supported format for image
   */
  async getBestFormat(
    availableFormats: Array<"avif" | "webp" | "jpeg">,
  ): Promise<"avif" | "webp" | "jpeg"> {
    // Check formats in order of preference
    for (const format of availableFormats) {
      const supported = await this.supportsFormat(format);
      if (supported) return format;
    }
    return "jpeg"; // Ultimate fallback
  }

  /**
   * Build image URL with best format
   */
  async buildImageUrl(
    basePath: string,
    filename: string,
    width?: number,
  ): Promise<string> {
    const format = await this.getBestFormat(["avif", "webp", "jpeg"]);
    const widthSuffix = width ? `-${width}` : "";
    return `${basePath}/${filename}${widthSuffix}.${format}`;
  }
}

// Usage
const formatDetector = new ImageFormatDetector();

async function loadOptimalImage(imageId: string): Promise<void> {
  const format = await formatDetector.getBestFormat(["avif", "webp", "jpeg"]);
  console.log(`Loading ${imageId} in ${format} format`);

  const imgUrl = await formatDetector.buildImageUrl("/images", imageId, 800);
  // Use imgUrl for image loading
}
```

## 4. Lazy Loading Strategies

### Native Lazy Loading

```html
<!-- Native lazy loading with loading attribute -->
<img
  src="image-800.jpg"
  srcset="image-400.jpg 400w, image-800.jpg 800w, image-1200.jpg 1200w"
  sizes="(max-width: 768px) 100vw, 800px"
  alt="Product image"
  loading="lazy"
  decoding="async"
/>

<!-- Eager loading for above-the-fold images -->
<img
  src="hero-1200.jpg"
  srcset="hero-800.jpg 800w, hero-1200.jpg 1200w, hero-1600.jpg 1600w"
  sizes="100vw"
  alt="Hero banner"
  loading="eager"
  fetchpriority="high"
/>
```

### Intersection Observer Lazy Loading

```typescript
interface LazyLoadConfig {
  rootMargin?: string;
  threshold?: number;
  loadingClass?: string;
  loadedClass?: string;
  errorClass?: string;
}

class LazyImageLoader {
  private observer: IntersectionObserver;
  private config: Required<LazyLoadConfig>;

  constructor(config: LazyLoadConfig = {}) {
    this.config = {
      rootMargin: config.rootMargin || "50px",
      threshold: config.threshold || 0.01,
      loadingClass: config.loadingClass || "lazy-loading",
      loadedClass: config.loadedClass || "lazy-loaded",
      errorClass: config.errorClass || "lazy-error",
    };

    this.observer = new IntersectionObserver(
      this.handleIntersection.bind(this),
      {
        rootMargin: this.config.rootMargin,
        threshold: this.config.threshold,
      },
    );
  }

  /**
   * Observe images for lazy loading
   */
  observe(elements: NodeListOf<HTMLImageElement> | HTMLImageElement[]): void {
    elements.forEach((img) => {
      // Skip if already loaded or no data-src
      if (img.classList.contains(this.config.loadedClass) || !img.dataset.src) {
        return;
      }

      this.observer.observe(img);
    });
  }

  /**
   * Handle intersection changes
   */
  private handleIntersection(
    entries: IntersectionObserverEntry[],
    observer: IntersectionObserver,
  ): void {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target as HTMLImageElement;
        this.loadImage(img);
        observer.unobserve(img);
      }
    });
  }

  /**
   * Load image from data attributes
   */
  private loadImage(img: HTMLImageElement): void {
    img.classList.add(this.config.loadingClass);

    // Set srcset if available
    if (img.dataset.srcset) {
      img.srcset = img.dataset.srcset;
    }

    // Set sizes if available
    if (img.dataset.sizes) {
      img.sizes = img.dataset.sizes;
    }

    // Set src
    const src = img.dataset.src!;

    img.onload = () => {
      img.classList.remove(this.config.loadingClass);
      img.classList.add(this.config.loadedClass);

      // Clean up data attributes
      delete img.dataset.src;
      delete img.dataset.srcset;
      delete img.dataset.sizes;
    };

    img.onerror = () => {
      img.classList.remove(this.config.loadingClass);
      img.classList.add(this.config.errorClass);
      console.error(`Failed to load image: ${src}`);
    };

    img.src = src;
  }

  /**
   * Disconnect observer
   */
  disconnect(): void {
    this.observer.disconnect();
  }
}

// Usage
const lazyLoader = new LazyImageLoader({
  rootMargin: "100px",
  threshold: 0.01,
});

// Observe all lazy images
const lazyImages = document.querySelectorAll<HTMLImageElement>("img[data-src]");
lazyLoader.observe(lazyImages);
```

### HTML for Intersection Observer Lazy Loading

```html
<!-- Lazy loaded with Intersection Observer -->
<img
  class="lazy-image"
  src="placeholder-10x10.jpg"
  data-src="product-800.jpg"
  data-srcset="product-400.jpg 400w,
               product-800.jpg 800w,
               product-1200.jpg 1200w"
  data-sizes="(max-width: 768px) 100vw, 600px"
  alt="Product image"
  loading="lazy"
/>

<style>
  .lazy-image {
    opacity: 0;
    transition: opacity 0.3s;
  }

  .lazy-image.lazy-loading {
    opacity: 0.5;
  }

  .lazy-image.lazy-loaded {
    opacity: 1;
  }

  .lazy-image.lazy-error {
    opacity: 1;
    background: #f0f0f0;
  }
</style>
```

## 5. Blur-Up Technique (LQIP)

### Low Quality Image Placeholder

```typescript
interface BlurUpConfig {
  placeholderQuality?: number; // 5-20
  transitionDuration?: number; // ms
  blurAmount?: number; // px
}

class BlurUpImageLoader {
  private config: Required<BlurUpConfig>;

  constructor(config: BlurUpConfig = {}) {
    this.config = {
      placeholderQuality: config.placeholderQuality || 10,
      transitionDuration: config.transitionDuration || 300,
      blurAmount: config.blurAmount || 20,
    };
  }

  /**
   * Create blur-up container with placeholder
   */
  createBlurUpImage(
    placeholder: string,
    fullImage: string,
    alt: string,
    srcset?: string,
    sizes?: string,
  ): HTMLDivElement {
    const container = document.createElement("div");
    container.className = "blur-up-container";
    container.style.position = "relative";
    container.style.overflow = "hidden";

    // Placeholder (blurred)
    const placeholderImg = document.createElement("img");
    placeholderImg.className = "blur-up-placeholder";
    placeholderImg.src = placeholder;
    placeholderImg.alt = alt;
    placeholderImg.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      filter: blur(${this.config.blurAmount}px);
      transform: scale(1.1);
      transition: opacity ${this.config.transitionDuration}ms;
    `;

    // Full image
    const fullImg = document.createElement("img");
    fullImg.className = "blur-up-full";
    fullImg.src = fullImage;
    if (srcset) fullImg.srcset = srcset;
    if (sizes) fullImg.sizes = sizes;
    fullImg.alt = alt;
    fullImg.loading = "lazy";
    fullImg.decoding = "async";
    fullImg.style.cssText = `
      position: relative;
      width: 100%;
      height: auto;
      opacity: 0;
      transition: opacity ${this.config.transitionDuration}ms;
    `;

    // Load handler
    fullImg.onload = () => {
      fullImg.style.opacity = "1";
      placeholderImg.style.opacity = "0";

      // Remove placeholder after transition
      setTimeout(() => {
        if (placeholderImg.parentNode) {
          container.removeChild(placeholderImg);
        }
      }, this.config.transitionDuration);
    };

    container.appendChild(placeholderImg);
    container.appendChild(fullImg);

    return container;
  }

  /**
   * Generate placeholder from full image (client-side)
   */
  async generatePlaceholder(
    imageUrl: string,
    width: number = 40,
  ): Promise<string> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.crossOrigin = "anonymous";

      img.onload = () => {
        const canvas = document.createElement("canvas");
        const aspectRatio = img.height / img.width;
        canvas.width = width;
        canvas.height = Math.round(width * aspectRatio);

        const ctx = canvas.getContext("2d")!;
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

        resolve(canvas.toDataURL("image/jpeg", 0.5));
      };

      img.onerror = reject;
      img.src = imageUrl;
    });
  }
}

// Usage
const blurUpLoader = new BlurUpImageLoader({
  blurAmount: 20,
  transitionDuration: 400,
});

const container = blurUpLoader.createBlurUpImage(
  "product-placeholder-40.jpg", // Tiny, blurred placeholder
  "product-800.jpg", // Full resolution
  "Product image",
  "product-400.jpg 400w, product-800.jpg 800w, product-1200.jpg 1200w",
  "(max-width: 768px) 100vw, 600px",
);

document.getElementById("gallery")?.appendChild(container);
```

### CSS-Only Blur-Up

```html
<div class="blur-up-wrapper">
  <img
    class="blur-up-placeholder"
    src="tiny-placeholder-20x15.jpg"
    alt="Product"
    aria-hidden="true"
  />
  <img
    class="blur-up-full"
    src="product-800.jpg"
    srcset="product-400.jpg 400w, product-800.jpg 800w, product-1200.jpg 1200w"
    sizes="(max-width: 768px) 100vw, 600px"
    alt="Product"
    loading="lazy"
  />
</div>

<style>
  .blur-up-wrapper {
    position: relative;
    overflow: hidden;
    background: #f0f0f0;
  }

  .blur-up-placeholder {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    filter: blur(20px);
    transform: scale(1.1);
    transition: opacity 0.4s ease-out;
  }

  .blur-up-full {
    position: relative;
    width: 100%;
    height: auto;
    display: block;
    opacity: 0;
    transition: opacity 0.4s ease-in;
  }

  /* When full image loads */
  .blur-up-full.loaded {
    opacity: 1;
  }

  .blur-up-full.loaded + .blur-up-placeholder {
    opacity: 0;
  }
</style>

<script>
  // Add loaded class when image loads
  document.querySelectorAll(".blur-up-full").forEach((img) => {
    if (img.complete) {
      img.classList.add("loaded");
    } else {
      img.addEventListener("load", () => {
        img.classList.add("loaded");
      });
    }
  });
</script>
```

## 6. Aspect Ratio Preservation

### Modern Aspect Ratio CSS

```css
/* Modern approach: aspect-ratio property */
.image-container {
  aspect-ratio: 16 / 9;
  width: 100%;
  overflow: hidden;
}

.image-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
}

/* Different ratios for different contexts */
.avatar {
  aspect-ratio: 1 / 1; /* Square */
}

.hero {
  aspect-ratio: 21 / 9; /* Ultra-wide */
}

.portrait {
  aspect-ratio: 3 / 4;
}

/* Responsive aspect ratios */
.card-image {
  aspect-ratio: 4 / 3;
}

@media (min-width: 768px) {
  .card-image {
    aspect-ratio: 16 / 9;
  }
}
```

### TypeScript Aspect Ratio Handler

```typescript
interface AspectRatioConfig {
  width: number;
  height: number;
  objectFit?: "cover" | "contain" | "fill" | "none" | "scale-down";
  objectPosition?: string;
}

class AspectRatioManager {
  /**
   * Calculate aspect ratio from dimensions
   */
  static calculateRatio(width: number, height: number): string {
    const gcd = this.greatestCommonDivisor(width, height);
    return `${width / gcd} / ${height / gcd}`;
  }

  /**
   * Get greatest common divisor
   */
  private static greatestCommonDivisor(a: number, b: number): number {
    return b === 0 ? a : this.greatestCommonDivisor(b, a % b);
  }

  /**
   * Apply aspect ratio to element
   */
  static applyAspectRatio(
    element: HTMLElement,
    config: AspectRatioConfig,
  ): void {
    const ratio = this.calculateRatio(config.width, config.height);
    element.style.aspectRatio = ratio;

    if (config.objectFit) {
      element.style.objectFit = config.objectFit;
    }

    if (config.objectPosition) {
      element.style.objectPosition = config.objectPosition;
    }
  }

  /**
   * Create responsive aspect ratio container
   */
  static createContainer(
    config: AspectRatioConfig,
    breakpoints?: Map<number, AspectRatioConfig>,
  ): HTMLDivElement {
    const container = document.createElement("div");
    container.className = "aspect-ratio-container";

    // Base styles
    this.applyAspectRatio(container, config);
    container.style.width = "100%";
    container.style.overflow = "hidden";

    // Responsive styles via media queries
    if (breakpoints) {
      const styleSheet = document.createElement("style");
      const uniqueClass = `aspect-${Date.now()}`;
      container.classList.add(uniqueClass);

      let css = "";
      breakpoints.forEach((bpConfig, minWidth) => {
        const ratio = this.calculateRatio(bpConfig.width, bpConfig.height);
        css += `
          @media (min-width: ${minWidth}px) {
            .${uniqueClass} {
              aspect-ratio: ${ratio};
            }
          }
        `;
      });

      styleSheet.textContent = css;
      document.head.appendChild(styleSheet);
    }

    return container;
  }

  /**
   * Preserve aspect ratio with explicit width/height
   */
  static setDimensionsWithRatio(
    img: HTMLImageElement,
    displayWidth: number,
    aspectRatio: number,
  ): void {
    img.width = displayWidth;
    img.height = Math.round(displayWidth / aspectRatio);
  }
}

// Usage examples
// Apply 16:9 aspect ratio
const heroImage = document.querySelector<HTMLElement>(".hero-image")!;
AspectRatioManager.applyAspectRatio(heroImage, {
  width: 16,
  height: 9,
  objectFit: "cover",
  objectPosition: "center",
});

// Create responsive container
const breakpoints = new Map<number, AspectRatioConfig>([
  [768, { width: 16, height: 9 }],
  [1024, { width: 21, height: 9 }],
]);

const container = AspectRatioManager.createContainer(
  { width: 4, height: 3 }, // Mobile default
  breakpoints,
);
```

## 7. Performance Monitoring

```typescript
interface ImagePerformanceMetrics {
  url: string;
  loadTime: number;
  fileSize: number;
  renderTime: number;
  naturalWidth: number;
  naturalHeight: number;
  displayWidth: number;
  displayHeight: number;
  wastedPixels: number;
  format: string;
}

class ImagePerformanceMonitor {
  private metrics: ImagePerformanceMetrics[] = [];
  private observer: PerformanceObserver;

  constructor() {
    // Monitor resource timing for images
    this.observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (entry.initiatorType === "img") {
          this.recordImageLoad(entry as PerformanceResourceTiming);
        }
      });
    });

    this.observer.observe({ entryTypes: ["resource"] });
  }

  /**
   * Record image load metrics
   */
  private recordImageLoad(entry: PerformanceResourceTiming): void {
    const img = document.querySelector<HTMLImageElement>(
      `img[src*="${entry.name.split("/").pop()}"]`,
    );

    if (!img) return;

    const metric: ImagePerformanceMetrics = {
      url: entry.name,
      loadTime: entry.duration,
      fileSize: entry.transferSize,
      renderTime: entry.responseEnd - entry.requestStart,
      naturalWidth: img.naturalWidth,
      naturalHeight: img.naturalHeight,
      displayWidth: img.width,
      displayHeight: img.height,
      wastedPixels: this.calculateWastedPixels(img),
      format: this.extractFormat(entry.name),
    };

    this.metrics.push(metric);
  }

  /**
   * Calculate wasted pixels (oversized images)
   */
  private calculateWastedPixels(img: HTMLImageElement): number {
    const naturalPixels = img.naturalWidth * img.naturalHeight;
    const displayPixels = img.width * img.height * window.devicePixelRatio ** 2;
    return Math.max(0, naturalPixels - displayPixels);
  }

  /**
   * Extract format from URL
   */
  private extractFormat(url: string): string {
    const match = url.match(/\.(jpg|jpeg|png|webp|avif|gif)(\?|$)/i);
    return match ? match[1].toUpperCase() : "Unknown";
  }

  /**
   * Get performance summary
   */
  getSummary(): {
    totalImages: number;
    totalSize: number;
    averageLoadTime: number;
    oversizedImages: number;
    formatBreakdown: Record<string, number>;
  } {
    const totalSize = this.metrics.reduce((sum, m) => sum + m.fileSize, 0);
    const avgLoadTime =
      this.metrics.reduce((sum, m) => sum + m.loadTime, 0) /
      this.metrics.length;
    const oversized = this.metrics.filter((m) => m.wastedPixels > 10000).length;

    const formatBreakdown: Record<string, number> = {};
    this.metrics.forEach((m) => {
      formatBreakdown[m.format] = (formatBreakdown[m.format] || 0) + 1;
    });

    return {
      totalImages: this.metrics.length,
      totalSize: Math.round(totalSize / 1024), // KB
      averageLoadTime: Math.round(avgLoadTime),
      oversizedImages: oversized,
      formatBreakdown,
    };
  }

  /**
   * Get largest contentful paint image
   */
  getLCPImage(): ImagePerformanceMetrics | null {
    return this.metrics.reduce(
      (largest, current) => {
        const currentPixels = current.displayWidth * current.displayHeight;
        const largestPixels = largest
          ? largest.displayWidth * largest.displayHeight
          : 0;
        return currentPixels > largestPixels ? current : largest;
      },
      null as ImagePerformanceMetrics | null,
    );
  }

  /**
   * Disconnect observer
   */
  disconnect(): void {
    this.observer.disconnect();
  }
}

// Usage
const perfMonitor = new ImagePerformanceMonitor();

// After page load, check metrics
window.addEventListener("load", () => {
  setTimeout(() => {
    const summary = perfMonitor.getSummary();
    console.log("Image Performance Summary:", summary);

    const lcp = perfMonitor.getLCPImage();
    if (lcp) {
      console.log("LCP Image:", lcp);
    }
  }, 2000);
});
```

## 8. Complete Implementation Example

```typescript
/**
 * Comprehensive responsive image component
 */
interface ResponsiveImageProps {
  src: string;
  alt: string;
  variants: ImageVariant[];
  formats?: Array<"avif" | "webp" | "jpeg">;
  sizes: string;
  aspectRatio?: { width: number; height: number };
  lazy?: boolean;
  priority?: boolean;
  placeholder?: string;
  onLoad?: () => void;
  onError?: (error: Error) => void;
}

class ResponsiveImage {
  private container: HTMLElement;
  private props: ResponsiveImageProps;
  private formatDetector: ImageFormatDetector;
  private lazyLoader?: LazyImageLoader;

  constructor(props: ResponsiveImageProps) {
    this.props = props;
    this.formatDetector = new ImageFormatDetector();
    this.container = this.createContainer();

    if (props.lazy && !props.priority) {
      this.lazyLoader = new LazyImageLoader();
    }
  }

  /**
   * Create responsive image container
   */
  private createContainer(): HTMLElement {
    const wrapper = document.createElement("div");
    wrapper.className = "responsive-image-wrapper";

    // Apply aspect ratio if provided
    if (this.props.aspectRatio) {
      const ratio = AspectRatioManager.calculateRatio(
        this.props.aspectRatio.width,
        this.props.aspectRatio.height,
      );
      wrapper.style.aspectRatio = ratio;
    }

    wrapper.style.cssText += `
      position: relative;
      overflow: hidden;
      width: 100%;
    `;

    return wrapper;
  }

  /**
   * Build picture element with all sources
   */
  private async buildPictureElement(): Promise<HTMLPictureElement> {
    const picture = document.createElement("picture");
    const formats = this.props.formats || ["avif", "webp", "jpeg"];

    // Add source for each format
    for (const format of formats) {
      const supported = await this.formatDetector.supportsFormat(format);
      if (!supported && format !== "jpeg") continue;

      const source = document.createElement("source");
      source.type = `image/${format}`;

      // Build srcset for this format
      source.srcset = this.props.variants
        .map((v) => `${this.replaceExtension(v.path, format)} ${v.width}w`)
        .join(", ");

      source.sizes = this.props.sizes;
      picture.appendChild(source);
    }

    // Add img fallback
    const img = this.createImgElement();
    picture.appendChild(img);

    return picture;
  }

  /**
   * Create img element
   */
  private createImgElement(): HTMLImageElement {
    const img = document.createElement("img");
    img.alt = this.props.alt;
    img.src = this.props.src;

    // Build srcset
    img.srcset = this.props.variants
      .map((v) => `${v.path} ${v.width}w`)
      .join(", ");

    img.sizes = this.props.sizes;

    // Loading strategy
    if (this.props.priority) {
      img.loading = "eager";
      img.fetchPriority = "high";
    } else if (this.props.lazy) {
      img.loading = "lazy";
    }

    img.decoding = "async";

    // Event handlers
    img.onload = () => {
      img.classList.add("loaded");
      this.props.onLoad?.();
    };

    img.onerror = () => {
      const error = new Error(`Failed to load image: ${this.props.src}`);
      this.props.onError?.(error);
    };

    return img;
  }

  /**
   * Replace file extension
   */
  private replaceExtension(path: string, format: string): string {
    return path.replace(/\.[^.]+$/, `.${format}`);
  }

  /**
   * Render the image
   */
  async render(): Promise<HTMLElement> {
    // Add placeholder if provided
    if (this.props.placeholder) {
      const placeholder = document.createElement("img");
      placeholder.src = this.props.placeholder;
      placeholder.alt = "";
      placeholder.className = "responsive-image-placeholder";
      placeholder.style.cssText = `
        position: absolute;
        width: 100%;
        height: 100%;
        filter: blur(20px);
        transform: scale(1.1);
      `;
      this.container.appendChild(placeholder);
    }

    // Build picture element
    const picture = await this.buildPictureElement();
    this.container.appendChild(picture);

    // Apply lazy loading if needed
    if (this.lazyLoader) {
      this.lazyLoader.observe([picture.querySelector("img")!]);
    }

    return this.container;
  }
}

// Usage example
async function createProductImage(): Promise<HTMLElement> {
  const responsiveImage = new ResponsiveImage({
    src: "product-800.jpg",
    alt: "Product showcase",
    variants: [
      { width: 400, path: "product-400.jpg" },
      { width: 800, path: "product-800.jpg" },
      { width: 1200, path: "product-1200.jpg" },
      { width: 1600, path: "product-1600.jpg" },
    ],
    formats: ["avif", "webp", "jpeg"],
    sizes: "(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 600px",
    aspectRatio: { width: 4, height: 3 },
    lazy: true,
    placeholder: "product-placeholder-40.jpg",
    onLoad: () => console.log("Image loaded successfully"),
    onError: (error) => console.error("Image load error:", error),
  });

  return await responsiveImage.render();
}
```

## 10 Key Takeaways

1. **srcset + sizes Required**: Always use `srcset` with width descriptors (w) and `sizes` attribute for flexible images. Browser selects optimal image based on viewport, DPR, and network.

2. **Picture for Art Direction**: Use `<picture>` when you need different crops/compositions for different viewports, or when serving multiple formats (AVIF/WebP/JPEG).

3. **Format Strategy**: AVIF first (best compression), WebP second (wide support), JPEG fallback (universal). Can save 60-75% bandwidth vs JPEG alone.

4. **Native Lazy Loading**: Use `loading="lazy"` for below-fold images. Combine with Intersection Observer for advanced control (rootMargin, custom placeholders).

5. **LCP Optimization**: Mark above-fold hero images with `loading="eager"` and `fetchpriority="high"`. Never lazy load LCP images.

6. **Blur-Up Technique**: Show tiny placeholder (20-40px wide) with blur filter while full image loads. Prevents layout shift and improves perceived performance.

7. **Aspect Ratio Preservation**: Use CSS `aspect-ratio` property to prevent CLS. Set explicit width/height attributes on img elements for size hints.

8. **Sizes Accuracy Matters**: Incorrect `sizes` attribute causes browser to select wrong image. Use browser DevTools to verify actual rendered sizes.

9. **Avoid Over-Serving**: Monitor "wasted pixels" - images much larger than display size. Aim for naturalSize ≤ displaySize × 1.5.

10. **Performance Monitoring**: Track image metrics (load time, file size, format distribution). Use PerformanceObserver to identify optimization opportunities and measure LCP impact.
