# Microdata & Schema.org Structured Data

## Core Concepts

Structured data provides explicit semantic meaning to web content, enabling search engines to understand context beyond raw HTML. Schema.org defines a shared vocabulary for describing entities (people, products, events, articles) that search engines use to generate rich snippets, knowledge panels, and enhanced search results. Modern SEO requires implementing structured data using JSON-LD, Microdata, or RDFa formats to improve visibility and click-through rates.

### Schema.org Vocabulary

Schema.org is a collaborative project between Google, Microsoft, Yahoo, and Yandex that defines schemas for structured data markup. It includes over 800 types (Person, Organization, Product, Article, Event) and properties that describe entity attributes and relationships.

### JSON-LD vs Microdata vs RDFa

Three formats exist for implementing structured data: **JSON-LD** (JavaScript Object Notation for Linked Data) embeds data in `<script>` tags, **Microdata** uses HTML attributes (`itemscope`, `itemprop`), and **RDFa** (Resource Description Framework in Attributes) extends HTML with semantic attributes. Google recommends JSON-LD for its simplicity and maintainability.

### Rich Snippets

Rich snippets are enhanced search results that display additional information (star ratings, prices, availability, author) extracted from structured data. They increase visibility, improve click-through rates, and provide better user experiences in search results.

---

## HTML/CSS/TypeScript Code Examples

### Example 1: JSON-LD vs Microdata vs RDFa Comparison

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Structured Data Format Comparison</title>

    <!-- JSON-LD Format (Google's Preferred) -->
    <script type="application/ld+json">
      {
        "@context": "https://schema.org",
        "@type": "Person",
        "name": "Jane Smith",
        "jobTitle": "Senior Software Engineer",
        "email": "jane.smith@example.com",
        "url": "https://example.com/jane-smith",
        "image": "https://example.com/images/jane.jpg",
        "telephone": "+1-555-123-4567",
        "address": {
          "@type": "PostalAddress",
          "streetAddress": "123 Tech Street",
          "addressLocality": "San Francisco",
          "addressRegion": "CA",
          "postalCode": "94102",
          "addressCountry": "US"
        },
        "worksFor": {
          "@type": "Organization",
          "name": "Tech Corp",
          "url": "https://techcorp.com"
        }
      }
    </script>
  </head>
  <body>
    <!-- Microdata Format -->
    <div itemscope itemtype="https://schema.org/Person">
      <h1 itemprop="name">Jane Smith</h1>
      <p>
        <span itemprop="jobTitle">Senior Software Engineer</span> at
        <span
          itemprop="worksFor"
          itemscope
          itemtype="https://schema.org/Organization"
        >
          <span itemprop="name">Tech Corp</span>
        </span>
      </p>
      <p>
        Email:
        <a href="mailto:jane.smith@example.com" itemprop="email"
          >jane.smith@example.com</a
        >
      </p>
      <p>Phone: <span itemprop="telephone">+1-555-123-4567</span></p>
      <img
        src="https://example.com/images/jane.jpg"
        alt="Jane Smith"
        itemprop="image"
      />
      <div
        itemprop="address"
        itemscope
        itemtype="https://schema.org/PostalAddress"
      >
        <p>
          <span itemprop="streetAddress">123 Tech Street</span><br />
          <span itemprop="addressLocality">San Francisco</span>,
          <span itemprop="addressRegion">CA</span>
          <span itemprop="postalCode">94102</span>
          <span itemprop="addressCountry">US</span>
        </p>
      </div>
    </div>

    <!-- RDFa Format -->
    <div vocab="https://schema.org/" typeof="Person">
      <h1 property="name">Jane Smith</h1>
      <p>
        <span property="jobTitle">Senior Software Engineer</span> at
        <span property="worksFor" typeof="Organization">
          <span property="name">Tech Corp</span>
        </span>
      </p>
      <p>
        Email:
        <a href="mailto:jane.smith@example.com" property="email"
          >jane.smith@example.com</a
        >
      </p>
      <p>Phone: <span property="telephone">+1-555-123-4567</span></p>
      <img
        src="https://example.com/images/jane.jpg"
        alt="Jane Smith"
        property="image"
      />
      <div property="address" typeof="PostalAddress">
        <p>
          <span property="streetAddress">123 Tech Street</span><br />
          <span property="addressLocality">San Francisco</span>,
          <span property="addressRegion">CA</span>
          <span property="postalCode">94102</span>
          <span property="addressCountry">US</span>
        </p>
      </div>
    </div>
  </body>
</html>
```

### Example 2: Person Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Person",
    "@id": "https://example.com/people/john-doe",
    "name": "John Doe",
    "alternateName": "Johnny D",
    "givenName": "John",
    "familyName": "Doe",
    "gender": "Male",
    "birthDate": "1990-05-15",
    "nationality": {
      "@type": "Country",
      "name": "United States"
    },
    "jobTitle": "Full Stack Developer",
    "email": "john.doe@example.com",
    "telephone": "+1-555-987-6543",
    "url": "https://johndoe.dev",
    "image": "https://example.com/images/john-doe.jpg",
    "sameAs": [
      "https://twitter.com/johndoe",
      "https://linkedin.com/in/johndoe",
      "https://github.com/johndoe",
      "https://stackoverflow.com/users/12345/johndoe"
    ],
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "456 Developer Lane",
      "addressLocality": "Austin",
      "addressRegion": "TX",
      "postalCode": "78701",
      "addressCountry": "US"
    },
    "worksFor": {
      "@type": "Organization",
      "name": "Web Solutions Inc",
      "url": "https://websolutions.com"
    },
    "alumniOf": {
      "@type": "EducationalOrganization",
      "name": "Stanford University",
      "url": "https://stanford.edu"
    },
    "award": ["Best Developer 2025", "Innovation Award 2024"],
    "knowsAbout": [
      "JavaScript",
      "TypeScript",
      "React",
      "Node.js",
      "Python",
      "AWS"
    ],
    "knowsLanguage": [
      {
        "@type": "Language",
        "name": "English"
      },
      {
        "@type": "Language",
        "name": "Spanish"
      }
    ]
  }
</script>
```

### Example 3: Organization Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "@id": "https://techcorp.com/#organization",
    "name": "Tech Corp",
    "legalName": "Tech Corporation Inc.",
    "url": "https://techcorp.com",
    "logo": "https://techcorp.com/logo.png",
    "foundingDate": "2010-03-15",
    "founders": [
      {
        "@type": "Person",
        "name": "Alice Johnson"
      },
      {
        "@type": "Person",
        "name": "Bob Williams"
      }
    ],
    "slogan": "Innovation Through Technology",
    "description": "Leading provider of innovative software solutions for businesses worldwide.",
    "email": "info@techcorp.com",
    "telephone": "+1-800-TECH-CORP",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "789 Innovation Drive",
      "addressLocality": "Silicon Valley",
      "addressRegion": "CA",
      "postalCode": "94025",
      "addressCountry": "US"
    },
    "contactPoint": [
      {
        "@type": "ContactPoint",
        "telephone": "+1-800-TECH-CORP",
        "contactType": "customer service",
        "areaServed": "US",
        "availableLanguage": ["English", "Spanish"]
      },
      {
        "@type": "ContactPoint",
        "telephone": "+1-800-555-0199",
        "contactType": "sales",
        "areaServed": "Worldwide",
        "availableLanguage": "English"
      }
    ],
    "sameAs": [
      "https://twitter.com/techcorp",
      "https://facebook.com/techcorp",
      "https://linkedin.com/company/techcorp",
      "https://youtube.com/techcorp"
    ],
    "numberOfEmployees": {
      "@type": "QuantitativeValue",
      "value": 500
    },
    "memberOf": {
      "@type": "Organization",
      "name": "Tech Industry Association"
    },
    "owns": {
      "@type": "Product",
      "name": "TechPlatform Pro"
    }
  }
