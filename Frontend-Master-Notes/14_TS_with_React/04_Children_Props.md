# Children Props in TypeScript

## Core Concept

React children props represent the content between component opening and closing tags. TypeScript provides specific types for children, enabling type-safe composition patterns and preventing common errors.

---

## ReactNode Type

```typescript
import { ReactNode } from 'react';

// Most flexible - accepts anything renderable
interface ContainerProps {
  children: ReactNode;
}

function Container({ children }: ContainerProps) {
  return <div className="container">{children}</div>;
}

// Usage - accepts multiple types
<Container>
  Hello World  {/* string */}
</Container>

<Container>
  {42}  {/* number */}
</Container>

<Container>
  <div>Content</div>  {/* JSX element */}
</Container>

<Container>
  {null}  {/* null/undefined */}
</Container>

<Container>
  {[1, 2, 3].map(n => <div key={n}>{n}</div>)}  {/* array */}
</Container>
```

---

## ReactElement Type

```typescript
import { ReactElement } from 'react';

// Only accepts JSX elements (not strings/numbers/null)
interface CardProps {
  children: ReactElement;
}

function Card({ children }: CardProps) {
  return <div className="card">{children}</div>;
}

// ✅ Works
<Card>
  <div>Content</div>
</Card>

// ❌ Type error
<Card>
  Hello World
</Card>

// ❌ Type error
<Card>
  {null}
</Card>
```

---

## Specific Component Types

```typescript
import { ReactElement } from 'react';

// Only accept specific components
interface TabsProps {
  children: ReactElement<TabProps> | ReactElement<TabProps>[];
}

interface TabProps {
  label: string;
  children: ReactNode;
}

function Tab({ label, children }: TabProps) {
  return <div>{children}</div>;
}

function Tabs({ children }: TabsProps) {
  return <div className="tabs">{children}</div>;
}

// ✅ Works
<Tabs>
  <Tab label="First">Content 1</Tab>
  <Tab label="Second">Content 2</Tab>
</Tabs>

// ❌ Type error - not a Tab component
<Tabs>
  <div>Invalid</div>
</Tabs>
```

---

## JSX.Element Type

```typescript
// Similar to ReactElement but more restrictive
interface LayoutProps {
  children: JSX.Element;
}

function Layout({ children }: LayoutProps) {
  return <main>{children}</main>;
}

// ✅ Works
<Layout>
  <div>Content</div>
</Layout>

// ❌ Type error - no array support by default
<Layout>
  <div>One</div>
  <div>Two</div>
</Layout>
```

---

## PropsWithChildren Utility

```typescript
import { PropsWithChildren } from 'react';

// Automatically adds children: ReactNode
interface BoxProps {
  color: string;
}

function Box({ color, children }: PropsWithChildren<BoxProps>) {
  return <div style={{ color }}>{children}</div>;
}

// Equivalent to:
interface BoxPropsManual {
  color: string;
  children?: ReactNode;
}
```

---

## Optional vs Required Children

```typescript
// Optional children (most common)
interface CardProps {
  title: string;
  children?: ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <div>
      <h2>{title}</h2>
      {children}
    </div>
  );
}

// Required children
interface SectionProps {
  children: ReactNode;
}

function Section({ children }: SectionProps) {
  if (!children) {
    throw new Error('Section requires children');
  }
  return <section>{children}</section>;
}
```

---

## Multiple Children Slots

```typescript
interface LayoutProps {
  header: ReactNode;
  sidebar: ReactNode;
  content: ReactNode;
  footer?: ReactNode;
}

function Layout({ header, sidebar, content, footer }: LayoutProps) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{content}</main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

// Usage
<Layout
  header={<Header />}
  sidebar={<Sidebar />}
  content={<MainContent />}
  footer={<Footer />}
/>
```

---

## Render Props Pattern

```typescript
interface DataListProps<T> {
  data: T[];
  renderItem: (item: T, index: number) => ReactNode;
  renderEmpty?: () => ReactNode;
}

function DataList<T>({ data, renderItem, renderEmpty }: DataListProps<T>) {
  if (data.length === 0 && renderEmpty) {
    return <>{renderEmpty()}</>;
  }

  return (
    <ul>
      {data.map((item, index) => (
        <li key={index}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Usage
<DataList
  data={users}
  renderItem={(user) => <UserCard user={user} />}
  renderEmpty={() => <p>No users found</p>}
/>
```

---

## Function as Children (FaCC)

```typescript
interface ToggleProps {
  children: (isOpen: boolean, toggle: () => void) => ReactNode;
}

function Toggle({ children }: ToggleProps) {
  const [isOpen, setIsOpen] = useState(false);
  const toggle = () => setIsOpen(!isOpen);

  return <>{children(isOpen, toggle)}</>;
}

// Usage
<Toggle>
  {(isOpen, toggle) => (
    <>
      <button onClick={toggle}>
        {isOpen ? 'Hide' : 'Show'}
      </button>
      {isOpen && <div>Content</div>}
    </>
  )}
</Toggle>
```

