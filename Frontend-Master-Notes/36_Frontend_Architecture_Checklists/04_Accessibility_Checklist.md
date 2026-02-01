# Accessibility Checklist (A11y)

## Table of Contents

- [Introduction](#introduction)
- [Semantic HTML](#semantic-html)
- [Keyboard Navigation](#keyboard-navigation)
- [Screen Reader Support](#screen-reader-support)
- [ARIA Attributes](#aria-attributes)
- [Color and Contrast](#color-and-contrast)
- [Forms and Inputs](#forms-and-inputs)
- [Interactive Elements](#interactive-elements)
- [Media and Content](#media-and-content)
- [Focus Management](#focus-management)
- [Testing and Validation](#testing-and-validation)
- [Documentation](#documentation)
- [Key Takeaways](#key-takeaways)

## Introduction

Accessibility ensures your application is usable by everyone, including users with disabilities. It's not just the right thing to do - it's often legally required (WCAG 2.1, ADA compliance).

### Accessibility Principles (POUR)

```typescript
interface AccessibilityPrinciples {
  perceivable: "Information must be presentable to users";
  operable: "Interface components must be operable";
  understandable: "Information and operation must be understandable";
  robust: "Content must be robust enough for assistive technologies";
}

const wcagLevels = {
  A: "Minimum level of accessibility",
  AA: "Mid-range and most common target (WCAG 2.1 AA)",
  AAA: "Highest and most complex level",
};

// Target: WCAG 2.1 Level AA compliance
```

## Semantic HTML

### ✅ Use Proper HTML Elements

```typescript
// Bad: Divs for everything
function BadExample() {
  return (
    <div className="header">
      <div className="nav">
        <div onClick={() => navigate('/')}>Home</div>
        <div onClick={() => navigate('/about')}>About</div>
      </div>
      <div className="main">
        <div className="heading">Welcome</div>
        <div>Content here</div>
      </div>
    </div>
  );
}

// Good: Semantic HTML
function GoodExample() {
  return (
    <>
      <header>
        <nav aria-label="Main navigation">
          <a href="/">Home</a>
          <a href="/about">About</a>
        </nav>
      </header>
      <main>
        <h1>Welcome</h1>
        <p>Content here</p>
      </main>
    </>
  );
}

// Semantic structure
const semanticStructure = `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Title</title>
  </head>
  <body>
    <a href="#main-content" class="skip-link">Skip to main content</a>

    <header>
      <nav aria-label="Main navigation">
        <!-- Navigation links -->
      </nav>
    </header>

    <main id="main-content">
      <article>
        <h1>Article Title</h1>
        <section>
          <h2>Section Heading</h2>
          <p>Content...</p>
        </section>
      </article>

      <aside>
        <h2>Related Content</h2>
        <!-- Sidebar content -->
      </aside>
    </main>

    <footer>
      <p>&copy; 2024 Company Name</p>
    </footer>
  </body>
</html>
`;
```

### ✅ Heading Hierarchy

```typescript
function AccessibleHeadings() {
  return (
    <article>
      <h1>Main Article Title</h1>
      {/* Don't skip levels */}

      <section>
        <h2>First Section</h2>
        <p>Content...</p>

        <h3>Subsection</h3>
        <p>Content...</p>

        <h3>Another Subsection</h3>
        <p>Content...</p>
      </section>

      <section>
        <h2>Second Section</h2>
        <p>Content...</p>
      </section>
    </article>
  );
}

// Heading validation utility
function validateHeadingHierarchy(): void {
  const headings = Array.from(document.querySelectorAll('h1, h2, h3, h4, h5, h6'));
  const levels = headings.map(h => parseInt(h.tagName[1]));

  for (let i = 1; i < levels.length; i++) {
    const jump = levels[i] - levels[i - 1];
    if (jump > 1) {
      console.warn(`Heading level skipped: h${levels[i - 1]} to h${levels[i]}`);
    }
  }
}
```

## Keyboard Navigation

### ✅ Keyboard Accessibility

```typescript
// All interactive elements must be keyboard accessible
function KeyboardAccessibleButton() {
  const handleClick = () => console.log('Clicked');
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      handleClick();
    }
  };

  return (
    <button onClick={handleClick}>
      Click Me
    </button>
  );
}

// Custom interactive element
function CustomButton({ onClick, children }: { onClick: () => void; children: React.ReactNode }) {
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      onClick();
    }
  };

  return (
    <div
      role="button"
      tabIndex={0}
      onClick={onClick}
      onKeyDown={handleKeyDown}
      className="custom-button"
    >
      {children}
    </div>
  );
}

// Skip navigation link
function SkipNavigation() {
  return (
    <a
      href="#main-content"
      className="skip-link"
      style={{
        position: 'absolute',
        left: '-9999px',
        zIndex: 999,
        padding: '1rem',
        background: 'white',
        ':focus': {
          left: '0'
        }
      }}
    >
      Skip to main content
    </a>
  );
}

// Keyboard trap for modals
function useKeyboardTrap(ref: RefObject<HTMLElement>) {
  useEffect(() => {
    if (!ref.current) return;

    const focusableElements = ref.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    };

    ref.current.addEventListener('keydown', handleKeyDown);
    firstElement?.focus();

    return () => {
      ref.current?.removeEventListener('keydown', handleKeyDown);
    };
  }, [ref]);
}

// Accessible modal
function AccessibleModal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousActiveElement = useRef<HTMLElement | null>(null);

  useKeyboardTrap(modalRef);

  useEffect(() => {
    if (isOpen) {
      previousActiveElement.current = document.activeElement as HTMLElement;
    } else {
      previousActiveElement.current?.focus();
    }
  }, [isOpen]);

  useEffect(() => {
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    if (isOpen) {
      document.addEventListener('keydown', handleEscape);
      return () => document.removeEventListener('keydown', handleEscape);
    }
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      className="modal-overlay"
    >
      <div className="modal-content">
        <h2 id="modal-title">Modal Title</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>
  );
}
```

### ✅ Focus Indicators

```css
/* Focus styles - NEVER remove outline without replacement */

/* Bad */
*:focus {
  outline: none; /* Never do this! */
}

/* Good - Custom focus styles */
*:focus {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

/* Better - Use :focus-visible for keyboard-only focus */
*:focus:not(:focus-visible) {
  outline: none;
}

*:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

/* Skip link styling */
.skip-link {
  position: absolute;
  left: -9999px;
  z-index: 999;
}

.skip-link:focus {
  left: 0;
  top: 0;
  padding: 1rem;
  background: white;
  color: black;
}

/* High contrast focus indicator */
@media (prefers-contrast: high) {
  *:focus-visible {
    outline: 3px solid currentColor;
    outline-offset: 3px;
  }
}
```

## Screen Reader Support

### ✅ ARIA Labels and Descriptions

```typescript
// Accessible icon buttons
function AccessibleIconButton() {
  return (
    <>
      {/* Bad */}
      <button>
        <SearchIcon />
      </button>

      {/* Good - aria-label */}
      <button aria-label="Search">
        <SearchIcon aria-hidden="true" />
      </button>

      {/* Good - visually hidden text */}
      <button>
        <SearchIcon aria-hidden="true" />
        <span className="sr-only">Search</span>
      </button>
    </>
  );
}

// Visually hidden but accessible to screen readers
const srOnlyStyles = {
  position: 'absolute',
  width: '1px',
  height: '1px',
  padding: 0,
  margin: '-1px',
  overflow: 'hidden',
  clip: 'rect(0, 0, 0, 0)',
  whiteSpace: 'nowrap',
  borderWidth: 0
} as const;

function VisuallyHidden({ children }: { children: React.ReactNode }) {
  return <span style={srOnlyStyles}>{children}</span>;
}

// Accessible image
function AccessibleImage() {
  return (
    <>
      {/* Decorative image */}
      <img src="/decorative.png" alt="" role="presentation" />

      {/* Informative image */}
      <img src="/chart.png" alt="Sales increased by 25% in Q4 2024" />

      {/* Complex image with description */}
      <figure>
        <img
          src="/complex-diagram.png"
          alt="System architecture diagram"
          aria-describedby="diagram-description"
        />
        <figcaption id="diagram-description">
          The diagram shows a three-tier architecture with a React frontend,
          Node.js API layer, and PostgreSQL database.
        </figcaption>
      </figure>
    </>
  );
}

// Live regions for dynamic content
function LiveRegion() {
  const [message, setMessage] = useState('');

  return (
    <>
      {/* Polite: Wait for user to finish current activity */}
      <div role="status" aria-live="polite">
        {message}
      </div>

      {/* Assertive: Interrupt immediately */}
      <div role="alert" aria-live="assertive">
        Error: Form submission failed
      </div>

      {/* Off: No announcements */}
      <div aria-live="off">
        This content won't be announced
      </div>
    </>
  );
}

// Accessible loading states
function LoadingButton({ isLoading, children }: { isLoading: boolean; children: React.ReactNode }) {
  return (
    <button disabled={isLoading} aria-busy={isLoading}>
      {isLoading ? (
        <>
          <Spinner aria-hidden="true" />
          <span className="sr-only">Loading...</span>
        </>
      ) : (
        children
      )}
    </button>
  );
}
```

## ARIA Attributes

### ✅ Proper ARIA Usage

```typescript
// Accordion
function AccessibleAccordion() {
  const [openIndex, setOpenIndex] = useState<number | null>(null);

  return (
    <div>
      {items.map((item, index) => (
        <div key={index}>
          <h3>
            <button
              aria-expanded={openIndex === index}
              aria-controls={`panel-${index}`}
              id={`accordion-${index}`}
              onClick={() => setOpenIndex(openIndex === index ? null : index)}
            >
              {item.title}
            </button>
          </h3>
          <div
            id={`panel-${index}`}
            role="region"
            aria-labelledby={`accordion-${index}`}
            hidden={openIndex !== index}
          >
            {item.content}
          </div>
        </div>
      ))}
    </div>
  );
}

// Tabs
function AccessibleTabs() {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <div>
      <div role="tablist" aria-label="Content sections">
        {tabs.map((tab, index) => (
          <button
            key={index}
            role="tab"
            aria-selected={activeTab === index}
            aria-controls={`tabpanel-${index}`}
            id={`tab-${index}`}
            tabIndex={activeTab === index ? 0 : -1}
            onClick={() => setActiveTab(index)}
            onKeyDown={(e) => {
              if (e.key === 'ArrowRight') {
                setActiveTab((activeTab + 1) % tabs.length);
              } else if (e.key === 'ArrowLeft') {
                setActiveTab((activeTab - 1 + tabs.length) % tabs.length);
              }
            }}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {tabs.map((tab, index) => (
        <div
          key={index}
          role="tabpanel"
          id={`tabpanel-${index}`}
          aria-labelledby={`tab-${index}`}
          hidden={activeTab !== index}
          tabIndex={0}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}

// Combobox (autocomplete)
function AccessibleCombobox() {
  const [value, setValue] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);

  return (
    <div>
      <label id="combobox-label" htmlFor="combobox-input">
        Search
      </label>
      <div role="combobox" aria-expanded={isOpen} aria-haspopup="listbox">
        <input
          id="combobox-input"
          type="text"
          aria-autocomplete="list"
          aria-controls="listbox"
          aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
          value={value}
          onChange={(e) => setValue(e.target.value)}
          onFocus={() => setIsOpen(true)}
        />
      </div>
      {isOpen && (
        <ul id="listbox" role="listbox" aria-labelledby="combobox-label">
          {results.map((result, index) => (
            <li
              key={index}
              id={`option-${index}`}
              role="option"
              aria-selected={activeIndex === index}
              onClick={() => setValue(result)}
            >
              {result}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

// Menu
function AccessibleMenu() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button
        aria-haspopup="menu"
        aria-expanded={isOpen}
        aria-controls="menu"
        onClick={() => setIsOpen(!isOpen)}
      >
        Actions
      </button>

      {isOpen && (
        <ul id="menu" role="menu">
          <li role="none">
            <button role="menuitem" onClick={handleEdit}>
              Edit
            </button>
          </li>
          <li role="none">
            <button role="menuitem" onClick={handleDelete}>
              Delete
            </button>
          </li>
        </ul>
      )}
    </div>
  );
}
```

## Color and Contrast

### ✅ WCAG Contrast Requirements

```typescript
// Contrast ratio checker
function checkContrastRatio(foreground: string, background: string): {
  ratio: number;
  passesAA: boolean;
  passesAAA: boolean;
} {
  const luminance = (r: number, g: number, b: number): number => {
    const [rs, gs, bs] = [r, g, b].map((c) => {
      c = c / 255;
      return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
    });
    return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
  };

  const hexToRgb = (hex: string): [number, number, number] => {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    return result
      ? [parseInt(result[1], 16), parseInt(result[2], 16), parseInt(result[3], 16)]
      : [0, 0, 0];
  };

  const [fr, fg, fb] = hexToRgb(foreground);
  const [br, bg, bb] = hexToRgb(background);

  const l1 = luminance(fr, fg, fb);
  const l2 = luminance(br, bg, bb);

  const ratio = (Math.max(l1, l2) + 0.05) / (Math.min(l1, l2) + 0.05);

  return {
    ratio,
    passesAA: ratio >= 4.5, // Normal text: 4.5:1
    passesAAA: ratio >= 7 // Enhanced: 7:1
  };
}

// Usage
const buttonContrast = checkContrastRatio('#ffffff', '#0066cc');
console.log(`Contrast ratio: ${buttonContrast.ratio.toFixed(2)}:1`);
console.log(`Passes AA: ${buttonContrast.passesAA}`);

// Don't rely on color alone
function AccessibleStatus({ status }: { status: 'success' | 'error' | 'warning' }) {
  const icons = {
    success: <CheckIcon />,
    error: <ErrorIcon />,
    warning: <WarningIcon />
  };

  const labels = {
    success: 'Success',
    error: 'Error',
    warning: 'Warning'
  };

  return (
    <div className={`status status-${status}`}>
      {icons[status]}
      <span>{labels[status]}</span>
    </div>
  );
}
```

```css
/* High contrast mode support */
@media (prefers-contrast: high) {
  .button {
    border: 2px solid currentColor;
  }
}

/* Reduced motion support */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

/* Dark mode support */
@media (prefers-color-scheme: dark) {
  :root {
    --text-color: #ffffff;
    --background-color: #1a1a1a;
  }
}

/* Color contrast examples */
.good-contrast {
  color: #000000; /* Black text */
  background: #ffffff; /* White background */
  /* Ratio: 21:1 - Passes AAA */
}

.minimum-contrast {
  color: #595959; /* Dark gray text */
  background: #ffffff; /* White background */
  /* Ratio: 4.6:1 - Passes AA for normal text */
}

.large-text {
  font-size: 18px;
  font-weight: bold;
  color: #767676; /* Medium gray */
  background: #ffffff; /* White background */
  /* Ratio: 3.1:1 - Passes AA for large text (18pt+) */
}
```

## Forms and Inputs

### ✅ Accessible Forms

```typescript
// Accessible form
function AccessibleForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});

  return (
    <form onSubmit={handleSubmit} noValidate>
      {/* Text input */}
      <div>
        <label htmlFor="name">
          Name <span aria-label="required">*</span>
        </label>
        <input
          id="name"
          type="text"
          required
          aria-required="true"
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? 'name-error' : undefined}
        />
        {errors.name && (
          <div id="name-error" role="alert" className="error">
            {errors.name}
          </div>
        )}
      </div>

      {/* Email with help text */}
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          aria-describedby="email-help"
        />
        <div id="email-help" className="help-text">
          We'll never share your email
        </div>
      </div>

      {/* Radio group */}
      <fieldset>
        <legend>Choose a plan</legend>
        <div>
          <input type="radio" id="basic" name="plan" value="basic" />
          <label htmlFor="basic">Basic</label>
        </div>
        <div>
          <input type="radio" id="pro" name="plan" value="pro" />
          <label htmlFor="pro">Pro</label>
        </div>
      </fieldset>

      {/* Checkbox */}
      <div>
        <input
          type="checkbox"
          id="terms"
          required
          aria-required="true"
          aria-invalid={!!errors.terms}
          aria-describedby="terms-error"
        />
        <label htmlFor="terms">
          I agree to the <a href="/terms">terms and conditions</a>
        </label>
        {errors.terms && (
          <div id="terms-error" role="alert" className="error">
            {errors.terms}
          </div>
        )}
      </div>

      {/* Select */}
      <div>
        <label htmlFor="country">Country</label>
        <select id="country">
          <option value="">Select a country</option>
          <option value="us">United States</option>
          <option value="uk">United Kingdom</option>
        </select>
      </div>

      <button type="submit">Submit</button>
    </form>
  );
}

// Custom checkbox
function AccessibleCheckbox({
  checked,
  onChange,
  label
}: {
  checked: boolean;
  onChange: (checked: boolean) => void;
  label: string;
}) {
  return (
    <label className="checkbox-wrapper">
      <input
        type="checkbox"
        checked={checked}
        onChange={(e) => onChange(e.target.checked)}
        className="sr-only"
      />
      <span
        role="checkbox"
        aria-checked={checked}
        tabIndex={0}
        onKeyDown={(e) => {
          if (e.key === ' ' || e.key === 'Enter') {
            e.preventDefault();
            onChange(!checked);
          }
        }}
        className="custom-checkbox"
      >
        {checked && <CheckIcon />}
      </span>
      <span>{label}</span>
    </label>
  );
}
```

## Interactive Elements

### ✅ Accessible Components

```typescript
// Tooltip
function AccessibleTooltip({ trigger, content }: TooltipProps) {
  const [isVisible, setIsVisible] = useState(false);
  const tooltipId = useId();

  return (
    <>
      <button
        aria-describedby={isVisible ? tooltipId : undefined}
        onMouseEnter={() => setIsVisible(true)}
        onMouseLeave={() => setIsVisible(false)}
        onFocus={() => setIsVisible(true)}
        onBlur={() => setIsVisible(false)}
      >
        {trigger}
      </button>
      {isVisible && (
        <div id={tooltipId} role="tooltip">
          {content}
        </div>
      )}
    </>
  );
}

// Table
function AccessibleTable() {
  return (
    <table>
      <caption>Quarterly Sales Report</caption>
      <thead>
        <tr>
          <th scope="col">Quarter</th>
          <th scope="col">Revenue</th>
          <th scope="col">Growth</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <th scope="row">Q1</th>
          <td>$1.2M</td>
          <td>+5%</td>
        </tr>
      </tbody>
    </table>
  );
}

// Breadcrumbs
function AccessibleBreadcrumbs({ items }: { items: { label: string; href: string }[] }) {
  return (
    <nav aria-label="Breadcrumb">
      <ol className="breadcrumb">
        {items.map((item, index) => (
          <li key={index}>
            {index < items.length - 1 ? (
              <a href={item.href}>{item.label}</a>
            ) : (
              <span aria-current="page">{item.label}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}

// Pagination
function AccessiblePagination({ currentPage, totalPages, onPageChange }: PaginationProps) {
  return (
    <nav aria-label="Pagination">
      <ul className="pagination">
        <li>
          <button
            onClick={() => onPageChange(currentPage - 1)}
            disabled={currentPage === 1}
            aria-label="Previous page"
          >
            Previous
          </button>
        </li>
        {Array.from({ length: totalPages }, (_, i) => i + 1).map((page) => (
          <li key={page}>
            <button
              onClick={() => onPageChange(page)}
              aria-label={`Page ${page}`}
              aria-current={page === currentPage ? 'page' : undefined}
              disabled={page === currentPage}
            >
              {page}
            </button>
          </li>
        ))}
        <li>
          <button
            onClick={() => onPageChange(currentPage + 1)}
            disabled={currentPage === totalPages}
            aria-label="Next page"
          >
            Next
          </button>
        </li>
      </ul>
    </nav>
  );
}
```

## Media and Content

### ✅ Accessible Media

```typescript
// Video with captions and transcript
function AccessibleVideo() {
  return (
    <>
      <video controls>
        <source src="/video.mp4" type="video/mp4" />
        <track
          kind="captions"
          src="/captions-en.vtt"
          srclang="en"
          label="English"
          default
        />
        <track
          kind="captions"
          src="/captions-es.vtt"
          srclang="es"
          label="Español"
        />
        <track
          kind="descriptions"
          src="/descriptions.vtt"
          srclang="en"
          label="Audio descriptions"
        />
        Your browser doesn't support video.
      </video>

      {/* Transcript */}
      <details>
        <summary>Video transcript</summary>
        <div>
          <p>Full transcript of video content...</p>
        </div>
      </details>
    </>
  );
}

// Audio player
function AccessibleAudioPlayer() {
  return (
    <div>
      <audio controls>
        <source src="/audio.mp3" type="audio/mpeg" />
        Your browser doesn't support audio.
      </audio>

      {/* Provide transcript */}
      <details>
        <summary>Audio transcript</summary>
        <p>Transcript content...</p>
      </details>
    </div>
  );
}

// Animations with reduced motion
function ResponsiveAnimation() {
  const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  return (
    <motion.div
      initial={{ opacity: 0, y: prefersReducedMotion ? 0 : 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{
        duration: prefersReducedMotion ? 0.01 : 0.3
      }}
    >
      Content
    </motion.div>
  );
}
```

## Focus Management

### ✅ Focus Control

```typescript
// Announce route changes to screen readers
function RouteAnnouncer() {
  const location = useLocation();
  const [announcement, setAnnouncement] = useState('');

  useEffect(() => {
    // Announce page title
    const title = document.title;
    setAnnouncement(`Navigated to ${title}`);
  }, [location]);

  return (
    <div role="status" aria-live="polite" aria-atomic="true" className="sr-only">
      {announcement}
    </div>
  );
}

// Focus management after actions
function useAnnouncement() {
  const announce = useCallback((message: string, priority: 'polite' | 'assertive' = 'polite') => {
    const announcement = document.createElement('div');
    announcement.setAttribute('role', priority === 'assertive' ? 'alert' : 'status');
    announcement.setAttribute('aria-live', priority);
    announcement.className = 'sr-only';
    announcement.textContent = message;

    document.body.appendChild(announcement);

    setTimeout(() => {
      document.body.removeChild(announcement);
    }, 1000);
  }, []);

  return announce;
}

// Usage
function DeleteButton() {
  const announce = useAnnouncement();

  const handleDelete = async () => {
    await deleteItem();
    announce('Item deleted successfully', 'polite');
  };

  return <button onClick={handleDelete}>Delete</button>;
}
```

## Testing and Validation

### ✅ Automated Testing

```typescript
// accessibility.test.ts
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Accessibility', () => {
  it('should have no accessibility violations', async () => {
    const { container } = render(<App />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should have proper heading hierarchy', () => {
    const { container } = render(<Page />);
    const headings = container.querySelectorAll('h1, h2, h3, h4, h5, h6');

    expect(headings[0].tagName).toBe('H1');
    // Verify no heading levels are skipped
  });

  it('should have keyboard navigation', () => {
    const { getByRole } = render(<Button onClick={jest.fn()} />);
    const button = getByRole('button');

    expect(button).toHaveFocus();
    userEvent.keyboard('{Enter}');
    expect(onClick).toHaveBeenCalled();
  });
});
```

### ✅ Manual Testing Checklist

```markdown
## Manual A11y Testing

### Keyboard Testing

- [ ] All interactive elements reachable via Tab
- [ ] Tab order is logical
- [ ] No keyboard traps
- [ ] Skip navigation link works
- [ ] Focus indicators visible
- [ ] Escape closes modals/dropdowns

### Screen Reader Testing

- [ ] Test with NVDA (Windows)
- [ ] Test with JAWS (Windows)
- [ ] Test with VoiceOver (Mac/iOS)
- [ ] Test with TalkBack (Android)
- [ ] All images have alt text
- [ ] Form labels associated
- [ ] ARIA labels present and accurate
- [ ] Live regions announce updates

### Visual Testing

- [ ] 400% zoom works (WCAG 1.4.10)
- [ ] Text resizing works
- [ ] Color contrast passes AA
- [ ] No information by color alone
- [ ] Works in high contrast mode
- [ ] Works with dark mode

### Browser Extensions

- [ ] axe DevTools scan passes
- [ ] WAVE evaluation passes
- [ ] Lighthouse a11y score > 90
```

## Documentation

### ✅ Accessibility Documentation

```markdown
# Accessibility Statement

## Our Commitment

We are committed to ensuring digital accessibility for people with disabilities. We continually improve the user experience for everyone and apply the relevant accessibility standards.

## Conformance Status

This website strives to conform to WCAG 2.1 Level AA standards.

## Feedback

We welcome your feedback on the accessibility of our site. Please contact us:

- Email: accessibility@example.com
- Phone: (555) 123-4567

## Known Issues

- Legacy PDFs may not be fully accessible (being remediated)
- Third-party embedded content may have limitations

## Compatibility

This website is designed to be compatible with:

- Modern browsers (Chrome, Firefox, Safari, Edge)
- Screen readers (NVDA, JAWS, VoiceOver, TalkBack)
- Keyboard navigation
- Voice control software

## Assessment

Last assessment: January 2024
Assessment method: Self-assessment and automated testing
```

## Key Takeaways

1. **Semantic HTML**: Use proper HTML elements - they provide built-in accessibility features

2. **Keyboard Navigation**: Ensure all functionality is accessible via keyboard, maintain logical tab order

3. **Screen Reader Support**: Provide meaningful labels, use ARIA appropriately, announce dynamic changes

4. **Color Contrast**: Meet WCAG AA standards (4.5:1 for normal text, 3:1 for large text)

5. **Focus Management**: Visible focus indicators, trap focus in modals, manage focus after actions

6. **Form Accessibility**: Associate labels, provide error messages, use appropriate input types

7. **ARIA Usage**: Use ARIA to enhance semantics, but HTML first whenever possible

8. **Testing**: Automated tests (axe, WAVE) + manual testing with screen readers and keyboard

9. **Documentation**: Provide accessibility statement, document known issues, offer multiple contact methods

10. **Responsive**: Works with zoom, text resizing, reduced motion, high contrast, dark mode

---

**WCAG 2.1 Level AA Compliance Checklist**:

```bash
✅ Perceivable
□ Text alternatives for images (1.1.1)
□ Captions for videos (1.2.2)
□ Audio descriptions (1.2.5)
□ Color contrast 4.5:1 (1.4.3)
□ Text resizable 200% (1.4.4)
□ Text images avoided (1.4.5)

✅ Operable
□ Keyboard accessible (2.1.1)
□ No keyboard trap (2.1.2)
□ Bypass blocks (2.4.1)
□ Page titles (2.4.2)
□ Focus order (2.4.3)
□ Link purpose (2.4.4)
□ Multiple ways to find pages (2.4.5)
□ Headings and labels (2.4.6)
□ Visible focus (2.4.7)

✅ Understandable
□ Language of page (3.1.1)
□ Language of parts (3.1.2)
□ Consistent navigation (3.2.3)
□ Consistent identification (3.2.4)
□ Error identification (3.3.1)
□ Labels or instructions (3.3.2)
□ Error suggestions (3.3.3)
□ Error prevention (3.3.4)

✅ Robust
□ Valid HTML (4.1.1)
□ Name, role, value (4.1.2)
□ Status messages (4.1.3)

# Accessible and inclusive! ♿
```
