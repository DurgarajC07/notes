# SEO Checklist - Search Engine Optimization

## Table of Contents

- [Introduction](#introduction)
- [Technical SEO](#technical-seo)
- [On-Page SEO](#on-page-seo)
- [Meta Tags](#meta-tags)
- [Structured Data](#structured-data)
- [Performance](#performance)
- [Mobile Optimization](#mobile-optimization)
- [Content Strategy](#content-strategy)
- [Link Building](#link-building)
- [Analytics and Monitoring](#analytics-and-monitoring)
- [Automation Scripts](#automation-scripts)
- [Key Takeaways](#key-takeaways)

## Introduction

Search Engine Optimization ensures your content is discoverable and ranks well in search results. Modern SEO combines technical excellence, content quality, and user experience.

### SEO Fundamentals

```typescript
interface SEOConfig {
  title: string;
  description: string;
  keywords?: string[];
  canonical?: string;
  ogImage?: string;
  robots?: "index,follow" | "noindex,nofollow";
  structuredData?: StructuredData[];
}

const seoDefaults: SEOConfig = {
  title: "Default Title - Brand Name",
  description: "Default description (155-160 characters)",
  robots: "index,follow",
  canonical: "https://example.com",
};

// SEO best practices
const seoBestPractices = {
  titleLength: { min: 50, max: 60 }, // Characters
  descriptionLength: { min: 120, max: 160 },
  urlLength: { max: 75 },
  h1PerPage: 1,
  imageAltText: "required",
  internalLinks: { min: 2, max: 10 },
};
```

## Technical SEO

### ✅ robots.txt

```txt
# /public/robots.txt
User-agent: *
Allow: /

# Block admin pages
Disallow: /admin/
Disallow: /api/
Disallow: /private/

# Block duplicate content
Disallow: /*?sort=*
Disallow: /*?filter=*

# Allow specific bots
User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

# Sitemap location
Sitemap: https://example.com/sitemap.xml
Sitemap: https://example.com/sitemap-news.xml
```

### ✅ Sitemap Generation

```typescript
// generateSitemap.ts
import { writeFileSync } from "fs";
import { format } from "date-fns";

interface SitemapUrl {
  loc: string;
  lastmod?: string;
  changefreq?:
    | "always"
    | "hourly"
    | "daily"
    | "weekly"
    | "monthly"
    | "yearly"
    | "never";
  priority?: number;
}

function generateSitemap(urls: SitemapUrl[]): string {
  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${urls
  .map(
    (url) => `  <url>
    <loc>${escapeXml(url.loc)}</loc>
    ${url.lastmod ? `<lastmod>${url.lastmod}</lastmod>` : ""}
    ${url.changefreq ? `<changefreq>${url.changefreq}</changefreq>` : ""}
    ${url.priority ? `<priority>${url.priority}</priority>` : ""}
  </url>`,
  )
  .join("\n")}
</urlset>`;

  return xml;
}

function escapeXml(unsafe: string): string {
  return unsafe.replace(/[<>&'"]/g, (c) => {
    switch (c) {
      case "<":
        return "&lt;";
      case ">":
        return "&gt;";
      case "&":
        return "&amp;";
      case "'":
        return "&apos;";
      case '"':
        return "&quot;";
      default:
        return c;
    }
  });
}

// Dynamic sitemap generation
async function generateDynamicSitemap() {
  const baseUrl = "https://example.com";
  const today = format(new Date(), "yyyy-MM-dd");

  // Static pages
  const staticPages: SitemapUrl[] = [
    {
      loc: `${baseUrl}/`,
      lastmod: today,
      changefreq: "daily",
      priority: 1.0,
    },
    {
      loc: `${baseUrl}/about`,
      lastmod: today,
      changefreq: "monthly",
      priority: 0.8,
    },
    {
      loc: `${baseUrl}/contact`,
      lastmod: today,
      changefreq: "monthly",
      priority: 0.6,
    },
  ];

  // Dynamic pages from database
  const posts = await fetchPosts();
  const postPages: SitemapUrl[] = posts.map((post) => ({
    loc: `${baseUrl}/blog/${post.slug}`,
    lastmod: format(new Date(post.updatedAt), "yyyy-MM-dd"),
    changefreq: "weekly",
    priority: 0.7,
  }));

  const products = await fetchProducts();
  const productPages: SitemapUrl[] = products.map((product) => ({
    loc: `${baseUrl}/products/${product.slug}`,
    lastmod: format(new Date(product.updatedAt), "yyyy-MM-dd"),
    changefreq: "daily",
    priority: 0.9,
  }));

  const allPages = [...staticPages, ...postPages, ...productPages];
  const sitemap = generateSitemap(allPages);

  writeFileSync("./public/sitemap.xml", sitemap);
  console.log(`✅ Sitemap generated with ${allPages.length} URLs`);
}

// Sitemap index for large sites
function generateSitemapIndex(sitemaps: string[]): string {
  const today = format(new Date(), "yyyy-MM-dd");

  return `<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${sitemaps
  .map(
    (sitemap) => `  <sitemap>
    <loc>${sitemap}</loc>
    <lastmod>${today}</lastmod>
  </sitemap>`,
  )
  .join("\n")}
</sitemapindex>`;
}
```

### ✅ Canonical URLs

```typescript
// Prevent duplicate content
function CanonicalLink({ url }: { url: string }) {
  return (
    <Head>
      <link rel="canonical" href={url} />
    </Head>
  );
}

// Canonical URL helper
function getCanonicalUrl(pathname: string): string {
  const baseUrl = 'https://example.com';

  // Remove trailing slash
  const path = pathname.replace(/\/$/, '');

  // Remove query parameters for canonical
  return `${baseUrl}${path}`;
}

// Usage in page
export default function ProductPage() {
  const router = useRouter();
  const canonicalUrl = getCanonicalUrl(router.pathname);

  return (
    <>
      <Head>
        <link rel="canonical" href={canonicalUrl} />
      </Head>
      {/* Page content */}
    </>
  );
}
```

### ✅ URL Structure

```typescript
// Good URL patterns
const goodUrls = {
  blog: "/blog/seo-best-practices",
  product: "/products/wireless-headphones",
  category: "/categories/electronics",
  user: "/users/john-doe",
};

// Bad URL patterns to avoid
const badUrls = {
  queryParams: "/product?id=12345",
  numbers: "/post/12345",
  underscores: "/blog/seo_best_practices",
  long: "/blog/this-is-a-very-long-url-that-contains-too-many-words",
};

// Slug generator
function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, "") // Remove special characters
    .replace(/[\s_-]+/g, "-") // Replace spaces and underscores with hyphens
    .replace(/^-+|-+$/g, ""); // Remove leading/trailing hyphens
}

// Usage
const title = "SEO Best Practices for 2024!";
const slug = generateSlug(title);
// Output: 'seo-best-practices-for-2024'
```

### ✅ Redirect Handling

```typescript
// next.config.js
module.exports = {
  async redirects() {
    return [
      // Permanent redirect (301)
      {
        source: "/old-blog/:slug",
        destination: "/blog/:slug",
        permanent: true,
      },
      // Temporary redirect (302)
      {
        source: "/temp-page",
        destination: "/new-page",
        permanent: false,
      },
      // Wildcard redirects
      {
        source: "/blog/:year/:month/:slug",
        destination: "/blog/:slug",
        permanent: true,
      },
    ];
  },
};

// Custom redirect middleware
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Redirect www to non-www
  if (request.headers.get("host")?.startsWith("www.")) {
    const newUrl = request.nextUrl.clone();
    newUrl.host = newUrl.host.replace("www.", "");
    return NextResponse.redirect(newUrl, 301);
  }

  // Force HTTPS
  if (request.headers.get("x-forwarded-proto") !== "https") {
    const newUrl = request.nextUrl.clone();
    newUrl.protocol = "https:";
    return NextResponse.redirect(newUrl, 301);
  }

  return NextResponse.next();
}
```

## On-Page SEO

### ✅ Page Titles

```typescript
// SEO-optimized titles
interface PageTitleProps {
  title: string;
  siteName?: string;
  separator?: string;
}

function generatePageTitle({
  title,
  siteName = 'Brand Name',
  separator = '|'
}: PageTitleProps): string {
  // Primary keyword at the beginning
  // 50-60 characters total
  return `${title} ${separator} ${siteName}`.slice(0, 60);
}

// Usage examples
const titles = {
  homepage: generatePageTitle({ title: 'Home' }),
  product: generatePageTitle({ title: 'Wireless Headphones - Noise Cancelling' }),
  blog: generatePageTitle({ title: 'SEO Best Practices 2024' }),
  category: generatePageTitle({ title: 'Electronics' })
};

// Dynamic title component
function SEOHead({ title, description, keywords }: SEOConfig) {
  const fullTitle = generatePageTitle({ title });

  return (
    <Head>
      <title>{fullTitle}</title>
      <meta name="description" content={description} />
      {keywords && <meta name="keywords" content={keywords.join(', ')} />}
    </Head>
  );
}
```

### ✅ Meta Descriptions

```typescript
// Meta description generator
function generateMetaDescription(content: string, maxLength = 160): string {
  if (content.length <= maxLength) {
    return content;
  }

  // Truncate at word boundary
  const truncated = content.slice(0, maxLength);
  const lastSpace = truncated.lastIndexOf(" ");

  return truncated.slice(0, lastSpace) + "...";
}

// Extract description from content
function extractDescription(htmlContent: string): string {
  // Remove HTML tags
  const text = htmlContent.replace(/<[^>]*>/g, "");

  // Get first paragraph or sentence
  const firstParagraph = text.split("\n\n")[0];

  return generateMetaDescription(firstParagraph);
}

// Usage
const blogPost = {
  title: "SEO Best Practices for 2024",
  content: "<p>Learn the latest SEO techniques...</p>",
  metaDescription: extractDescription(blogPost.content),
};
```

### ✅ Heading Structure

```typescript
// Proper heading hierarchy
function ArticlePage() {
  return (
    <article>
      <h1>Main Article Title (H1)</h1>

      <section>
        <h2>Introduction (H2)</h2>
        <p>Content...</p>

        <h3>Background (H3)</h3>
        <p>Content...</p>
      </section>

      <section>
        <h2>Main Content (H2)</h2>

        <h3>Subsection 1 (H3)</h3>
        <p>Content...</p>

        <h4>Detail (H4)</h4>
        <p>Content...</p>

        <h3>Subsection 2 (H3)</h3>
        <p>Content...</p>
      </section>

      <section>
        <h2>Conclusion (H2)</h2>
        <p>Content...</p>
      </section>
    </article>
  );
}

// Heading validation
function validateHeadingStructure(): { valid: boolean; errors: string[] } {
  const headings = Array.from(document.querySelectorAll('h1, h2, h3, h4, h5, h6'));
  const errors: string[] = [];

  // Check for single H1
  const h1Count = headings.filter(h => h.tagName === 'H1').length;
  if (h1Count === 0) {
    errors.push('Missing H1 tag');
  } else if (h1Count > 1) {
    errors.push('Multiple H1 tags found');
  }

  // Check hierarchy
  const levels = headings.map(h => parseInt(h.tagName[1]));
  for (let i = 1; i < levels.length; i++) {
    if (levels[i] - levels[i - 1] > 1) {
      errors.push(`Heading hierarchy skip: H${levels[i - 1]} to H${levels[i]}`);
    }
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

### ✅ Image Optimization

```typescript
// Image SEO component
interface SEOImageProps {
  src: string;
  alt: string;
  title?: string;
  width: number;
  height: number;
}

function SEOImage({ src, alt, title, width, height }: SEOImageProps) {
  return (
    <img
      src={src}
      alt={alt}
      title={title}
      width={width}
      height={height}
      loading="lazy"
      decoding="async"
    />
  );
}

// Next.js Image with SEO
import Image from 'next/image';

function OptimizedImage({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={800}
      height={600}
      loading="lazy"
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,..."
    />
  );
}

// Generate alt text from filename
function generateAltText(filename: string): string {
  return filename
    .replace(/\.[^/.]+$/, '') // Remove extension
    .replace(/[-_]/g, ' ')    // Replace dashes/underscores
    .replace(/\b\w/g, l => l.toUpperCase()); // Capitalize
}

// Usage
const filename = 'wireless-headphones-black.jpg';
const altText = generateAltText(filename);
// Output: 'Wireless Headphones Black'
```

## Meta Tags

### ✅ Open Graph Tags

```typescript
// Open Graph component
interface OpenGraphProps {
  title: string;
  description: string;
  image: string;
  url: string;
  type?: 'website' | 'article' | 'product';
  siteName?: string;
}

function OpenGraphTags({
  title,
  description,
  image,
  url,
  type = 'website',
  siteName = 'Brand Name'
}: OpenGraphProps) {
  return (
    <Head>
      {/* Open Graph */}
      <meta property="og:type" content={type} />
      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={image} />
      <meta property="og:url" content={url} />
      <meta property="og:site_name" content={siteName} />

      {/* Twitter Card */}
      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={title} />
      <meta name="twitter:description" content={description} />
      <meta name="twitter:image" content={image} />
      <meta name="twitter:site" content="@brandname" />
      <meta name="twitter:creator" content="@authorname" />
    </Head>
  );
}

// Article-specific Open Graph
function ArticleOpenGraph({ article }: { article: Article }) {
  return (
    <Head>
      <meta property="og:type" content="article" />
      <meta property="article:published_time" content={article.publishedAt} />
      <meta property="article:modified_time" content={article.updatedAt} />
      <meta property="article:author" content={article.author} />
      <meta property="article:section" content={article.category} />
      {article.tags.map(tag => (
        <meta key={tag} property="article:tag" content={tag} />
      ))}
    </Head>
  );
}

// Product Open Graph
function ProductOpenGraph({ product }: { product: Product }) {
  return (
    <Head>
      <meta property="og:type" content="product" />
      <meta property="product:price:amount" content={product.price} />
      <meta property="product:price:currency" content="USD" />
      <meta property="product:availability" content={product.inStock ? 'in stock' : 'out of stock'} />
      <meta property="product:condition" content="new" />
    </Head>
  );
}
```

## Structured Data

### ✅ JSON-LD Schema

```typescript
// Organization schema
const organizationSchema = {
  '@context': 'https://schema.org',
  '@type': 'Organization',
  name: 'Company Name',
  url: 'https://example.com',
  logo: 'https://example.com/logo.png',
  contactPoint: {
    '@type': 'ContactPoint',
    telephone: '+1-555-123-4567',
    contactType: 'customer service',
    areaServed: 'US',
    availableLanguage: ['en', 'es']
  },
  sameAs: [
    'https://facebook.com/company',
    'https://twitter.com/company',
    'https://linkedin.com/company/company'
  ]
};

// Article schema
function generateArticleSchema(article: Article) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: article.title,
    description: article.description,
    image: article.image,
    datePublished: article.publishedAt,
    dateModified: article.updatedAt,
    author: {
      '@type': 'Person',
      name: article.author,
      url: `https://example.com/authors/${article.authorSlug}`
    },
    publisher: {
      '@type': 'Organization',
      name: 'Company Name',
      logo: {
        '@type': 'ImageObject',
        url: 'https://example.com/logo.png'
      }
    },
    mainEntityOfPage: {
      '@type': 'WebPage',
      '@id': `https://example.com/blog/${article.slug}`
    }
  };
}

// Product schema
function generateProductSchema(product: Product) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    image: product.images,
    description: product.description,
    sku: product.sku,
    mpn: product.mpn,
    brand: {
      '@type': 'Brand',
      name: product.brand
    },
    offers: {
      '@type': 'Offer',
      url: `https://example.com/products/${product.slug}`,
      priceCurrency: 'USD',
      price: product.price,
      priceValidUntil: '2024-12-31',
      availability: product.inStock
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
      seller: {
        '@type': 'Organization',
        name: 'Company Name'
      }
    },
    aggregateRating: product.rating && {
      '@type': 'AggregateRating',
      ratingValue: product.rating.average,
      reviewCount: product.rating.count
    },
    review: product.reviews?.map(review => ({
      '@type': 'Review',
      reviewRating: {
        '@type': 'Rating',
        ratingValue: review.rating,
        bestRating: 5
      },
      author: {
        '@type': 'Person',
        name: review.author
      },
      reviewBody: review.body,
      datePublished: review.publishedAt
    }))
  };
}

// Breadcrumb schema
function generateBreadcrumbSchema(breadcrumbs: Breadcrumb[]) {
  return {
    '@context': 'https://schema.org',
    '@type': 'BreadcrumbList',
    itemListElement: breadcrumbs.map((crumb, index) => ({
      '@type': 'ListItem',
      position: index + 1,
      name: crumb.label,
      item: `https://example.com${crumb.href}`
    }))
  };
}

// FAQ schema
function generateFAQSchema(faqs: FAQ[]) {
  return {
    '@context': 'https://schema.org',
    '@type': 'FAQPage',
    mainEntity: faqs.map(faq => ({
      '@type': 'Question',
      name: faq.question,
      acceptedAnswer: {
        '@type': 'Answer',
        text: faq.answer
      }
    }))
  };
}

// Structured data component
function StructuredData({ data }: { data: object }) {
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
    />
  );
}

// Usage in page
export default function ProductPage({ product }: { product: Product }) {
  const productSchema = generateProductSchema(product);
  const breadcrumbSchema = generateBreadcrumbSchema([
    { label: 'Home', href: '/' },
    { label: 'Products', href: '/products' },
    { label: product.name, href: `/products/${product.slug}` }
  ]);

  return (
    <>
      <Head>
        <StructuredData data={productSchema} />
        <StructuredData data={breadcrumbSchema} />
      </Head>
      {/* Page content */}
    </>
  );
}
```

## Performance

### ✅ Core Web Vitals

```typescript
// Measure Core Web Vitals
import { getCLS, getFID, getFCP, getLCP, getTTFB } from "web-vitals";

function sendToAnalytics(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
  });

  // Send to analytics
  navigator.sendBeacon("/api/analytics", body);
}

// Track all metrics
getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);

// Core Web Vitals thresholds
const coreWebVitals = {
  LCP: { good: 2500, needsImprovement: 4000 }, // ms
  FID: { good: 100, needsImprovement: 300 }, // ms
  CLS: { good: 0.1, needsImprovement: 0.25 }, // score
};

// Performance monitoring
function reportWebVitals(metric: NextWebVitalsMetric) {
  const { name, value, rating, id } = metric;

  console.log(`${name}: ${value} (${rating})`);

  // Send to analytics
  if (typeof window !== "undefined") {
    (window as any).gtag?.("event", name, {
      event_category: "Web Vitals",
      value: Math.round(name === "CLS" ? value * 1000 : value),
      event_label: id,
      non_interaction: true,
    });
  }
}
```

### ✅ Lazy Loading

```typescript
// Lazy load images
function LazyImage({ src, alt }: { src: string; alt: string }) {
  return (
    <img
      src={src}
      alt={alt}
      loading="lazy"
      decoding="async"
    />
  );
}

// Lazy load components
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function Page() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}

