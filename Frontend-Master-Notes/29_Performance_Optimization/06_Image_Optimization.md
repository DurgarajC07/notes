# Image Optimization

## Core Concepts

Image optimization is critical for web performance as images typically comprise 50-70% of page weight. Modern image optimization involves format selection, responsive delivery, lazy loading, compression, and strategic use of CDNs. The goal is to deliver the smallest possible file while maintaining acceptable visual quality across all devices and network conditions.

### Key Concepts

- **Format Selection**: Choosing optimal format (WebP, AVIF, JPEG, PNG)
- **Responsive Images**: Delivering appropriate sizes for different viewports
- **Lazy Loading**: Deferring image load until needed
- **Compression**: Balancing file size vs visual quality
- **Art Direction**: Different crops/compositions for different screen sizes
- **Progressive Loading**: Blur-up technique for perceived performance
- **CDN Optimization**: Automatic format conversion and resizing

## TypeScript/JavaScript Code Examples

### Example 1: Modern Format Detection and Fallback

```typescript
// Detect browser support for modern image formats
class ImageFormatDetector {
  private supportCache: Map<string, boolean> = new Map();

  async supportsFormat(format: "webp" | "avif" | "jpeg2000"): Promise<boolean> {
    // Check cache first
    if (this.supportCache.has(format)) {
      return this.supportCache.get(format)!;
    }

    const supported = await this.detectSupport(format);
    this.supportCache.set(format, supported);
    return supported;
  }

  private async detectSupport(format: string): Promise<boolean> {
    // Test data URIs for each format
    const testImages: Record<string, string> = {
      webp: "data:image/webp;base64,UklGRiQAAABXRUJQVlA4IBgAAAAwAQCdASoBAAEAAwA0JaQAA3AA/vuUAAA=",
      avif: "data:image/avif;base64,AAAAIGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZk1BMUIAAADybWV0YQAAAAAAAAAoaGRscgAAAAAAAAAAcGljdAAAAAAAAAAAAAAAAGxpYmF2aWYAAAAADnBpdG0AAAAAAAEAAAAeaWxvYwAAAABEAAABAAEAAAABAAABGgAAAB0AAAAoaWluZgAAAAAAAQAAABppbmZlAgAAAAABAABhdjAxQ29sb3IAAAAAamlwcnAAAABLaXBjbwAAABRpc3BlAAAAAAAAAAEAAAABAAAAEHBpeGkAAAAAAwgICAAAAAxhdjFDgQ0MAAAAABNjb2xybmNseAACAAIAAYAAAAAXaXBtYQAAAAAAAAABAAEEAQKDBAAAACVtZGF0EgAKCBgANogQEAwgMg8f8D///8WfhwB8+ErK42A=",
      jpeg2000:
        "data:image/jp2;base64,/0//UQAyAAAAAAABAAAAAgAAAAAAAAAAAAAABAAAAAQAAAAAAAAAAAAEBwEBBwEBBwEBBwEB/1IADAAAAAEAAAQEAAH/XAAEQED/ZAAlAAFDcmVhdGVkIGJ5IE9wZW5KUEVHIHZlcnNpb24gMi4wLjD/kAAKAAAAAABYAAH/UwAJAQAABAQAAf9dAAUBQED/UwAJAgAABAQAAf9dAAUCQED/UwAJAwAABAQAAf9dAAUDQED/k wAKAAAAAABgAAH/UwAJBAAABAQAAf9dAAUEQED/UwAJBQAABAQAAf9dAAUFQED/UwAJBgAABAQAAf9dAAUGQED/k wAKAAAAAABoAAH/UwAJBwAABAQAAf9dAAUHQED/UwAJCAAABAQAAf9dAAUIQED/UwAJCQAABAQAAf9dAAUJQED/2Q==",
    };

    return new Promise((resolve) => {
      const img = new Image();

      img.onload = () => {
        resolve(img.width === 1 && img.height === 1);
      };

      img.onerror = () => {
        resolve(false);
      };

      img.src = testImages[format];
    });
  }

  // Get best supported format
  async getBestFormat(): Promise<"avif" | "webp" | "jpeg"> {
    if (await this.supportsFormat("avif")) {
      return "avif";
    } else if (await this.supportsFormat("webp")) {
      return "webp";
    } else {
      return "jpeg";
    }
  }

  // Clear cache
  clearCache(): void {
    this.supportCache.clear();
  }
}

// Image URL builder with format selection
class ImageURLBuilder {
  private detector = new ImageFormatDetector();
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async buildURL(
    imagePath: string,
    options: ImageOptions = {},
  ): Promise<string> {
    const format = options.format || (await this.detector.getBestFormat());
    const width = options.width || "";
    const quality = options.quality || 80;

    // Build URL with parameters
    const params = new URLSearchParams();
    if (width) params.set("w", String(width));
    params.set("q", String(quality));
    params.set("fm", format);

    return `${this.baseUrl}${imagePath}?${params.toString()}`;
  }

  // Build srcset for responsive images
  async buildSrcSet(
    imagePath: string,
    widths: number[],
    options: ImageOptions = {},
  ): Promise<string> {
    const format = options.format || (await this.detector.getBestFormat());

    const srcsetParts = await Promise.all(
      widths.map(async (width) => {
        const url = await this.buildURL(imagePath, {
          ...options,
          width,
          format,
        });
        return `${url} ${width}w`;
      }),
    );

    return srcsetParts.join(", ");
  }
}

interface ImageOptions {
  width?: number;
  quality?: number;
  format?: "avif" | "webp" | "jpeg";
}

// Usage
const imageBuilder = new ImageURLBuilder("https://cdn.example.com");

async function loadOptimizedImage(
  container: HTMLElement,
  imagePath: string,
): Promise<void> {
  const img = document.createElement("img");

  // Get best format
  const detector = new ImageFormatDetector();
  const format = await detector.getBestFormat();

  // Build URLs for different formats
  const srcWebP = await imageBuilder.buildURL(imagePath, { format: "webp" });
  const srcAVIF = await imageBuilder.buildURL(imagePath, { format: "avif" });
  const srcJPEG = await imageBuilder.buildURL(imagePath, { format: "jpeg" });

  // Use picture element for format fallback
  const picture = document.createElement("picture");

  // AVIF source (best compression)
  if (await detector.supportsFormat("avif")) {
    const sourceAVIF = document.createElement("source");
    sourceAVIF.srcset = srcAVIF;
    sourceAVIF.type = "image/avif";
    picture.appendChild(sourceAVIF);
  }

  // WebP source (good compression)
  if (await detector.supportsFormat("webp")) {
    const sourceWebP = document.createElement("source");
    sourceWebP.srcset = srcWebP;
    sourceWebP.type = "image/webp";
    picture.appendChild(sourceWebP);
  }

  // JPEG fallback
  img.src = srcJPEG;
  img.alt = "Optimized image";

  picture.appendChild(img);
  container.appendChild(picture);
}
```