</script>
```

### Example 4: E-commerce Product Schema with Rich Snippets

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Product",
    "@id": "https://shop.example.com/products/premium-wireless-headphones",
    "name": "Premium Wireless Headphones NC1000",
    "description": "Professional noise-cancelling wireless headphones with 30-hour battery life, premium sound quality, and comfortable over-ear design.",
    "image": [
      "https://shop.example.com/images/headphones-front.jpg",
      "https://shop.example.com/images/headphones-side.jpg",
      "https://shop.example.com/images/headphones-case.jpg"
    ],
    "sku": "NC1000-BLK",
    "mpn": "NC1000",
    "gtin13": "1234567890123",
    "brand": {
      "@type": "Brand",
      "name": "AudioTech",
      "logo": "https://shop.example.com/brands/audiotech-logo.png"
    },
    "manufacturer": {
      "@type": "Organization",
      "name": "AudioTech Industries"
    },
    "model": "NC1000",
    "color": "Black",
    "material": "Aluminum, Leather, Plastic",
    "weight": {
      "@type": "QuantitativeValue",
      "value": 250,
      "unitCode": "GRM"
    },
    "offers": {
      "@type": "Offer",
      "url": "https://shop.example.com/products/premium-wireless-headphones",
      "priceCurrency": "USD",
      "price": 299.99,
      "priceValidUntil": "2026-12-31",
      "availability": "https://schema.org/InStock",
      "itemCondition": "https://schema.org/NewCondition",
      "seller": {
        "@type": "Organization",
        "name": "Tech Store"
      },
      "shippingDetails": {
        "@type": "OfferShippingDetails",
        "shippingRate": {
          "@type": "MonetaryAmount",
          "value": 0,
          "currency": "USD"
        },
        "shippingDestination": {
          "@type": "DefinedRegion",
          "addressCountry": "US"
        },
        "deliveryTime": {
          "@type": "ShippingDeliveryTime",
          "handlingTime": {
            "@type": "QuantitativeValue",
            "minValue": 1,
            "maxValue": 2,
            "unitCode": "DAY"
          },
          "transitTime": {
            "@type": "QuantitativeValue",
            "minValue": 3,
            "maxValue": 5,
            "unitCode": "DAY"
          }
        }
      },
      "hasMerchantReturnPolicy": {
        "@type": "MerchantReturnPolicy",
        "returnPolicyCategory": "https://schema.org/MerchantReturnFiniteReturnWindow",
        "merchantReturnDays": 30,
        "returnMethod": "https://schema.org/ReturnByMail",
        "returnFees": "https://schema.org/FreeReturn"
      }
    },
    "aggregateRating": {
      "@type": "AggregateRating",
      "ratingValue": 4.7,
      "reviewCount": 342,
      "bestRating": 5,
      "worstRating": 1
    },
    "review": [
      {
        "@type": "Review",
        "reviewRating": {
          "@type": "Rating",
          "ratingValue": 5,
          "bestRating": 5
        },
        "author": {
          "@type": "Person",
          "name": "Sarah Johnson"
        },
        "datePublished": "2026-01-15",
        "reviewBody": "Best headphones I've ever owned. The noise cancellation is incredible and battery life exceeds expectations.",
        "publisher": {
          "@type": "Organization",
          "name": "Tech Store"
        }
      },
      {
        "@type": "Review",
        "reviewRating": {
          "@type": "Rating",
          "ratingValue": 4,
          "bestRating": 5
        },
        "author": {
          "@type": "Person",
          "name": "Mike Chen"
        },
        "datePublished": "2026-01-20",
        "reviewBody": "Great sound quality and comfortable for long listening sessions. Only downside is the price.",
        "publisher": {
          "@type": "Organization",
          "name": "Tech Store"
        }
      }
    ],
    "additionalProperty": [
      {
        "@type": "PropertyValue",
        "name": "Battery Life",
        "value": "30 hours"
      },
      {
        "@type": "PropertyValue",
        "name": "Connectivity",
        "value": "Bluetooth 5.2"
      },
      {
        "@type": "PropertyValue",
        "name": "Noise Cancellation",
        "value": "Active ANC"
      }
    ],
    "category": "Electronics > Audio > Headphones",
    "audience": {
      "@type": "PeopleAudience",
      "suggestedMinAge": 12
    }
  }
</script>
```