// Intersection Observer for custom lazy loading
function useLazyLoad(ref: RefObject<HTMLElement>) {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { rootMargin: '50px' }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, [ref]);

  return isVisible;
}
```

## Mobile Optimization

### ✅ Mobile-First Design

```typescript
// Responsive meta tags
<Head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
</Head>

// Mobile-friendly tap targets
const styles = {
  button: {
    minHeight: '44px',
    minWidth: '44px',
    padding: '12px 24px'
  }
};

// Responsive images
function ResponsiveImage({ src, alt }: { src: string; alt: string }) {
  return (
    <picture>
      <source
        media="(max-width: 640px)"
        srcSet={`${src}-small.jpg 1x, ${src}-small@2x.jpg 2x`}
      />
      <source
        media="(max-width: 1024px)"
        srcSet={`${src}-medium.jpg 1x, ${src}-medium@2x.jpg 2x`}
      />
      <img
        src={`${src}-large.jpg`}
        srcSet={`${src}-large@2x.jpg 2x`}
        alt={alt}
        loading="lazy"
      />
    </picture>
  );
}
```

## Content Strategy

### ✅ Content Optimization

```typescript
// Keyword density calculator
function calculateKeywordDensity(content: string, keyword: string): number {
  const text = content.toLowerCase();
  const keywordLower = keyword.toLowerCase();

  const wordCount = text.split(/\s+/).length;
  const keywordCount = (text.match(new RegExp(keywordLower, "g")) || []).length;

  return (keywordCount / wordCount) * 100;
}