### Example 2: Responsive Images with Srcset and Sizes

```typescript
// Generate responsive image configuration
class ResponsiveImageGenerator {
  private breakpoints: number[];
  private imageCDN: string;

  constructor(
    imageCDN: string,
    breakpoints: number[] = [320, 640, 960, 1280, 1920, 2560],
  ) {
    this.imageCDN = imageCDN;
    this.breakpoints = breakpoints;
  }

  // Generate srcset attribute
  generateSrcSet(imagePath: string, options: ResponsiveOptions = {}): string {
    const widths = options.widths || this.breakpoints;
    const format = options.format || "webp";
    const quality = options.quality || 80;

    return widths
      .map((width) => {
        const url = this.buildImageURL(imagePath, width, format, quality);
        return `${url} ${width}w`;
      })
      .join(", ");
  }

  // Generate sizes attribute
  generateSizes(config: SizesConfig[]): string {
    return config
      .map(({ media, size }) => {
        if (media) {
          return `${media} ${size}`;
        }
        return size;
      })
      .join(", ");
  }

  // Build complete picture element
  createPictureElement(
    imagePath: string,
    alt: string,
    options: PictureOptions = {},
  ): HTMLPictureElement {
    const picture = document.createElement("picture");

    // Art direction sources
    if (options.sources) {
      options.sources.forEach((source) => {
        const sourceEl = document.createElement("source");
        sourceEl.media = source.media;
        sourceEl.srcset = this.generateSrcSet(source.src, options);
        if (source.type) {
          sourceEl.type = source.type;
        }
        picture.appendChild(sourceEl);
      });
    }

    // Main img element
    const img = document.createElement("img");
    img.srcset = this.generateSrcSet(imagePath, options);
    img.sizes = options.sizes || "100vw";
    img.alt = alt;
    img.loading = options.loading || "lazy";
    img.decoding = options.decoding || "async";

    // Add inline dimensions to prevent layout shift
    if (options.width && options.height) {
      img.width = options.width;
      img.height = options.height;
    }

    picture.appendChild(img);
    return picture;
  }

  private buildImageURL(
    path: string,
    width: number,
    format: string,
    quality: number,
  ): string {
    return `${this.imageCDN}${path}?w=${width}&fm=${format}&q=${quality}`;
  }

  // Calculate optimal sizes based on layout
  calculateSizes(containerWidth: number, viewportWidth: number): string {
    const percentage = (containerWidth / viewportWidth) * 100;
    return `${Math.round(percentage)}vw`;
  }
}

interface ResponsiveOptions {
  widths?: number[];
  format?: string;
  quality?: number;
}

interface SizesConfig {
  media?: string;
  size: string;
}

interface PictureOptions extends ResponsiveOptions {
  sources?: Array<{
    media: string;
    src: string;
    type?: string;
  }>;
  sizes?: string;
  loading?: "lazy" | "eager";
  decoding?: "sync" | "async" | "auto";
  width?: number;
  height?: number;
}

// Usage for hero image
const imageGen = new ResponsiveImageGenerator("https://cdn.example.com");

const heroImage = imageGen.createPictureElement("/hero.jpg", "Hero banner", {
  widths: [640, 960, 1280, 1920, 2560],
  format: "webp",
  quality: 85,
  sizes: "100vw",
  loading: "eager", // Hero is above fold
  width: 1920,
  height: 1080,
});

document.getElementById("hero")?.appendChild(heroImage);

// Usage with art direction
const artDirectedImage = imageGen.createPictureElement(
  "/product-desktop.jpg",
  "Product image",
  {
    sources: [
      {
        media: "(max-width: 640px)",
        src: "/product-mobile.jpg",
      },
      {
        media: "(max-width: 960px)",
        src: "/product-tablet.jpg",
      },
    ],
    sizes: imageGen.generateSizes([
      { media: "(max-width: 640px)", size: "100vw" },
      { media: "(max-width: 960px)", size: "50vw" },
      { size: "33vw" },
    ]),
    loading: "lazy",
  },
);
```

### Example 3: Advanced Lazy Loading

