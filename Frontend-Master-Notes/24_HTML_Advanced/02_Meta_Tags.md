# Meta Tags & SEO Optimization

## Core Concepts

Meta tags are HTML elements that provide structured metadata about a web page to browsers, search engines, and social media platforms. They don't display on the page but significantly impact SEO, social sharing, and user experience. Modern web development requires understanding essential meta tags, Open Graph Protocol, Twitter Cards, and their integration with Progressive Web Apps.

### Essential Meta Tags

Meta tags live in the `<head>` section and control how search engines index content, how browsers render pages, and how social platforms display shared links. They include character encoding, viewport settings, descriptions, keywords, and authorship information.

### Open Graph Protocol

Developed by Facebook, Open Graph Protocol standardizes how web pages appear when shared on social media. It uses meta tags with the `property` attribute prefixed with `og:` to control title, image, description, and content type.

### Twitter Cards

Twitter Cards enhance tweets with rich media attachments. They use `name="twitter:"` meta tags to define card types (summary, large image, player) and content that appears in expanded tweets.

---

## HTML/CSS/TypeScript Code Examples

### Example 1: Essential Meta Tags Foundation

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <!-- Character Encoding - MUST be first -->
    <meta charset="UTF-8" />

    <!-- Viewport for responsive design -->
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1.0, viewport-fit=cover"
    />

    <!-- IE Compatibility Mode -->
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />

    <!-- Page Title (50-60 characters for SEO) -->
    <title>Complete Meta Tags Guide | Frontend Masters 2026</title>

    <!-- Meta Description (150-160 characters) -->
    <meta
      name="description"
      content="Master essential meta tags, Open Graph Protocol, and Twitter Cards for optimal SEO and social media sharing. Complete guide with production examples."
    />

    <!-- Keywords (less important in modern SEO) -->
    <meta
      name="keywords"
      content="meta tags, SEO, Open Graph, Twitter Cards, HTML5, web development"
    />

    <!-- Author Information -->
    <meta name="author" content="Frontend Masters Team" />

    <!-- Copyright -->
    <meta name="copyright" content="© 2026 Frontend Masters" />

    <!-- Robots Meta Tag -->
    <meta
      name="robots"
      content="index, follow, max-snippet:-1, max-image-preview:large, max-video-preview:-1"
    />
  </head>
  <body>
    <!-- Content -->
  </body>
</html>
```

### Example 2: Viewport Meta Tag Variations

```html
<!-- Standard Responsive Viewport -->
<meta name="viewport" content="width=device-width, initial-scale=1.0" />

<!-- Prevent User Zoom (use carefully, affects accessibility) -->
<meta
  name="viewport"
  content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
/>

<!-- iOS Safe Area Support (iPhone X notch) -->
<meta
  name="viewport"
  content="width=device-width, initial-scale=1.0, viewport-fit=cover"
/>

<!-- Fixed Width Viewport (legacy mobile sites) -->
<meta name="viewport" content="width=1024" />

<!-- High DPI Display Optimization -->
<meta
  name="viewport"
  content="width=device-width, initial-scale=1.0, minimum-scale=1.0"
/>
```

```css
/* CSS to work with viewport-fit=cover */
body {
  padding: env(safe-area-inset-top) env(safe-area-inset-right)
    env(safe-area-inset-bottom) env(safe-area-inset-left);
}

/* iOS notch handling */
.header {
  padding-top: max(1rem, env(safe-area-inset-top));
}
```

### Example 3: Comprehensive Open Graph Protocol

```html
<head>
  <!-- Basic Open Graph Tags -->
  <meta property="og:title" content="Complete Meta Tags Guide 2026" />
  <meta property="og:type" content="article" />
  <meta
    property="og:url"
    content="https://frontendmasters.com/guides/meta-tags"
  />
  <meta
    property="og:image"
    content="https://frontendmasters.com/images/meta-guide-og.jpg"
  />
  <meta property="og:image:width" content="1200" />
  <meta property="og:image:height" content="630" />
  <meta property="og:image:alt" content="Meta Tags Guide Cover Image" />

  <!-- Additional Open Graph -->
  <meta
    property="og:description"
    content="Master essential meta tags for SEO and social sharing."
  />
  <meta property="og:site_name" content="Frontend Masters" />
  <meta property="og:locale" content="en_US" />
  <meta property="og:locale:alternate" content="es_ES" />
  <meta property="og:locale:alternate" content="fr_FR" />

  <!-- Article-Specific Open Graph -->
  <meta property="article:published_time" content="2026-02-01T09:00:00+00:00" />
  <meta property="article:modified_time" content="2026-02-01T15:30:00+00:00" />
  <meta
    property="article:author"
    content="https://frontendmasters.com/authors/john-doe"
  />
  <meta property="article:section" content="Web Development" />
  <meta property="article:tag" content="HTML" />
  <meta property="article:tag" content="SEO" />
  <meta property="article:tag" content="Meta Tags" />

  <!-- Facebook App ID (for analytics) -->
  <meta property="fb:app_id" content="123456789012345" />
</head>
```

### Example 4: Twitter Cards Implementation

```html
<head>
  <!-- Twitter Summary Card -->
  <meta name="twitter:card" content="summary" />
  <meta name="twitter:site" content="@frontendmasters" />
  <meta name="twitter:creator" content="@johndoe" />
  <meta name="twitter:title" content="Complete Meta Tags Guide 2026" />
  <meta
    name="twitter:description"
    content="Master essential meta tags for SEO and social sharing."
  />
  <meta
    name="twitter:image"
    content="https://frontendmasters.com/images/meta-guide-twitter.jpg"
  />
  <meta name="twitter:image:alt" content="Meta Tags Guide Cover" />