### Example 5: Article Schema for Blog Posts

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "10 Advanced CSS Techniques Every Developer Should Know in 2026",
    "alternativeHeadline": "Master Modern CSS: Container Queries, Cascade Layers, and More",
    "image": [
      "https://blog.example.com/images/css-techniques-1200x630.jpg",
      "https://blog.example.com/images/css-techniques-1200x1200.jpg",
      "https://blog.example.com/images/css-techniques-800x600.jpg"
    ],
    "author": {
      "@type": "Person",
      "name": "Jane Smith",
      "url": "https://blog.example.com/authors/jane-smith",
      "image": "https://blog.example.com/authors/jane-smith.jpg",
      "jobTitle": "Senior Frontend Developer",
      "worksFor": {
        "@type": "Organization",
        "name": "Frontend Masters"
      },
      "sameAs": [
        "https://twitter.com/janesmith",
        "https://github.com/janesmith"
      ]
    },
    "publisher": {
      "@type": "Organization",
      "name": "Dev Blog",
      "logo": {
        "@type": "ImageObject",
        "url": "https://blog.example.com/logo-600x60.png",
        "width": 600,
        "height": 60
      },
      "url": "https://blog.example.com"
    },
    "datePublished": "2026-02-01T09:00:00+00:00",
    "dateModified": "2026-02-01T15:30:00+00:00",
    "mainEntityOfPage": {
      "@type": "WebPage",
      "@id": "https://blog.example.com/articles/advanced-css-techniques-2026"
    },
    "description": "Discover 10 advanced CSS techniques including container queries, cascade layers, and modern layout patterns. Complete guide with production-ready code examples.",
    "articleSection": "CSS",
    "keywords": [
      "CSS",
      "web development",
      "frontend",
      "container queries",
      "cascade layers"
    ],
    "wordCount": 3500,
    "timeRequired": "PT12M",
    "articleBody": "Full article text here...",
    "inLanguage": "en-US",
    "copyrightYear": 2026,
    "copyrightHolder": {
      "@type": "Organization",
      "name": "Dev Blog"
    },
    "speakable": {
      "@type": "SpeakableSpecification",
      "cssSelector": ["h1", "h2", ".summary"]
    },
    "isAccessibleForFree": true,
    "isPartOf": {
      "@type": "Blog",
      "name": "Dev Blog",
      "url": "https://blog.example.com"
    },
    "about": {
      "@type": "Thing",
      "name": "CSS"
    },
    "citation": [
      {
        "@type": "CreativeWork",
        "name": "CSS Specifications",
        "url": "https://w3.org/Style/CSS/"
      }
    ]
  }
</script>
```

### Example 6: Review and Rating Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Review",
    "itemReviewed": {
      "@type": "Product",
      "name": "Premium Wireless Headphones NC1000",
      "image": "https://shop.example.com/images/headphones.jpg",
      "brand": {
        "@type": "Brand",
        "name": "AudioTech"
      },
      "offers": {
        "@type": "Offer",
        "priceCurrency": "USD",
        "price": 299.99,
        "availability": "https://schema.org/InStock"
      },
      "aggregateRating": {
        "@type": "AggregateRating",
        "ratingValue": 4.7,
        "reviewCount": 342
      }
    },
    "reviewRating": {
      "@type": "Rating",
      "ratingValue": 5,
      "bestRating": 5,
      "worstRating": 1
    },
    "author": {
      "@type": "Person",
      "name": "Sarah Johnson",
      "sameAs": "https://example.com/users/sarah-johnson"
    },
    "datePublished": "2026-01-15",
    "reviewBody": "These headphones exceeded all my expectations. The noise cancellation is phenomenal - I can't hear anything when it's activated. Battery life easily lasts me a full week of daily commuting. Sound quality is crisp and clear across all genres. Highly recommended!",
    "publisher": {
      "@type": "Organization",
      "name": "Tech Store Reviews"
    },
    "positiveNotes": {
      "@type": "ItemList",
      "itemListElement": [
        "Excellent noise cancellation",
        "Long battery life",
        "Comfortable fit",
        "Superior sound quality"
      ]
    },
    "negativeNotes": {
      "@type": "ItemList",
      "itemListElement": [
        "Somewhat expensive",
        "Carrying case could be smaller"
      ]
    }
  }
</script>

<!-- Standalone Rating Schema -->
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Product",
    "name": "Premium Wireless Headphones NC1000",
    "aggregateRating": {
      "@type": "AggregateRating",
      "ratingValue": 4.7,
      "ratingCount": 342,
      "reviewCount": 342,
      "bestRating": 5,
      "worstRating": 1
    }
  }
</script>
```

