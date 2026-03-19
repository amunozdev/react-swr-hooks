# SWR Patterns

## Fetcher Architecture

SWR requires an explicit fetcher function. Define a typed, reusable fetcher at module scope:

```typescript
// lib/swr-fetcher.ts
const fetcher = async <T>(url: string): Promise<T> => {
  const res = await fetch(url);
  if (!res.ok) {
    const error = new Error('Fetch failed');
    (error as any).status = res.status;
    throw error;
  }
  return res.json();
};

export default fetcher;
```

Pass this fetcher to individual `useSWR` calls or set it globally via `SWRConfig`:

```typescript
<SWRConfig value={{ fetcher }}>
  {children}
</SWRConfig>
```

**Rule:** Never define fetchers inline inside `useSWR` — an inline function creates a new reference each render, defeating deduplication.

## Basic Query Hook Template

Use `useSWR` for all read operations. Keys come from a centralized `swrKeys` factory — never hardcode strings.

```typescript
import useSWR from 'swr';
import { swrKeys } from 'lib/swr-keys';
import fetcher from 'lib/swr-fetcher';
import type {
  FetchResponse,
  UseHookParams,
  UseHookReturn,
} from './use-hook-name.types';

export function useHookName({
  id,
  enabled = true,
}: UseHookParams): UseHookReturn {
  const { data, isLoading, error, isValidating, mutate } = useSWR<FetchResponse>(
    enabled ? swrKeys.resource.byId(id) : null,
    fetcher,
    {
      dedupingInterval: 5000,
      revalidateOnFocus: true,
    }
  );

  return {
    data: data ?? null,
    isLoading,
    isValidating,
    error: error instanceof Error ? error.message : null,
    mutate,
  };
}
```

Key distinction: `isLoading` means first load with no cached data. `isValidating` means a background revalidation is in progress (cached data is still shown). Never show a skeleton during `isValidating` — use a subtle indicator instead.

## Mutation Hook Template

Use `useSWRMutation` for write operations that trigger on demand (form submissions, button clicks).

```typescript
import useSWRMutation from 'swr/mutation';
import { useSWRConfig } from 'swr';
import { swrKeys } from 'lib/swr-keys';
import type {
  MutationRequest,
  MutationResponse,
  UseMutationReturn,
} from './use-hook-name.types';

async function mutateApi(
  url: string,
  { arg }: { arg: MutationRequest }
): Promise<MutationResponse> {
  const response = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(arg),
  });

  const data: MutationResponse = await response.json();
  if (!data.success) throw new Error(data.error || 'Operation failed');
  return data;
}

export function useHookName(): UseMutationReturn {
  const { mutate } = useSWRConfig();

  const { trigger, isMutating, error } = useSWRMutation(
    swrKeys.resource.create(),
    mutateApi,
    {
      onSuccess: (_data, _key, _config) => {
        mutate(swrKeys.resource.lists());
      },
    }
  );

  return {
    trigger,
    isMutating,
    error: error instanceof Error ? error.message : null,
  };
}
```

| Scenario | Use |
|----------|-----|
| Remote mutation + cache revalidation | `useSWRMutation` with `trigger` |
| Local optimistic update of cached data | Bound `mutate` from `useSWR` |

## Infinite Query Hook Template

Use `useSWRInfinite` for paginated lists, cursor-based pagination, or infinite scroll.

