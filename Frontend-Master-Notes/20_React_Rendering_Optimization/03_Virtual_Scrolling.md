# Virtual Scrolling and Windowing

## Core Concept

Virtual scrolling (windowing) only renders visible items in a large list, dramatically improving performance for thousands of items by reducing DOM nodes.

---

## The Problem

```typescript
// ❌ BAD - renders 10,000 DOM nodes
function BadList({ items }: { items: Item[] }) {
  return (
    <div style={{ height: '500px', overflow: 'auto' }}>
      {items.map(item => (
        <div key={item.id} style={{ height: '50px' }}>
          {item.name}
        </div>
      ))}
    </div>
  );
}

// Problem: If items.length = 10,000
// - 10,000 DOM nodes created
// - Slow initial render
// - Slow scrolling
// - High memory usage
```

---

## react-window

Lightweight virtual scrolling:

```typescript
import { FixedSizeList } from 'react-window';

interface Item {
  id: number;
  name: string;
}

interface Props {
  items: Item[];
}

function VirtualList({ items }: Props) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <FixedSizeList
      height={500}        // Container height
      itemCount={items.length}
      itemSize={50}       // Each row height
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// Renders only ~10 visible items at a time!
```

---

## Variable Size Lists

Items with different heights:

```typescript
import { VariableSizeList } from 'react-window';

function VariableList({ items }: Props) {
  // Function to get item height
  const getItemSize = (index: number) => {
    return items[index].type === 'header' ? 80 : 50;
  };

  const Row = ({ index, style }: any) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <VariableSizeList
      height={500}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

---

## Grid Virtualization

For two-dimensional data:

```typescript
import { FixedSizeGrid } from 'react-window';

function VirtualGrid() {
  const Cell = ({ columnIndex, rowIndex, style }: any) => (
    <div style={style}>
      Row {rowIndex}, Col {columnIndex}
    </div>
  );

  return (
    <FixedSizeGrid
      columnCount={1000}
      columnWidth={100}
      height={600}
      rowCount={1000}
      rowHeight={35}
      width={800}
    >
      {Cell}
    </FixedSizeGrid>
  );
}
```

---

## react-virtualized

More feature-rich library:

```typescript
import { List, AutoSizer } from 'react-virtualized';

function AutoSizedList({ items }: Props) {
  const rowRenderer = ({ key, index, style }: any) => (
    <div key={key} style={style}>
      {items[index].name}
    </div>
  );

  return (
    <AutoSizer>
      {({ height, width }) => (
        <List
          height={height}
          rowCount={items.length}
          rowHeight={50}
          rowRenderer={rowRenderer}
          width={width}
        />
      )}
    </AutoSizer>
  );
}
```

---

## Infinite Loading

Combine with data fetching:

```typescript
import { FixedSizeList } from 'react-window';
import InfiniteLoader from 'react-window-infinite-loader';