```typescript
// Intelligent lazy loading with priorities
class LazyImageLoader {
  private observer: IntersectionObserver;
  private loadedImages = new WeakSet<HTMLImageElement>();
  private priorityQueue: HTMLImageElement[] = [];
  private loading = false;

  constructor(options: LazyLoadOptions = {}) {
    this.observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      {
        rootMargin: options.rootMargin || "50px",
        threshold: options.threshold || 0.01,
      },
    );
  }

  // Observe image for lazy loading
  observe(img: HTMLImageElement, priority: number = 0): void {
    img.dataset.priority = String(priority);
    img.dataset.src = img.getAttribute("src") || "";
    img.dataset.srcset = img.getAttribute("srcset") || "";

    // Remove actual src to prevent loading
    img.removeAttribute("src");
    img.removeAttribute("srcset");

    this.observer.observe(img);
  }

  private handleIntersection(entries: IntersectionObserverEntry[]): void {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target as HTMLImageElement;
        this.addToQueue(img);
        this.observer.unobserve(img);
      }
    });

    if (!this.loading) {
      this.processQueue();
    }
  }

  private addToQueue(img: HTMLImageElement): void {
    const priority = parseInt(img.dataset.priority || "0", 10);

    // Insert based on priority
    const insertIndex = this.priorityQueue.findIndex(
      (queuedImg) => parseInt(queuedImg.dataset.priority || "0", 10) < priority,
    );

    if (insertIndex === -1) {
      this.priorityQueue.push(img);
    } else {
      this.priorityQueue.splice(insertIndex, 0, img);
    }
  }

  private async processQueue(): Promise<void> {
    if (this.loading || this.priorityQueue.length === 0) {
      return;
    }

    this.loading = true;

    while (this.priorityQueue.length > 0) {
      const img = this.priorityQueue.shift()!;

      if (!this.loadedImages.has(img)) {
        await this.loadImage(img);
        this.loadedImages.add(img);
      }
    }

    this.loading = false;
  }

  private loadImage(img: HTMLImageElement): Promise<void> {
    return new Promise((resolve, reject) => {
      const src = img.dataset.src;
      const srcset = img.dataset.srcset;

      if (!src && !srcset) {
        resolve();
        return;
      }

      // Create temp image to preload
      const tempImg = new Image();

      tempImg.onload = () => {
        // Apply to actual image
        if (srcset) img.srcset = srcset;
        if (src) img.src = src;

        img.classList.add("loaded");
        resolve();
      };

      tempImg.onerror = () => {
        console.error("Failed to load image:", src);
        img.classList.add("error");
        reject(new Error("Image load failed"));
      };

      // Start loading
      if (srcset) tempImg.srcset = srcset;
      if (src) tempImg.src = src;
    });
  }

  // Disconnect observer
  disconnect(): void {
    this.observer.disconnect();
    this.priorityQueue = [];
  }

  // Get loading statistics
  getStats(): { loaded: number; queued: number } {
    return {
      loaded: this.loadedImages.size,
      queued: this.priorityQueue.length,
    };
  }
}

interface LazyLoadOptions {
  rootMargin?: string;
  threshold?: number;
}

// Usage
const lazyLoader = new LazyImageLoader({
  rootMargin: "100px",
  threshold: 0.01,
});

// High priority (above fold)
document.querySelectorAll(".hero-image").forEach((img) => {
  lazyLoader.observe(img as HTMLImageElement, 10);
});

// Normal priority (visible soon)
document.querySelectorAll(".product-image").forEach((img) => {
  lazyLoader.observe(img as HTMLImageElement, 5);
});

// Low priority (far below fold)
document.querySelectorAll(".gallery-image").forEach((img) => {
  lazyLoader.observe(img as HTMLImageElement, 1);
});

// Native lazy loading fallback
class NativeLazyLoader {
  static setup(images: NodeListOf<HTMLImageElement>): void {
    // Check for native support
    if ("loading" in HTMLImageElement.prototype) {
      images.forEach((img) => {
        img.loading = "lazy";
      });
    } else {
      // Fallback to IntersectionObserver
      const lazyLoader = new LazyImageLoader();
      images.forEach((img) => lazyLoader.observe(img));
    }
  }
}
```

### Example 4: Blur-Up Progressive Loading

```typescript
// Progressive image loading with blur-up technique
class ProgressiveImageLoader {
  private placeholders = new WeakMap<HTMLImageElement, HTMLCanvasElement>();

  // Create low-quality placeholder
  async createPlaceholder(
    img: HTMLImageElement,
    quality: number = 10,
  ): Promise<void> {
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    // Get small version of image
    const lowQualitySrc = this.getLowQualityURL(img.dataset.src!, quality);
    const lowQualityImg = await this.loadLowQuality(lowQualitySrc);

    // Draw to canvas
    canvas.width = lowQualityImg.width;
    canvas.height = lowQualityImg.height;
    ctx.drawImage(lowQualityImg, 0, 0);

    // Apply blur
    ctx.filter = "blur(10px)";
    ctx.drawImage(canvas, 0, 0);

    // Set as background
    const dataURL = canvas.toDataURL("image/jpeg", 0.8);
    img.style.backgroundImage = `url(${dataURL})`;
    img.style.backgroundSize = "cover";

    this.placeholders.set(img, canvas);
  }

  // Load full quality image
  async loadFullQuality(img: HTMLImageElement): Promise<void> {
    const fullSrc = img.dataset.src!;
    const fullSrcset = img.dataset.srcset;

    return new Promise((resolve, reject) => {
      const tempImg = new Image();

      tempImg.onload = () => {
        // Fade in full quality image
        img.style.opacity = "0";

        if (fullSrcset) img.srcset = fullSrcset;
        img.src = fullSrc;

        // Animate fade-in
        requestAnimationFrame(() => {
          img.style.transition = "opacity 0.3s ease-in";
          img.style.opacity = "1";
        });

        // Remove placeholder background after fade
        setTimeout(() => {
          img.style.backgroundImage = "";
        }, 300);

        resolve();
      };

      tempImg.onerror = reject;

      if (fullSrcset) tempImg.srcset = fullSrcset;
      tempImg.src = fullSrc;
    });
  }

  private getLowQualityURL(src: string, quality: number): string {
    // Modify URL to get low quality version
    return src.replace(/(\.\w+)$/, `-lq$1`);
  }

  private loadLowQuality(src: string): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve(img);
      img.onerror = reject;
      img.src = src;
    });
  }

  // Complete progressive load
  async progressiveLoad(img: HTMLImageElement): Promise<void> {
    // Show placeholder first
    await this.createPlaceholder(img);

    // Then load full quality
    await this.loadFullQuality(img);
  }
}

// Usage
const progressiveLoader = new ProgressiveImageLoader();

async function setupProgressiveImages(): Promise<void> {
  const images = document.querySelectorAll("img[data-src]");

  for (const img of Array.from(images)) {
    await progressiveLoader.progressiveLoad(img as HTMLImageElement);
  }
}

// Base64 encoded tiny placeholder
class TinyPlaceholder {
  // Generate tiny 10x10 placeholder
  static generate(width: number, height: number, color: string): string {
    const canvas = document.createElement("canvas");
    canvas.width = 10;
    canvas.height = 10;

    const ctx = canvas.getContext("2d");
    if (!ctx) return "";

    // Fill with dominant color
    ctx.fillStyle = color;
    ctx.fillRect(0, 0, 10, 10);

    return canvas.toDataURL("image/jpeg", 0.3);
  }

  // Apply to image
  static apply(img: HTMLImageElement, placeholder: string): void {
    img.style.backgroundImage = `url(${placeholder})`;
    img.style.backgroundSize = "cover";
  }
}

// Usage with dominant color
const placeholder = TinyPlaceholder.generate(1920, 1080, "#4A90E2");
const img = document.querySelector("img") as HTMLImageElement;
TinyPlaceholder.apply(img, placeholder);
```