```typescript
import useSWRInfinite from 'swr/infinite';
import { swrKeys } from 'lib/swr-keys';
import fetcher from 'lib/swr-fetcher';
import type { PageResponse, UseInfiniteHookReturn } from './use-hook-name.types';

const LIMIT = 20;

function getKey(pageIndex: number, previousPageData: PageResponse | null) {
  if (previousPageData && !previousPageData.nextCursor) return null;
  if (pageIndex === 0) return swrKeys.resource.list({ limit: LIMIT });
  return swrKeys.resource.list({ limit: LIMIT, cursor: previousPageData!.nextCursor });
}

export function useHookName(): UseInfiniteHookReturn {
  const { data, size, setSize, isLoading, isValidating } = useSWRInfinite<PageResponse>(
    getKey,
    fetcher,
  );

  // Flatten pages into a single list
  const items = data ? data.flatMap((page) => page.items) : [];
  const hasNextPage = data ? Boolean(data[data.length - 1]?.nextCursor) : false;
  const isFetchingNextPage = isValidating && data !== undefined && data.length === size;

  const loadMore = () => setSize(size + 1);

  return { items, loadMore, hasNextPage, isFetchingNextPage, isLoading };
}
```

Key points for infinite queries:
- `getKey` receives `pageIndex` and `previousPageData` — return `null` to stop fetching
- `data` is an array of pages; flatten with `flatMap`
- Advance pagination with `setSize(size + 1)`
- Use `isValidating` combined with `data.length === size` to detect next-page fetching vs background revalidation

## Optimistic Updates Template

Use optimistic updates when the UI should respond instantly (toggling favorites, sending messages).

```typescript
import useSWR, { useSWRConfig } from 'swr';
import { swrKeys } from 'lib/swr-keys';
import fetcher from 'lib/swr-fetcher';
import type { Item } from './use-hook-name.types';

export function useToggleFavorite(itemId: string) {
  const key = swrKeys.items.byId(itemId);
  const { data, mutate: boundMutate } = useSWR<Item>(key, fetcher);
  const { mutate: globalMutate } = useSWRConfig();

  const toggle = async () => {
    await boundMutate(
      async (current) => {
        const res = await fetch(`/api/items/${itemId}/favorite`, { method: 'POST' });
        const updated: Item = await res.json();
        return updated;
      },
      {
        optimisticData: data ? { ...data, isFavorite: !data.isFavorite } : undefined,
        rollbackOnError: true,
        revalidate: true,
      }
    );

    globalMutate(swrKeys.items.lists());
  };

  return { data, toggle };
}
```

Three-step flow:
1. **Optimistic** — `optimisticData` updates UI immediately
2. **Rollback** — `rollbackOnError: true` reverts if the mutation throws
3. **Revalidation** — `revalidate: true` refetches from server to reconcile

## Subscription Hook Template

Use `useSWRSubscription` for real-time data from WebSocket, SSE, or any push source.

```typescript
import useSWRSubscription from 'swr/subscription';
import type { SWRSubscription } from 'swr/subscription';
import { swrKeys } from 'lib/swr-keys';
import type { PriceUpdate } from './use-hook-name.types';

const subscribe: SWRSubscription<string, PriceUpdate, Error> = (key, { next }) => {
  const ws = new WebSocket(key);

  ws.onmessage = (event) => {
    try {
      const data: PriceUpdate = JSON.parse(event.data);
      next(null, data);
    } catch (err) {
      next(err as Error);
    }
  };

  ws.onerror = (event) => next(new Error('WebSocket error'));

  return () => ws.close();
};

export function useLivePrice(symbol: string) {
  const { data, error } = useSWRSubscription(
    swrKeys.prices.live(symbol),
    subscribe,
  );

  return { price: data ?? null, error };
}
```

The `subscribe` callback receives `{ next }` — call `next(null, data)` to push updates and `next(error)` on failure. Return a cleanup function to close the connection.

## SWR Middleware

Middleware wraps every `useSWR` call. Pattern: `(useSWRNext) => (key, fetcher, config) => { ... }`.

### Logger Middleware

```typescript
import type { Middleware } from 'swr';

const logger: Middleware = (useSWRNext) => (key, fetcher, config) => {
  const start = Date.now();
  const swr = useSWRNext(key, fetcher, config);

  if (swr.data && !swr.isValidating) {
    console.debug(`[SWR] ${String(key)} — ${Date.now() - start}ms`);
  }

  return swr;
};
```

### Auth Middleware