// Readability score (Flesch Reading Ease)
function calculateReadability(content: string): number {
  const sentences = content.split(/[.!?]+/).length;
  const words = content.split(/\s+/).length;
  const syllables = countSyllables(content);

  // Flesch Reading Ease formula
  return 206.835 - 1.015 * (words / sentences) - 84.6 * (syllables / words);
}

function countSyllables(text: string): number {
  // Simplified syllable counting
  return (
    text
      .toLowerCase()
      .replace(/[^a-z]/g, "")
      .match(/[aeiouy]+/g)?.length || 0
  );
}

// Content analysis
interface ContentAnalysis {
  wordCount: number;
  readingTime: number;
  keywordDensity: number;
  readabilityScore: number;
  headingCount: number;
  imageCount: number;
  linkCount: number;
}

function analyzeContent(content: string, keyword: string): ContentAnalysis {
  const words = content.split(/\s+/).length;
  const readingSpeed = 200; // words per minute

  return {
    wordCount: words,
    readingTime: Math.ceil(words / readingSpeed),
    keywordDensity: calculateKeywordDensity(content, keyword),
    readabilityScore: calculateReadability(content),
    headingCount: (content.match(/<h[1-6]>/g) || []).length,
    imageCount: (content.match(/<img/g) || []).length,
    linkCount: (content.match(/<a href/g) || []).length,
  };
}
```

## Link Building

### ✅ Internal Linking

```typescript
// Internal link component
function InternalLink({ href, children }: { href: string; children: React.ReactNode }) {
  return (
    <Link href={href} prefetch>
      {children}
    </Link>
  );
}