### Example 7: BreadcrumbList Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": [
      {
        "@type": "ListItem",
        "position": 1,
        "name": "Home",
        "item": "https://shop.example.com/"
      },
      {
        "@type": "ListItem",
        "position": 2,
        "name": "Electronics",
        "item": "https://shop.example.com/electronics"
      },
      {
        "@type": "ListItem",
        "position": 3,
        "name": "Audio Equipment",
        "item": "https://shop.example.com/electronics/audio"
      },
      {
        "@type": "ListItem",
        "position": 4,
        "name": "Headphones",
        "item": "https://shop.example.com/electronics/audio/headphones"
      },
      {
        "@type": "ListItem",
        "position": 5,
        "name": "Premium Wireless Headphones NC1000",
        "item": "https://shop.example.com/products/premium-wireless-headphones"
      }
    ]
  }
</script>

<!-- HTML Breadcrumb with Microdata -->
<nav aria-label="Breadcrumb">
  <ol
    itemscope
    itemtype="https://schema.org/BreadcrumbList"
    style="list-style: none; display: flex; gap: 0.5rem;"
  >
    <li
      itemprop="itemListElement"
      itemscope
      itemtype="https://schema.org/ListItem"
    >
      <a itemprop="item" href="https://shop.example.com/">
        <span itemprop="name">Home</span>
      </a>
      <meta itemprop="position" content="1" />
    </li>
    <li>/</li>
    <li
      itemprop="itemListElement"
      itemscope
      itemtype="https://schema.org/ListItem"
    >
      <a itemprop="item" href="https://shop.example.com/electronics">
        <span itemprop="name">Electronics</span>
      </a>
      <meta itemprop="position" content="2" />
    </li>
    <li>/</li>
    <li
      itemprop="itemListElement"
      itemscope
      itemtype="https://schema.org/ListItem"
    >
      <a itemprop="item" href="https://shop.example.com/electronics/audio">
        <span itemprop="name">Audio Equipment</span>
      </a>
      <meta itemprop="position" content="3" />
    </li>
    <li>/</li>
    <li
      itemprop="itemListElement"
      itemscope
      itemtype="https://schema.org/ListItem"
    >
      <span itemprop="name">Headphones</span>
      <meta itemprop="position" content="4" />
    </li>
  </ol>
</nav>
```

### Example 8: FAQ Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    "mainEntity": [
      {
        "@type": "Question",
        "name": "What is structured data?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "<p>Structured data is a standardized format for providing information about a page and classifying the page content. It helps search engines understand the content better and display rich results.</p>"
        }
      },
      {
        "@type": "Question",
        "name": "Why should I use Schema.org markup?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "<p>Schema.org markup helps search engines understand your content better, which can lead to rich snippets in search results. These enhanced listings often receive higher click-through rates and better visibility.</p>"
        }
      },
      {
        "@type": "Question",
        "name": "Which format is best: JSON-LD, Microdata, or RDFa?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "<p>Google recommends JSON-LD because it's easier to implement and maintain. JSON-LD doesn't require modifying your HTML structure, making it cleaner and more flexible than Microdata or RDFa.</p>"
        }
      },
      {
        "@type": "Question",
        "name": "How do I test my structured data?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "<p>Use Google's Rich Results Test (https://search.google.com/test/rich-results) or Schema Markup Validator (https://validator.schema.org) to test and validate your structured data implementation.</p>"
        }
      },
      {
        "@type": "Question",
        "name": "Does structured data improve my search rankings?",
        "acceptedAnswer": {
          "@type": "Answer",
          "text": "<p>Structured data doesn't directly improve rankings, but it can increase click-through rates by making your listings more attractive in search results. Higher CTR can indirectly improve rankings over time.</p>"
        }
      }
    ]
  }
</script>
```

