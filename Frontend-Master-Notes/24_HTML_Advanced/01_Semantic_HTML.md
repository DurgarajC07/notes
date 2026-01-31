# Semantic HTML

## Core Concept

Semantic HTML refers to using HTML markup that conveys the meaning and structure of content rather than just its presentation. Semantic elements clearly describe their purpose to both browsers and developers, improving accessibility, SEO, maintainability, and enabling better machine parsing of web content.

### Why Semantic HTML Matters

1. **Accessibility**: Screen readers and assistive technologies rely on semantic structure
2. **SEO**: Search engines better understand content hierarchy and relevance
3. **Maintainability**: Code is self-documenting and easier to understand
4. **Future-proof**: Better compatibility with future browsers and devices
5. **Styling hooks**: Semantic elements provide natural CSS selectors

### HTML5 Semantic Element Categories

- **Sectioning**: `<article>`, `<section>`, `<nav>`, `<aside>`
- **Structural**: `<header>`, `<footer>`, `<main>`
- **Content**: `<figure>`, `<figcaption>`, `<time>`, `<address>`
- **Text-level**: `<mark>`, `<strong>`, `<em>`, `<abbr>`

## HTML/CSS/TypeScript Code Examples

### Example 1: Basic Document Structure with Semantic Elements

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Semantic HTML Document</title>
</head>
<body>
    <!-- Main header with site-wide navigation -->
    <header class="site-header">
        <h1>Company Name</h1>
        <nav aria-label="Primary navigation">
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/products">Products</a></li>
                <li><a href="/about">About</a></li>
                <li><a href="/contact">Contact</a></li>
            </ul>
        </nav>
    </header>

    <!-- Main content area -->
    <main id="main-content">
        <article class="blog-post">
            <header>
                <h2>Understanding Semantic HTML</h2>
                <p>
                    Published on <time datetime="2026-02-01">February 1, 2026</time>
                    by <address><a href="/author/john">John Doe</a></address>
                </p>
            </header>

            <section>
                <h3>Introduction</h3>
                <p>Content goes here...</p>
            </section>

            <section>
                <h3>Main Concepts</h3>
                <p>More content...</p>
            </section>

            <footer>
                <p>Tags: <a href="/tag/html">#html</a> <a href="/tag/semantic">#semantic</a></p>
            </footer>
        </article>
    </main>

    <!-- Sidebar with related content -->
    <aside class="sidebar">
        <section>
            <h3>Related Articles</h3>
            <ul>
                <li><a href="/article1">Article 1</a></li>
                <li><a href="/article2">Article 2</a></li>
            </ul>
        </section>
    </aside>

    <!-- Site footer -->
    <footer class="site-footer">
        <p>&copy; 2026 Company Name. All rights reserved.</p>
    </footer>
</body>
</html>
```

### Example 2: Article with Proper Heading Hierarchy

```html
<article class="research-paper">
  <header>
    <h1>The Impact of Semantic HTML on Web Development</h1>
    <p class="byline">
      By <span class="author">Dr. Jane Smith</span>
      <time datetime="2026-01-15" pubdate>January 15, 2026</time>
    </p>
  </header>

  <section id="abstract">
    <h2>Abstract</h2>
    <p>This paper examines the benefits of semantic HTML...</p>
  </section>

  <section id="introduction">
    <h2>1. Introduction</h2>
    <p>Semantic HTML has revolutionized...</p>

    <section>
      <h3>1.1 Background</h3>
      <p>Historical context...</p>
    </section>

    <section>
      <h3>1.2 Motivation</h3>
      <p>Why this matters...</p>
    </section>
  </section>

  <section id="methodology">
    <h2>2. Methodology</h2>
    <p>Our research approach...</p>

    <section>
      <h3>2.1 Data Collection</h3>
      <p>We collected data from...</p>

      <section>
        <h4>2.1.1 Sample Selection</h4>
        <p>Participants were chosen...</p>
      </section>
    </section>
  </section>

  <section id="results">
    <h2>3. Results</h2>
    <p>Key findings include...</p>
  </section>

  <footer>
    <h2>References</h2>
    <ol>
      <li>Smith, J. (2025). <cite>HTML Standards</cite></li>
      <li>Doe, J. (2024). <cite>Web Accessibility</cite></li>
    </ol>
  </footer>
</article>
```

### Example 3: Navigation Patterns with Semantic Elements

```html
<!-- Primary site navigation -->
<nav aria-label="Main navigation" class="main-nav">
  <ul>
    <li><a href="/" aria-current="page">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/services">Services</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- Breadcrumb navigation -->
<nav aria-label="Breadcrumb" class="breadcrumb">
  <ol>
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/products/electronics">Electronics</a></li>
    <li aria-current="page">Smartphones</li>
  </ol>
</nav>

<!-- Table of contents -->
<nav aria-label="Table of contents" class="toc">
  <h2>Contents</h2>
  <ol>
    <li><a href="#introduction">Introduction</a></li>
    <li><a href="#methodology">Methodology</a></li>
    <li><a href="#results">Results</a></li>
    <li><a href="#conclusion">Conclusion</a></li>
  </ol>
</nav>

<!-- Pagination navigation -->
<nav aria-label="Pagination" class="pagination">
  <ul>
    <li><a href="/page/1" aria-label="Previous page">Previous</a></li>
    <li><a href="/page/1">1</a></li>
    <li><a href="/page/2">2</a></li>
    <li><a href="/page/3" aria-current="page">3</a></li>
    <li><a href="/page/4">4</a></li>
    <li><a href="/page/5">5</a></li>
    <li><a href="/page/5" aria-label="Next page">Next</a></li>
  </ul>
</nav>
```

### Example 4: Section vs Article Decision Matrix

```html
<!-- Use ARTICLE when content is independently distributable -->
<article class="blog-post">
  <h2>10 Tips for Better Code</h2>
  <p>This post can be syndicated, shared, or read independently...</p>

  <!-- Sections within article for thematic grouping -->
  <section>
    <h3>Performance Tips</h3>
    <p>Optimize your code...</p>
  </section>

  <section>
    <h3>Readability Tips</h3>
    <p>Write clean code...</p>
  </section>
</article>