</head>
```

```html
<head>
  <!-- Twitter Large Image Card -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:site" content="@frontendmasters" />
  <meta name="twitter:creator" content="@johndoe" />
  <meta name="twitter:title" content="Complete Meta Tags Guide 2026" />
  <meta
    name="twitter:description"
    content="Master essential meta tags for SEO and social sharing."
  />
  <meta
    name="twitter:image"
    content="https://frontendmasters.com/images/meta-guide-large.jpg"
  />
  <meta name="twitter:image:alt" content="Meta Tags Guide Cover Image" />
</head>
```

```html
<head>
  <!-- Twitter Player Card (for video/audio) -->
  <meta name="twitter:card" content="player" />
  <meta name="twitter:site" content="@frontendmasters" />
  <meta name="twitter:title" content="Meta Tags Video Tutorial" />
  <meta name="twitter:description" content="Learn meta tags in 10 minutes" />
  <meta
    name="twitter:image"
    content="https://frontendmasters.com/videos/meta-tags-thumb.jpg"
  />
  <meta
    name="twitter:player"
    content="https://frontendmasters.com/embed/meta-tags-video"
  />
  <meta name="twitter:player:width" content="1280" />
  <meta name="twitter:player:height" content="720" />
  <meta
    name="twitter:player:stream"
    content="https://frontendmasters.com/videos/meta-tags.mp4"
  />
  <meta name="twitter:player:stream:content_type" content="video/mp4" />
</head>
```

### Example 5: Canonical URLs and Duplicate Content

```html
<head>
  <!-- Canonical URL - Tells search engines the preferred version -->
  <link rel="canonical" href="https://frontendmasters.com/guides/meta-tags" />

  <!-- Alternate Language Versions -->
  <link
    rel="alternate"
    hreflang="en"
    href="https://frontendmasters.com/guides/meta-tags"
  />
  <link
    rel="alternate"
    hreflang="es"
    href="https://frontendmasters.com/es/guides/meta-tags"
  />
  <link
    rel="alternate"
    hreflang="fr"
    href="https://frontendmasters.com/fr/guides/meta-tags"
  />
  <link
    rel="alternate"
    hreflang="x-default"
    href="https://frontendmasters.com/guides/meta-tags"
  />

  <!-- Pagination -->
  <link rel="prev" href="https://frontendmasters.com/guides/meta-tags?page=1" />
  <link rel="next" href="https://frontendmasters.com/guides/meta-tags?page=3" />
</head>
```

### Example 6: Robots Meta Tag Control

```html
<!-- Allow indexing and following links (default) -->
<meta name="robots" content="index, follow" />

<!-- Prevent indexing but allow following links -->
<meta name="robots" content="noindex, follow" />

<!-- Allow indexing but prevent following links -->
<meta name="robots" content="index, nofollow" />

<!-- Prevent both indexing and following links -->
<meta name="robots" content="noindex, nofollow" />

<!-- Prevent caching -->
<meta name="robots" content="noarchive" />

<!-- Prevent showing snippets in search results -->
<meta name="robots" content="nosnippet" />

<!-- Prevent image indexing -->
<meta name="robots" content="noimageindex" />

<!-- Combined robots directives -->
<meta
  name="robots"
  content="index, follow, max-snippet:-1, max-image-preview:large, max-video-preview:-1"
/>

<!-- Target specific search engines -->
<meta name="googlebot" content="index, follow" />
<meta name="bingbot" content="index, follow" />

<!-- Prevent translation -->
<meta name="google" content="notranslate" />
```

### Example 7: Theme Color and Browser Customization

```html
<head>
  <!-- Theme Color (Chrome, Firefox, Opera, Safari) -->
  <meta name="theme-color" content="#2563eb" />

  <!-- Theme Color with Media Query -->
  <meta
    name="theme-color"
    content="#2563eb"
    media="(prefers-color-scheme: light)"
  />
  <meta
    name="theme-color"
    content="#1e293b"
    media="(prefers-color-scheme: dark)"
  />

  <!-- Apple Status Bar Style -->
  <meta
    name="apple-mobile-web-app-status-bar-style"
    content="black-translucent"
  />

  <!-- Apple Web App Capable (standalone mode) -->
  <meta name="apple-mobile-web-app-capable" content="yes" />

  <!-- Apple Web App Title -->
  <meta name="apple-mobile-web-app-title" content="FM Guide" />

  <!-- Windows Tile Color -->
  <meta name="msapplication-TileColor" content="#2563eb" />
  <meta name="msapplication-TileImage" content="/icons/mstile-144x144.png" />
</head>
```

```typescript
// Dynamic theme color with TypeScript
class ThemeManager {
  private metaThemeColor: HTMLMetaElement | null;

  constructor() {
    this.metaThemeColor = document.querySelector('meta[name="theme-color"]');
  }

  setThemeColor(color: string): void {
    if (this.metaThemeColor) {
      this.metaThemeColor.setAttribute("content", color);
    }
  }

  setThemeByMode(mode: "light" | "dark"): void {
    const colors = {
      light: "#2563eb",
      dark: "#1e293b",
    };
    this.setThemeColor(colors[mode]);
  }

  detectSystemTheme(): "light" | "dark" {
    return window.matchMedia("(prefers-color-scheme: dark)").matches
      ? "dark"
      : "light";
  }

  initializeTheme(): void {
    const savedTheme = localStorage.getItem("theme") as "light" | "dark" | null;
    const theme = savedTheme || this.detectSystemTheme();
    this.setThemeByMode(theme);

    // Listen for system theme changes
    window
      .matchMedia("(prefers-color-scheme: dark)")
      .addEventListener("change", (e) => {
        if (!localStorage.getItem("theme")) {
          this.setThemeByMode(e.matches ? "dark" : "light");
        }
      });
  }
}

