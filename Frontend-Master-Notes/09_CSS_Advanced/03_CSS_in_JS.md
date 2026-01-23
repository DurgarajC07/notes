# 🎨 CSS-in-JS Deep Dive

> CSS-in-JS allows you to write CSS directly in JavaScript, providing scoped styles, dynamic styling, and better developer experience. Popular libraries include styled-components and Emotion.

---

## 📖 1. Concept Explanation

### What is CSS-in-JS?

**Write CSS in JavaScript using template literals or objects:**

```javascript
// styled-components
import styled from "styled-components";

const Button = styled.button`
  background: ${(props) => (props.primary ? "blue" : "white")};
  color: ${(props) => (props.primary ? "white" : "blue")};
  padding: 12px 24px;
  border-radius: 4px;

  &:hover {
    opacity: 0.8;
  }
`;

// Usage
<Button primary>Click Me</Button>;
```

---

## 🧠 2. Why It Matters

### Traditional CSS Problems

```css
/* styles.css */
.button {
  background: blue;
}

/* Another file */
.button {
  background: red; /* Conflict! */
}

/* Global scope pollution */
/* No dynamic styling based on props */
/* Dead code elimination difficult */
```

---

### CSS-in-JS Benefits

```javascript
// ✅ Scoped styles (no conflicts)
const Button1 = styled.button`
  background: blue;
`;

const Button2 = styled.button`
  background: red;
`;

// ✅ Dynamic styling
const Button = styled.button`
  background: ${(props) => (props.variant === "primary" ? "blue" : "gray")};
`;

// ✅ Dead code elimination (unused components removed)
// ✅ TypeScript support
// ✅ Theming
```

---

## ⚙️ 3. styled-components

### Basic Syntax

```javascript
import styled from "styled-components";

// Element styled with template literal
const Title = styled.h1`
  font-size: 24px;
  color: #333;
  margin-bottom: 16px;
`;

const Container = styled.div`
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
`;

// Usage
function App() {
  return (
    <Container>
      <Title>Hello World</Title>
    </Container>
  );
}
```

---

### Props-based Styling

```javascript
const Button = styled.button`
  background: ${props => {
    switch (props.variant) {
      case 'primary': return 'blue';
      case 'danger': return 'red';
      case 'success': return 'green';
      default: return 'gray';
    }
  }};

  color: white;
  padding: ${props => props.size === 'large' ? '16px 32px' : '8px 16px'};
  font-size: ${props => props.size === 'large' ? '18px' : '14px'};
  border: none;
  border-radius: 4px;
  cursor: pointer;

  &:hover {
    opacity: 0.8;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

// TypeScript
interface ButtonProps {
  variant?: 'primary' | 'danger' | 'success';
  size?: 'small' | 'large';
}

const TypedButton = styled.button<ButtonProps>`
  background: ${props => props.variant === 'primary' ? 'blue' : 'gray'};
  padding: ${props => props.size === 'large' ? '16px 32px' : '8px 16px'};
`;

// Usage
<Button variant="primary" size="large">Submit</Button>
<Button variant="danger">Delete</Button>
<TypedButton variant="primary" size="large">Typed</TypedButton>
```

---

### Extending Styles

```javascript
const Button = styled.button`
  padding: 12px 24px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
`;

// Extend and override
const PrimaryButton = styled(Button)`
  background: blue;
  color: white;
`;

const OutlinedButton = styled(Button)`
  background: transparent;
  border: 2px solid blue;
  color: blue;
`;

// Extend with different element
const LinkButton = styled(Button).attrs({ as: 'a' })`
  text-decoration: none;
`;

// Usage
<PrimaryButton>Primary</PrimaryButton>
<OutlinedButton>Outlined</OutlinedButton>
<LinkButton href="/about">Link</LinkButton>
```

---

### Theming

```javascript
import { ThemeProvider } from 'styled-components';

const lightTheme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#212529'
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px'
  }
};

const darkTheme = {
  colors: {
    primary: '#0d6efd',
    secondary: '#6c757d',
    background: '#212529',
    text: '#f8f9fa'
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px'
  }
};