<!-- Use SECTION for thematic grouping within page -->
<section class="features">
  <h2>Our Features</h2>

  <!-- Articles within section when items are independently distributable -->
  <article class="feature-card">
    <h3>Feature 1</h3>
    <p>Description...</p>
  </article>

  <article class="feature-card">
    <h3>Feature 2</h3>
    <p>Description...</p>
  </article>
</section>

<!-- Nested sections for hierarchical content -->
<section class="documentation">
  <h2>Documentation</h2>

  <section class="getting-started">
    <h3>Getting Started</h3>
    <p>Introduction...</p>
  </section>

  <section class="advanced">
    <h3>Advanced Topics</h3>
    <p>Deep dive...</p>
  </section>
</section>
```

### Example 5: Aside Element Usage Patterns

```html
<main>
  <article class="main-content">
    <h1>Understanding Climate Change</h1>
    <p>Climate change is affecting our planet...</p>

    <!-- Aside tangentially related to main content -->
    <aside class="callout">
      <h3>Did You Know?</h3>
      <p>Global temperatures have risen 1.1°C since pre-industrial times.</p>
    </aside>

    <p>Scientists agree that human activities...</p>

    <!-- Aside for supplementary information -->
    <aside class="definition">
      <h4>Greenhouse Gases</h4>
      <p>
        Gases that trap heat in the atmosphere, including CO2, methane, and
        nitrous oxide.
      </p>
    </aside>
  </article>
</main>

<!-- Page-level aside for related but independent content -->
<aside class="sidebar">
  <section class="related-articles">
    <h3>Related Topics</h3>
    <ul>
      <li><a href="/renewable-energy">Renewable Energy</a></li>
      <li><a href="/carbon-footprint">Carbon Footprint</a></li>
    </ul>
  </section>

  <section class="newsletter">
    <h3>Subscribe</h3>
    <form>
      <label for="email">Email:</label>
      <input type="email" id="email" required />
      <button type="submit">Subscribe</button>
    </form>
  </section>
</aside>
```

### Example 6: Header and Footer Contexts

```html
<!-- Site-level header -->
<header class="site-header">
  <a href="/" class="logo">
    <img src="logo.svg" alt="Company Logo" />
  </a>
  <nav aria-label="Primary">
    <ul>
      <li><a href="/products">Products</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>

<main>
  <!-- Article header -->
  <article>
    <header class="article-header">
      <h1>Article Title</h1>
      <p class="meta">
        <time datetime="2026-02-01">Feb 1, 2026</time>
        by <a href="/author/jane">Jane Doe</a>
      </p>
      <p class="summary">Brief description of the article content.</p>
    </header>

    <p>Article content...</p>

    <!-- Article footer -->
    <footer class="article-footer">
      <p>
        Tags:
        <a href="/tag/html">#html</a>
        <a href="/tag/semantic">#semantic</a>
      </p>
      <p>
        Share:
        <a href="#" aria-label="Share on Twitter">Twitter</a>
        <a href="#" aria-label="Share on Facebook">Facebook</a>
      </p>
    </footer>
  </article>

  <!-- Section header and footer -->
  <section class="comments">
    <header>
      <h2>Comments (42)</h2>
      <button type="button">Sort by newest</button>
    </header>

    <article class="comment">
      <p>Great article!</p>
    </article>

    <footer>
      <button type="button">Load more comments</button>
    </footer>
  </section>
</main>

<!-- Site-level footer -->
<footer class="site-footer">
  <nav aria-label="Footer navigation">
    <ul>
      <li><a href="/privacy">Privacy</a></li>
      <li><a href="/terms">Terms</a></li>
      <li><a href="/contact">Contact</a></li>
    </ul>
  </nav>
  <p>&copy; 2026 Company Name</p>
</footer>
```

### Example 7: Main Element and Landmark Roles

```html
<!DOCTYPE html>
<html lang="en">
  <body>
    <!-- Skip to main content link for keyboard users -->
    <a href="#main" class="skip-link">Skip to main content</a>

    <!-- Header landmark (implicit role="banner") -->
    <header>
      <h1>Site Title</h1>
      <nav aria-label="Primary">
        <!-- Navigation content -->
      </nav>
    </header>

    <!-- Main landmark (implicit role="main") -->
    <!-- Only ONE main element per page -->
    <main id="main">
      <h2>Page Heading</h2>
      <p>This is the primary content of the page...</p>

      <article>
        <h3>Article Heading</h3>
        <p>Article content...</p>
      </article>
    </main>

    <!-- Complementary landmark (implicit role="complementary") -->
    <aside>
      <h2>Sidebar</h2>
      <p>Related information...</p>
    </aside>

    <!-- Footer landmark (implicit role="contentinfo") -->
    <footer>
      <p>Footer content...</p>
    </footer>
  </body>
</html>
```

### Example 8: Time Element with Datetime Attribute

```html
<!-- Absolute date -->
<p>Published on <time datetime="2026-02-01">February 1, 2026</time></p>

<!-- Date with time -->
<p>
  Event starts at <time datetime="2026-02-01T14:30:00">2:30 PM on Feb 1</time>
</p>

<!-- Date with timezone -->
<p>
  Webinar:
  <time datetime="2026-02-01T14:00:00-05:00">2 PM EST, Feb 1, 2026</time>
</p>

<!-- Duration -->
<p>Video length: <time datetime="PT2H30M">2 hours 30 minutes</time></p>

<!-- Year only -->
<p>Copyright <time datetime="2026">2026</time></p>

<!-- Relative time with machine-readable datetime -->
<article>
  <header>
    <h2>Blog Post Title</h2>
    <p>
      Posted
      <time datetime="2026-01-28" title="January 28, 2026">4 days ago</time>
    </p>
  </header>
</article>

<!-- Event scheduling -->
<div class="event">
  <h3>Annual Conference</h3>
  <p>
    From <time datetime="2026-06-15">June 15</time> to
    <time datetime="2026-06-17">June 17, 2026</time>
  </p>
</div>

<!-- Birthday with year -->
<p>Born on <time datetime="1990-05-15">May 15, 1990</time></p>

<!-- Publication date with pubdate (deprecated but shown for reference) -->
<article>
  <h2>Article Title</h2>
  <p>Published: <time datetime="2026-02-01">Feb 1, 2026</time></p>