---

## Cloning and Modifying Children

```typescript
import { Children, cloneElement, ReactElement } from 'react';

interface ButtonGroupProps {
  size: 'small' | 'medium' | 'large';
  children: ReactElement | ReactElement[];
}

function ButtonGroup({ size, children }: ButtonGroupProps) {
  return (
    <div className="button-group">
      {Children.map(children, (child) => {
        if (!child) return null;

        // Clone and inject props
        return cloneElement(child, {
          size,
          ...child.props
        });
      })}
    </div>
  );
}

// Usage
<ButtonGroup size="large">
  <Button>One</Button>
  <Button>Two</Button>
</ButtonGroup>
```

---

## Validating Children Type

```typescript
import { Children, isValidElement, ReactElement } from 'react';

interface TabsProps {
  children: ReactNode;
}

function Tabs({ children }: TabsProps) {
  // Validate all children are Tab components
  Children.forEach(children, (child) => {
    if (!isValidElement(child) || child.type !== Tab) {
      throw new Error('Tabs only accepts Tab components as children');
    }
  });

  return <div className="tabs">{children}</div>;
}
```

---

## Children Utilities

```typescript
import { Children, ReactNode } from 'react';

interface ListProps {
  children: ReactNode;
}

function List({ children }: ListProps) {
  const count = Children.count(children);
  const array = Children.toArray(children);
  const first = array[0];
  const last = array[array.length - 1];

  return (
    <div>
      <div>Total items: {count}</div>
      <ul>
        {Children.map(children, (child, index) => (
          <li key={index}>{child}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Compound Components

```typescript
interface AccordionProps {
  children: ReactNode;
}

interface AccordionItemProps {
  title: string;
  children: ReactNode;
}

function Accordion({ children }: AccordionProps) {
  return <div className="accordion">{children}</div>;
}

function AccordionItem({ title, children }: AccordionItemProps) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="accordion-item">
      <button onClick={() => setIsOpen(!isOpen)}>
        {title}
      </button>
      {isOpen && <div>{children}</div>}
    </div>
  );
}

// Attach as namespace
Accordion.Item = AccordionItem;

// Usage
<Accordion>
  <Accordion.Item title="Section 1">
    Content 1
  </Accordion.Item>
  <Accordion.Item title="Section 2">
    Content 2
  </Accordion.Item>
</Accordion>
```

---

## Strict Children Typing

```typescript
// Accept only one child
interface SingleChildProps {
  children: ReactElement;
}

// Accept exactly two children
interface TwoChildrenProps {
  children: [ReactElement, ReactElement];
}

// Accept 1-3 children
interface OneToThreeProps {
  children: ReactElement | [ReactElement, ReactElement] | [ReactElement, ReactElement, ReactElement];
}

// Type-safe tuple children
interface SplitPanelProps {
  children: [ReactElement, ReactElement];
}

function SplitPanel({ children }: SplitPanelProps) {
  const [left, right] = children;

  return (
    <div className="split">
      <div className="left">{left}</div>
      <div className="right">{right}</div>
    </div>
  );
}

// Usage
<SplitPanel>
  <LeftPanel />
  <RightPanel />
</SplitPanel>
```

---

## Real-World: Modal with Slots

```typescript
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  children: {
    header?: ReactNode;
    body: ReactNode;
    footer?: ReactNode;
  };
}

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children.header && (
          <div className="modal-header">
            {children.header}
            <button onClick={onClose}>×</button>
          </div>
        )}

        <div className="modal-body">
          {children.body}
        </div>

        {children.footer && (
          <div className="modal-footer">
            {children.footer}
          </div>
        )}
      </div>
    </div>
  );
}

// Usage
<Modal
  isOpen={isOpen}
  onClose={handleClose}
  children={{
    header: <h2>Confirm Action</h2>,
    body: <p>Are you sure?</p>,
    footer: (
      <>
        <Button onClick={handleClose}>Cancel</Button>
        <Button onClick={handleConfirm}>Confirm</Button>
      </>
    )
  }}
/>
```

---

## Best Practices

✅ **Use `ReactNode` for maximum flexibility**  
✅ **Use `ReactElement` when you need JSX only**  
✅ **Use `PropsWithChildren` utility**  
✅ **Make children optional** unless required  
✅ **Validate children types** at runtime when needed  
✅ **Use render props** for complex logic sharing  
❌ **Don't use `any` for children**  
❌ **Don't mutate children** directly  
❌ **Don't forget null checks**

---

## Key Takeaways

1. **`ReactNode` is most flexible** - strings, numbers, JSX, null
2. **`ReactElement` only accepts JSX** elements
3. **`PropsWithChildren` adds children automatically**
4. **Render props enable** flexible composition
5. **Clone children to inject props**
6. **Validate children types** for compound components
7. **Multiple slots better than children** for complex layouts
