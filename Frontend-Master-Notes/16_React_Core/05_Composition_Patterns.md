# Composition Patterns in React

## Core Concept

Composition is React's way of building complex UIs from smaller, reusable components. It favors **composition over inheritance**.

---

## Children Prop

```typescript
interface Props {
  title: string;
  children: ReactNode;
}

function Card({ title, children }: Props) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="content">{children}</div>
    </div>
  );
}

// Usage
<Card title="My Card">
  <p>Card content here</p>
  <button>Action</button>
</Card>
```

---

## Render Props

```typescript
interface Props {
  data: number[];
  render: (data: number[]) => ReactNode;
}

function DataProvider({ data, render }: Props) {
  return <div>{render(data)}</div>;
}

// Usage
<DataProvider
  data={[1, 2, 3]}
  render={(data) => (
    <ul>
      {data.map(item => <li key={item}>{item}</li>)}
    </ul>
  )}
/>
```

---

## Compound Components

```typescript
interface TabsContextType {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}

const TabsContext = createContext<TabsContextType | null>(null);

function Tabs({ children }: { children: ReactNode }) {
  const [activeTab, setActiveTab] = useState('tab1');

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: ReactNode }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ id, children }: { id: string; children: ReactNode }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('Tab must be used within Tabs');

  const { activeTab, setActiveTab } = context;

  return (
    <button
      className={activeTab === id ? 'active' : ''}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const context = useContext(TabsContext);
  if (!context) throw new Error('TabPanel must be used within Tabs');

  const { activeTab } = context;

  if (activeTab !== id) return null;
  return <div>{children}</div>;
}

// Export compound component
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

export { Tabs };

// Usage
<Tabs>
  <Tabs.List>
    <Tabs.Tab id="tab1">Tab 1</Tabs.Tab>
    <Tabs.Tab id="tab2">Tab 2</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="tab1">Content 1</Tabs.Panel>
  <Tabs.Panel id="tab2">Content 2</Tabs.Panel>
</Tabs>
```

---

## Container/Presentational Pattern

```typescript
// Presentational (UI only)
interface UserListProps {
  users: User[];
  onUserClick: (user: User) => void;
}

function UserList({ users, onUserClick }: UserListProps) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Container (logic)
function UserListContainer() {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    fetchUsers().then(setUsers);
  }, []);

  const handleUserClick = (user: User) => {
    console.log('Clicked:', user.name);
  };

  return <UserList users={users} onUserClick={handleUserClick} />;
}
```

---

## Slot Pattern

```typescript
interface LayoutProps {
  header: ReactNode;
  sidebar: ReactNode;
  main: ReactNode;
  footer: ReactNode;
}

function Layout({ header, sidebar, main, footer }: LayoutProps) {
  return (
    <div className="layout">
      <header>{header}</header>
      <div className="content">
        <aside>{sidebar}</aside>
        <main>{main}</main>
      </div>
      <footer>{footer}</footer>
    </div>
  );
}

// Usage
<Layout
  header={<Header />}
  sidebar={<Sidebar />}
  main={<MainContent />}
  footer={<Footer />}
/>
```

---

## Best Practices

✅ **Favor composition** over inheritance  
✅ **Use children prop** for generic containers  
✅ **Render props** for sharing stateful logic  
✅ **Compound components** for related UI elements  
✅ **Slots** for flexible layouts  
❌ **Don't overuse** - keep it simple when possible

---

## Key Takeaways

1. **Children prop** is the simplest composition
2. **Render props** share logic flexibly
3. **Compound components** model complex relationships
4. **Container/Presentational** separates logic from UI
5. **Composition > Inheritance** in React