</article>
```

### Example 9: Address Element for Contact Information

```html
<!-- Author contact information -->
<article>
  <h2>About the Author</h2>
  <address>
    <p>Written by <a href="mailto:jane@example.com">Jane Doe</a></p>
    <p>Visit her <a href="https://janedoe.com">website</a></p>
  </address>
</article>

<!-- Company address in footer -->
<footer>
  <address>
    <strong>Acme Corporation</strong><br />
    123 Business Street<br />
    Suite 456<br />
    San Francisco, CA 94105<br />
    <a href="tel:+14155551234">+1 (415) 555-1234</a><br />
    <a href="mailto:info@acme.com">info@acme.com</a>
  </address>
</footer>

<!-- Article author information -->
<article>
  <h1>Understanding Semantic HTML</h1>
  <p>Content...</p>
  <footer>
    <address>
      For questions about this article, contact
      <a href="mailto:support@example.com">our support team</a>.
    </address>
  </footer>
</article>

<!-- Multiple address contexts -->
<body>
  <header>
    <h1>Contact Us</h1>
  </header>

  <main>
    <section>
      <h2>Headquarters</h2>
      <address>
        100 Main St<br />
        New York, NY 10001<br />
        <a href="tel:+12125551234">+1 (212) 555-1234</a>
      </address>
    </section>

    <section>
      <h2>West Coast Office</h2>
      <address>
        456 Market St<br />
        San Francisco, CA 94105<br />
        <a href="tel:+14155555678">+1 (415) 555-5678</a>
      </address>
    </section>
  </main>
</body>
```

### Example 10: Figure and Figcaption Elements

```html
<!-- Basic figure with caption -->
<figure>
  <img src="chart.png" alt="Sales data for Q4 2025" />
  <figcaption>Figure 1: Quarterly sales growth showing 15% increase</figcaption>
</figure>

<!-- Code snippet as figure -->
<figure>
  <pre><code>
function greet(name) {
    return `Hello, ${name}!`;
}
    </code></pre>
  <figcaption>Listing 1: Basic greeting function in JavaScript</figcaption>
</figure>

<!-- Multiple images in single figure -->
<figure>
  <img src="photo1.jpg" alt="Mountain landscape" />
  <img src="photo2.jpg" alt="Forest trail" />
  <img src="photo3.jpg" alt="Lake view" />
  <figcaption>Figure 2: National park photo series, Summer 2026</figcaption>
</figure>

<!-- Blockquote as figure -->
<figure>
  <blockquote>
    <p>The only way to do great work is to love what you do.</p>
  </blockquote>
  <figcaption>
    — <cite>Steve Jobs</cite>, Stanford Commencement, 2005
  </figcaption>
</figure>

<!-- Video with caption -->
<figure>
  <video controls width="640" height="360">
    <source src="tutorial.mp4" type="video/mp4" />
    <track kind="captions" src="captions.vtt" srclang="en" label="English" />
  </video>
  <figcaption>Video 1: Product setup tutorial (5:30)</figcaption>
</figure>

<!-- Table as figure -->
<figure>
  <table>
    <thead>
      <tr>
        <th>Year</th>
        <th>Revenue</th>
        <th>Growth</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>2024</td>
        <td>$10M</td>
        <td>20%</td>
      </tr>
      <tr>
        <td>2025</td>
        <td>$15M</td>
        <td>50%</td>
      </tr>
    </tbody>
  </table>
  <figcaption>Table 1: Company revenue growth 2024-2025</figcaption>
</figure>

<!-- SVG diagram as figure -->
<figure>
  <svg width="200" height="100" role="img" aria-labelledby="svg-title">
    <title id="svg-title">Process flow diagram</title>
    <rect x="10" y="10" width="80" height="40" fill="blue" />
    <rect x="110" y="10" width="80" height="40" fill="green" />
  </svg>
  <figcaption>Figure 3: Two-step process workflow</figcaption>
</figure>
```

### Example 11: Document Outline Algorithm (Deprecated but Educational)

```html
<!-- Modern approach: Use h1-h6 with proper nesting -->
<body>
  <header>
    <h1>Site Name</h1>
    <nav aria-label="Primary">
      <h2 class="visually-hidden">Main Navigation</h2>
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
      </ul>
    </nav>
  </header>

  <main>
    <article>
      <h2>Main Article Title</h2>
      <p>Content...</p>

      <section>
        <h3>Section 1</h3>
        <p>Content...</p>

        <section>
          <h4>Subsection 1.1</h4>
          <p>Content...</p>
        </section>

        <section>
          <h4>Subsection 1.2</h4>
          <p>Content...</p>
        </section>
      </section>

      <section>
        <h3>Section 2</h3>
        <p>Content...</p>
      </section>
    </article>

    <article>
      <h2>Secondary Article</h2>
      <p>Content...</p>
    </article>
  </main>

  <aside>
    <h2>Sidebar</h2>
    <section>
      <h3>Related Links</h3>
      <ul>
        <li><a href="#">Link 1</a></li>
      </ul>
    </section>
  </aside>

  <footer>
    <h2 class="visually-hidden">Site Footer</h2>
    <p>&copy; 2026</p>
  </footer>
</body>
```

### Example 12: Mark Element for Highlighting

```html
<!-- Highlighting search results -->
<article>
  <h2>Search Results for "semantic"</h2>
  <p>
    <mark>Semantic</mark> HTML uses elements that clearly describe their
    meaning. Using <mark>semantic</mark> markup improves accessibility.
  </p>
</article>

<!-- Highlighting relevant text -->
<article>
  <h2>Important Notice</h2>
  <p>
    Please review the <mark>updated terms of service</mark> before continuing.
  </p>
</article>

<!-- Highlighting code differences -->
<figure>
  <pre><code>
// Old code
function add(a, b) {
    return a + b;
}

// New code with <mark>type safety</mark>
function add(a: number, b: number): number {
    return a + b;
}
    </code></pre>
</figure>

<!-- Reference highlighting -->
<article>
  <p>
    According to the study<sup><a href="#ref1">[1]</a></sup
    >, <mark>87% of users prefer semantic HTML</mark> for better accessibility.
  </p>
</article>
```

### Example 13: Strong vs B, Em vs I Elements

```html
<!-- STRONG: Semantic importance -->
<p><strong>Warning:</strong> This action cannot be undone.</p>