### Example 9: HowTo Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "HowTo",
    "name": "How to Implement Structured Data with JSON-LD",
    "description": "Step-by-step guide to implementing Schema.org structured data using JSON-LD format for better SEO.",
    "image": "https://blog.example.com/images/json-ld-tutorial.jpg",
    "totalTime": "PT30M",
    "estimatedCost": {
      "@type": "MonetaryAmount",
      "currency": "USD",
      "value": "0"
    },
    "supply": [
      {
        "@type": "HowToSupply",
        "name": "Text editor or IDE"
      },
      {
        "@type": "HowToSupply",
        "name": "Web browser"
      },
      {
        "@type": "HowToSupply",
        "name": "Schema.org documentation"
      }
    ],
    "tool": [
      {
        "@type": "HowToTool",
        "name": "Google Rich Results Test"
      },
      {
        "@type": "HowToTool",
        "name": "Schema Markup Validator"
      }
    ],
    "step": [
      {
        "@type": "HowToStep",
        "position": 1,
        "name": "Choose the appropriate Schema type",
        "text": "Identify which Schema.org type best describes your content (Product, Article, Event, etc.). Visit schema.org to browse available types.",
        "url": "https://blog.example.com/howto-structured-data#step1",
        "image": "https://blog.example.com/images/step1.jpg"
      },
      {
        "@type": "HowToStep",
        "position": 2,
        "name": "Add JSON-LD script tag",
        "text": "Add a <script type=\"application/ld+json\"> tag in the <head> section of your HTML document.",
        "url": "https://blog.example.com/howto-structured-data#step2",
        "image": "https://blog.example.com/images/step2.jpg"
      },
      {
        "@type": "HowToStep",
        "position": 3,
        "name": "Structure your data",
        "text": "Create a JSON object with @context set to https://schema.org and @type set to your chosen Schema type. Add required and recommended properties.",
        "url": "https://blog.example.com/howto-structured-data#step3",
        "image": "https://blog.example.com/images/step3.jpg"
      },
      {
        "@type": "HowToStep",
        "position": 4,
        "name": "Validate your markup",
        "text": "Test your structured data using Google Rich Results Test to ensure it's valid and eligible for rich results.",
        "url": "https://blog.example.com/howto-structured-data#step4",
        "image": "https://blog.example.com/images/step4.jpg"
      },
      {
        "@type": "HowToStep",
        "position": 5,
        "name": "Deploy and monitor",
        "text": "Deploy your changes and monitor Search Console for any structured data errors or warnings.",
        "url": "https://blog.example.com/howto-structured-data#step5",
        "image": "https://blog.example.com/images/step5.jpg"
      }
    ],
    "video": {
      "@type": "VideoObject",
      "name": "JSON-LD Tutorial Video",
      "description": "Video walkthrough of implementing JSON-LD structured data",
      "thumbnailUrl": "https://blog.example.com/video-thumb.jpg",
      "contentUrl": "https://blog.example.com/videos/json-ld-tutorial.mp4",
      "uploadDate": "2026-01-15T08:00:00+00:00",
      "duration": "PT15M"
    }
  }
</script>
```

### Example 10: Event Schema

```html
<script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Event",
    "name": "Web Development Conference 2026",
    "description": "Annual conference featuring the latest trends in web development, including talks on React, TypeScript, and modern CSS.",
    "image": ["https://events.example.com/images/webdev-conf-2026.jpg"],
    "startDate": "2026-06-15T09:00:00-07:00",
    "endDate": "2026-06-17T18:00:00-07:00",
    "eventStatus": "https://schema.org/EventScheduled",
    "eventAttendanceMode": "https://schema.org/MixedEventAttendanceMode",
    "location": [
      {
        "@type": "Place",
        "name": "San Francisco Convention Center",
        "address": {
          "@type": "PostalAddress",
          "streetAddress": "747 Howard Street",
          "addressLocality": "San Francisco",
          "addressRegion": "CA",
          "postalCode": "94103",
          "addressCountry": "US"
        },
        "geo": {
          "@type": "GeoCoordinates",
          "latitude": 37.7833,
          "longitude": -122.4167
        }
      },
      {
        "@type": "VirtualLocation",
        "url": "https://events.example.com/webdev-conf-2026/stream"
      }
    ],
    "offers": [
      {
        "@type": "Offer",
        "name": "Early Bird Ticket",
        "price": 299,
        "priceCurrency": "USD",
        "availability": "https://schema.org/InStock",
        "validFrom": "2026-01-01T00:00:00-08:00",
        "validThrough": "2026-03-31T23:59:59-08:00",
        "url": "https://events.example.com/webdev-conf-2026/tickets/early-bird"
      },
      {
        "@type": "Offer",
        "name": "Standard Ticket",
        "price": 399,
        "priceCurrency": "USD",
        "availability": "https://schema.org/InStock",
        "validFrom": "2026-04-01T00:00:00-08:00",
        "url": "https://events.example.com/webdev-conf-2026/tickets/standard"
      },
      {
        "@type": "Offer",
        "name": "Virtual Attendance",
        "price": 99,
        "priceCurrency": "USD",
        "availability": "https://schema.org/InStock",
        "validFrom": "2026-01-01T00:00:00-08:00",
        "url": "https://events.example.com/webdev-conf-2026/tickets/virtual"
      }
    ],
    "performer": [
      {
        "@type": "Person",
        "name": "Sarah Connor",
        "url": "https://sarahconnor.dev"
      },
      {
        "@type": "Person",
        "name": "John Smith",
        "url": "https://johnsmith.tech"
      }
    ],
    "organizer": {
      "@type": "Organization",
      "name": "Web Dev Events Inc",
      "url": "https://webdevevents.com"
    },
    "sponsor": [
      {
        "@type": "Organization",
        "name": "Tech Giant Corp",
        "url": "https://techgiant.com"
      }
    ]
  }