```typescript
const auth: Middleware = (useSWRNext) => (key, fetcher, config) => {
  const enhancedFetcher = async (...args: Parameters<typeof fetcher>) => {
    const token = localStorage.getItem('auth_token');
    if (!token) throw new Error('Not authenticated');
    return fetcher!(...args);
  };

  return useSWRNext(key, enhancedFetcher, config);
};
```

### Composing Middleware

Apply middleware globally via `SWRConfig`:

```typescript
<SWRConfig value={{ use: [logger, auth] }}>
  {children}
</SWRConfig>
```

Middleware executes in array order — place logger first to capture timing for all subsequent middleware.

## Cache Provider

Replace SWR's in-memory cache with localStorage for persistence across page reloads:

```typescript
function localStorageProvider() {
  const map = new Map<string, any>(
    JSON.parse(localStorage.getItem('swr-cache') || '[]')
  );

  window.addEventListener('beforeunload', () => {
    const entries = JSON.stringify(Array.from(map.entries()));
    localStorage.setItem('swr-cache', entries);
  });

  return map;
}

<SWRConfig value={{ provider: localStorageProvider }}>
  {children}
</SWRConfig>
```

## Key Patterns & Conditional Fetching

SWR keys identify cached data. Three key formats:

| Format | Example | When |
|--------|---------|------|
| String | `'/api/user'` | Simple, static endpoint |
| Array | `['/api/items', id, filters]` | Parameterized queries |
| Function | `() => id ? swrKeys.items.byId(id) : null` | Conditional fetching |

**Conditional fetching:** Pass `null` as key to disable the request entirely:

```typescript
const { data } = useSWR(userId ? swrKeys.user.byId(userId) : null, fetcher);
```

**Dependent queries:** Second query key depends on first query's data:

```typescript
const { data: user } = useSWR(swrKeys.user.me(), fetcher);
const { data: projects } = useSWR(
  user ? swrKeys.projects.byOwner(user.id) : null,
  fetcher,
);
```

## Loading States

Using the wrong loading state causes bad UX. Understand the distinction:

| State | Meaning | UI Pattern |
|-------|---------|------------|
| `isLoading` | First load, no cached data | Skeleton/spinner |
| `isValidating` | Background revalidation, cached data shown | Subtle indicator (spinner in corner) |
| `isMutating` | Mutation in progress via `useSWRMutation` | Disable button, show spinner |

**Rule:** Never show a skeleton when `isValidating` is true and cached data exists — use a subtle background indicator instead.

## Revalidation Strategies

SWR revalidates stale data automatically. Configure per-key or globally:

| Option | Default | Description |
|--------|---------|-------------|
| `revalidateOnFocus` | `true` | Revalidate when window regains focus |
| `revalidateOnReconnect` | `true` | Revalidate when network reconnects |
| `refreshInterval` | `0` (off) | Polling interval in milliseconds |
| `revalidateIfStale` | `true` | Revalidate on mount if data is stale |
| `revalidateOnMount` | — | Force revalidation on mount (overrides `revalidateIfStale`) |

Keep `revalidateOnFocus: true` globally — it is a core SWR feature. Only disable it per-key when you have a specific reason (e.g., expensive queries that rarely change).

## Debounce + SWR

Combine `useDebounce` with conditional key for search inputs and typeahead:

```typescript
import useSWR from 'swr';
import { useDebounce } from 'hooks/use-debounce';
import { swrKeys } from 'lib/swr-keys';
import fetcher from 'lib/swr-fetcher';

export function useUserSearch(query: string) {
  const debouncedQuery = useDebounce(query, 300);

  const { data, isLoading } = useSWR(
    debouncedQuery.length >= 2 ? swrKeys.users.search(debouncedQuery) : null,
    fetcher,
  );

  return { results: data?.users ?? [], isLoading };
}
```

Key points:
- Debounce the input, not the SWR call — SWR deduplicates by key automatically
- Use `null` key to prevent queries for too-short input
- The debounce delay (300ms) prevents excessive API calls while typing