<p>The password must contain <strong>at least 8 characters</strong>.</p>

<!-- B: Stylistic bold without importance -->
<p>
  <b>Product Name:</b> Widget 2000<br />
  <b>Price:</b> $99.99
</p>

<!-- EM: Semantic emphasis (stress) -->
<p>You must <em>not</em> share your password with anyone.</p>

<p>I <em>love</em> semantic HTML!</p>

<!-- I: Alternative voice, technical term, foreign phrase -->
<p>The term <i>semantic</i> comes from Greek.</p>

<p>The ship <i>HMS Victory</i> was launched in 1765.</p>

<p><i lang="fr">Bonjour</i> means "hello" in French.</p>

<!-- Combining for proper meaning -->
<article>
  <h2>Safety Guidelines</h2>
  <p>
    <strong>Important:</strong> Do <em>not</em> operate machinery while tired.
  </p>
  <p><b>Note:</b> See section <i>5.2 Operating Procedures</i> for details.</p>
</article>
```

### Example 14: Abbreviation and Definition Elements

```html
<!-- Abbreviation with expansion -->
<p>
  The <abbr title="World Wide Web Consortium">W3C</abbr>
  defines web standards.
</p>

<p>
  <abbr title="HyperText Markup Language">HTML</abbr> is the foundation of web
  pages.
</p>

<!-- Definition list for terms -->
<dl>
  <dt>HTML</dt>
  <dd>
    HyperText Markup Language - the standard markup language for web pages
  </dd>

  <dt>CSS</dt>
  <dd>Cascading Style Sheets - used for styling HTML elements</dd>

  <dt>JavaScript</dt>
  <dd>A programming language for interactive web content</dd>
</dl>

<!-- Multiple definitions for one term -->
<dl>
  <dt>Semantic</dt>
  <dd>Relating to meaning in language or logic</dd>
  <dd>In HTML: elements that convey meaning about their content</dd>
</dl>

<!-- Definition with abbreviation -->
<article>
  <h2>Web Technologies</h2>
  <dl>
    <dt><abbr title="Application Programming Interface">API</abbr></dt>
    <dd>
      A set of protocols and tools for building software applications. APIs
      define how software components should interact.
    </dd>

    <dt><abbr title="Representational State Transfer">REST</abbr></dt>
    <dd>An architectural style for designing networked applications.</dd>
  </dl>
</article>
```

### Example 15: Details and Summary Elements

```html
<!-- Basic accordion -->
<details>
  <summary>What is semantic HTML?</summary>
  <p>
    Semantic HTML uses elements that clearly describe their meaning to both the
    browser and the developer, improving accessibility and SEO.
  </p>
</details>

<!-- Open by default -->
<details open>
  <summary>Important Notice</summary>
  <p>This content is visible by default.</p>
</details>

<!-- FAQ section -->
<section class="faq">
  <h2>Frequently Asked Questions</h2>

  <details>
    <summary>How do I get started?</summary>
    <p>
      Begin by replacing div elements with appropriate semantic elements like
      article, section, and nav.
    </p>
  </details>

  <details>
    <summary>What about browser support?</summary>
    <p>
      All modern browsers support semantic HTML5 elements. Use polyfills for
      older browsers if needed.
    </p>
  </details>

  <details>
    <summary>Will this affect my SEO?</summary>
    <p>
      Yes! Semantic HTML helps search engines better understand your content
      structure, potentially improving rankings.
    </p>
  </details>
</section>

<!-- Complex nested details -->
<details>
  <summary>Chapter 1: Introduction</summary>
  <p>Introduction content...</p>

  <details>
    <summary>Section 1.1: Background</summary>
    <p>Background information...</p>
  </details>

  <details>
    <summary>Section 1.2: Objectives</summary>
    <p>Project objectives...</p>
  </details>
</details>
```

### Example 16: TypeScript Interface for Semantic HTML Components

```typescript
// Type-safe semantic HTML component structure
interface SemanticArticle {
  id: string;
  title: string;
  author: {
    name: string;
    email: string;
    url?: string;
  };
  publishDate: Date;
  lastModified?: Date;
  content: HTMLContent[];
  tags: string[];
  category: string;
  metadata: ArticleMetadata;
}

interface ArticleMetadata {
  readingTime: number; // in minutes
  wordCount: number;
  language: string;
  isPublished: boolean;
}

interface HTMLContent {
  type: "section" | "aside" | "figure" | "blockquote";
  heading?: string;
  content: string | HTMLElement;
  id?: string;
}

// Semantic HTML generator
class SemanticHTMLGenerator {
  generateArticle(article: SemanticArticle): string {
    return `
            <article id="${article.id}" 
                     itemscope 
                     itemtype="https://schema.org/Article">
                <header>
                    <h1 itemprop="headline">${article.title}</h1>
                    <address>
                        <a href="mailto:${article.author.email}" 
                           itemprop="author">
                            ${article.author.name}
                        </a>
                    </address>
                    <time datetime="${article.publishDate.toISOString()}" 
                          itemprop="datePublished">
                        ${this.formatDate(article.publishDate)}
                    </time>
                </header>
                
                ${this.generateSections(article.content)}
                
                <footer>
                    <p>Tags: ${this.generateTags(article.tags)}</p>
                    <p>Reading time: ${article.metadata.readingTime} minutes</p>
                </footer>
            </article>
        `;
  }

  private generateSections(contents: HTMLContent[]): string {
    return contents
      .map((content) => {
        switch (content.type) {
          case "section":
            return `
                        <section ${content.id ? `id="${content.id}"` : ""}>
                            ${content.heading ? `<h2>${content.heading}</h2>` : ""}
                            ${content.content}
                        </section>
                    `;
          case "aside":
            return `<aside>${content.content}</aside>`;
          case "figure":
            return `<figure>${content.content}</figure>`;
          default:
            return `<div>${content.content}</div>`;
        }
      })
      .join("\n");
  }

  private generateTags(tags: string[]): string {
    return tags
      .map((tag) => `<a href="/tag/${tag}" rel="tag">#${tag}</a>`)
      .join(" ");
  }