### Example 5: Image CDN Integration

```typescript
// Integrate with image CDN services
class ImageCDN {
  private provider: CDNProvider;
  private baseURL: string;

  constructor(provider: CDNProvider, baseURL: string) {
    this.provider = provider;
    this.baseURL = baseURL;
  }

  // Build optimized URL
  buildURL(path: string, transforms: ImageTransforms): string {
    switch (this.provider) {
      case "cloudinary":
        return this.buildCloudinaryURL(path, transforms);
      case "imgix":
        return this.buildImgixURL(path, transforms);
      case "cloudflare":
        return this.buildCloudflareURL(path, transforms);
      default:
        return `${this.baseURL}${path}`;
    }
  }

  private buildCloudinaryURL(
    path: string,
    transforms: ImageTransforms,
  ): string {
    const parts: string[] = [];

    if (transforms.width) parts.push(`w_${transforms.width}`);
    if (transforms.height) parts.push(`h_${transforms.height}`);
    if (transforms.quality) parts.push(`q_${transforms.quality}`);
    if (transforms.format) parts.push(`f_${transforms.format}`);
    if (transforms.crop) parts.push(`c_${transforms.crop}`);
    if (transforms.gravity) parts.push(`g_${transforms.gravity}`);
    if (transforms.dpr) parts.push(`dpr_${transforms.dpr}`);

    // Auto format and quality
    if (transforms.auto) {
      parts.push("f_auto", "q_auto");
    }

    const transformString = parts.join(",");
    return `${this.baseURL}/${transformString}${path}`;
  }

  private buildImgixURL(path: string, transforms: ImageTransforms): string {
    const params = new URLSearchParams();

    if (transforms.width) params.set("w", String(transforms.width));
    if (transforms.height) params.set("h", String(transforms.height));
    if (transforms.quality) params.set("q", String(transforms.quality));
    if (transforms.format) params.set("fm", transforms.format);
    if (transforms.crop) params.set("fit", transforms.crop);
    if (transforms.dpr) params.set("dpr", String(transforms.dpr));

    // Auto optimization
    if (transforms.auto) {
      params.set("auto", "format,compress");
    }

    return `${this.baseURL}${path}?${params.toString()}`;
  }

  private buildCloudflareURL(
    path: string,
    transforms: ImageTransforms,
  ): string {
    const parts: string[] = [];

    if (transforms.width) parts.push(`width=${transforms.width}`);
    if (transforms.height) parts.push(`height=${transforms.height}`);
    if (transforms.quality) parts.push(`quality=${transforms.quality}`);
    if (transforms.format) parts.push(`format=${transforms.format}`);
    if (transforms.crop) parts.push(`fit=${transforms.crop}`);

    const transformString = parts.join(",");
    return `${this.baseURL}/cdn-cgi/image/${transformString}${path}`;
  }

  // Generate responsive srcset
  generateSrcSet(
    path: string,
    widths: number[],
    transforms: Omit<ImageTransforms, "width"> = {},
  ): string {
    return widths
      .map((width) => {
        const url = this.buildURL(path, { ...transforms, width });
        return `${url} ${width}w`;
      })
      .join(", ");
  }

  // Get optimal DPR
  getOptimalDPR(): number {
    return window.devicePixelRatio || 1;
  }

  // Generate picture element with format variations
  generatePicture(
    path: string,
    transforms: ImageTransforms,
    alt: string,
  ): HTMLPictureElement {
    const picture = document.createElement("picture");

    // AVIF source
    const sourceAVIF = document.createElement("source");
    sourceAVIF.srcset = this.buildURL(path, { ...transforms, format: "avif" });
    sourceAVIF.type = "image/avif";
    picture.appendChild(sourceAVIF);

    // WebP source
    const sourceWebP = document.createElement("source");
    sourceWebP.srcset = this.buildURL(path, { ...transforms, format: "webp" });
    sourceWebP.type = "image/webp";
    picture.appendChild(sourceWebP);

    // Fallback
    const img = document.createElement("img");
    img.src = this.buildURL(path, { ...transforms, format: "jpg" });
    img.alt = alt;
    img.loading = "lazy";
    picture.appendChild(img);

    return picture;
  }
}

type CDNProvider = "cloudinary" | "imgix" | "cloudflare";

interface ImageTransforms {
  width?: number;
  height?: number;
  quality?: number;
  format?: string;
  crop?: string;
  gravity?: string;
  dpr?: number;
  auto?: boolean;
}

// Usage with Cloudinary
const cloudinary = new ImageCDN(
  "cloudinary",
  "https://res.cloudinary.com/demo/image/upload",
);

const productImageURL = cloudinary.buildURL("/product.jpg", {
  width: 800,
  quality: 85,
  format: "webp",
  crop: "fill",
  gravity: "auto",
  auto: true,
});

// Generate responsive srcset
const srcset = cloudinary.generateSrcSet("/hero.jpg", [640, 960, 1280, 1920], {
  quality: 85,
  format: "webp",
  auto: true,
  dpr: cloudinary.getOptimalDPR(),
});

// Generate picture with multiple formats
const picture = cloudinary.generatePicture(
  "/banner.jpg",
  {
    width: 1920,
    height: 600,
    quality: 90,
    crop: "fill",
  },
  "Banner image",
);

document.getElementById("banner")?.appendChild(picture);
```