</script>
```

### Example 11: TypeScript Schema Generator

```typescript
interface SchemaBase {
  "@context": string;
  "@type": string;
  "@id"?: string;
}

interface PersonSchema extends SchemaBase {
  "@type": "Person";
  name: string;
  jobTitle?: string;
  email?: string;
  url?: string;
  image?: string;
  sameAs?: string[];
  address?: PostalAddressSchema;
  worksFor?: OrganizationSchema;
}

interface OrganizationSchema extends SchemaBase {
  "@type": "Organization";
  name: string;
  url?: string;
  logo?: string;
  contactPoint?: ContactPointSchema[];
  address?: PostalAddressSchema;
  sameAs?: string[];
}

interface ProductSchema extends SchemaBase {
  "@type": "Product";
  name: string;
  description: string;
  image: string[];
  sku?: string;
  brand?: BrandSchema;
  offers: OfferSchema;
  aggregateRating?: AggregateRatingSchema;
  review?: ReviewSchema[];
}

interface OfferSchema {
  "@type": "Offer";
  url: string;
  priceCurrency: string;
  price: number;
  availability: string;
  seller?: OrganizationSchema;
}

interface AggregateRatingSchema {
  "@type": "AggregateRating";
  ratingValue: number;
  reviewCount: number;
  bestRating?: number;
  worstRating?: number;
}

interface ReviewSchema {
  "@type": "Review";
  reviewRating: RatingSchema;
  author: PersonSchema;
  datePublished: string;
  reviewBody: string;
}

interface RatingSchema {
  "@type": "Rating";
  ratingValue: number;
  bestRating?: number;
}

interface ArticleSchema extends SchemaBase {
  "@type": "Article";
  headline: string;
  image: string[];
  author: PersonSchema;
  publisher: OrganizationSchema;
  datePublished: string;
  dateModified?: string;
  description?: string;
}

interface PostalAddressSchema {
  "@type": "PostalAddress";
  streetAddress: string;
  addressLocality: string;
  addressRegion: string;
  postalCode: string;
  addressCountry: string;
}

interface ContactPointSchema {
  "@type": "ContactPoint";
  telephone: string;
  contactType: string;
  areaServed?: string;
  availableLanguage?: string[];
}

interface BrandSchema {
  "@type": "Brand";
  name: string;
  logo?: string;
}

class SchemaGenerator {
  private context: string = "https://schema.org";

  generatePerson(data: Omit<PersonSchema, "@context" | "@type">): string {
    const schema: PersonSchema = {
      "@context": this.context,
      "@type": "Person",
      ...data,
    };
    return this.toJSON(schema);
  }

  generateOrganization(
    data: Omit<OrganizationSchema, "@context" | "@type">,
  ): string {
    const schema: OrganizationSchema = {
      "@context": this.context,
      "@type": "Organization",
      ...data,
    };
    return this.toJSON(schema);
  }

  generateProduct(data: Omit<ProductSchema, "@context" | "@type">): string {
    const schema: ProductSchema = {
      "@context": this.context,
      "@type": "Product",
      ...data,
    };
    return this.toJSON(schema);
  }

  generateArticle(data: Omit<ArticleSchema, "@context" | "@type">): string {
    const schema: ArticleSchema = {
      "@context": this.context,
      "@type": "Article",
      ...data,
    };
    return this.toJSON(schema);
  }

  generateBreadcrumb(items: Array<{ name: string; url: string }>): string {
    const schema = {
      "@context": this.context,
      "@type": "BreadcrumbList",
      itemListElement: items.map((item, index) => ({
        "@type": "ListItem",
        position: index + 1,
        name: item.name,
        item: item.url,
      })),
    };
    return this.toJSON(schema);
  }

  generateFAQ(questions: Array<{ question: string; answer: string }>): string {
    const schema = {
      "@context": this.context,
      "@type": "FAQPage",
      mainEntity: questions.map((q) => ({
        "@type": "Question",
        name: q.question,
        acceptedAnswer: {
          "@type": "Answer",
          text: q.answer,
        },
      })),
    };
    return this.toJSON(schema);
  }

  private toJSON(schema: any): string {
    return JSON.stringify(schema, null, 2);
  }

  injectSchema(schema: string, elementId?: string): void {
    const script = document.createElement("script");
    script.type = "application/ld+json";
    script.textContent = schema;

    if (elementId) {
      script.id = elementId;
    }

    document.head.appendChild(script);
  }

  removeSchema(elementId: string): void {
    const script = document.getElementById(elementId);
    if (script) {
      script.remove();
    }
  }

  updateSchema(elementId: string, schema: string): void {
    this.removeSchema(elementId);
    this.injectSchema(schema, elementId);
  }
}

// Usage Examples
const generator = new SchemaGenerator();

// Generate Person Schema
const personSchema = generator.generatePerson({
  name: "Jane Smith",
  jobTitle: "Senior Software Engineer",
  email: "jane@example.com",
  url: "https://janesmith.dev",
  sameAs: ["https://twitter.com/janesmith", "https://github.com/janesmith"],
});

// Inject into page
generator.injectSchema(personSchema, "person-schema");