function InfiniteList() {
  const [items, setItems] = useState<Item[]>([]);
  const [hasMore, setHasMore] = useState(true);

  const loadMoreItems = async (startIndex: number, stopIndex: number) => {
    const newItems = await fetchItems(startIndex, stopIndex);
    setItems(prev => [...prev, ...newItems]);
    if (newItems.length === 0) setHasMore(false);
  };

  const isItemLoaded = (index: number) => !hasMore || index < items.length;

  const Row = ({ index, style }: any) => {
    if (!isItemLoaded(index)) {
      return <div style={style}>Loading...</div>;
    }
    return <div style={style}>{items[index].name}</div>;
  };

  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={hasMore ? items.length + 1 : items.length}
      loadMoreItems={loadMoreItems}
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          height={500}
          itemCount={items.length}
          itemSize={50}
          onItemsRendered={onItemsRendered}
          ref={ref}
          width="100%"
        >
          {Row}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

---

## Scrolling to Items

```typescript
import { useRef } from 'react';
import { FixedSizeList } from 'react-window';

function ScrollableList({ items }: Props) {
  const listRef = useRef<FixedSizeList>(null);

  const scrollToItem = (index: number) => {
    listRef.current?.scrollToItem(index, 'center');
  };

  const Row = ({ index, style }: any) => (
    <div style={style}>{items[index].name}</div>
  );

  return (
    <>
      <button onClick={() => scrollToItem(500)}>
        Scroll to Item 500
      </button>
      <FixedSizeList
        ref={listRef}
        height={500}
        itemCount={items.length}
        itemSize={50}
        width="100%"
      >
        {Row}
      </FixedSizeList>
    </>
  );
}
```

---

## Dynamic Heights with Measurement

```typescript
import { VariableSizeList } from 'react-window';
import { useRef, useEffect } from 'react';

function DynamicHeightList({ items }: Props) {
  const listRef = useRef<VariableSizeList>(null);
  const rowHeights = useRef<{ [key: number]: number }>({});

  const getItemSize = (index: number) => {
    return rowHeights.current[index] || 50; // Default 50px
  };

  const setRowHeight = (index: number, height: number) => {
    if (rowHeights.current[index] !== height) {
      rowHeights.current[index] = height;
      listRef.current?.resetAfterIndex(index);
    }
  };

  const Row = ({ index, style }: any) => {
    const rowRef = useRef<HTMLDivElement>(null);

    useEffect(() => {
      if (rowRef.current) {
        setRowHeight(index, rowRef.current.clientHeight);
      }
    }, [index]);

    return (
      <div style={style}>
        <div ref={rowRef}>
          {items[index].content}
        </div>
      </div>
    );
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={500}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

---

## Real-World: Chat Messages

```typescript
import { VariableSizeList } from 'react-window';

interface Message {
  id: string;
  text: string;
  sender: string;
  timestamp: Date;
}

function ChatWindow({ messages }: { messages: Message[] }) {
  const listRef = useRef<VariableSizeList>(null);

  // Scroll to bottom on new message
  useEffect(() => {
    listRef.current?.scrollToItem(messages.length - 1);
  }, [messages.length]);

  const getItemSize = (index: number) => {
    const message = messages[index];
    const lines = Math.ceil(message.text.length / 50);
    return 60 + (lines * 20); // Base height + text lines
  };

  const Row = ({ index, style }: any) => {
    const message = messages[index];
    return (
      <div style={style} className="message">
        <div className="sender">{message.sender}</div>
        <div className="text">{message.text}</div>
        <div className="time">{message.timestamp.toLocaleTimeString()}</div>
      </div>
    );
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      itemCount={messages.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

---

## Performance Comparison

```typescript
// Without virtualization
// 10,000 items × 50px = 500,000px tall
// 10,000 DOM nodes
// Initial render: ~2000ms
// Memory: ~50MB

// With virtualization
// Renders ~15 visible items
// 15 DOM nodes
// Initial render: ~50ms
// Memory: ~5MB

// 40x faster! 10x less memory!
```

---

## Best Practices

✅ **Use for 100+ items** - overhead not worth it for small lists  
✅ **Fixed sizes when possible** - faster than variable  
✅ **Measure dynamic heights** accurately  
✅ **Reset cache** when content changes  
✅ **Add overscan** for smoother scrolling  
❌ **Don't virtualize small lists** - adds complexity  
❌ **Don't forget item keys** - causes bugs  
❌ **Don't use with CSS animations** - breaks positioning

---

## Overscan

Render extra items above/below viewport:

```typescript
<FixedSizeList
  height={500}
  itemCount={items.length}
  itemSize={50}
  overscanCount={5}  // Render 5 extra items above/below
  width="100%"
>
  {Row}
</FixedSizeList>
```

---

## Key Takeaways

1. **Virtualization renders only visible items**
2. **react-window is lightweight** and fast
3. **FixedSizeList for uniform heights**
4. **VariableSizeList for dynamic heights**
5. **Combine with infinite loading** for large datasets
6. **Use overscan** for smoother scrolling
7. **40x performance improvement** for large lists