// Styled component using theme
const Container = styled.div`
  background: ${props => props.theme.colors.background};
  color: ${props => props.theme.colors.text};
  padding: ${props => props.theme.spacing.medium};
`;

const Button = styled.button`
  background: ${props => props.theme.colors.primary};
  color: white;
  padding: ${props => props.theme.spacing.small} ${props => props.theme.spacing.medium};
  border-radius: 4px;
  border: none;
`;

// App with theme
function App() {
  const [isDark, setIsDark] = useState(false);

  return (
    <ThemeProvider theme={isDark ? darkTheme : lightTheme}>
      <Container>
        <h1>Themed App</h1>
        <Button onClick={() => setIsDark(!isDark)}>
          Toggle Theme
        </Button>
      </Container>
    </ThemeProvider>
  );
}

// TypeScript theme
declare module 'styled-components' {
  export interface DefaultTheme {
    colors: {
      primary: string;
      secondary: string;
      background: string;
      text: string;
    };
    spacing: {
      small: string;
      medium: string;
      large: string;
    };
  }
}
```

---

### Global Styles

```javascript
import { createGlobalStyle } from "styled-components";

const GlobalStyles = createGlobalStyle`
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
  
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu;
    background: ${(props) => props.theme.colors.background};
    color: ${(props) => props.theme.colors.text};
  }
  
  a {
    color: ${(props) => props.theme.colors.primary};
    text-decoration: none;
  }
`;

// Usage
function App() {
  return (
    <ThemeProvider theme={lightTheme}>
      <GlobalStyles />
      <Container>
        <h1>App Content</h1>
      </Container>
    </ThemeProvider>
  );
}
```

---

### Animations

```javascript
import styled, { keyframes } from 'styled-components';

const fadeIn = keyframes`
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

const rotate = keyframes`
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
`;

const FadeInBox = styled.div`
  animation: ${fadeIn} 0.5s ease-out;
`;

const Spinner = styled.div`
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  animation: ${rotate} 1s linear infinite;
`;

// Usage
<FadeInBox>
  <h2>Fades in on mount</h2>
</FadeInBox>

<Spinner />
```

---

## ⚙️ 4. Emotion

### Basic Syntax

```javascript
/** @jsxImportSource @emotion/react */
import { css } from "@emotion/react";
import styled from "@emotion/styled";

// css prop
const Container = () => (
  <div
    css={css`
      max-width: 1200px;
      margin: 0 auto;
      padding: 20px;
    `}
  >
    <h1>Hello</h1>
  </div>
);

// styled API (like styled-components)
const Button = styled.button`
  background: blue;
  color: white;
  padding: 12px 24px;
  border-radius: 4px;
`;
```

---

### Object Styles

```javascript
import { css } from "@emotion/react";

// Object syntax
const containerStyle = css({
  maxWidth: 1200,
  margin: "0 auto",
  padding: 20,
  backgroundColor: "#f5f5f5",

  // Nested
  "& h1": {
    fontSize: 24,
    color: "#333",
  },

  // Media query
  "@media (max-width: 768px)": {
    padding: 10,
  },
});

// Usage
<div css={containerStyle}>
  <h1>Title</h1>
</div>;
```

---

### Composition

```javascript
import { css } from '@emotion/react';

const baseButton = css`
  padding: 12px 24px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-size: 14px;
`;

const primaryButton = css`
  ${baseButton}
  background: blue;
  color: white;

  &:hover {
    background: darkblue;
  }
`;

const dangerButton = css`
  ${baseButton}
  background: red;
  color: white;

  &:hover {
    background: darkred;
  }
`;

// Usage
<button css={primaryButton}>Primary</button>
<button css={dangerButton}>Danger</button>
```

---

## ✅ 5. Performance Optimization

### 1. Avoid Creating Styled Components Inside Render

```javascript
// ❌ BAD: New component every render
function Parent() {
  const StyledDiv = styled.div`
    color: red;
  `; // New component instance!

  return <StyledDiv>Hello</StyledDiv>;
}

// ✅ GOOD: Define outside
const StyledDiv = styled.div`
  color: red;
`;

function Parent() {
  return <StyledDiv>Hello</StyledDiv>;
}
```