// Generate Product Schema
const productSchema = generator.generateProduct({
  name: "Premium Headphones",
  description: "High-quality wireless headphones",
  image: ["https://example.com/headphones.jpg"],
  sku: "HP-1000",
  brand: {
    "@type": "Brand",
    name: "AudioTech",
  },
  offers: {
    "@type": "Offer",
    url: "https://shop.example.com/headphones",
    priceCurrency: "USD",
    price: 299.99,
    availability: "https://schema.org/InStock",
  },
  aggregateRating: {
    "@type": "AggregateRating",
    ratingValue: 4.7,
    reviewCount: 342,
  },
});

generator.injectSchema(productSchema, "product-schema");

// Generate Breadcrumb
const breadcrumbSchema = generator.generateBreadcrumb([
  { name: "Home", url: "https://example.com/" },
  { name: "Products", url: "https://example.com/products" },
  { name: "Headphones", url: "https://example.com/products/headphones" },
]);

generator.injectSchema(breadcrumbSchema, "breadcrumb-schema");
```

### Example 12: Schema Validation Utility

```typescript
interface ValidationResult {
  isValid: boolean;
  errors: string[];
  warnings: string[];
}

class SchemaValidator {
  private requiredFields: Record<string, string[]> = {
    Person: ["@type", "name"],
    Organization: ["@type", "name"],
    Product: ["@type", "name", "image", "offers"],
    Article: [
      "@type",
      "headline",
      "image",
      "author",
      "publisher",
      "datePublished",
    ],
    Review: ["@type", "itemReviewed", "reviewRating", "author"],
    Event: ["@type", "name", "startDate", "location"],
  };

  validate(schemaJSON: string): ValidationResult {
    const result: ValidationResult = {
      isValid: true,
      errors: [],
      warnings: [],
    };

    try {
      const schema = JSON.parse(schemaJSON);

      // Check for @context
      if (!schema["@context"]) {
        result.errors.push("Missing @context property");
        result.isValid = false;
      } else if (schema["@context"] !== "https://schema.org") {
        result.warnings.push('@context should be "https://schema.org"');
      }

      // Check for @type
      if (!schema["@type"]) {
        result.errors.push("Missing @type property");
        result.isValid = false;
        return result;
      }

      // Validate required fields for type
      const type = schema["@type"];
      const required = this.requiredFields[type];

      if (required) {
        required.forEach((field) => {
          if (!schema[field]) {
            result.errors.push(`Missing required field: ${field}`);
            result.isValid = false;
          }
        });
      }

      // Type-specific validation
      if (type === "Product") {
        this.validateProduct(schema, result);
      } else if (type === "Article") {
        this.validateArticle(schema, result);
      }
    } catch (error) {
      result.errors.push(`Invalid JSON: ${error.message}`);
      result.isValid = false;
    }

    return result;
  }

  private validateProduct(schema: any, result: ValidationResult): void {
    // Validate offers
    if (schema.offers) {
      if (!schema.offers.price) {
        result.errors.push("Product offers must have price");
        result.isValid = false;
      }
      if (!schema.offers.priceCurrency) {
        result.errors.push("Product offers must have priceCurrency");
        result.isValid = false;
      }
      if (!schema.offers.availability) {
        result.warnings.push("Product offers should include availability");
      }
    }

    // Validate images
    if (!Array.isArray(schema.image)) {
      result.warnings.push("Product image should be an array");
    }

    // Check for aggregate rating
    if (schema.aggregateRating) {
      if (!schema.aggregateRating.ratingValue) {
        result.errors.push("aggregateRating must have ratingValue");
        result.isValid = false;
      }
      if (!schema.aggregateRating.reviewCount) {
        result.errors.push("aggregateRating must have reviewCount");
        result.isValid = false;
      }
    }
  }

  private validateArticle(schema: any, result: ValidationResult): void {
    // Validate author
    if (schema.author && !schema.author["@type"]) {
      result.errors.push("Article author must have @type");
      result.isValid = false;
    }

    // Validate publisher
    if (schema.publisher) {
      if (!schema.publisher["@type"]) {
        result.errors.push("Article publisher must have @type");
        result.isValid = false;
      }
      if (!schema.publisher.logo) {
        result.warnings.push("Article publisher should have logo");
      }
    }

    // Validate dates
    if (schema.datePublished && !this.isValidISO8601(schema.datePublished)) {
      result.errors.push("datePublished must be valid ISO 8601 format");
      result.isValid = false;
    }

    if (schema.dateModified && !this.isValidISO8601(schema.dateModified)) {
      result.errors.push("dateModified must be valid ISO 8601 format");
      result.isValid = false;
    }
  }

  private isValidISO8601(dateString: string): boolean {
    const date = new Date(dateString);
    return date instanceof Date && !isNaN(date.getTime());
  }

  async validateWithGoogle(url: string): Promise<any> {
    // Note: This would require a backend proxy to call Google's API
    const apiUrl = `https://search.google.com/test/rich-results?url=${encodeURIComponent(url)}`;
    console.log("Test your URL at:", apiUrl);
    return { message: "Use Google Rich Results Test manually", url: apiUrl };
  }