// Usage
const themeManager = new ThemeManager();
themeManager.initializeTheme();
```

### Example 8: Apple Touch Icons and Favicon System

```html
<head>
  <!-- Standard Favicon -->
  <link rel="icon" type="image/x-icon" href="/favicon.ico" />
  <link
    rel="icon"
    type="image/png"
    sizes="32x32"
    href="/icons/favicon-32x32.png"
  />
  <link
    rel="icon"
    type="image/png"
    sizes="16x16"
    href="/icons/favicon-16x16.png"
  />

  <!-- Apple Touch Icons -->
  <link
    rel="apple-touch-icon"
    sizes="180x180"
    href="/icons/apple-touch-icon.png"
  />
  <link
    rel="apple-touch-icon"
    sizes="152x152"
    href="/icons/apple-touch-icon-152x152.png"
  />
  <link
    rel="apple-touch-icon"
    sizes="120x120"
    href="/icons/apple-touch-icon-120x120.png"
  />
  <link
    rel="apple-touch-icon"
    sizes="76x76"
    href="/icons/apple-touch-icon-76x76.png"
  />

  <!-- Apple Touch Startup Images (splash screens) -->
  <link
    rel="apple-touch-startup-image"
    href="/icons/splash-2048x2732.png"
    media="(device-width: 1024px) and (device-height: 1366px) and (-webkit-device-pixel-ratio: 2)"
  />

  <!-- Android Chrome Icons -->
  <link
    rel="icon"
    type="image/png"
    sizes="192x192"
    href="/icons/android-chrome-192x192.png"
  />
  <link
    rel="icon"
    type="image/png"
    sizes="512x512"
    href="/icons/android-chrome-512x512.png"
  />

  <!-- Safari Pinned Tab -->
  <link rel="mask-icon" href="/icons/safari-pinned-tab.svg" color="#2563eb" />

  <!-- Microsoft Tiles -->
  <meta name="msapplication-config" content="/browserconfig.xml" />
</head>
```

```xml
<!-- browserconfig.xml -->
<?xml version="1.0" encoding="utf-8"?>
<browserconfig>
  <msapplication>
    <tile>
      <square150x150logo src="/icons/mstile-150x150.png"/>
      <square310x310logo src="/icons/mstile-310x310.png"/>
      <wide310x150logo src="/icons/mstile-310x150.png"/>
      <TileColor>#2563eb</TileColor>
    </tile>
  </msapplication>
</browserconfig>
```

### Example 9: PWA Manifest Link and Configuration

```html
<head>
  <!-- PWA Manifest Link -->
  <link rel="manifest" href="/manifest.json" />

  <!-- Fallback for older browsers -->
  <meta name="mobile-web-app-capable" content="yes" />
  <meta name="application-name" content="Frontend Masters" />