  private formatDate(date: Date): string {
    return date.toLocaleDateString("en-US", {
      year: "numeric",
      month: "long",
      day: "numeric",
    });
  }
}
```

### Example 17: Semantic HTML Validation TypeScript Utility

```typescript
// Semantic HTML structure validator
interface ValidationRule {
  element: string;
  requiredParents?: string[];
  requiredChildren?: string[];
  requiredAttributes?: string[];
  maxOccurrences?: number;
}

class SemanticValidator {
  private rules: ValidationRule[] = [
    {
      element: "main",
      maxOccurrences: 1,
      requiredParents: ["body"],
    },
    {
      element: "article",
      requiredChildren: ["h1", "h2", "h3", "h4", "h5", "h6"],
    },
    {
      element: "nav",
      requiredAttributes: ["aria-label", "aria-labelledby"],
    },
    {
      element: "time",
      requiredAttributes: ["datetime"],
    },
  ];

  validate(html: string): ValidationResult[] {
    const parser = new DOMParser();
    const doc = parser.parseFromString(html, "text/html");
    const results: ValidationResult[] = [];

    this.rules.forEach((rule) => {
      const elements = doc.querySelectorAll(rule.element);

      // Check max occurrences
      if (rule.maxOccurrences && elements.length > rule.maxOccurrences) {
        results.push({
          element: rule.element,
          type: "error",
          message: `Too many <${rule.element}> elements. Max: ${rule.maxOccurrences}`,
        });
      }

      elements.forEach((element, index) => {
        // Check required parents
        if (rule.requiredParents) {
          const parent = element.parentElement;
          if (
            parent &&
            !rule.requiredParents.includes(parent.tagName.toLowerCase())
          ) {
            results.push({
              element: rule.element,
              type: "warning",
              message: `<${rule.element}> should be child of: ${rule.requiredParents.join(", ")}`,
            });
          }
        }

        // Check required children
        if (rule.requiredChildren) {
          const hasRequiredChild = rule.requiredChildren.some((childTag) =>
            element.querySelector(childTag),
          );
          if (!hasRequiredChild) {
            results.push({
              element: rule.element,
              type: "warning",
              message: `<${rule.element}> should contain: ${rule.requiredChildren.join(", ")}`,
            });
          }
        }

        // Check required attributes
        if (rule.requiredAttributes) {
          rule.requiredAttributes.forEach((attr) => {
            if (!element.hasAttribute(attr)) {
              results.push({
                element: rule.element,
                type: "error",
                message: `<${rule.element}> missing required attribute: ${attr}`,
              });
            }
          });
        }
      });
    });

    return results;
  }
}

interface ValidationResult {
  element: string;
  type: "error" | "warning" | "info";
  message: string;
}

// Usage
const validator = new SemanticValidator();
const results = validator.validate(document.documentElement.outerHTML);
results.forEach((result) => {
  console.log(
    `[${result.type.toUpperCase()}] ${result.element}: ${result.message}`,
  );
});
```

### Example 18: SEO-Optimized Semantic Article Component

```typescript
import React from 'react';

interface Article {
    id: string;
    title: string;
    description: string;
    content: string;
    author: Author;
    publishDate: Date;
    modifiedDate?: Date;
    imageUrl?: string;
    category: string;
    tags: string[];
}

interface Author {
    name: string;
    email: string;
    url?: string;
    imageUrl?: string;
}

const SemanticArticle: React.FC<{ article: Article }> = ({ article }) => {
    const structuredData = {
        "@context": "https://schema.org",
        "@type": "Article",
        "headline": article.title,
        "description": article.description,
        "image": article.imageUrl,
        "author": {
            "@type": "Person",
            "name": article.author.name,
            "url": article.author.url
        },
        "datePublished": article.publishDate.toISOString(),
        "dateModified": article.modifiedDate?.toISOString(),
        "publisher": {
            "@type": "Organization",
            "name": "Your Company",
            "logo": {
                "@type": "ImageObject",
                "url": "https://yoursite.com/logo.png"
            }
        }
    };

    return (
        <>
            <script
                type="application/ld+json"
                dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
            />

            <article
                itemScope
                itemType="https://schema.org/Article"
                className="semantic-article"
            >
                <header className="article-header">
                    <h1 itemProp="headline">{article.title}</h1>

                    <div className="article-meta">
                        <address itemProp="author" itemScope itemType="https://schema.org/Person">
                            {article.author.imageUrl && (
                                <img
                                    src={article.author.imageUrl}
                                    alt={article.author.name}
                                    itemProp="image"
                                />
                            )}
                            <a
                                href={article.author.url || `mailto:${article.author.email}`}
                                itemProp="url"
                            >
                                <span itemProp="name">{article.author.name}</span>
                            </a>
                        </address>

                        <time
                            dateTime={article.publishDate.toISOString()}
                            itemProp="datePublished"
                        >
                            {article.publishDate.toLocaleDateString()}
                        </time>

                        {article.modifiedDate && (
                            <time
                                dateTime={article.modifiedDate.toISOString()}
                                itemProp="dateModified"
                                className="modified-date"
                            >
                                Updated: {article.modifiedDate.toLocaleDateString()}
                            </time>
                        )}
                    </div>

                    {article.imageUrl && (
                        <figure>
                            <img
                                src={article.imageUrl}
                                alt={article.title}
                                itemProp="image"
                            />
                        </figure>
                    )}
                </header>

                <div
                    className="article-content"
                    itemProp="articleBody"
                    dangerouslySetInnerHTML={{ __html: article.content }}
                />

                <footer className="article-footer">
                    <nav aria-label="Article tags">
                        <ul className="tags">
                            {article.tags.map(tag => (
                                <li key={tag}>
                                    <a href={`/tag/${tag}`} rel="tag">
                                        #{tag}
                                    </a>
                                </li>
                            ))}
                        </ul>
                    </nav>

                    <div className="article-share">
                        <h3>Share this article</h3>
                        <button
                            onClick={() => shareArticle(article)}
                            aria-label="Share article"
                        >
                            Share
                        </button>
                    </div>
                </footer>
            </article>
        </>
    );
};

function shareArticle(article: Article) {
    if (navigator.share) {
        navigator.share({
            title: article.title,
            text: article.description,
            url: window.location.href
        });
    }
}

export default SemanticArticle;
```

### Example 19: CSS Styling for Semantic Elements

```css
/* Semantic HTML styling best practices */

