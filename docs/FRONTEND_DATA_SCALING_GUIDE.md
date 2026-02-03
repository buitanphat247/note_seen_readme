# 📊 Hướng Dẫn Mở Rộng Data Scale cho Frontend

**Ngày tạo:** 2026-01-22  
**Version:** 1.0  
**Mục tiêu:** Hướng dẫn chi tiết cách scale frontend application khi data tăng lên

---

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Caching Strategies](#caching-strategies)
3. [Pagination & Infinite Scroll](#pagination--infinite-scroll)
4. [Virtual Scrolling](#virtual-scrolling)
5. [Data Prefetching](#data-prefetching)
6. [Code Splitting & Lazy Loading](#code-splitting--lazy-loading)
7. [State Management Optimization](#state-management-optimization)
8. [API Optimization](#api-optimization)
9. [Memory Management](#memory-management)
10. [Performance Monitoring](#performance-monitoring)
11. [Best Practices](#best-practices)

---

## 🎯 Tổng Quan

Khi hệ thống phát triển, lượng data tăng lên đáng kể. Frontend cần được optimize để:
- ✅ Load nhanh hơn với data lớn
- ✅ Giảm memory usage
- ✅ Cải thiện user experience
- ✅ Tối ưu network requests
- ✅ Handle large datasets efficiently

---

## 💾 Caching Strategies

### 1. **Browser Caching**

#### HTTP Cache Headers
```typescript
// Next.js API Route
export async function GET(request: Request) {
  const data = await fetchData();
  
  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
      // s-maxage: Cache trên CDN 60s
      // stale-while-revalidate: Serve stale data trong 300s khi đang revalidate
    },
  });
}
```

#### Service Worker Caching
```typescript
// app/workers/cache-strategy.ts
export const CACHE_STRATEGIES = {
  // Cache first, fallback to network
  CACHE_FIRST: 'cache-first',
  // Network first, fallback to cache
  NETWORK_FIRST: 'network-first',
  // Stale while revalidate
  STALE_WHILE_REVALIDATE: 'stale-while-revalidate',
  // Network only
  NETWORK_ONLY: 'network-only',
};

// Implementation
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/data')) {
    event.respondWith(
      caches.open('data-cache-v1').then((cache) => {
        return cache.match(event.request).then((cachedResponse) => {
          if (cachedResponse) {
            // Return cached data immediately
            fetch(event.request).then((networkResponse) => {
              // Update cache in background
              cache.put(event.request, networkResponse.clone());
            });
            return cachedResponse;
          }
          // No cache, fetch from network
          return fetch(event.request).then((response) => {
            cache.put(event.request, response.clone());
            return response;
          });
        });
      })
    );
  }
});
```

### 2. **In-Memory Caching**

#### React Query / TanStack Query
```typescript
// lib/hooks/useDataCache.ts
import { useQuery, useQueryClient } from '@tanstack/react-query';

const CACHE_KEYS = {
  USER_LIST: ['users'],
  USER_DETAIL: (id: string) => ['user', id],
  POSTS: ['posts'],
  POST_DETAIL: (id: string) => ['post', id],
};

// Cache configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      refetchOnWindowFocus: false,
      refetchOnReconnect: true,
    },
  },
});

// Usage
export function useUserList() {
  return useQuery({
    queryKey: CACHE_KEYS.USER_LIST,
    queryFn: async () => {
      const response = await fetch('/api/users');
      return response.json();
    },
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
  });
}

// Prefetch data
export function usePrefetchUser(id: string) {
  const queryClient = useQueryClient();
  
  return useCallback(() => {
    queryClient.prefetchQuery({
      queryKey: CACHE_KEYS.USER_DETAIL(id),
      queryFn: async () => {
        const response = await fetch(`/api/users/${id}`);
        return response.json();
      },
    });
  }, [id, queryClient]);
}
```

#### Custom Cache Hook
```typescript
// lib/hooks/useCache.ts
import { useState, useEffect, useRef } from 'react';

interface CacheEntry<T> {
  data: T;
  timestamp: number;
  expiresAt: number;
}

export function useCache<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 5 * 60 * 1000 // 5 minutes default
) {
  const cacheRef = useRef<Map<string, CacheEntry<T>>>(new Map());
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const cached = cacheRef.current.get(key);
    const now = Date.now();

    // Check if cache is valid
    if (cached && cached.expiresAt > now) {
      setData(cached.data);
      setLoading(false);
      return;
    }

    // Fetch new data
    setLoading(true);
    fetcher()
      .then((result) => {
        cacheRef.current.set(key, {
          data: result,
          timestamp: now,
          expiresAt: now + ttl,
        });
        setData(result);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, [key, ttl]);

  // Cleanup expired cache entries
  useEffect(() => {
    const interval = setInterval(() => {
      const now = Date.now();
      for (const [key, entry] of cacheRef.current.entries()) {
        if (entry.expiresAt <= now) {
          cacheRef.current.delete(key);
        }
      }
    }, 60000); // Check every minute

    return () => clearInterval(interval);
  }, []);

  return { data, loading, error };
}
```

### 3. **IndexedDB Caching (Large Data)**

```typescript
// lib/utils/indexeddb-cache.ts
export class IndexedDBCache {
  private dbName = 'app-cache';
  private version = 1;
  private db: IDBDatabase | null = null;

  async init(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        if (!db.objectStoreNames.contains('data')) {
          db.createObjectStore('data', { keyPath: 'key' });
        }
      };
    });
  }

  async set<T>(key: string, data: T, ttl: number = 24 * 60 * 60 * 1000): Promise<void> {
    if (!this.db) await this.init();

    const entry = {
      key,
      data,
      timestamp: Date.now(),
      expiresAt: Date.now() + ttl,
    };

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['data'], 'readwrite');
      const store = transaction.objectStore('data');
      const request = store.put(entry);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async get<T>(key: string): Promise<T | null> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['data'], 'readonly');
      const store = transaction.objectStore('data');
      const request = store.get(key);

      request.onsuccess = () => {
        const entry = request.result;
        if (!entry) {
          resolve(null);
          return;
        }

        // Check if expired
        if (entry.expiresAt <= Date.now()) {
          this.delete(key);
          resolve(null);
          return;
        }

        resolve(entry.data);
      };

      request.onerror = () => reject(request.error);
    });
  }

  async delete(key: string): Promise<void> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['data'], 'readwrite');
      const store = transaction.objectStore('data');
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async clear(): Promise<void> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['data'], 'readwrite');
      const store = transaction.objectStore('data');
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }
}
```

---

## 📄 Pagination & Infinite Scroll

### 1. **Cursor-based Pagination**

```typescript
// lib/hooks/useInfiniteScroll.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseInfiniteScrollOptions<T> {
  fetcher: (cursor?: string) => Promise<{
    data: T[];
    nextCursor?: string;
    hasMore: boolean;
  }>;
  threshold?: number; // Trigger when 200px from bottom
}

export function useInfiniteScroll<T>({
  fetcher,
  threshold = 200,
}: UseInfiniteScrollOptions<T>) {
  const [data, setData] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [cursor, setCursor] = useState<string | undefined>();
  const [hasMore, setHasMore] = useState(true);
  const observerRef = useRef<IntersectionObserver | null>(null);
  const loadMoreRef = useRef<HTMLDivElement | null>(null);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;

    setLoading(true);
    setError(null);

    try {
      const result = await fetcher(cursor);
      setData((prev) => [...prev, ...result.data]);
      setCursor(result.nextCursor);
      setHasMore(result.hasMore);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [cursor, fetcher, hasMore, loading]);

  useEffect(() => {
    loadMore();
  }, []); // Initial load

  useEffect(() => {
    if (!loadMoreRef.current || !hasMore) return;

    observerRef.current = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasMore && !loading) {
          loadMore();
        }
      },
      { rootMargin: `${threshold}px` }
    );

    observerRef.current.observe(loadMoreRef.current);

    return () => {
      if (observerRef.current) {
        observerRef.current.disconnect();
      }
    };
  }, [hasMore, loading, loadMore, threshold]);

  return {
    data,
    loading,
    error,
    hasMore,
    loadMoreRef,
    reload: () => {
      setData([]);
      setCursor(undefined);
      setHasMore(true);
      loadMore();
    },
  };
}

// Usage
function UserList() {
  const {
    data: users,
    loading,
    hasMore,
    loadMoreRef,
  } = useInfiniteScroll({
    fetcher: async (cursor) => {
      const response = await fetch(
        `/api/users?cursor=${cursor || ''}&limit=20`
      );
      const result = await response.json();
      return {
        data: result.users,
        nextCursor: result.nextCursor,
        hasMore: result.hasMore,
      };
    },
  });

  return (
    <div>
      {users.map((user) => (
        <UserCard key={user.id} user={user} />
      ))}
      {hasMore && <div ref={loadMoreRef}>Loading more...</div>}
      {loading && <Spinner />}
    </div>
  );
}
```

### 2. **Offset-based Pagination**

```typescript
// lib/hooks/usePagination.ts
import { useState, useCallback } from 'react';

interface UsePaginationOptions<T> {
  fetcher: (page: number, pageSize: number) => Promise<{
    data: T[];
    total: number;
  }>;
  pageSize?: number;
  initialPage?: number;
}

export function usePagination<T>({
  fetcher,
  pageSize = 20,
  initialPage = 1,
}: UsePaginationOptions<T>) {
  const [data, setData] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [page, setPage] = useState(initialPage);
  const [total, setTotal] = useState(0);

  const totalPages = Math.ceil(total / pageSize);

  const fetchPage = useCallback(async (pageNum: number) => {
    setLoading(true);
    setError(null);

    try {
      const result = await fetcher(pageNum, pageSize);
      setData(result.data);
      setTotal(result.total);
      setPage(pageNum);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [fetcher, pageSize]);

  const goToPage = useCallback((pageNum: number) => {
    if (pageNum >= 1 && pageNum <= totalPages) {
      fetchPage(pageNum);
    }
  }, [fetchPage, totalPages]);

  const nextPage = useCallback(() => {
    if (page < totalPages) {
      goToPage(page + 1);
    }
  }, [page, totalPages, goToPage]);

  const prevPage = useCallback(() => {
    if (page > 1) {
      goToPage(page - 1);
    }
  }, [page, goToPage]);

  return {
    data,
    loading,
    error,
    page,
    totalPages,
    total,
    goToPage,
    nextPage,
    prevPage,
    refresh: () => fetchPage(page),
  };
}
```

---

## 🎯 Virtual Scrolling

### React Window / React Virtualized

```typescript
// components/VirtualizedList.tsx
import { FixedSizeList as List } from 'react-window';
import { useMemo } from 'react';

interface VirtualizedListProps<T> {
  items: T[];
  height: number;
  itemHeight: number;
  renderItem: (props: { index: number; style: React.CSSProperties }) => React.ReactNode;
}

export function VirtualizedList<T>({
  items,
  height,
  itemHeight,
  renderItem,
}: VirtualizedListProps<T>) {
  const Row = useMemo(
    () => ({ index, style }: { index: number; style: React.CSSProperties }) => {
      return <div style={style}>{renderItem({ index, style })}</div>;
    },
    [renderItem]
  );

  return (
    <List
      height={height}
      itemCount={items.length}
      itemSize={itemHeight}
      width="100%"
    >
      {Row}
    </List>
  );
}

// Usage với infinite scroll
function LargeUserList() {
  const { data, loading, hasMore, loadMoreRef } = useInfiniteScroll({
    fetcher: async (cursor) => {
      // ... fetch logic
    },
  });

  return (
    <VirtualizedList
      items={data}
      height={600}
      itemHeight={80}
      renderItem={({ index, style }) => (
        <div style={style}>
          <UserCard user={data[index]} />
        </div>
      )}
    />
  );
}
```

---

## 🚀 Data Prefetching

### 1. **Link Prefetching**

```typescript
// components/PrefetchLink.tsx
import Link from 'next/link';
import { useRouter } from 'next/navigation';
import { useCallback } from 'react';

interface PrefetchLinkProps {
  href: string;
  children: React.ReactNode;
  prefetchData?: () => Promise<void>;
}

export function PrefetchLink({ href, children, prefetchData }: PrefetchLinkProps) {
  const router = useRouter();

  const handleMouseEnter = useCallback(() => {
    // Prefetch route
    router.prefetch(href);
    
    // Prefetch data if provided
    if (prefetchData) {
      prefetchData();
    }
  }, [href, router, prefetchData]);

  return (
    <Link href={href} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

### 2. **Viewport-based Prefetching**

```typescript
// lib/hooks/useViewportPrefetch.ts
import { useEffect, useRef } from 'react';

export function useViewportPrefetch(
  fetcher: () => Promise<void>,
  options: { threshold?: number } = {}
) {
  const ref = useRef<HTMLDivElement>(null);
  const { threshold = 0.1 } = options;

  useEffect(() => {
    if (!ref.current) return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          fetcher();
          observer.disconnect();
        }
      },
      { threshold }
    );

    observer.observe(ref.current);

    return () => observer.disconnect();
  }, [fetcher, threshold]);

  return ref;
}

// Usage
function UserCard({ userId }: { userId: string }) {
  const prefetchRef = useViewportPrefetch(async () => {
    // Prefetch user details when card enters viewport
    await prefetchUserDetails(userId);
  });

  return <div ref={prefetchRef}>User Card</div>;
}
```

---

## 📦 Code Splitting & Lazy Loading

### 1. **Route-based Code Splitting**

```typescript
// app/(root)/features/[type]/page.tsx
import dynamic from 'next/dynamic';

// Lazy load heavy components
const VocabularyFeature = dynamic(() => import('@/components/features/vocabulary/VocabularyFeature'), {
  loading: () => <VocabularySkeleton />,
  ssr: false, // Disable SSR if not needed
});

const WritingFeature = dynamic(() => import('@/components/features/writing/WritingFeature'), {
  loading: () => <WritingSkeleton />,
});

export default function FeaturePage({ params }: { params: { type: string } }) {
  const FeatureComponent = {
    vocabulary: VocabularyFeature,
    writing: WritingFeature,
    listening: () => import('@/components/features/listening/ListeningFeature'),
  }[params.type];

  return <FeatureComponent />;
}
```

### 2. **Component-based Lazy Loading**

```typescript
// lib/utils/lazyLoad.ts
import { lazy, ComponentType } from 'react';

export function lazyLoad<T extends ComponentType<any>>(
  importFunc: () => Promise<{ default: T }>,
  fallback?: React.ReactNode
) {
  return lazy(importFunc);
}

// Usage
const HeavyChart = lazyLoad(
  () => import('@/components/charts/HeavyChart'),
  <ChartSkeleton />
);
```

### 3. **Data-based Lazy Loading**

```typescript
// lib/hooks/useLazyData.ts
import { useState, useEffect, useCallback } from 'react';

export function useLazyData<T>(
  fetcher: () => Promise<T>,
  options: { enabled?: boolean } = {}
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const load = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const result = await fetcher();
      setData(result);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [fetcher]);

  useEffect(() => {
    if (options.enabled !== false) {
      load();
    }
  }, [load, options.enabled]);

  return { data, loading, error, reload: load };
}
```

---

## 🗄️ State Management Optimization

### 1. **Context Splitting**

```typescript
// app/context/SocialDataContext.tsx
export const SocialDataContext = createContext<SocialDataContextType | undefined>(undefined);

// app/context/SocialActionsContext.tsx
export const SocialActionsContext = createContext<SocialActionsContextType | undefined>(undefined);

// Components chỉ subscribe context cần thiết
function UserList() {
  // Chỉ re-render khi data thay đổi, không re-render khi actions thay đổi
  const { users } = useContext(SocialDataContext);
  return <div>{/* ... */}</div>;
}
```

### 2. **Selective State Updates**

```typescript
// lib/hooks/useSelectiveState.ts
import { useState, useCallback } from 'react';

export function useSelectiveState<T>(initialState: T) {
  const [state, setState] = useState<T>(initialState);

  const updateField = useCallback(<K extends keyof T>(field: K, value: T[K]) => {
    setState((prev) => ({
      ...prev,
      [field]: value,
    }));
  }, []);

  const updateFields = useCallback((fields: Partial<T>) => {
    setState((prev) => ({
      ...prev,
      ...fields,
    }));
  }, []);

  return { state, setState, updateField, updateFields };
}
```

---

## 🌐 API Optimization

### 1. **Request Batching**

```typescript
// lib/utils/requestBatcher.ts
class RequestBatcher {
  private batch: Map<string, Promise<any>> = new Map();
  private batchTimeout: number = 50; // 50ms

  async batchRequest<T>(
    key: string,
    fetcher: () => Promise<T>
  ): Promise<T> {
    if (this.batch.has(key)) {
      return this.batch.get(key)!;
    }

    const promise = fetcher().finally(() => {
      this.batch.delete(key);
    });

    this.batch.set(key, promise);
    return promise;
  }
}

export const requestBatcher = new RequestBatcher();
```

### 2. **Request Deduplication**

```typescript
// lib/utils/requestDeduplicator.ts
class RequestDeduplicator {
  private pending: Map<string, Promise<any>> = new Map();

  async deduplicate<T>(
    key: string,
    fetcher: () => Promise<T>
  ): Promise<T> {
    if (this.pending.has(key)) {
      return this.pending.get(key)!;
    }

    const promise = fetcher().finally(() => {
      this.pending.delete(key);
    });

    this.pending.set(key, promise);
    return promise;
  }
}

export const requestDeduplicator = new RequestDeduplicator();
```

### 3. **GraphQL DataLoader Pattern**

```typescript
// lib/utils/dataLoader.ts
class DataLoader<TKey, TValue> {
  private batch: TKey[] = [];
  private cache: Map<TKey, TValue> = new Map();
  private batchFn: (keys: TKey[]) => Promise<TValue[]>;
  private batchTimeout: number = 50;

  constructor(
    batchFn: (keys: TKey[]) => Promise<TValue[]>,
    options: { batchTimeout?: number } = {}
  ) {
    this.batchFn = batchFn;
    this.batchTimeout = options.batchTimeout || 50;
  }

  async load(key: TKey): Promise<TValue> {
    // Check cache
    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }

    // Add to batch
    this.batch.push(key);

    // Schedule batch
    return new Promise((resolve, reject) => {
      setTimeout(async () => {
        const keys = [...this.batch];
        this.batch = [];

        try {
          const values = await this.batchFn(keys);
          keys.forEach((k, i) => {
            this.cache.set(k, values[i]);
            if (k === key) {
              resolve(values[i]);
            }
          });
        } catch (error) {
          reject(error);
        }
      }, this.batchTimeout);
    });
  }

  clear(key?: TKey) {
    if (key) {
      this.cache.delete(key);
    } else {
      this.cache.clear();
    }
  }
}
```

---

## 🧠 Memory Management

### 1. **Cleanup Unused Data**

```typescript
// lib/hooks/useCleanup.ts
import { useEffect, useRef } from 'react';

export function useCleanup(cleanupFn: () => void) {
  const cleanupRef = useRef(cleanupFn);

  useEffect(() => {
    cleanupRef.current = cleanupFn;
  }, [cleanupFn]);

  useEffect(() => {
    return () => {
      cleanupRef.current();
    };
  }, []);
}

// Usage
function LargeDataComponent() {
  const dataRef = useRef<LargeData[]>([]);

  useCleanup(() => {
    // Clear large data when component unmounts
    dataRef.current = [];
  });

  return <div>{/* ... */}</div>;
}
```

### 2. **WeakMap for Temporary Data**

```typescript
// lib/utils/weakCache.ts
const weakCache = new WeakMap<object, any>();

export function getCached<T>(key: object, fetcher: () => T): T {
  if (weakCache.has(key)) {
    return weakCache.get(key);
  }

  const value = fetcher();
  weakCache.set(key, value);
  return value;
}
```

---

## 📊 Performance Monitoring

### 1. **Web Vitals Tracking**

```typescript
// lib/utils/performance.ts
import { onCLS, onFID, onFCP, onLCP, onTTFB, onINP } from 'next/web-vitals';

export function trackWebVitals() {
  onCLS(console.log);
  onFID(console.log);
  onFCP(console.log);
  onLCP(console.log);
  onTTFB(console.log);
  onINP(console.log);
}
```

### 2. **Custom Performance Metrics**

```typescript
// lib/utils/customMetrics.ts
export function measurePerformance(name: string, fn: () => void) {
  const start = performance.now();
  fn();
  const end = performance.now();
  const duration = end - start;

  // Log to analytics
  if (typeof window !== 'undefined' && window.gtag) {
    window.gtag('event', 'performance', {
      event_category: 'Performance',
      event_label: name,
      value: Math.round(duration),
    });
  }

  return duration;
}
```

---

## ✅ Best Practices

### 1. **Data Loading Priorities**

```typescript
// Priority levels
const PRIORITY = {
  CRITICAL: 1, // Above the fold, initial render
  HIGH: 2,     // Visible on scroll
  MEDIUM: 3,   // Prefetch on hover
  LOW: 4,      // Lazy load on demand
};

// Load critical data first
async function loadPageData() {
  const [critical, high, medium] = await Promise.all([
    loadCriticalData(), // Above the fold
    loadHighPriorityData(), // Visible on scroll
    loadMediumPriorityData(), // Prefetch
  ]);

  return { critical, high, medium };
}
```

### 2. **Progressive Data Loading**

```typescript
// Load data progressively
async function loadProgressiveData() {
  // Step 1: Load skeleton data
  const skeleton = await loadSkeletonData();
  renderSkeleton(skeleton);

  // Step 2: Load essential data
  const essential = await loadEssentialData();
  renderEssential(essential);

  // Step 3: Load additional data
  const additional = await loadAdditionalData();
  renderAdditional(additional);
}
```

### 3. **Error Boundaries for Data Loading**

```typescript
// components/DataErrorBoundary.tsx
export class DataErrorBoundary extends Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  render() {
    if (this.state.hasError) {
      return <DataErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

---

## 📝 Summary

### Key Strategies:

1. **Caching**: Browser cache, in-memory cache, IndexedDB
2. **Pagination**: Cursor-based, offset-based, infinite scroll
3. **Virtual Scrolling**: Render only visible items
4. **Prefetching**: Link prefetch, viewport prefetch
5. **Code Splitting**: Route-based, component-based
6. **State Optimization**: Context splitting, selective updates
7. **API Optimization**: Batching, deduplication, DataLoader
8. **Memory Management**: Cleanup, WeakMap
9. **Performance Monitoring**: Web Vitals, custom metrics

### Implementation Order:

1. **Phase 1**: Implement caching và pagination
2. **Phase 2**: Add virtual scrolling cho large lists
3. **Phase 3**: Optimize state management
4. **Phase 4**: Add prefetching và code splitting
5. **Phase 5**: Monitor và optimize based on metrics

---

**Last Updated:** 2026-01-22  
**Author:** AI Code Reviewer  
**Version:** 1.0