// Related posts
function RelatedPosts({ currentPost, category }: { currentPost: Post; category: string }) {
  const relatedPosts = posts
    .filter(post =>
      post.id !== currentPost.id &&
      post.category === category
    )
    .slice(0, 3);

  return (
    <aside>
      <h2>Related Articles</h2>
      <ul>
        {relatedPosts.map(post => (
          <li key={post.id}>
            <InternalLink href={`/blog/${post.slug}`}>
              {post.title}
            </InternalLink>
          </li>
        ))}
      </ul>
    </aside>
  );
}

// Contextual internal links
function addInternalLinks(content: string, links: { keyword: string; url: string }[]): string {
  let result = content;

  links.forEach(({ keyword, url }) => {
    const regex = new RegExp(`\\b${keyword}\\b`, 'i');
    result = result.replace(regex, `<a href="${url}">${keyword}</a>`);
  });

  return result;
}
```

### ✅ External Links

```typescript
// External link component
function ExternalLink({ href, children }: { href: string; children: React.ReactNode }) {
  return (
    <a
      href={href}
      target="_blank"
      rel="noopener noreferrer nofollow"
    >
      {children}
    </a>
  );
}

// Determine if link is external
function isExternalLink(href: string): boolean {
  return href.startsWith('http') && !href.includes(window.location.hostname);
}