### Example 6: Intersection Observer with Priority Loading

```typescript
// Advanced intersection observer for smart image loading
class SmartImageObserver {
  private observer: IntersectionObserver;
  private observers: Map<HTMLImageElement, IntersectionObserver> = new Map();
  private loadQueue: PriorityQueue<ImageLoadTask>;

  constructor() {
    this.loadQueue = new PriorityQueue<ImageLoadTask>(
      (a, b) => b.priority - a.priority,
    );

    this.setupDefaultObserver();
  }

  private setupDefaultObserver(): void {
    this.observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      {
        rootMargin: "50px 0px",
        threshold: [0, 0.25, 0.5, 0.75, 1],
      },
    );
  }

  // Observe image with custom priority
  observe(img: HTMLImageElement, options: ObserveOptions = {}): void {
    const priority = options.priority || this.calculatePriority(img);
    const rootMargin = options.rootMargin || "50px";

    // Create custom observer if needed
    const observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries, priority),
      { rootMargin },
    );

    observer.observe(img);
    this.observers.set(img, observer);
  }

  private handleIntersection(
    entries: IntersectionObserverEntry[],
    basePriority: number = 0,
  ): void {
    entries.forEach((entry) => {
      const img = entry.target as HTMLImageElement;

      if (entry.isIntersecting) {
        // Calculate dynamic priority based on intersection ratio
        const priority = basePriority + entry.intersectionRatio * 10;

        this.loadQueue.enqueue({
          img,
          priority,
          timestamp: Date.now(),
        });

        // Start processing queue
        this.processQueue();

        // Unobserve after queuing
        const observer = this.observers.get(img);
        if (observer) {
          observer.unobserve(img);
        }
      }
    });
  }

  private async processQueue(): Promise<void> {
    while (!this.loadQueue.isEmpty()) {
      const task = this.loadQueue.dequeue();
      if (!task) break;

      await this.loadImage(task.img);
    }
  }

  private async loadImage(img: HTMLImageElement): Promise<void> {
    const src = img.dataset.src;
    const srcset = img.dataset.srcset;

    if (!src && !srcset) return;

    return new Promise((resolve, reject) => {
      const tempImg = new Image();

      tempImg.onload = () => {
        if (srcset) img.srcset = srcset;
        if (src) img.src = src;

        img.classList.add("loaded");
        resolve();
      };

      tempImg.onerror = reject;

      if (srcset) tempImg.srcset = srcset;
      if (src) tempImg.src = src;
    });
  }

  private calculatePriority(img: HTMLImageElement): number {
    let priority = 0;

    // Check if above the fold
    const rect = img.getBoundingClientRect();
    if (rect.top < window.innerHeight) {
      priority += 10;
    }

    // Check size (larger images get higher priority)
    const area = rect.width * rect.height;
    priority += Math.min(area / 10000, 5);

    // Check if marked as important
    if (img.classList.contains("priority")) {
      priority += 20;
    }

    return priority;
  }

  cleanup(): void {
    for (const observer of this.observers.values()) {
      observer.disconnect();
    }
    this.observers.clear();
  }
}

interface ObserveOptions {
  priority?: number;
  rootMargin?: string;
}

interface ImageLoadTask {
  img: HTMLImageElement;
  priority: number;
  timestamp: number;
}

// Simple priority queue implementation
class PriorityQueue<T> {
  private items: T[] = [];
  private comparator: (a: T, b: T) => number;

  constructor(comparator: (a: T, b: T) => number) {
    this.comparator = comparator;
  }

  enqueue(item: T): void {
    this.items.push(item);
    this.items.sort(this.comparator);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

// Usage
const smartObserver = new SmartImageObserver();

// High priority hero images
document.querySelectorAll(".hero img").forEach((img) => {
  smartObserver.observe(img as HTMLImageElement, {
    priority: 100,
    rootMargin: "0px",
  });
});

// Normal priority content images
document.querySelectorAll(".content img").forEach((img) => {
  smartObserver.observe(img as HTMLImageElement, {
    priority: 50,
  });
});

// Low priority gallery images
document.querySelectorAll(".gallery img").forEach((img) => {
  smartObserver.observe(img as HTMLImageElement, {
    priority: 10,
    rootMargin: "200px",
  });
});
```

### Example 7: Adaptive Image Quality

