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

1. Confirm directory structure: `hooks/use-hook-name/` with `index.ts` + `use-hook-name.ts` + optional `.types.ts` + optional `.test.ts`
2. Confirm barrel export: `export { useHookName } from './use-hook-name'`
3. Verify data fetching uses `useSWR` / `useSWRInfinite` / `useSWRMutation` (never manual fetch + useState)
4. Verify keys come from `swrKeys` factory (never hardcoded strings)
5. Verify config values from centralized `lib/swr-config.ts`
6. Check for no anti-patterns from the checklist above
7. Check for no manual `useMemo`/`useCallback` (if React Compiler is enabled; acceptable otherwise)
8. Verify types properly defined: no `any`, explicit return type, `.types.ts` for 3+ interfaces
9. Confirm hook is under ~300 lines with single responsibility
10. Check for no duplicated utilities — check `lib/utils/` first
11. Verify loading states used correctly: `isLoading` for skeletons, `isValidating` for background
12. Verify conditional fetching uses `null` key (not `enabled` flag)
13. Verify optimistic updates include `rollbackOnError: true`
14. Run project build command and confirm it passes
15. Run project lint command and confirm it passes

## Troubleshooting

| Failure | Cause | Fix |
|---------|-------|-----|
| Build fails with type errors after hook creation | Return type annotation doesn't match actual return | Check the `UseHookNameReturn` type matches the hook's return object exactly |
| Tests fail with `act()` warning | Assertions run before async state settles | Wrap assertions in `waitFor(() => { ... })` |
| `useSWR` returns `undefined` data but network shows 200 | Key mismatch or fetcher not receiving the key correctly | Verify the key passed to `useSWR` matches what `swrKeys` factory produces; check fetcher receives the serialized key |
| Optimistic update doesn't rollback on error | Missing `rollbackOnError: true` in mutate options | Add `rollbackOnError: true` to the `mutate()` options object |
| Hook causes infinite re-renders | Calling `mutate()` inside `useEffect` without proper deps | Move `mutate()` calls to event handlers or `onSuccess` callbacks — never inside `useEffect` |
| SWR makes duplicate requests on mount | `dedupingInterval` too low or provider creates new cache each render | Set `dedupingInterval` in `SWRConfig`; ensure `provider` function is stable (not inline) |
