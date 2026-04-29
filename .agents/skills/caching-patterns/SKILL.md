# Caching Patterns

Common caching strategies for Node.js/TypeScript applications.

## In-Memory Cache with TTL

```typescript
interface CacheEntry<T> {
  value: T;
  expiresAt: number;
}

class MemoryCache<T = unknown> {
  private store = new Map<string, CacheEntry<T>>();
  private defaultTTL: number;

  constructor(defaultTTLSeconds = 60) {
    this.defaultTTL = defaultTTLSeconds * 1000;
  }

  set(key: string, value: T, ttlSeconds?: number): void {
    const ttl = ttlSeconds ? ttlSeconds * 1000 : this.defaultTTL;
    this.store.set(key, {
      value,
      expiresAt: Date.now() + ttl,
    });
  }

  get(key: string): T | null {
    const entry = this.store.get(key);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return null;
    }
    return entry.value;
  }

  delete(key: string): void {
    this.store.delete(key);
  }

  // Remove expired entries
  purge(): void {
    const now = Date.now();
    for (const [key, entry] of this.store.entries()) {
      if (now > entry.expiresAt) this.store.delete(key);
    }
  }
}
```

## Cache-Aside Pattern

```typescript
async function getOrFetch<T>(
  cache: MemoryCache<T>,
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds?: number
): Promise<T> {
  const cached = cache.get(key);
  if (cached !== null) return cached;

  const value = await fetcher();
  cache.set(key, value, ttlSeconds);
  return value;
}

// Usage
const userCache = new MemoryCache<User>(300);

async function getUser(id: string): Promise<User> {
  return getOrFetch(
    userCache,
    `user:${id}`,
    () => db.users.findById(id),
    300
  );
}
```

## Redis Cache Wrapper

```typescript
import { Redis } from 'ioredis';

class RedisCache {
  constructor(private client: Redis) {}

  async get<T>(key: string): Promise<T | null> {
    const raw = await this.client.get(key);
    if (!raw) return null;
    try {
      return JSON.parse(raw) as T;
    } catch {
      return null;
    }
  }

  async set<T>(key: string, value: T, ttlSeconds?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    if (ttlSeconds) {
      await this.client.setex(key, ttlSeconds, serialized);
    } else {
      await this.client.set(key, serialized);
    }
  }

  async delete(key: string): Promise<void> {
    await this.client.del(key);
  }

  // Invalidate all keys matching a pattern
  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern);
    if (keys.length > 0) {
      await this.client.del(...keys);
    }
  }
}
```

## Memoize Decorator

```typescript
function memoize<T extends (...args: unknown[]) => unknown>(
  fn: T,
  keyFn?: (...args: Parameters<T>) => string
): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>) => {
    const key = keyFn ? keyFn(...args) : JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args) as ReturnType<T>;
    cache.set(key, result);
    return result;
  }) as T;
}

// Async version
function memoizeAsync<T extends (...args: unknown[]) => Promise<unknown>>(
  fn: T
): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    const promise = fn(...args) as ReturnType<T>;
    cache.set(key, promise);
    // Don't cache rejected promises
    promise.catch(() => cache.delete(key));
    return promise;
  }) as T;
}
```

## Notes

- Prefer Redis for distributed/multi-instance deployments
- Use `MemoryCache` for single-process or short-lived data
- Always set TTLs — unbounded caches cause memory leaks
- Invalidate by pattern when updating related entities
- Memoize pure or near-pure functions; avoid for side-effectful ones