```typescript
// Adjust image quality based on network conditions
class AdaptiveImageQuality {
  private connection: any;
  private qualityLevels: QualityLevel[] = [
    { name: "high", quality: 90, conditions: ["4g", "wifi"] },
    { name: "medium", quality: 75, conditions: ["3g"] },
    { name: "low", quality: 60, conditions: ["2g", "slow-2g"] },
  ];

  constructor() {
    this.connection =
      (navigator as any).connection ||
      (navigator as any).mozConnection ||
      (navigator as any).webkitConnection;

    this.setupConnectionListener();
  }

  private setupConnectionListener(): void {
    if (this.connection) {
      this.connection.addEventListener("change", () => {
        console.log("Connection changed:", this.getConnectionType());
      });
    }
  }

  // Get current connection type
  getConnectionType(): string {
    if (!this.connection) return "4g";
    return this.connection.effectiveType || "4g";
  }

  // Get optimal quality for current connection
  getOptimalQuality(): number {
    const connectionType = this.getConnectionType();
    const saveData = this.connection?.saveData || false;

    // Data saver mode: use lowest quality
    if (saveData) {
      return 50;
    }

    // Find matching quality level
    const level = this.qualityLevels.find((l) =>
      l.conditions.includes(connectionType),
    );

    return level?.quality || 85;
  }

  // Get optimal format for current connection
  getOptimalFormat(): "avif" | "webp" | "jpeg" {
    const connectionType = this.getConnectionType();
    const saveData = this.connection?.saveData || false;

    if (saveData) {
      return "jpeg"; // Faster decode
    }

    switch (connectionType) {
      case "slow-2g":
      case "2g":
        return "jpeg"; // Smaller files, simpler decode
      case "3g":
        return "webp"; // Good balance
      case "4g":
      default:
        return "avif"; // Best compression
    }
  }

  // Should load high-res images?
  shouldLoadHighRes(): boolean {
    const connectionType = this.getConnectionType();
    const saveData = this.connection?.saveData || false;

    return !saveData && ["4g", "wifi"].includes(connectionType);
  }

  // Get optimal image size multiplier
  getSizeMultiplier(): number {
    const connectionType = this.getConnectionType();
    const saveData = this.connection?.saveData || false;

    if (saveData) return 0.5;

    switch (connectionType) {
      case "slow-2g":
      case "2g":
        return 0.5;
      case "3g":
        return 0.75;
      case "4g":
      default:
        return 1.0;
    }
  }
}

interface QualityLevel {
  name: string;
  quality: number;
  conditions: string[];
}

// Usage
const adaptiveQuality = new AdaptiveImageQuality();

function loadAdaptiveImage(
  img: HTMLImageElement,
  basePath: string,
  baseWidth: number,
): void {
  const quality = adaptiveQuality.getOptimalQuality();
  const format = adaptiveQuality.getOptimalFormat();
  const sizeMultiplier = adaptiveQuality.getSizeMultiplier();
  const width = Math.round(baseWidth * sizeMultiplier);

  const url = `${basePath}?w=${width}&q=${quality}&fm=${format}`;

  img.src = url;
  console.log(`Loading: ${width}px @ ${quality}% quality (${format})`);
}

// Adaptive srcset generation
class AdaptiveSrcSet {
  private adaptiveQuality = new AdaptiveImageQuality();

  generate(basePath: string, widths: number[]): string {
    const quality = this.adaptiveQuality.getOptimalQuality();
    const format = this.adaptiveQuality.getOptimalFormat();
    const sizeMultiplier = this.adaptiveQuality.getSizeMultiplier();

    return widths
      .map((w) => {
        const adjustedWidth = Math.round(w * sizeMultiplier);
        const url = `${basePath}?w=${adjustedWidth}&q=${quality}&fm=${format}`;
        return `${url} ${adjustedWidth}w`;
      })
      .join(", ");
  }
}

const adaptiveSrcSet = new AdaptiveSrcSet();
const srcset = adaptiveSrcSet.generate("/image.jpg", [640, 960, 1280, 1920]);
```

### Example 8: Image Preloading Strategy

```typescript
// Strategic image preloading
class ImagePreloader {
  private preloadQueue: string[] = [];
  private preloaded = new Set<string>();
  private loading = new Set<string>();
  private maxConcurrent = 2;

  // Preload critical images
  async preload(
    urls: string[],
    priority: "high" | "low" = "low",
  ): Promise<void> {
    // Add to queue
    this.preloadQueue.push(...urls.filter((url) => !this.preloaded.has(url)));

    // Start loading
    await this.processQueue(priority);
  }

  private async processQueue(priority: "high" | "low"): Promise<void> {
    const concurrent =
      priority === "high" ? this.maxConcurrent * 2 : this.maxConcurrent;

    while (this.loading.size < concurrent && this.preloadQueue.length > 0) {
      const url = this.preloadQueue.shift()!;

      if (!this.loading.has(url) && !this.preloaded.has(url)) {
        this.loading.add(url);
        this.loadImage(url, priority)
          .then(() => {
            this.preloaded.add(url);
            this.loading.delete(url);
            this.processQueue(priority);
          })
          .catch((err) => {
            console.error(`Failed to preload: ${url}`, err);
            this.loading.delete(url);
          });
      }
    }
  }

  private loadImage(url: string, priority: "high" | "low"): Promise<void> {
    return new Promise((resolve, reject) => {
      const link = document.createElement("link");
      link.rel = "preload";
      link.as = "image";
      link.href = url;

      if (priority === "high") {
        link.setAttribute("importance", "high");
      }

      link.onload = () => resolve();
      link.onerror = () => reject(new Error(`Failed to load: ${url}`));

      document.head.appendChild(link);
    });
  }

  // Preload next page images
  async preloadNextPage(urls: string[]): Promise<void> {
    // Use requestIdleCallback for low-priority preloading
    if ("requestIdleCallback" in window) {
      requestIdleCallback(() => {
        this.preload(urls, "low");
      });
    } else {
      setTimeout(() => {
        this.preload(urls, "low");
      }, 100);
    }
  }

  // Get preload statistics
  getStats(): { preloaded: number; loading: number; queued: number } {
    return {
      preloaded: this.preloaded.size,
      loading: this.loading.size,
      queued: this.preloadQueue.length,
    };
  }

  // Clear preload cache
  clear(): void {
    this.preloadQueue = [];
    this.preloaded.clear();
  }
}

// Usage
const preloader = new ImagePreloader();

// Preload hero images immediately
preloader.preload(["/hero-1.webp", "/hero-2.webp", "/hero-3.webp"], "high");

// Preload next section images
preloader.preloadNextPage([
  "/product-1.webp",
  "/product-2.webp",
  "/product-3.webp",
]);

// On route change, preload new page images
function onRouteChange(newRoute: string): void {
  const nextPageImages = getImagesForRoute(newRoute);
  preloader.preloadNextPage(nextPageImages);
}

function getImagesForRoute(route: string): string[] {
  // Get images for route
  return [];
}

// Hover intent preloading
class HoverPreloader {
  private preloader = new ImagePreloader();
  private hoverTimers = new Map<HTMLElement, number>();
  private readonly HOVER_DELAY = 100; // ms

  setupHoverPreload(links: NodeListOf<HTMLAnchorElement>): void {
    links.forEach((link) => {
      link.addEventListener("mouseenter", () => {
        this.onHoverStart(link);
      });

      link.addEventListener("mouseleave", () => {
        this.onHoverEnd(link);
      });
    });
  }

  private onHoverStart(link: HTMLAnchorElement): void {
    const timer = window.setTimeout(() => {
      const url = link.href;
      const images = this.extractImagesFromPage(url);
      this.preloader.preloadNextPage(images);
    }, this.HOVER_DELAY);

    this.hoverTimers.set(link, timer);
  }

  private onHoverEnd(link: HTMLAnchorElement): void {
    const timer = this.hoverTimers.get(link);
    if (timer) {
      clearTimeout(timer);
      this.hoverTimers.delete(link);
    }
  }

  private extractImagesFromPage(url: string): string[] {
    // Extract image URLs from page
    // This would typically involve fetching and parsing the page
    return [];
  }
}
```