---

### 2. Use css Prop for Dynamic Styles

```javascript
// ❌ LESS OPTIMAL: New styled component for each variant
const RedButton = styled.button`
  background: red;
`;

const BlueButton = styled.button`
  background: blue;
`;

function App() {
  return isRed ? <RedButton /> : <BlueButton />;
}

// ✅ BETTER: Single component with props
const Button = styled.button`
  background: ${(props) => props.color};
`;

function App() {
  return <Button color={isRed ? "red" : "blue"} />;
}

// ✅ BEST: css prop (no styled component)
/** @jsxImportSource @emotion/react */
import { css } from "@emotion/react";

function App() {
  return (
    <button
      css={css`
        background: ${isRed ? "red" : "blue"};
      `}
    >
      Click
    </button>
  );
}
```

---

### 3. Memoize Complex Styles

```javascript
import { useMemo } from "react";
import { css } from "@emotion/react";

function DynamicComponent({ theme, size, variant }) {
  // ❌ BAD: Recomputes every render
  const styles = css`
    background: ${theme.colors[variant]};
    padding: ${theme.spacing[size]};
    /* ... complex calculations ... */
  `;

  // ✅ GOOD: Memoized
  const styles = useMemo(
    () => css`
      background: ${theme.colors[variant]};
      padding: ${theme.spacing[size]};
      /* ... complex calculations ... */
    `,
    [theme, size, variant],
  );

  return <div css={styles}>Content</div>;
}
```

---

## 🧪 6. Real-World Example: Component Library

