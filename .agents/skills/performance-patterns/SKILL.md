# Performance Patterns Skill

Patterns for optimizing JavaScript/TypeScript application performance including caching, memoization, lazy loading, and efficient data handling.

## Patterns

### Memoization

```typescript
// Generic memoize function with TTL support
function memoize<T extends (...args: any[]) => any>(
  fn: T,
  options: { ttl?: number; maxSize?: number } = {}
): T {
  const { ttl, maxSize = 100 } = options;
  const cache = new Map<string, { value: ReturnType<T>; expires?: number }>();

  return function (...args: Parameters<T>): ReturnType<T> {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached) {
      if (!cached.expires || cached.expires > Date.now()) {
        return cached.value;
      }
      cache.delete(key);
    }

    const value = fn(...args);

    if (cache.size >= maxSize) {
      // Evict oldest entry
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, {
      value,
      expires: ttl ? Date.now() + ttl : undefined,
    });

    return value;
  } as T;
}
```

### Debounce & Throttle

```typescript
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout>;
  return function (...args: Parameters<T>) {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

function throttle<T extends (...args: any[]) => any>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  return function (...args: Parameters<T>) {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}
```

### Lazy Loading

```typescript
// Lazy singleton factory
function lazy<T>(factory: () => T): () => T {
  let instance: T | undefined;
  return () => {
    if (instance === undefined) {
      instance = factory();
    }
    return instance;
  };
}

// Example: lazy database connection
const getDb = lazy(() => createDatabaseConnection(process.env.DATABASE_URL!));
```

### Batch Processing

```typescript
// Process items in chunks to avoid blocking the event loop
async function processInBatches<T, R>(
  items: T[],
  processor: (batch: T[]) => Promise<R[]>,
  batchSize = 50
): Promise<R[]> {
  const results: R[] = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await processor(batch);
    results.push(...batchResults);

    // Yield to event loop between batches
    if (i + batchSize < items.length) {
      await new Promise((resolve) => setImmediate(resolve));
    }
  }

  return results;
}
```

### Request Deduplication

```typescript
// Deduplicate concurrent identical requests
class RequestDeduplicator {
  private inflight = new Map<string, Promise<any>>();

  async dedupe<T>(key: string, fn: () => Promise<T>): Promise<T> {
    if (this.inflight.has(key)) {
      return this.inflight.get(key) as Promise<T>;
    }

    const promise = fn().finally(() => this.inflight.delete(key));
    this.inflight.set(key, promise);
    return promise;
  }
}

export const deduplicator = new RequestDeduplicator();

// Usage
// const user = await deduplicator.dedupe(`user:${id}`, () => fetchUser(id));
```

### Virtual List (React)

```tsx
import { useRef, useState, useCallback } from 'react';

interface UseVirtualListOptions {
  itemHeight: number;
  overscan?: number;
}

function useVirtualList<T>(
  items: T[],
  containerHeight: number,
  { itemHeight, overscan = 3 }: UseVirtualListOptions
) {
  const [scrollTop, setScrollTop] = useState(0);

  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const endIndex = Math.min(
    items.length - 1,
    Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan
  );

  const visibleItems = items.slice(startIndex, endIndex + 1).map((item, i) => ({
    item,
    index: startIndex + i,
    offsetTop: (startIndex + i) * itemHeight,
  }));

  const totalHeight = items.length * itemHeight;

  const onScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  }, []);

  return { visibleItems, totalHeight, onScroll };
}
```

## Usage Guidelines

- Use `memoize` for expensive pure functions; avoid for functions with side effects
- Apply `debounce` to user input handlers (search, resize); `throttle` for scroll/mousemove
- Use `processInBatches` when handling large datasets to keep the app responsive
- Apply `RequestDeduplicator` in service layers to prevent redundant API calls
- Prefer `lazy` for expensive initializations that may not always be needed
- Profile before optimizing — measure, then apply the appropriate pattern