### Example 9: Image Error Handling and Fallback

```typescript
// Robust image loading with fallbacks
class ImageFallbackHandler {
  private fallbackChain: Map<string, string[]> = new Map();
  private failedImages = new Set<string>();

  // Register fallback chain for image
  registerFallbacks(primarySrc: string, fallbacks: string[]): void {
    this.fallbackChain.set(primarySrc, fallbacks);
  }

  // Setup error handling for image
  setupErrorHandling(img: HTMLImageElement): void {
    const originalSrc = img.src || img.dataset.src;
    if (!originalSrc) return;

    img.addEventListener("error", () => {
      this.handleImageError(img, originalSrc);
    });
  }

  private handleImageError(img: HTMLImageElement, failedSrc: string): void {
    // Mark as failed
    this.failedImages.add(failedSrc);

    // Try next fallback
    const fallbacks = this.fallbackChain.get(failedSrc);
    if (fallbacks && fallbacks.length > 0) {
      const nextFallback = fallbacks.shift()!;
      console.log(`Trying fallback: ${nextFallback}`);

      // Register remaining fallbacks for new source
      if (fallbacks.length > 0) {
        this.fallbackChain.set(nextFallback, fallbacks);
      }

      img.src = nextFallback;
    } else {
      // No more fallbacks, show placeholder
      this.showPlaceholder(img);
    }
  }

  private showPlaceholder(img: HTMLImageElement): void {
    // Generate placeholder
    const placeholder = this.generatePlaceholder(
      img.width || 300,
      img.height || 200,
      img.alt,
    );

    img.src = placeholder;
    img.classList.add("image-error");
  }

  private generatePlaceholder(
    width: number,
    height: number,
    alt: string,
  ): string {
    const canvas = document.createElement("canvas");
    canvas.width = width;
    canvas.height = height;

    const ctx = canvas.getContext("2d");
    if (!ctx) return "";

    // Gray background
    ctx.fillStyle = "#f0f0f0";
    ctx.fillRect(0, 0, width, height);

    // Icon
    ctx.fillStyle = "#999";
    ctx.font = `${Math.min(width, height) / 4}px Arial`;
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText("🖼️", width / 2, height / 2);

    // Alt text
    if (alt) {
      ctx.font = "14px Arial";
      ctx.fillText(alt, width / 2, height / 2 + 30);
    }

    return canvas.toDataURL("image/png");
  }

  // Retry failed images
  async retryFailed(): Promise<void> {
    const failed = Array.from(this.failedImages);
    this.failedImages.clear();

    for (const src of failed) {
      try {
        await this.testImage(src);
        console.log(`Retry successful: ${src}`);
      } catch (error) {
        this.failedImages.add(src);
      }
    }
  }

  private testImage(src: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve();
      img.onerror = () => reject();
      img.src = src;
    });
  }

  // Get failed image count
  getFailedCount(): number {
    return this.failedImages.size;
  }
}

// Usage
const fallbackHandler = new ImageFallbackHandler();

// Setup fallback chain
fallbackHandler.registerFallbacks("/image.avif", [
  "/image.webp",
  "/image.jpg",
  "/placeholder.jpg",
]);

// Setup error handling for all images
document.querySelectorAll("img").forEach((img) => {
  fallbackHandler.setupErrorHandling(img as HTMLImageElement);
});

// Retry failed images after network recovers
window.addEventListener("online", () => {
  fallbackHandler.retryFailed();
});
```

### Example 10: Image Performance Monitor