/* Main layout structure */
body {
  display: grid;
  grid-template-areas:
    "header header"
    "main aside"
    "footer footer";
  grid-template-columns: 1fr 300px;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
  margin: 0;
}

header {
  grid-area: header;
}
main {
  grid-area: main;
}
aside {
  grid-area: aside;
}
footer {
  grid-area: footer;
}

/* Article styling */
article {
  max-width: 70ch;
  margin: 0 auto;
  padding: 2rem;
}

article > header {
  margin-bottom: 2rem;
  border-bottom: 2px solid #e0e0e0;
  padding-bottom: 1rem;
}

article h1 {
  font-size: 2.5rem;
  line-height: 1.2;
  margin-bottom: 0.5rem;
}

article time {
  color: #666;
  font-size: 0.9rem;
}

article address {
  font-style: normal;
  display: inline;
}

/* Section styling with visual hierarchy */
section {
  margin: 2rem 0;
}

section > h2 {
  font-size: 1.8rem;
  margin-top: 2rem;
  margin-bottom: 1rem;
  border-left: 4px solid #007bff;
  padding-left: 1rem;
}

section > h3 {
  font-size: 1.4rem;
  margin-top: 1.5rem;
}

/* Aside styling */
aside {
  background-color: #f8f9fa;
  padding: 1.5rem;
  border-left: 3px solid #007bff;
  margin: 1.5rem 0;
}

aside h3 {
  margin-top: 0;
  font-size: 1.2rem;
}

/* Navigation styling */
nav[aria-label="Primary"] {
  background-color: #333;
}

nav[aria-label="Primary"] ul {
  list-style: none;
  display: flex;
  gap: 1rem;
  margin: 0;
  padding: 1rem;
}

nav[aria-label="Primary"] a {
  color: white;
  text-decoration: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
}

nav[aria-label="Primary"] a[aria-current="page"] {
  background-color: #007bff;
  font-weight: bold;
}

/* Breadcrumb navigation */
nav[aria-label="Breadcrumb"] ol {
  list-style: none;
  display: flex;
  gap: 0.5rem;
  padding: 0.5rem 0;
  margin: 0;
}

nav[aria-label="Breadcrumb"] li:not(:last-child)::after {
  content: "›";
  margin-left: 0.5rem;
  color: #666;
}

/* Figure and figcaption */
figure {
  margin: 2rem 0;
  text-align: center;
}

figure img {
  max-width: 100%;
  height: auto;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

figcaption {
  margin-top: 0.5rem;
  font-size: 0.9rem;
  color: #666;
  font-style: italic;
}

/* Details and summary */
details {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 0.5rem;
  margin: 1rem 0;
}

summary {
  cursor: pointer;
  font-weight: bold;
  padding: 0.5rem;
  user-select: none;
}

summary:hover {
  background-color: #f0f0f0;
}

details[open] summary {
  margin-bottom: 0.5rem;
  border-bottom: 1px solid #ddd;
}

/* Mark element */
mark {
  background-color: #ffeb3b;
  padding: 0.1rem 0.2rem;
  border-radius: 2px;
}

/* Time element */
time {
  white-space: nowrap;
}

/* Address element */
address {
  font-style: normal;
  line-height: 1.6;
}

/* Skip link for accessibility */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: white;
  padding: 8px;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  body {
    grid-template-areas:
      "header"
      "main"
      "aside"
      "footer";
    grid-template-columns: 1fr;
  }

  article {
    padding: 1rem;
  }

  nav[aria-label="Primary"] ul {
    flex-direction: column;
  }
}
```

### Example 20: Semantic HTML Linting Configuration

```typescript
// ESLint configuration for semantic HTML in React/JSX
const semanticHTMLRules = {
  rules: {
    // Enforce semantic elements over divs
    "jsx-a11y/no-noninteractive-element-to-interactive-role": "error",
    "jsx-a11y/heading-has-content": "error",
    "jsx-a11y/html-has-lang": "error",

    // Custom rule: Prefer semantic elements
    "prefer-semantic-elements": {
      article: ["div.article", "div.post", "div.entry"],
      section: ["div.section", "div.group"],
      nav: ["div.nav", "div.navigation", "div.menu"],
      aside: ["div.sidebar", "div.aside"],
      header: ["div.header", "div.top"],
      footer: ["div.footer", "div.bottom"],
      main: ["div.main", "div.content"],
    },
  },
};

// TypeScript type definitions for semantic HTML checking
type SemanticElement =
  | "article"
  | "section"
  | "nav"
  | "aside"
  | "header"
  | "footer"
  | "main"
  | "figure"
  | "time"
  | "address";

interface SemanticStructure {
  element: SemanticElement;
  children?: SemanticStructure[];
  attributes: Record<string, string>;
  isRequired: boolean;
}

// HTML validator
class SemanticHTMLLinter {
  private violations: string[] = [];

  lint(html: Document): string[] {
    this.violations = [];

    // Check for single main element
    const mainElements = html.querySelectorAll("main");
    if (mainElements.length === 0) {
      this.violations.push("Missing <main> element");
    } else if (mainElements.length > 1) {
      this.violations.push("Multiple <main> elements found. Only one allowed.");
    }

    // Check article structure
    html.querySelectorAll("article").forEach((article) => {
      const heading = article.querySelector("h1, h2, h3, h4, h5, h6");
      if (!heading) {
        this.violations.push("<article> should contain a heading");
      }
    });

    // Check nav accessibility
    html.querySelectorAll("nav").forEach((nav) => {
      if (
        !nav.hasAttribute("aria-label") &&
        !nav.hasAttribute("aria-labelledby")
      ) {
        this.violations.push("<nav> should have aria-label or aria-labelledby");
      }
    });

    // Check time datetime attribute
    html.querySelectorAll("time").forEach((time) => {
      if (!time.hasAttribute("datetime")) {
        this.violations.push("<time> should have datetime attribute");
      }
    });

    // Check for divitis (excessive div usage)
    const divs = html.querySelectorAll("div");
    const semanticElements = html.querySelectorAll(
      "article, section, nav, aside, header, footer, main",
    );
    const ratio = divs.length / (semanticElements.length || 1);

    if (ratio > 5) {
      this.violations.push(
        `High div-to-semantic ratio (${ratio.toFixed(1)}:1). Consider using more semantic elements.`,
      );
    }

    return this.violations;
  }
}