```javascript
import styled from 'styled-components';

// Theme
export const theme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    success: '#28a745',
    danger: '#dc3545',
    warning: '#ffc107',
    info: '#17a2b8',
    light: '#f8f9fa',
    dark: '#343a40'
  },
  spacing: (multiplier = 1) => `${8 * multiplier}px`,
  breakpoints: {
    xs: '0px',
    sm: '576px',
    md: '768px',
    lg: '992px',
    xl: '1200px'
  }
};

// Button component
interface ButtonProps {
  variant?: keyof typeof theme.colors;
  size?: 'small' | 'medium' | 'large';
  fullWidth?: boolean;
}

export const Button = styled.button<ButtonProps>`
  background: ${props => theme.colors[props.variant || 'primary']};
  color: white;
  padding: ${props => {
    switch (props.size) {
      case 'small': return '6px 12px';
      case 'large': return '16px 32px';
      default: return '12px 24px';
    }
  }};
  font-size: ${props => {
    switch (props.size) {
      case 'small': return '12px';
      case 'large': return '18px';
      default: return '14px';
    }
  }};
  border: none;
  border-radius: 4px;
  cursor: pointer;
  width: ${props => props.fullWidth ? '100%' : 'auto'};
  transition: opacity 0.2s;

  &:hover {
    opacity: 0.8;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

// Card component
export const Card = styled.div`
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  padding: ${theme.spacing(3)};
  margin-bottom: ${theme.spacing(2)};
`;

export const CardHeader = styled.div`
  font-size: 20px;
  font-weight: 600;
  margin-bottom: ${theme.spacing(2)};
  color: ${theme.colors.dark};
`;

export const CardBody = styled.div`
  color: ${theme.colors.secondary};
  line-height: 1.6;
`;

// Grid system
export const Container = styled.div`
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 ${theme.spacing(2)};

  @media (min-width: ${theme.breakpoints.md}) {
    padding: 0 ${theme.spacing(3)};
  }
`;

export const Row = styled.div`
  display: flex;
  flex-wrap: wrap;
  margin: 0 -${theme.spacing()};
`;

interface ColProps {
  xs?: number;
  sm?: number;
  md?: number;
  lg?: number;
  xl?: number;
}

export const Col = styled.div<ColProps>`
  padding: 0 ${theme.spacing()};
  flex: 0 0 ${props => props.xs ? `${(props.xs / 12) * 100}%` : '100%'};

  @media (min-width: ${theme.breakpoints.sm}) {
    flex: 0 0 ${props => props.sm ? `${(props.sm / 12) * 100}%` : '100%'};
  }

  @media (min-width: ${theme.breakpoints.md}) {
    flex: 0 0 ${props => props.md ? `${(props.md / 12) * 100}%` : '100%'};
  }

  @media (min-width: ${theme.breakpoints.lg}) {
    flex: 0 0 ${props => props.lg ? `${(props.lg / 12) * 100}%` : '100%'};
  }

  @media (min-width: ${theme.breakpoints.xl}) {
    flex: 0 0 ${props => props.xl ? `${(props.xl / 12) * 100}%` : '100%'};
  }
`;

// Usage
function App() {
  return (
    <Container>
      <Row>
        <Col xs={12} md={6} lg={4}>
          <Card>
            <CardHeader>Card 1</CardHeader>
            <CardBody>Content here</CardBody>
            <Button variant="primary" size="medium">
              Click Me
            </Button>
          </Card>
        </Col>

        <Col xs={12} md={6} lg={4}>
          <Card>
            <CardHeader>Card 2</CardHeader>
            <CardBody>More content</CardBody>
            <Button variant="success" fullWidth>
              Full Width
            </Button>
          </Card>
        </Col>
      </Row>
    </Container>
  );
}
```

---

## ❓ 7. Interview Questions

### Q1: CSS-in-JS vs traditional CSS?

**Answer:**

| Feature                   | CSS-in-JS                      | Traditional CSS                |
| ------------------------- | ------------------------------ | ------------------------------ |
| **Scoping**               | Automatic (unique class names) | Manual (BEM, modules)          |
| **Dynamic styles**        | Easy (props, theme)            | Hard (inline styles, CSS vars) |
| **Dead code elimination** | Automatic                      | Manual                         |
| **TypeScript**            | Native support                 | Limited                        |
| **Performance**           | Runtime cost                   | Faster (no JS)                 |
| **Bundle size**           | Larger (includes library)      | Smaller                        |
| **Learning curve**        | Steeper                        | Easier                         |

**When to use CSS-in-JS:**

- Component libraries
- Complex theming
- Dynamic prop-based styling
- TypeScript projects

**When to use traditional CSS:**

- Static websites
- Performance-critical apps
- Simple styling needs

---

### Q2: styled-components vs Emotion?

**Answer:**

| Feature         | styled-components | Emotion           |
| --------------- | ----------------- | ----------------- |
| **Bundle size** | ~15KB             | ~7KB (core)       |
| **API**         | styled API        | styled + css prop |
| **Performance** | Good              | Slightly better   |
| **SSR**         | Built-in          | Requires setup    |
| **Adoption**    | More popular      | Growing           |

**styled-components:**

```javascript
const Button = styled.button`
  background: blue;
`;
```

**Emotion (css prop):**

```javascript
<button
  css={css`
    background: blue;
  `}
>
  Click
</button>
```

**Recommendation:** Both excellent! Choose based on team preference.

---

### Q3: How to optimize CSS-in-JS performance?

**Answer:**

**1. Define components outside render:**

```javascript
// ✅ GOOD
const Button = styled.button`...`;

function App() {
  return <Button />;
}
```

**2. Use css prop for dynamic styles:**

```javascript
// ✅ BETTER
<button css={css`background: ${color};`}>
```

**3. Memoize complex styles:**

```javascript
const styles = useMemo(() => css`...`, [deps]);
```

**4. Use object styles (faster than parsing template literals):**

```javascript
const styles = css({ background: "blue" });
```

**5. Enable production build:**

- Removes dev warnings
- Minifies class names
- Smaller bundle

---

## 🎯 Summary

**CSS-in-JS benefits:**

- ✅ Scoped styles (no conflicts)
- ✅ Dynamic styling
- ✅ Theming
- ✅ TypeScript support
- ✅ Dead code elimination

**styled-components:**

- Template literal syntax
- ThemeProvider for theming
- createGlobalStyle for global styles

**Emotion:**

- Lighter (7KB)
- css prop + styled API
- Object styles

**Best practices:**

- Define styled components outside render
- Use css prop for dynamic styles
- Memoize complex computations
- Enable production builds

Master CSS-in-JS for modern component styling! 🎨