```typescript
// Monitor image loading performance
class ImagePerformanceMonitor {
  private metrics: ImageMetrics[] = [];
  private observer: PerformanceObserver | null = null;

  start(): void {
    if (!("PerformanceObserver" in window)) {
      console.warn("PerformanceObserver not supported");
      return;
    }

    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === "resource" && this.isImage(entry.name)) {
          this.recordMetric(entry as PerformanceResourceTiming);
        }
      }
    });

    try {
      this.observer.observe({ entryTypes: ["resource"] });
    } catch (e) {
      console.error("Failed to start performance observer:", e);
    }
  }

  private isImage(url: string): boolean {
    return /\.(jpg|jpeg|png|gif|webp|avif|svg)(\?|$)/i.test(url);
  }

  private recordMetric(entry: PerformanceResourceTiming): void {
    const metric: ImageMetrics = {
      url: entry.name,
      size: entry.transferSize,
      duration: entry.duration,
      startTime: entry.startTime,
      protocol: entry.nextHopProtocol,
      cached: entry.transferSize === 0,
    };

    this.metrics.push(metric);

    // Alert if slow loading
    if (entry.duration > 1000) {
      console.warn(`Slow image load: ${entry.name} (${entry.duration}ms)`);
    }

    // Alert if large file
    if (entry.transferSize > 500000) {
      // 500KB
      console.warn(
        `Large image: ${entry.name} (${(entry.transferSize / 1024).toFixed(2)}KB)`,
      );
    }
  }

  // Generate performance report
  generateReport(): ImagePerformanceReport {
    if (this.metrics.length === 0) {
      return {
        totalImages: 0,
        totalSize: 0,
        avgDuration: 0,
        slowestImage: null,
        largestImage: null,
        cacheHitRate: 0,
      };
    }

    const totalSize = this.metrics.reduce((sum, m) => sum + m.size, 0);
    const totalDuration = this.metrics.reduce((sum, m) => sum + m.duration, 0);
    const cachedCount = this.metrics.filter((m) => m.cached).length;

    const slowest = this.metrics.reduce((max, m) =>
      m.duration > max.duration ? m : max,
    );

    const largest = this.metrics.reduce((max, m) =>
      m.size > max.size ? m : max,
    );

    return {
      totalImages: this.metrics.length,
      totalSize,
      avgDuration: totalDuration / this.metrics.length,
      slowestImage: slowest,
      largestImage: largest,
      cacheHitRate: (cachedCount / this.metrics.length) * 100,
    };
  }

  // Get metrics
  getMetrics(): ImageMetrics[] {
    return [...this.metrics];
  }

  // Clear metrics
  clear(): void {
    this.metrics = [];
  }

  // Stop monitoring
  stop(): void {
    if (this.observer) {
      this.observer.disconnect();
      this.observer = null;
    }
  }
}

interface ImageMetrics {
  url: string;
  size: number;
  duration: number;
  startTime: number;
  protocol: string;
  cached: boolean;
}

interface ImagePerformanceReport {
  totalImages: number;
  totalSize: number;
  avgDuration: number;
  slowestImage: ImageMetrics | null;
  largestImage: ImageMetrics | null;
  cacheHitRate: number;
}

// Usage
const perfMonitor = new ImagePerformanceMonitor();
perfMonitor.start();

// Get report after page load
window.addEventListener("load", () => {
  setTimeout(() => {
    const report = perfMonitor.generateReport();
    console.log("Image Performance Report:", report);
    console.log(`Total size: ${(report.totalSize / 1024 / 1024).toFixed(2)}MB`);
    console.log(`Avg duration: ${report.avgDuration.toFixed(2)}ms`);
    console.log(`Cache hit rate: ${report.cacheHitRate.toFixed(2)}%`);
  }, 1000);
});
```

## Real-World Usage

### E-Commerce Product Gallery

```typescript
class ProductGallery {
  private container: HTMLElement;
  private lazyLoader = new LazyImageLoader();
  private cdn = new ImageCDN("cloudinary", "https://res.cloudinary.com/demo");
  private adaptiveQuality = new AdaptiveImageQuality();

  constructor(container: HTMLElement) {
    this.container = container;
  }

  loadProducts(products: Product[]): void {
    products.forEach((product) => {
      const productEl = this.createProductElement(product);
      this.container.appendChild(productEl);
    });
  }

  private createProductElement(product: Product): HTMLElement {
    const div = document.createElement("div");
    div.className = "product-card";

    // Get optimal quality for network
    const quality = this.adaptiveQuality.getOptimalQuality();
    const format = this.adaptiveQuality.getOptimalFormat();

    // Generate responsive srcset
    const srcset = this.cdn.generateSrcSet(product.imageUrl, [320, 640, 960], {
      quality,
      format,
    });

    const img = document.createElement("img");
    img.dataset.srcset = srcset;
    img.dataset.src = this.cdn.buildURL(product.imageUrl, {
      width: 640,
      quality,
      format,
    });
    img.alt = product.name;

    // Setup lazy loading
    this.lazyLoader.observe(img, product.featured ? 10 : 5);

    div.appendChild(img);
    return div;
  }
}

interface Product {
  id: string;
  name: string;
  imageUrl: string;
  featured: boolean;
}
```

## Production Patterns

### 1. **Format Fallback Chain**

AVIF → WebP → JPEG with automatic detection

### 2. **Responsive Image Sizes**

Generate multiple sizes for different viewports

### 3. **Progressive Loading**

Blur-up technique for perceived performance

### 4. **Lazy Loading with Priority**

Load critical images first, defer others

### 5. **CDN Integration**

Use image CDN for automatic optimization

## Best Practices

1. **Use modern formats** (AVIF, WebP) with JPEG fallback
2. **Implement responsive images** with srcset and sizes
3. **Apply lazy loading** for below-fold images
4. **Set explicit dimensions** to prevent layout shift
5. **Use loading="lazy"** for native lazy loading
6. **Optimize for Core Web Vitals** (LCP, CLS)
7. **Compress images** at 80-85% quality
8. **Use CDN** for automatic format conversion
9. **Implement blur-up** for progressive loading
10. **Monitor performance** with PerformanceObserver
11. **Set appropriate cache headers**
12. **Use art direction** for different screen sizes
13. **Preload critical images** above the fold
14. **Adapt to network conditions**
15. **Provide alt text** for accessibility

## 10 Key Takeaways

1. **AVIF provides 30-50% better compression than WebP** and 50-70% better than JPEG while maintaining visual quality
2. **Lazy loading is essential** - defer off-screen images to reduce initial page weight and improve Time to Interactive
3. **Srcset and sizes enable responsive images** - browser automatically selects optimal image based on viewport and DPR
4. **Image CDNs automate optimization** - format conversion, compression, and resizing happen automatically at edge
5. **Blur-up technique improves perceived performance** - show low-quality placeholder instantly, then fade in high-quality
6. **Modern formats need fallbacks** - use picture element with multiple source elements for format fallback chain
7. **Set explicit width/height to prevent CLS** - dimensions allow browser to reserve space before image loads
8. **loading="lazy" has 95%+ browser support** - native lazy loading is performant and requires zero JavaScript
9. **Prioritize above-fold images** - use loading="eager" and preload for critical images to optimize LCP
10. **Network-adaptive loading maximizes UX** - deliver lower quality on slow connections, higher quality on fast networks