export { SemanticHTMLLinter, semanticHTMLRules };
```

## Real-World Usage

### E-commerce Product Page

```html
<main>
  <article class="product" itemscope itemtype="https://schema.org/Product">
    <header>
      <h1 itemprop="name">Wireless Headphones Pro</h1>
    </header>

    <figure>
      <img
        src="headphones.jpg"
        alt="Wireless Headphones Pro"
        itemprop="image"
      />
      <figcaption>Premium noise-cancelling headphones</figcaption>
    </figure>

    <section class="product-details">
      <h2>Product Details</h2>
      <dl>
        <dt>Price</dt>
        <dd itemprop="offers" itemscope itemtype="https://schema.org/Offer">
          <span itemprop="price">$299.99</span>
          <meta itemprop="priceCurrency" content="USD" />
        </dd>

        <dt>Availability</dt>
        <dd>
          <span itemprop="availability" content="https://schema.org/InStock">
            In Stock
          </span>
        </dd>
      </dl>
    </section>

    <section class="description">
      <h2>Description</h2>
      <p itemprop="description">
        Premium wireless headphones with active noise cancellation...
      </p>
    </section>

    <aside class="specifications">
      <h3>Technical Specifications</h3>
      <dl>
        <dt>Battery Life</dt>
        <dd>30 hours</dd>
        <dt>Weight</dt>
        <dd>250g</dd>
      </dl>
    </aside>
  </article>
</main>
```

### Blog with Multiple Articles

```html
<main>
    <section class="featured">
        <h2>Featured Posts</h2>

        <article class="post-preview">
            <header>
                <h3><a href="/post/semantic-html">Understanding Semantic HTML</a></h3>
                <p class="meta">
                    <time datetime="2026-01-15">January 15, 2026</time>
                    by <address><a href="/author/jane">Jane Doe</a></address>
                </p>
            </header>
            <p>Learn how semantic HTML improves accessibility and SEO...</p>
            <a href="/post/semantic-html">Read more</a>
        </article>
    </section>

    <nav aria-label="Blog pagination">
        <a href="/page/2">Older posts</a>
    </nav>
</main>
```

### Documentation Site Navigation

```html
<body>
  <header>
    <h1>Documentation</h1>
    <nav aria-label="Primary">
      <ul>
        <li><a href="/docs">Docs</a></li>
        <li><a href="/api">API</a></li>
        <li><a href="/examples">Examples</a></li>
      </ul>
    </nav>
  </header>

  <div class="layout">
    <aside class="sidebar">
      <nav aria-label="Documentation sections">
        <h2>Table of Contents</h2>
        <ul>
          <li><a href="#getting-started">Getting Started</a></li>
          <li><a href="#core-concepts">Core Concepts</a></li>
          <li><a href="#advanced">Advanced</a></li>
        </ul>
      </nav>
    </aside>

    <main>
      <article>
        <section id="getting-started">
          <h2>Getting Started</h2>
          <p>Installation instructions...</p>
        </section>
      </article>
    </main>
  </div>
</body>
```

## Production Patterns

### Pattern 1: Component-Based Semantic Structure

```typescript
// React component with semantic HTML
interface BlogPostProps {
    post: {
        id: string;
        title: string;
        content: string;
        author: string;
        publishDate: Date;
        tags: string[];
    };
}

const BlogPost: React.FC<BlogPostProps> = ({ post }) => {
    return (
        <article className="blog-post">
            <header>
                <h2>{post.title}</h2>
                <address>By {post.author}</address>
                <time dateTime={post.publishDate.toISOString()}>
                    {post.publishDate.toLocaleDateString()}
                </time>
            </header>

            <div className="content">
                {post.content}
            </div>

            <footer>
                <nav aria-label="Post tags">
                    {post.tags.map(tag => (
                        <a key={tag} href={`/tag/${tag}`}>#{tag}</a>
                    ))}
                </nav>
            </footer>
        </article>
    );
};
```

### Pattern 2: Progressive Enhancement with Semantic HTML

```html
<!-- Works without CSS/JS, enhanced with them -->
<details class="enhanced-accordion">
  <summary>Click to expand</summary>
  <div class="content">
    <p>Content that works even if JavaScript fails</p>
  </div>
</details>

<script>
  // Progressive enhancement
  document.querySelectorAll(".enhanced-accordion").forEach((details) => {
    // Add animation or enhanced behavior
    details.addEventListener("toggle", (e) => {
      if (details.open) {
        // Smooth animation
        const content = details.querySelector(".content");
        content.style.animation = "slideDown 0.3s ease";
      }
    });
  });
</script>
```

### Pattern 3: SEO-Optimized Semantic Structure

```html
<article itemscope itemtype="https://schema.org/NewsArticle">
  <meta itemprop="datePublished" content="2026-02-01T10:00:00Z" />
  <meta itemprop="dateModified" content="2026-02-01T14:30:00Z" />

  <header>
    <h1 itemprop="headline">Breaking News Title</h1>
    <p itemprop="description">Brief description for search results</p>
  </header>

  <figure itemprop="image" itemscope itemtype="https://schema.org/ImageObject">
    <img src="image.jpg" alt="Article image" itemprop="url" />
    <meta itemprop="width" content="1200" />
    <meta itemprop="height" content="630" />
  </figure>

  <div itemprop="articleBody">
    <p>Article content...</p>
  </div>

  <footer>
    <address itemprop="author" itemscope itemtype="https://schema.org/Person">
      <span itemprop="name">John Doe</span>
    </address>
  </footer>
</article>
```

### Pattern 4: Accessible Navigation Patterns

```typescript
// TypeScript utility for managing semantic navigation
class SemanticNavigation {
  private nav: HTMLElement;
  private links: NodeListOf<HTMLAnchorElement>;

  constructor(navElement: HTMLElement) {
    this.nav = navElement;
    this.links = navElement.querySelectorAll("a");
    this.init();
  }

  private init(): void {
    this.setCurrentPage();
    this.addKeyboardSupport();
  }

  private setCurrentPage(): void {
    const currentPath = window.location.pathname;
    this.links.forEach((link) => {
      if (link.pathname === currentPath) {
        link.setAttribute("aria-current", "page");
        link.classList.add("current");
      }
    });
  }

