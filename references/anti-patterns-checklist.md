# Anti-Patterns & Verification Checklist

## Anti-Patterns Checklist

Before finishing any hook, verify NONE of these are present:

| Anti-Pattern | Why it's wrong | Correct alternative |
|-------------|----------------|---------------------|
| `setState()` during render | Infinite loop, breaks React Compiler (if enabled) | Derive in `useMemo` or `useReducer` |
| `useCallback(...)()` (IIFE) | Not memoization, runs every render | Extract to `useEffect` or event handler |
| `setTimeout` to wait for state | Race condition, fragile timing | `useEffect` with proper dependency |
| Error detection via string matching | Breaks with i18n, fragile | Use error codes or typed errors |
| `console.error` for user-facing errors | Not user-visible | Return error state from the hook |
| `new Map()` / `new Set()` in render | New reference every render | `useRef` or `useState` |
| `eslint-disable react-hooks/exhaustive-deps` | Hides real dependency bugs | Fix the dependency array properly |
| `useEffect` for data fetching | Loses cache, dedup, revalidation | Use `useSWR` |
| Manual `useMemo` / `useCallback` (when React Compiler is enabled) | Redundant, compiler handles it | Remove — compiler optimizes at build time |
| `fetch()` + `useState` + `useEffect` | Manual state management, no cache | Use `useSWR` / `useSWRMutation` |
| Hardcoded SWR key strings | No centralized cache control | Use `swrKeys` factory |
| `mutate()` inside `useEffect` | Creates revalidation loops | Call in event handlers or `onSuccess` |
| Inline fetcher in `useSWR` | New ref each render, defeats dedup | Module-scope or config fetcher |
| `revalidateOnFocus: false` globally | Disables core SWR feature | Per-key override only |

Note: `useEffect` is NOT valid for data fetching, but IS valid for side effects like IntersectionObserver setup, DOM manipulation, or event listeners.

## Verification Checklist

After creating or modifying a hook, verify:

- [ ] Directory structure: `hooks/use-hook-name/` with `index.ts` + `use-hook-name.ts` + optional `.types.ts` + optional `.test.ts`
- [ ] Barrel export: `export { useHookName } from './use-hook-name'`
- [ ] Data fetching uses `useSWR` / `useSWRInfinite` / `useSWRMutation` (never manual fetch + useState)
- [ ] Keys come from `swrKeys` factory (never hardcoded strings)
- [ ] Config values from centralized `lib/swr-config.ts`
- [ ] No anti-patterns from the checklist above
- [ ] No manual `useMemo`/`useCallback` (if React Compiler is enabled; acceptable otherwise)
- [ ] Types properly defined: no `any`, explicit return type, `.types.ts` for 3+ interfaces
- [ ] Hook is under ~300 lines with single responsibility
- [ ] No duplicated utilities — check `lib/utils/` first
- [ ] Loading states used correctly: `isLoading` for skeletons, `isValidating` for background
- [ ] Conditional fetching uses `null` key (not `enabled` flag)
- [ ] Optimistic updates include `rollbackOnError: true`
- [ ] Project build command passes
- [ ] Project lint command passes