</head>
```

```json
{
  "name": "Frontend Masters Notes",
  "short_name": "FM Notes",
  "description": "Comprehensive frontend development notes and guides",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait-primary",
  "background_color": "#ffffff",
  "theme_color": "#2563eb",
  "categories": ["education", "productivity"],
  "lang": "en-US",
  "dir": "ltr",
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "shortcuts": [
    {
      "name": "JavaScript Notes",
      "short_name": "JS",
      "description": "Jump to JavaScript fundamentals",
      "url": "/javascript",
      "icons": [{ "src": "/icons/js-96x96.png", "sizes": "96x96" }]
    },
    {
      "name": "React Guides",
      "short_name": "React",
      "description": "Access React documentation",
      "url": "/react",
      "icons": [{ "src": "/icons/react-96x96.png", "sizes": "96x96" }]
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    },
    {
      "src": "/screenshots/mobile.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ]
}
```

### Example 10: Security-Related Meta Tags

```html
<head>
  <!-- Content Security Policy -->
  <meta
    http-equiv="Content-Security-Policy"
    content="default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.frontendmasters.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://api.frontendmasters.com"
  />

  <!-- Referrer Policy -->
  <meta name="referrer" content="strict-origin-when-cross-origin" />

  <!-- DNS Prefetch Control -->
  <meta http-equiv="x-dns-prefetch-control" content="on" />
  <link rel="dns-prefetch" href="https://fonts.googleapis.com" />
  <link rel="dns-prefetch" href="https://cdn.frontendmasters.com" />

  <!-- Preconnect for Performance -->
  <link rel="preconnect" href="https://fonts.googleapis.com" crossorigin />
  <link rel="preconnect" href="https://api.frontendmasters.com" crossorigin />

  <!-- Feature Policy / Permissions Policy -->
  <meta
    http-equiv="Permissions-Policy"
    content="geolocation=(), microphone=(), camera=(), payment=(), usb=()"
  />

  <!-- X-Frame-Options (prevent clickjacking) -->
  <meta http-equiv="X-Frame-Options" content="SAMEORIGIN" />

  <!-- X-Content-Type-Options -->
  <meta http-equiv="X-Content-Type-Options" content="nosniff" />
</head>
```

### Example 11: TypeScript Meta Tag Manager

```typescript
interface MetaTag {
  name?: string;
  property?: string;
  content: string;
  httpEquiv?: string;
}

interface OpenGraphData {
  title: string;
  type: string;
  url: string;
  image: string;
  description: string;
  siteName?: string;
  locale?: string;
}

interface TwitterCardData {
  card: "summary" | "summary_large_image" | "app" | "player";
  site?: string;
  creator?: string;
  title: string;
  description: string;
  image: string;
  imageAlt?: string;
}

class MetaTagManager {
  private head: HTMLHeadElement;

  constructor() {
    this.head = document.head;
  }

  private createMetaElement(tag: MetaTag): HTMLMetaElement {
    const meta = document.createElement("meta");

    if (tag.name) meta.setAttribute("name", tag.name);
    if (tag.property) meta.setAttribute("property", tag.property);
    if (tag.httpEquiv) meta.setAttribute("http-equiv", tag.httpEquiv);
    meta.setAttribute("content", tag.content);

    return meta;
  }

  private setOrUpdateMeta(selector: string, tag: MetaTag): void {
    let meta = this.head.querySelector<HTMLMetaElement>(selector);

    if (meta) {
      meta.setAttribute("content", tag.content);
    } else {
      meta = this.createMetaElement(tag);
      this.head.appendChild(meta);
    }
  }

  setTitle(title: string): void {
    document.title = title;
  }

  setDescription(description: string): void {
    this.setOrUpdateMeta('meta[name="description"]', {
      name: "description",
      content: description,
    });
  }

  setKeywords(keywords: string[]): void {
    this.setOrUpdateMeta('meta[name="keywords"]', {
      name: "keywords",
      content: keywords.join(", "),
    });
  }

  setRobots(content: string): void {
    this.setOrUpdateMeta('meta[name="robots"]', { name: "robots", content });
  }

  setCanonical(url: string): void {
    let link = this.head.querySelector<HTMLLinkElement>(
      'link[rel="canonical"]',
    );

    if (link) {
      link.setAttribute("href", url);
    } else {
      link = document.createElement("link");
      link.setAttribute("rel", "canonical");
      link.setAttribute("href", url);
      this.head.appendChild(link);
    }
  }

  setOpenGraph(data: OpenGraphData): void {
    const ogTags: MetaTag[] = [
      { property: "og:title", content: data.title },
      { property: "og:type", content: data.type },
      { property: "og:url", content: data.url },
      { property: "og:image", content: data.image },
      { property: "og:description", content: data.description },
    ];

    if (data.siteName) {
      ogTags.push({ property: "og:site_name", content: data.siteName });
    }

    if (data.locale) {
      ogTags.push({ property: "og:locale", content: data.locale });
    }

    ogTags.forEach((tag) => {
      this.setOrUpdateMeta(`meta[property="${tag.property}"]`, tag);
    });
  }

  setTwitterCard(data: TwitterCardData): void {
    const twitterTags: MetaTag[] = [
      { name: "twitter:card", content: data.card },
      { name: "twitter:title", content: data.title },
      { name: "twitter:description", content: data.description },
      { name: "twitter:image", content: data.image },
    ];

    if (data.site) {
      twitterTags.push({ name: "twitter:site", content: data.site });
    }

    if (data.creator) {
      twitterTags.push({ name: "twitter:creator", content: data.creator });
    }

    if (data.imageAlt) {
      twitterTags.push({ name: "twitter:image:alt", content: data.imageAlt });
    }

    twitterTags.forEach((tag) => {
      this.setOrUpdateMeta(`meta[name="${tag.name}"]`, tag);
    });
  }

  setThemeColor(color: string, media?: string): void {
    const selector = media
      ? `meta[name="theme-color"][media="${media}"]`
      : 'meta[name="theme-color"]:not([media])';

    let meta = this.head.querySelector<HTMLMetaElement>(selector);

    if (meta) {
      meta.setAttribute("content", color);
    } else {
      meta = document.createElement("meta");
      meta.setAttribute("name", "theme-color");
      meta.setAttribute("content", color);
      if (media) meta.setAttribute("media", media);
      this.head.appendChild(meta);
    }
  }

  updatePageMeta(pageData: {
    title: string;
    description: string;
    url: string;
    image: string;
    keywords?: string[];
  }): void {
    this.setTitle(pageData.title);
    this.setDescription(pageData.description);
    this.setCanonical(pageData.url);

    if (pageData.keywords) {
      this.setKeywords(pageData.keywords);
    }

    this.setOpenGraph({
      title: pageData.title,
      type: "website",
      url: pageData.url,
      image: pageData.image,
      description: pageData.description,
      siteName: "Frontend Masters",
    });

    this.setTwitterCard({
      card: "summary_large_image",
      title: pageData.title,
      description: pageData.description,
      image: pageData.image,
      site: "@frontendmasters",
    });
  }

  removeAllMetaTags(selector: string): void {
    this.head.querySelectorAll(selector).forEach((el) => el.remove());
  }
}

// Usage Example
const metaManager = new MetaTagManager();

// Update meta tags for a specific page
metaManager.updatePageMeta({
  title: "Complete Meta Tags Guide | Frontend Masters",
  description:
    "Learn how to implement essential meta tags, Open Graph, and Twitter Cards for optimal SEO.",
  url: "https://frontendmasters.com/guides/meta-tags",
  image: "https://frontendmasters.com/images/meta-guide-og.jpg",
  keywords: ["meta tags", "SEO", "Open Graph", "Twitter Cards"],
});

// Set theme color with media query support
metaManager.setThemeColor("#2563eb", "(prefers-color-scheme: light)");
metaManager.setThemeColor("#1e293b", "(prefers-color-scheme: dark)");
```

### Example 12: E-commerce Product Page Meta Tags

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Basic Meta -->
    <title>Premium Wireless Headphones - Noise Cancelling | TechStore</title>
    <meta
      name="description"
      content="Premium wireless headphones with active noise cancellation, 30-hour battery life, and superior sound quality. Free shipping on orders over $50."
    />
    <meta
      name="keywords"
      content="wireless headphones, noise cancelling, bluetooth headphones, premium audio"
    />

    <!-- Canonical URL -->
    <link
      rel="canonical"
      href="https://techstore.com/products/premium-headphones-nc1000"
    />

    <!-- Open Graph for Products -->
    <meta property="og:type" content="product" />
    <meta
      property="og:title"
      content="Premium Wireless Headphones - Noise Cancelling"
    />
    <meta
      property="og:description"
      content="Premium wireless headphones with active noise cancellation and 30-hour battery life."
    />
    <meta
      property="og:url"
      content="https://techstore.com/products/premium-headphones-nc1000"
    />
    <meta
      property="og:image"
      content="https://techstore.com/images/products/headphones-nc1000-main.jpg"
    />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta property="og:site_name" content="TechStore" />

    <!-- Product-specific Open Graph -->
    <meta property="product:price:amount" content="299.99" />
    <meta property="product:price:currency" content="USD" />
    <meta property="product:availability" content="in stock" />
    <meta property="product:condition" content="new" />
    <meta property="product:brand" content="AudioTech" />
    <meta property="product:retailer_item_id" content="NC1000" />

    <!-- Twitter Card -->
    <meta name="twitter:card" content="summary_large_image" />
    <meta name="twitter:site" content="@techstore" />
    <meta
      name="twitter:title"
      content="Premium Wireless Headphones - Noise Cancelling"
    />
    <meta
      name="twitter:description"
      content="Premium wireless headphones with active noise cancellation and 30-hour battery life."
    />
    <meta
      name="twitter:image"
      content="https://techstore.com/images/products/headphones-nc1000-twitter.jpg"
    />

    <!-- Additional Product Meta -->
    <meta
      property="product:plural_title"
      content="Premium Wireless Headphones"
    />
    <meta property="product:price:sale_amount" content="249.99" />
    <meta property="product:price:sale_currency" content="USD" />

    <!-- Robots -->
    <meta name="robots" content="index, follow, max-image-preview:large" />
  </head>
  <body>
    <!-- Product content -->
  </body>
</html>
```

### Example 13: Blog Article Meta Tags

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Basic Meta -->
    <title>
      10 Advanced CSS Techniques Every Developer Should Know | Dev Blog
    </title>
    <meta
      name="description"
      content="Discover 10 advanced CSS techniques including custom properties, container queries, and modern layout patterns. Complete guide with code examples."
    />
    <meta name="author" content="Jane Smith" />
    <meta
      name="keywords"
      content="CSS, web development, CSS techniques, container queries, custom properties"
    />

    <!-- Article Dates -->
    <meta
      property="article:published_time"
      content="2026-02-01T09:00:00+00:00"
    />
    <meta
      property="article:modified_time"
      content="2026-02-01T15:30:00+00:00"
    />

    <!-- Canonical URL -->
    <link
      rel="canonical"
      href="https://devblog.com/articles/advanced-css-techniques-2026"
    />

    <!-- Open Graph -->
    <meta property="og:type" content="article" />
    <meta
      property="og:title"
      content="10 Advanced CSS Techniques Every Developer Should Know"
    />
    <meta
      property="og:description"
      content="Discover 10 advanced CSS techniques including custom properties, container queries, and modern layout patterns."
    />
    <meta
      property="og:url"
      content="https://devblog.com/articles/advanced-css-techniques-2026"
    />
    <meta
      property="og:image"
      content="https://devblog.com/images/articles/css-techniques-og.jpg"
    />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta property="og:site_name" content="Dev Blog" />
    <meta property="og:locale" content="en_US" />

    <!-- Article-Specific Open Graph -->
    <meta
      property="article:author"
      content="https://devblog.com/authors/jane-smith"
    />
    <meta property="article:section" content="CSS" />
    <meta property="article:tag" content="CSS" />
    <meta property="article:tag" content="Web Development" />
    <meta property="article:tag" content="Frontend" />
    <meta property="article:tag" content="Container Queries" />

    <!-- Twitter Card -->
    <meta name="twitter:card" content="summary_large_image" />
    <meta name="twitter:site" content="@devblog" />
    <meta name="twitter:creator" content="@janesmith" />
    <meta
      name="twitter:title"
      content="10 Advanced CSS Techniques Every Developer Should Know"
    />
    <meta
      name="twitter:description"
      content="Discover 10 advanced CSS techniques including custom properties, container queries, and modern layout patterns."
    />
    <meta
      name="twitter:image"
      content="https://devblog.com/images/articles/css-techniques-twitter.jpg"
    />
    <meta
      name="twitter:image:alt"
      content="Advanced CSS Techniques Article Cover"
    />

    <!-- Additional Meta -->
    <meta name="reading-time" content="12 minutes" />
    <link rel="author" href="https://devblog.com/authors/jane-smith" />
  </head>
  <body>
    <!-- Article content -->
  </body>
</html>
```

### Example 14: Dynamic Meta Tag Updates with TypeScript (SPA)

```typescript
interface RouteMetaData {
  title: string;
  description: string;
  keywords?: string[];
  image?: string;
  type?: string;
  noIndex?: boolean;
}

class SPAMetaManager {
  private metaManager: MetaTagManager;
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.metaManager = new MetaTagManager();
    this.baseUrl = baseUrl;
  }

  updateRoute(path: string, meta: RouteMetaData): void {
    const fullUrl = `${this.baseUrl}${path}`;
    const image = meta.image || `${this.baseUrl}/images/default-og.jpg`;

    // Update title
    const fullTitle = `${meta.title} | Frontend Masters`;
    this.metaManager.setTitle(fullTitle);

    // Update description
    this.metaManager.setDescription(meta.description);

    // Update keywords
    if (meta.keywords) {
      this.metaManager.setKeywords(meta.keywords);
    }

    // Update canonical URL
    this.metaManager.setCanonical(fullUrl);

    // Update robots
    if (meta.noIndex) {
      this.metaManager.setRobots("noindex, nofollow");
    } else {
      this.metaManager.setRobots("index, follow");
    }

    // Update Open Graph
    this.metaManager.setOpenGraph({
      title: fullTitle,
      type: meta.type || "website",
      url: fullUrl,
      image: image,
      description: meta.description,
      siteName: "Frontend Masters",
      locale: "en_US",
    });

    // Update Twitter Card
    this.metaManager.setTwitterCard({
      card: "summary_large_image",
      site: "@frontendmasters",
      title: fullTitle,
      description: meta.description,
      image: image,
    });

    // Update history state for browser back button
    if (window.history.state?.url !== fullUrl) {
      window.history.replaceState({ url: fullUrl }, fullTitle, path);
    }
  }

  handleRouteChange(router: { path: string; meta: RouteMetaData }): void {
    this.updateRoute(router.path, router.meta);
  }
}

// Route configuration with meta data
interface Route {
  path: string;
  meta: RouteMetaData;
}

const routes: Route[] = [
  {
    path: "/",
    meta: {
      title: "Home",
      description:
        "Frontend Masters - Comprehensive web development notes and guides",
      keywords: [
        "frontend",
        "web development",
        "javascript",
        "react",
        "typescript",
      ],
    },
  },
  {
    path: "/javascript/fundamentals",
    meta: {
      title: "JavaScript Fundamentals",
      description:
        "Master JavaScript fundamentals including variables, functions, scope, and closures",
      keywords: ["javascript", "fundamentals", "es6", "programming"],
      type: "article",
    },
  },
  {
    path: "/react/hooks",
    meta: {
      title: "React Hooks Deep Dive",
      description:
        "Complete guide to React Hooks including useState, useEffect, useContext, and custom hooks",
      keywords: ["react", "hooks", "useState", "useEffect", "frontend"],
      type: "article",
    },
  },
  {
    path: "/admin/settings",
    meta: {
      title: "Admin Settings",
      description: "Admin panel settings",
      noIndex: true,
    },
  },
];

// Usage with a router
const spaMetaManager = new SPAMetaManager("https://frontendmasters.com");

// Simulate route change
function navigateTo(path: string): void {
  const route = routes.find((r) => r.path === path);
  if (route) {
    spaMetaManager.handleRouteChange(route);
  }
}

// Example: Navigate to React Hooks page
navigateTo("/react/hooks");
```

### Example 15: Meta Tags Testing and Validation

```typescript
interface MetaTagValidation {
  isValid: boolean;
  errors: string[];
  warnings: string[];
}

class MetaTagValidator {
  validateTitle(): MetaTagValidation {
    const result: MetaTagValidation = {
      isValid: true,
      errors: [],
      warnings: [],
    };
    const title = document.title;

    if (!title) {
      result.isValid = false;
      result.errors.push("Title is missing");
    } else {
      if (title.length < 30) {
        result.warnings.push("Title is too short (< 30 characters)");
      }
      if (title.length > 60) {
        result.warnings.push("Title is too long (> 60 characters)");
      }
    }

    return result;
  }

  validateDescription(): MetaTagValidation {
    const result: MetaTagValidation = {
      isValid: true,
      errors: [],
      warnings: [],
    };
    const meta = document.querySelector<HTMLMetaElement>(
      'meta[name="description"]',
    );

    if (!meta) {
      result.isValid = false;
      result.errors.push("Meta description is missing");
    } else {
      const content = meta.content;
      if (content.length < 120) {
        result.warnings.push("Description is too short (< 120 characters)");
      }
      if (content.length > 160) {
        result.warnings.push("Description is too long (> 160 characters)");
      }
    }

    return result;
  }

  validateCanonical(): MetaTagValidation {
    const result: MetaTagValidation = {
      isValid: true,
      errors: [],
      warnings: [],
    };
    const canonical = document.querySelector<HTMLLinkElement>(
      'link[rel="canonical"]',
    );

    if (!canonical) {
      result.warnings.push("Canonical URL is missing");
    } else {
      const href = canonical.href;
      if (!href.startsWith("http://") && !href.startsWith("https://")) {
        result.errors.push("Canonical URL is not absolute");
        result.isValid = false;
      }
    }

    return result;
  }

  validateOpenGraph(): MetaTagValidation {
    const result: MetaTagValidation = {
      isValid: true,
      errors: [],
      warnings: [],
    };
    const requiredOgTags = ["og:title", "og:type", "og:url", "og:image"];

    requiredOgTags.forEach((property) => {
      const meta = document.querySelector<HTMLMetaElement>(
        `meta[property="${property}"]`,
      );
      if (!meta) {
        result.errors.push(`Missing Open Graph tag: ${property}`);
        result.isValid = false;
      }
    });

    // Validate og:image dimensions
    const ogImage = document.querySelector<HTMLMetaElement>(
      'meta[property="og:image"]',
    );
    if (ogImage) {
      const width = document.querySelector<HTMLMetaElement>(
        'meta[property="og:image:width"]',
      );
      const height = document.querySelector<HTMLMetaElement>(
        'meta[property="og:image:height"]',
      );

      if (!width || !height) {
        result.warnings.push(
          "og:image:width and og:image:height are recommended",
        );
      }
    }

    return result;
  }

  validateTwitterCard(): MetaTagValidation {
    const result: MetaTagValidation = {
      isValid: true,
      errors: [],
      warnings: [],
    };
    const requiredTwitterTags = [
      "twitter:card",
      "twitter:title",
      "twitter:description",
    ];

    requiredTwitterTags.forEach((name) => {
      const meta = document.querySelector<HTMLMetaElement>(
        `meta[name="${name}"]`,
      );
      if (!meta) {
        result.warnings.push(`Missing Twitter Card tag: ${name}`);
      }
    });

    return result;
  }

  validateAll(): { [key: string]: MetaTagValidation } {
    return {
      title: this.validateTitle(),
      description: this.validateDescription(),
      canonical: this.validateCanonical(),
      openGraph: this.validateOpenGraph(),
      twitterCard: this.validateTwitterCard(),
    };
  }

  generateReport(): string {
    const results = this.validateAll();
    let report = "=== Meta Tags Validation Report ===\n\n";

    Object.entries(results).forEach(([category, validation]) => {
      report += `${category.toUpperCase()}:\n`;
      report += `  Status: ${validation.isValid ? "✓ Valid" : "✗ Invalid"}\n`;

      if (validation.errors.length > 0) {
        report += `  Errors:\n`;
        validation.errors.forEach((error) => (report += `    - ${error}\n`));
      }

      if (validation.warnings.length > 0) {
        report += `  Warnings:\n`;
        validation.warnings.forEach(
          (warning) => (report += `    - ${warning}\n`),
        );
      }

      report += "\n";
    });

    return report;
  }
}

// Usage
const validator = new MetaTagValidator();
console.log(validator.generateReport());

// Automated testing
function testMetaTags(): void {
  const validator = new MetaTagValidator();
  const results = validator.validateAll();

  const hasErrors = Object.values(results).some((r) => !r.isValid);

  if (hasErrors) {
    console.error("Meta tag validation failed:");
    console.log(validator.generateReport());
    throw new Error("Meta tag validation failed");
  }

  console.log("✓ All meta tags validated successfully");
}
```

---

## Real-World Usage

### Multi-language Website Meta Tags

For international websites, proper meta tag localization is crucial. Use `hreflang` attributes to specify language variations and set appropriate Open Graph locales.

```html
<head>
  <link rel="alternate" hreflang="en" href="https://example.com/en/page" />
  <link rel="alternate" hreflang="es" href="https://example.com/es/page" />
  <link rel="alternate" hreflang="fr" href="https://example.com/fr/page" />
  <link
    rel="alternate"
    hreflang="x-default"
    href="https://example.com/en/page"
  />

  <meta property="og:locale" content="en_US" />
  <meta property="og:locale:alternate" content="es_ES" />
  <meta property="og:locale:alternate" content="fr_FR" />
</head>
```

### News Article Meta Tags

News articles require specific meta tags for article discovery and proper display in news aggregators and social media.

```html
<head>
  <meta property="og:type" content="article" />
  <meta property="article:published_time" content="2026-02-01T09:00:00+00:00" />
  <meta property="article:modified_time" content="2026-02-01T15:30:00+00:00" />
  <meta property="article:author" content="John Doe" />
  <meta property="article:section" content="Technology" />
  <meta property="article:tag" content="Web Development" />
  <meta name="news_keywords" content="web development, meta tags, SEO" />
</head>
```

### Video Content Meta Tags

Video content requires special meta tags for proper embedding and sharing on social platforms.

```html
<head>
  <meta property="og:type" content="video.other" />
  <meta property="og:video" content="https://example.com/video.mp4" />
  <meta
    property="og:video:secure_url"
    content="https://example.com/video.mp4"
  />
  <meta property="og:video:type" content="video/mp4" />
  <meta property="og:video:width" content="1280" />
  <meta property="og:video:height" content="720" />
  <meta property="og:video:duration" content="600" />

  <meta name="twitter:card" content="player" />
  <meta name="twitter:player" content="https://example.com/embed/video" />
  <meta name="twitter:player:width" content="1280" />
  <meta name="twitter:player:height" content="720" />
</head>
```

---

## Production Patterns

### Meta Tag Template System

Create reusable meta tag templates for different page types:

```typescript
interface PageTemplate {
  type: "home" | "article" | "product" | "category" | "profile";
  defaults: RouteMetaData;
}

const templates: Record<PageTemplate["type"], RouteMetaData> = {
  home: {
    title: "Home",
    description: "Welcome to our website",
    type: "website",
  },
  article: {
    title: "Article",
    description: "Read our latest article",
    type: "article",
  },
  product: {
    title: "Product",
    description: "View product details",
    type: "product",
  },
  category: {
    title: "Category",
    description: "Browse our category",
    type: "website",
  },
  profile: {
    title: "Profile",
    description: "User profile page",
    type: "profile",
    noIndex: true,
  },
};

class TemplateMetaManager extends SPAMetaManager {
  applyTemplate(
    templateType: PageTemplate["type"],
    overrides: Partial<RouteMetaData>,
  ): void {
    const template = templates[templateType];
    const meta = { ...template, ...overrides };
    this.updateRoute(window.location.pathname, meta);
  }
}
```

### Server-Side Rendering (SSR) Integration

For Next.js, implement meta tags with automatic SSR support:

```typescript
// pages/_app.tsx
import Head from 'next/head';
import type { AppProps } from 'next/app';

interface MetaTagsProps {
  title?: string;
  description?: string;
  image?: string;
  url?: string;
}

export function MetaTags({ title, description, image, url }: MetaTagsProps) {
  const siteName = 'Frontend Masters';
  const defaultTitle = 'Frontend Masters - Learn Web Development';
  const defaultDescription = 'Comprehensive web development notes and guides';
  const defaultImage = 'https://frontendmasters.com/images/default-og.jpg';

  const finalTitle = title ? `${title} | ${siteName}` : defaultTitle;
  const finalUrl = url || 'https://frontendmasters.com';

  return (
    <Head>
      <title>{finalTitle}</title>
      <meta name="description" content={description || defaultDescription} />
      <link rel="canonical" href={finalUrl} />

      {/* Open Graph */}
      <meta property="og:title" content={finalTitle} />
      <meta property="og:description" content={description || defaultDescription} />
      <meta property="og:image" content={image || defaultImage} />
      <meta property="og:url" content={finalUrl} />
      <meta property="og:type" content="website" />
      <meta property="og:site_name" content={siteName} />

      {/* Twitter */}
      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={finalTitle} />
      <meta name="twitter:description" content={description || defaultDescription} />
      <meta name="twitter:image" content={image || defaultImage} />
      <meta name="twitter:site" content="@frontendmasters" />
    </Head>
  );
}
```

### Automated Meta Tag Generation from CMS

```typescript
interface CMSContent {
  id: string;
  title: string;
  excerpt: string;
  content: string;
  featuredImage: string;
  author: string;
  publishedAt: string;
  updatedAt: string;
  categories: string[];
  tags: string[];
}

class CMSMetaGenerator {
  generateArticleMeta(content: CMSContent, baseUrl: string): RouteMetaData {
    const url = `${baseUrl}/articles/${content.id}`;

    return {
      title: content.title,
      description: content.excerpt || this.generateExcerpt(content.content),
      keywords: [...content.categories, ...content.tags],
      image: content.featuredImage,
      type: "article",
    };
  }

  private generateExcerpt(content: string, maxLength: number = 160): string {
    const stripped = content.replace(/<[^>]*>/g, "");
    return stripped.length > maxLength
      ? stripped.substring(0, maxLength - 3) + "..."
      : stripped;
  }

  generateMetaHTML(content: CMSContent, baseUrl: string): string {
    const meta = this.generateArticleMeta(content, baseUrl);
    const url = `${baseUrl}/articles/${content.id}`;

    return `
      <title>${meta.title} | Frontend Masters</title>
      <meta name="description" content="${meta.description}">
      <meta name="keywords" content="${meta.keywords?.join(", ")}">
      <link rel="canonical" href="${url}">
      
      <meta property="og:title" content="${meta.title}">
      <meta property="og:description" content="${meta.description}">
      <meta property="og:image" content="${meta.image}">
      <meta property="og:url" content="${url}">
      <meta property="og:type" content="article">
      <meta property="article:published_time" content="${content.publishedAt}">
      <meta property="article:modified_time" content="${content.updatedAt}">
      <meta property="article:author" content="${content.author}">
      
      <meta name="twitter:card" content="summary_large_image">
      <meta name="twitter:title" content="${meta.title}">
      <meta name="twitter:description" content="${meta.description}">
      <meta name="twitter:image" content="${meta.image}">
    `.trim();
  }
}
```

---

## Best Practices

### 1. **Keep Titles Concise and Descriptive**

Aim for 50-60 characters. Include primary keyword early. Format: `Primary Keyword - Secondary Keyword | Brand Name`

### 2. **Optimize Meta Descriptions**

Write compelling 150-160 character descriptions. Include call-to-action. Don't duplicate title content.

### 3. **Use Absolute URLs**

Always use absolute URLs for canonical tags, Open Graph images, and Twitter Card images. This prevents broken links.

### 4. **Optimize Images for Social Sharing**

- Open Graph: 1200×630px
- Twitter Summary: 120×120px minimum
- Twitter Large Image: 1200×628px
- Always include alt text

### 5. **Implement Proper Canonical URLs**

Prevent duplicate content issues. Use self-referencing canonical tags on all pages.

### 6. **Test Meta Tags**

Use Facebook Sharing Debugger, Twitter Card Validator, and Google Rich Results Test regularly.

### 7. **Dynamic Meta Tag Updates**

For SPAs, update meta tags on route changes. Use history API for proper browser back button support.

### 8. **Mobile-First Viewport**

Always include viewport meta tag. Use `viewport-fit=cover` for devices with notches.

### 9. **Security Headers**

Implement Content Security Policy, Referrer Policy, and X-Frame-Options via meta tags or HTTP headers.

### 10. **Performance Optimization**

Use dns-prefetch and preconnect for external resources. Minimize meta tag count for faster parsing.

### 11. **Schema.org Integration**

Combine Open Graph with JSON-LD structured data for maximum SEO benefit.

### 12. **Localization Support**

Implement hreflang tags for multi-language sites. Set appropriate og:locale values.

---

## 10 Key Takeaways

1. **Essential Meta Tags Are Non-Negotiable**: Every page must have charset, viewport, title, and description meta tags for basic functionality and SEO.

2. **Open Graph Protocol Dominates Social Sharing**: Properly configured Open Graph tags control how your content appears on Facebook, LinkedIn, and other social platforms - without them, sharing looks unprofessional.

3. **Twitter Cards Enhance Tweet Engagement**: Implement Twitter Card meta tags to create rich media experiences that increase click-through rates by 2-3x.

4. **Canonical URLs Prevent SEO Penalties**: Duplicate content confuses search engines - use canonical tags to specify the authoritative version of every page.

5. **Image Optimization Matters**: Social media platforms have specific image size requirements (1200×630px for Open Graph, 1200×628px for Twitter Large Image) - wrong sizes result in cropped or pixelated previews.

6. **Robots Meta Tags Control Indexing**: Use robots meta tags strategically to prevent admin pages, duplicate content, or low-value pages from appearing in search results.

7. **PWA Manifest Links Enable App-Like Experiences**: Connecting your manifest.json via meta tags enables home screen installation, offline functionality, and native app behavior.

8. **Theme Color Creates Brand Cohesion**: The theme-color meta tag customizes browser UI to match your brand, improving user experience on mobile devices.

9. **Dynamic Meta Tag Updates Are Essential for SPAs**: Single-page applications require JavaScript-based meta tag management to update SEO and social sharing data on route changes.

10. **Testing and Validation Prevent Costly Mistakes**: Always validate meta tags using Facebook Sharing Debugger, Twitter Card Validator, and Google Rich Results Test before deployment - broken meta tags can tank social engagement.