// Smart link component
function SmartLink({ href, children }: { href: string; children: React.ReactNode }) {
  if (isExternalLink(href)) {
    return (
      <a href={href} target="_blank" rel="noopener noreferrer">
        {children}
      </a>
    );
  }

  return <Link href={href}>{children}</Link>;
}
```

## Analytics and Monitoring

### ✅ SEO Monitoring

```typescript
// Track SEO metrics
interface SEOMetrics {
  pageViews: number;
  uniqueVisitors: number;
  bounceRate: number;
  avgTimeOnPage: number;
  searchImpressions: number;
  searchClicks: number;
  avgPosition: number;
  ctr: number;
}

// Google Search Console API
async function fetchSearchConsoleData(
  startDate: string,
  endDate: string
): Promise<SEOMetrics> {
  const response = await fetch('/api/search-console', {
    method: 'POST',
    body: JSON.stringify({
      startDate,
      endDate,
      dimensions: ['page', 'query']
    })
  });

  const data = await response.json();

  return {
    searchImpressions: data.rows.reduce((sum, row) => sum + row.impressions, 0),
    searchClicks: data.rows.reduce((sum, row) => sum + row.clicks, 0),
    avgPosition: data.rows.reduce((sum, row) => sum + row.position, 0) / data.rows.length,
    ctr: data.rows.reduce((sum, row) => sum + row.ctr, 0) / data.rows.length
  };
}