  generateValidationReport(schemaJSON: string): string {
    const result = this.validate(schemaJSON);
    let report = "=== Schema Validation Report ===\n\n";

    report += `Status: ${result.isValid ? "✓ Valid" : "✗ Invalid"}\n\n`;

    if (result.errors.length > 0) {
      report += "ERRORS:\n";
      result.errors.forEach((error) => (report += `  - ${error}\n`));
      report += "\n";
    }

    if (result.warnings.length > 0) {
      report += "WARNINGS:\n";
      result.warnings.forEach((warning) => (report += `  - ${warning}\n`));
      report += "\n";
    }

    if (
      result.isValid &&
      result.errors.length === 0 &&
      result.warnings.length === 0
    ) {
      report += "✓ No issues found\n";
    }

    return report;
  }
}

// Usage
const validator = new SchemaValidator();

const productSchema = `{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Premium Headphones",
  "image": ["https://example.com/headphones.jpg"],
  "offers": {
    "@type": "Offer",
    "price": 299.99,
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}`;

const result = validator.validate(productSchema);
console.log(result);

const report = validator.generateValidationReport(productSchema);
console.log(report);
```

---

## Real-World Usage

### E-commerce Product Pages

Implement comprehensive Product schema with offers, ratings, reviews, and availability. Include breadcrumb navigation and organization markup for complete rich snippet eligibility.

### Blog Articles and News Sites

Use Article schema with author, publisher, and datePublished. Add BreadcrumbList for navigation and speakable properties for voice search optimization.

### Local Business Websites

Combine Organization schema with LocalBusiness type, including opening hours, service area, price range, and customer reviews for local SEO.

### Recipe Websites

Implement Recipe schema with ingredients, instructions, nutrition information, cook time, and ratings for recipe rich snippets in search results.

---

## Production Patterns

### Dynamic Schema Generation

```typescript
// CMS integration - generate schema from database
function generateProductSchemaFromDB(product: DBProduct): string {
  return new SchemaGenerator().generateProduct({
    name: product.name,
    description: product.description,
    image: product.images,
    sku: product.sku,
    brand: { "@type": "Brand", name: product.brand },
    offers: {
      "@type": "Offer",
      url: product.url,
      price: product.price,
      priceCurrency: "USD",
      availability: product.inStock
        ? "https://schema.org/InStock"
        : "https://schema.org/OutOfStock",
    },
  });
}
```

### Server-Side Rendering Integration

```typescript
// Next.js example
export async function getStaticProps({ params }) {
  const product = await fetchProduct(params.id);

  const structuredData = {
    "@context": "https://schema.org",
    "@type": "Product",
    // ... product data
  };

  return {
    props: {
      product,
      structuredData,
    },
  };
}
```

---

## Best Practices

### 1. **Use JSON-LD Over Microdata**

Google recommends JSON-LD for easier implementation and maintenance. Keep structured data separate from HTML.

### 2. **Include All Required Properties**

Each Schema type has required fields. Missing them prevents rich snippet eligibility. Check Schema.org documentation.

### 3. **Provide Multiple Images**

Include multiple image sizes and aspect ratios (16:9, 4:3, 1:1) for different rich result formats.

### 4. **Keep Data Accurate and Synchronized**

Structured data must match visible page content. Mismatches can result in penalties.

### 5. **Use Specific Schema Types**

Use the most specific type available (e.g., SoftwareApplication instead of just Product).

### 6. **Validate Before Deployment**

Always test with Google Rich Results Test and Schema Markup Validator before going live.

### 7. **Include Organization Markup**

Add Organization schema sitewide for brand recognition and knowledge panel eligibility.

### 8. **Implement Breadcrumbs**

BreadcrumbList improves navigation in search results and helps search engines understand site structure.

### 9. **Add Reviews and Ratings**

Product and business reviews increase trust and CTR in search results.

### 10. **Monitor Search Console**

Check for structured data errors and enhancements in Google Search Console regularly.

---

## 10 Key Takeaways

1. **JSON-LD Is the Preferred Format**: Google recommends JSON-LD over Microdata and RDFa because it's cleaner, easier to maintain, and doesn't require modifying HTML structure.

2. **Schema.org Enables Rich Snippets**: Proper structured data implementation can generate star ratings, prices, breadcrumbs, and other rich features in search results, significantly increasing click-through rates.

3. **Required Properties Are Non-Negotiable**: Each Schema type has mandatory fields - missing them prevents rich snippet eligibility even if syntax is correct.

4. **Structured Data Doesn't Directly Impact Rankings**: While it doesn't boost SEO rankings directly, enhanced visibility and higher CTR from rich snippets can indirectly improve search performance.

5. **Product Schema Drives E-commerce Success**: Comprehensive Product markup with offers, ratings, availability, and prices creates compelling search results that convert better than plain listings.

6. **BreadcrumbList Improves Navigation**: Breadcrumb structured data displays navigation paths in search results, improving UX and helping users understand site hierarchy.

7. **Article Schema Benefits Content Publishers**: Proper Article markup with author, publisher, and dates enables article rich snippets and Google News eligibility.

8. **FAQ and HowTo Schemas Capture Featured Snippets**: These formats are optimized for position zero (featured snippets) in search results, providing maximum visibility.

9. **Validation Is Mandatory**: Always test structured data with Google Rich Results Test and Schema Markup Validator - syntax errors silently prevent rich snippets without warnings.

10. **Accuracy Is Critical for Trust**: Structured data must match visible page content exactly - discrepancies can result in manual actions and loss of rich snippet eligibility.
