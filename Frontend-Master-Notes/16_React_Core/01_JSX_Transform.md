# JSX Transform in React

## Core Concept

JSX is **syntactic sugar** that gets transformed into regular JavaScript function calls. Understanding this transformation is crucial for debugging, optimization, and understanding React internals.

---

## Classic Transform (React <17)

### **Before Transform**

```jsx
const element = <h1 className="greeting">Hello, world!</h1>;
```

### **After Transform**

```javascript
const element = React.createElement(
  "h1",
  { className: "greeting" },
  "Hello, world!",
);
```

### **React.createElement Signature**

```javascript
React.createElement(
  type, // string (HTML tag) or Component
  props, // object of attributes/props
  ...children, // child elements
);
```

---

## New Transform (React >=17)

### **Before Transform**

```jsx
function App() {
  return <h1>Hello World</h1>;
}
```

### **After Transform**

```javascript
import { jsx as _jsx } from "react/jsx-runtime";

function App() {
  return _jsx("h1", { children: "Hello World" });
}
```

**Key difference:** No longer needs `import React from 'react'` in every file!

---

## JSX to Function Call Examples

### **Simple Element**

```jsx
<div>Hello</div>
```

Becomes:

```javascript
_jsx("div", { children: "Hello" });
```

### **With Props**

```jsx
<button onClick={handleClick} disabled>
  Click Me
</button>
```

Becomes:

```javascript
_jsx("button", {
  onClick: handleClick,
  disabled: true,
  children: "Click Me",
});
```

### **Multiple Children**

```jsx
<div>
  <h1>Title</h1>
  <p>Paragraph</p>
</div>
```

Becomes:

```javascript
_jsxs("div", {
  children: [
    _jsx("h1", { children: "Title" }),
    _jsx("p", { children: "Paragraph" }),
  ],
});
```

### **Component**

```jsx
<MyComponent prop="value">
  <span>Child</span>
</MyComponent>
```

Becomes:

```javascript
_jsx(MyComponent, {
  prop: "value",
  children: _jsx("span", { children: "Child" }),
});
```

---

## JSX Rules from Transform Perspective

### **1. Capitalization Matters**

```jsx
// Lowercase = HTML element (string)
<div />  →  _jsx('div', {})

// Uppercase = Component (reference)
<Component />  →  _jsx(Component, {})
```

### **2. Props Are Objects**

```jsx
<div className="box" id="main" />
```

Becomes:

```javascript
_jsx("div", {
  className: "box",
  id: "main",
});
```

### **3. Children Are Props**

```jsx
<div>Content</div>
```

Becomes:

```javascript
_jsx("div", { children: "Content" });
```

### **4. Expressions in Braces**

```jsx
<div>{count * 2}</div>
```

Becomes:

```javascript
_jsx("div", { children: count * 2 });
```

### **5. Spread Props**

```jsx
<Component {...props} />
```

Becomes:

```javascript
_jsx(Component, { ...props });
```

---

## Fragment Handling

### **<React.Fragment>**

```jsx
<React.Fragment>
  <Child1 />
  <Child2 />
</React.Fragment>
```

Becomes:

```javascript
_jsxs(Fragment, {
  children: [_jsx(Child1, {}), _jsx(Child2, {})],
});
```

### **Short Syntax <>**

```jsx
<>
  <Child1 />
  <Child2 />
</>
```

Becomes same as above.

---

## Key Generation

```jsx
const items = ["a", "b", "c"];
const list = items.map((item) => <li>{item}</li>);
```

Becomes:

```javascript
const list = items.map((item) => _jsx("li", { children: item }));
```

**Problem:** No keys! Should be:

```jsx
const list = items.map((item) => <li key={item}>{item}</li>);
```

Becomes:

```javascript
const list = items.map((item) => _jsx("li", { children: item }, item));
```

---

## Conditional Rendering

### **Ternary**

```jsx
{
  isLoggedIn ? <UserGreeting /> : <GuestGreeting />;
}
```

Becomes:

```javascript
isLoggedIn ? _jsx(UserGreeting, {}) : _jsx(GuestGreeting, {});
```

### **&&** Operator\*\*

```jsx
{
  messages.length > 0 && <MessageList messages={messages} />;
}
```

Becomes:

```javascript
messages.length > 0 && _jsx(MessageList, { messages: messages });
```

---

## Performance Implications

### **Inline Objects/Functions**

```jsx
// BAD - creates new object every render
<Component style={{ color: "red" }} />;

// Transforms to new object each time
_jsx(Component, { style: { color: "red" } });

// GOOD - define outside
const style = { color: "red" };
<Component style={style} />;
```

### **Inline Arrow Functions**

```jsx
// BAD - creates new function every render
<button onClick={() => handleClick(id)}>Click</button>;

// GOOD - use useCallback or bind
const handleClickWithId = useCallback(() => handleClick(id), [id]);
<button onClick={handleClickWithId}>Click</button>;
```

---

## TypeScript and JSX

```tsx
// TSX allows type checking
interface Props {
  name: string;
  age: number;
}

function Greeting({ name, age }: Props) {
  return (
    <div>
      Hello {name}, {age}
    </div>
  );
}

// Type error caught at compile time
<Greeting name="Alice" age="30" />; // Error: age must be number
```

---

## Configuration (babel/tsconfig)

### **Babel (.babelrc)**

```json
{
  "presets": [
    [
      "@babel/preset-react",
      {
        "runtime": "automatic" // New JSX transform
      }
    ]
  ]
}
```

### **TypeScript (tsconfig.json)**

```json
{
  "compilerOptions": {
    "jsx": "react-jsx" // New transform (React 17+)
    // or "jsx": "react"  // Classic transform
  }
}
```

---

## Best Practices

✅ **Use React 17+ new transform** - no import React needed  
✅ **Always add keys to lists** - helps React reconciliation  
✅ **Extract constants outside render** - avoid recreating objects  
✅ **Use fragments <> to avoid wrapper divs**  
✅ **Understand transformed output** - helps debug issues  
❌ **Don't create inline objects/functions** in hot paths  
❌ **Don't forget key prop** in mapped elements

---

## Key Takeaways

1. **JSX is syntactic sugar** for function calls
2. **Lowercase = HTML, Uppercase = Component**
3. **Props become object properties**
4. **Children are props** (special prop)
5. **New transform eliminates React import** requirement
6. **Keys are essential** for list reconciliation
7. **Inline objects/functions** create new instances each render