// SEO Dashboard
function SEODashboard() {
  const [metrics, setMetrics] = useState<SEOMetrics | null>(null);

  useEffect(() => {
    const fetchMetrics = async () => {
      const data = await fetchSearchConsoleData('2024-01-01', '2024-01-31');
      setMetrics(data);
    };

    fetchMetrics();
  }, []);

  if (!metrics) return <div>Loading...</div>;

  return (
    <div className="seo-dashboard">
      <MetricCard title="Impressions" value={metrics.searchImpressions} />
      <MetricCard title="Clicks" value={metrics.searchClicks} />
      <MetricCard title="Avg Position" value={metrics.avgPosition.toFixed(1)} />
      <MetricCard title="CTR" value={`${(metrics.ctr * 100).toFixed(2)}%`} />
    </div>
  );
}
```

## Automation Scripts

### ✅ SEO Audit Script

```typescript
// seo-audit.ts
import { readFileSync, writeFileSync } from "fs";
import { glob } from "glob";
import * as cheerio from "cheerio";

interface SEOIssue {
  file: string;
  type: string;
  severity: "error" | "warning";
  message: string;
}

async function auditSEO(): Promise<SEOIssue[]> {
  const issues: SEOIssue[] = [];
  const files = await glob("src/**/*.{tsx,jsx,html}");

  for (const file of files) {
    const content = readFileSync(file, "utf-8");
    const $ = cheerio.load(content);

    // Check for missing title
    if ($("title").length === 0) {
      issues.push({
        file,
        type: "missing-title",
        severity: "error",
        message: "Missing <title> tag",
      });
    }

    // Check title length
    const title = $("title").text();
    if (title.length > 60) {
      issues.push({
        file,
        type: "title-too-long",
        severity: "warning",
        message: `Title too long (${title.length} chars): ${title}`,
      });
    }

    // Check for missing meta description
    if ($('meta[name="description"]').length === 0) {
      issues.push({
        file,
        type: "missing-description",
        severity: "error",
        message: "Missing meta description",
      });
    }

    // Check meta description length
    const description = $('meta[name="description"]').attr("content") || "";
    if (description.length > 160) {
      issues.push({
        file,
        type: "description-too-long",
        severity: "warning",
        message: `Description too long (${description.length} chars)`,
      });
    }

    // Check for multiple H1 tags
    const h1Count = $("h1").length;
    if (h1Count > 1) {
      issues.push({
        file,
        type: "multiple-h1",
        severity: "error",
        message: `Found ${h1Count} H1 tags (should be 1)`,
      });
    }

    // Check for images without alt text
    $("img").each((_, img) => {
      if (!$(img).attr("alt")) {
        issues.push({
          file,
          type: "missing-alt",
          severity: "warning",
          message: `Image missing alt text: ${$(img).attr("src")}`,
        });
      }
    });

    // Check for missing canonical
    if ($('link[rel="canonical"]').length === 0) {
      issues.push({
        file,
        type: "missing-canonical",
        severity: "warning",
        message: "Missing canonical link",
      });
    }
  }

  // Generate report
  const report = {
    timestamp: new Date().toISOString(),
    filesChecked: files.length,
    issuesFound: issues.length,
    errorCount: issues.filter((i) => i.severity === "error").length,
    warningCount: issues.filter((i) => i.severity === "warning").length,
    issues,
  };

  writeFileSync("seo-audit-report.json", JSON.stringify(report, null, 2));

  console.log(`✅ SEO Audit Complete`);
  console.log(`Files checked: ${files.length}`);
  console.log(`Errors: ${report.errorCount}`);
  console.log(`Warnings: ${report.warningCount}`);

  return issues;
}

