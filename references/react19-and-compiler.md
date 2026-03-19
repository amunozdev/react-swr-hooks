# React 19 Hooks & React Compiler

## React 19 Hooks

### `useOptimistic` — Native Optimistic UI

Alternative to SWR's `mutate` with `optimisticData` for simple optimistic state during async operations:

```typescript
import { useOptimistic, useTransition } from 'react';

export function useLikeButton(currentLikes: number, toggleLike: (itemId: string) => Promise<void>, itemId: string) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    currentLikes,
    (current, delta: number) => current + delta
  );
  const [, startTransition] = useTransition();

  const handleLike = () => {
    startTransition(async () => {
      addOptimisticLike(1);
      await toggleLike(itemId);
    });
  };

  return { likes: optimisticLikes, handleLike };
}
```

**When to use each:**
| Approach | When to use |
|----------|-------------|
| SWR `mutate` with `optimisticData` | Client-side API calls, cache revalidation, rollback needed |
| `useOptimistic` | Simple optimistic state during async ops in `startTransition` |

### `use()` — Conditional Context and Promise Reading

`use()` can be called conditionally (unlike `useContext`). Use it to read Context or resolve Promises passed as props:

```typescript
import { use } from 'react';
import { ThemeContext } from 'contexts/theme';

function ConditionalTheme({ showTheme }: { showTheme: boolean }) {
  if (!showTheme) return null;
  const theme = use(ThemeContext);
  return <div style={{ color: theme.primary }}>Themed</div>;
}
```

**Critical rule:** Never create Promises inside a component render — they recreate on every render, causing infinite suspend loops. Always create Promises outside the render path and pass them as props.

## React Compiler — No Manual Memoization

If your project has React Compiler enabled, it automatically inserts memoization equivalent to `React.memo`, `useMemo`, and `useCallback` at build time.

**Note:** If your project does not use React Compiler, manual `useMemo`/`useCallback` are acceptable and recommended for expensive computations and stable callback references.

### What this means for hooks

**Do NOT write** manual `useMemo` or `useCallback` in new hooks when React Compiler is enabled. The compiler handles it:

```typescript
// CORRECT — React Compiler memoizes automatically
function useFilteredItems(items: Item[], query: string) {
  const filtered = items.filter(item =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );
  return filtered;
}

// UNNECESSARY (when compiler is enabled) — manual memoization the compiler already handles
function useFilteredItems(items: Item[], query: string) {
  return useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(query.toLowerCase())
    ),
    [items, query]
  );
}
```

### When the compiler CANNOT help

- Mutating objects/arrays in place (the compiler analyzes data flow statically)
- Reading from refs imperatively during render
- Side effects during render (non-idempotent render functions)

In these cases, the compiler skips memoization and may emit a lint diagnostic. Fix the code pattern instead of adding manual memoization.

### Stable references at module scope

Even with the compiler, prefer module-level constants for values that never change:

```typescript
// CORRECT — stable reference, no allocation per render
const DEFAULT_FILTERS = ['worn', 'signed'] as const;

function useItemFilters() {
  const [filters, setFilters] = useState(DEFAULT_FILTERS);
  // ...
}
```