  private addKeyboardSupport(): void {
    this.nav.addEventListener("keydown", (e: KeyboardEvent) => {
      if (e.key === "ArrowRight" || e.key === "ArrowDown") {
        e.preventDefault();
        this.focusNext();
      } else if (e.key === "ArrowLeft" || e.key === "ArrowUp") {
        e.preventDefault();
        this.focusPrevious();
      }
    });
  }

  private focusNext(): void {
    const current = document.activeElement as HTMLAnchorElement;
    const currentIndex = Array.from(this.links).indexOf(current);
    const nextIndex = (currentIndex + 1) % this.links.length;
    this.links[nextIndex].focus();
  }

  private focusPrevious(): void {
    const current = document.activeElement as HTMLAnchorElement;
    const currentIndex = Array.from(this.links).indexOf(current);
    const prevIndex =
      currentIndex - 1 < 0 ? this.links.length - 1 : currentIndex - 1;
    this.links[prevIndex].focus();
  }
}

// Usage
const mainNav = document.querySelector(
  'nav[aria-label="Primary"]',
) as HTMLElement;
if (mainNav) {
  new SemanticNavigation(mainNav);
}
```

### Pattern 5: Semantic Form Structure

```html
<form action="/submit" method="post">
  <fieldset>
    <legend>Personal Information</legend>

    <div class="form-group">
      <label for="name">Full Name</label>
      <input type="text" id="name" name="name" required />
    </div>

    <div class="form-group">
      <label for="email">Email Address</label>
      <input type="email" id="email" name="email" required />
      <small>We'll never share your email</small>
    </div>
  </fieldset>

  <fieldset>
    <legend>Preferences</legend>

    <div class="form-group">
      <label>
        <input type="checkbox" name="newsletter" value="yes" />
        Subscribe to newsletter
      </label>
    </div>
  </fieldset>

  <footer>
    <button type="submit">Submit</button>
    <button type="reset">Reset</button>
  </footer>
</form>
```

## Best Practices

### 1. Use Semantic Elements Appropriately

- **DO**: Use `<article>` for independently distributable content
- **DON'T**: Use `<article>` just for styling purposes
- **DO**: Use `<section>` for thematic grouping
- **DON'T**: Use `<section>` as a replacement for `<div>` everywhere

### 2. Maintain Proper Heading Hierarchy

```html
<!-- CORRECT -->
<h1>Main Title</h1>
<h2>Section Title</h2>
<h3>Subsection Title</h3>

<!-- INCORRECT - Skipping levels -->
<h1>Main Title</h1>
<h3>Section Title</h3>
<!-- Skipped h2 -->
```

### 3. Always Include ARIA Labels for Navigation

```html
<!-- GOOD -->
<nav aria-label="Primary navigation">...</nav>
<nav aria-label="Footer navigation">...</nav>

<!-- BAD -->
<nav>...</nav>
<nav>...</nav>
```

### 4. Use Time Element with Datetime Attribute

```html
<!-- GOOD -->
<time datetime="2026-02-01T14:30:00">February 1, 2026 at 2:30 PM</time>

<!-- BAD -->
<span>February 1, 2026 at 2:30 PM</span>
```

### 5. Single Main Element Per Page

```html
<!-- CORRECT -->
<body>
  <header>...</header>
  <main>...</main>
  <!-- Only one main -->
  <footer>...</footer>
</body>

<!-- INCORRECT -->
<body>
  <main>...</main>
  <main>...</main>
  <!-- Multiple main elements -->
</body>
```

### 6. Meaningful Figure Captions

```html
<!-- GOOD -->
<figure>
  <img src="chart.png" alt="Sales data" />
  <figcaption>Figure 1: Q4 2025 sales showing 20% growth</figcaption>
</figure>

<!-- BAD -->
<figure>
  <img src="chart.png" alt="Sales data" />
  <figcaption>Image</figcaption>
</figure>
```

### 7. Semantic vs Presentational Elements

```html
<!-- Semantic (GOOD) -->
<strong>Important:</strong> Do not share passwords. <em>Never</em> click
suspicious links.

<!-- Presentational (BAD) -->
<b>Important:</b> Do not share passwords. <i>Never</i> click suspicious links.
```

### 8. Address Element for Contact Info Only

```html
<!-- CORRECT -->
<address>
  Contact: <a href="mailto:info@example.com">info@example.com</a>
</address>

<!-- INCORRECT - Not contact information -->
<address>123 Main Street, New York, NY 10001</address>
```

### 9. Section Requires Heading

```html
<!-- GOOD -->
<section>
  <h2>Section Title</h2>
  <p>Content...</p>
</section>

<!-- BAD - No heading -->
<section>
  <p>Content...</p>
</section>
```

### 10. Avoid Div-itis

```html
<!-- BAD - Div soup -->
<div class="article">
  <div class="header">
    <div class="title">Title</div>
  </div>
</div>

<!-- GOOD - Semantic -->
<article>
  <header>
    <h2>Title</h2>
  </header>
</article>
```

## 10 Key Takeaways

1. **Semantic HTML improves accessibility**: Screen readers rely on semantic structure to navigate and understand content, making websites usable for people with disabilities

2. **SEO benefits are significant**: Search engines better understand content hierarchy and relevance when using semantic elements, potentially improving search rankings

3. **Use article for syndicated content**: The `<article>` element should contain content that makes sense independently and could be distributed separately from the page

4. **Section requires context**: Always use `<section>` with a heading; if you can't provide meaningful heading, consider using `<div>` instead

5. **One main per page**: Only one `<main>` element should exist per page, containing the primary content (excluding headers, footers, and navigation)

6. **Nav needs labels**: When using multiple `<nav>` elements, provide unique `aria-label` or `aria-labelledby` attributes for screen reader users

7. **Time element for dates**: Always use `<time>` with `datetime` attribute for machine-readable dates, enabling better parsing and functionality

8. **Header/footer context matters**: `<header>` and `<footer>` elements can be used within multiple contexts (page, article, section) with different meanings

9. **Figure for referenced content**: Use `<figure>` and `<figcaption>` for content referenced from main text (images, diagrams, code listings, quotes)

10. **Semantic HTML is progressive enhancement**: Start with semantic markup that works without CSS/JavaScript, then enhance with styling and interactivity