// Run audit
auditSEO().catch(console.error);
```

## Key Takeaways

1. **Technical SEO**: Proper robots.txt, dynamic sitemap, canonical URLs, clean URL structure

2. **On-Page SEO**: Optimized titles (50-60 chars), meta descriptions (120-160 chars), proper heading hierarchy

3. **Structured Data**: Implement JSON-LD schemas (Organization, Article, Product, Breadcrumb, FAQ)

4. **Performance**: Optimize Core Web Vitals (LCP < 2.5s, FID < 100ms, CLS < 0.1)

5. **Mobile-First**: Responsive design, mobile-friendly tap targets (44x44px min), fast mobile load times

6. **Content Quality**: Keyword optimization (1-2% density), readability, comprehensive content

7. **Meta Tags**: Complete Open Graph and Twitter Card implementation for social sharing

8. **Internal Linking**: Strategic internal links, contextual linking, related content

9. **Image SEO**: Descriptive alt text, optimized file sizes, responsive images, lazy loading

10. **Monitoring**: Track Search Console metrics, Core Web Vitals, automated SEO audits

---

**Pre-Launch SEO Checklist**:

```bash
✅ Technical
□ Robots.txt configured
□ Sitemap generated and submitted
□ Canonical URLs set
□ 301 redirects for old URLs
□ HTTPS enforced
□ No broken links

✅ On-Page
□ Unique title for each page (50-60 chars)
□ Unique meta description (120-160 chars)
□ One H1 per page
□ Proper heading hierarchy
□ Alt text for all images
□ Internal linking strategy

✅ Structured Data
□ Organization schema
□ Article/Product schemas
□ Breadcrumb schema
□ FAQ schema (if applicable)
□ Validate with Rich Results Test

✅ Performance
□ Core Web Vitals passing
□ Page load time < 3s
□ Images optimized
□ Code minified
□ Lazy loading implemented

✅ Mobile
□ Mobile-friendly test passes
□ Responsive design
□ Touch targets 44x44px minimum
□ No horizontal scroll

✅ Content
□ Keyword research completed
□ Content quality > 300 words
□ Readability score good
□ No duplicate content
□ Fresh, updated content

✅ Analytics
□ Google Analytics installed
□ Google Search Console verified
□ Conversion tracking setup
□ Event tracking configured

# Ready for search engines! 🚀
```
