# Testing Hooks

Use `renderHook` from `@testing-library/react` (NOT the deprecated `@testing-library/react-hooks` package).

## Testing SWR hooks

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { SWRConfig } from 'swr';

function createTestWrapper() {
  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <SWRConfig value={{ dedupingInterval: 0, provider: () => new Map() }}>
        {children}
      </SWRConfig>
    );
  };
}

describe('useItemDetail', () => {
  it('returns item data on success', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValueOnce(
      new Response(JSON.stringify({ id: '1', name: 'Jersey' }), {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      })
    );

    const { result } = renderHook(() => useItemDetail('1'), {
      wrapper: createTestWrapper(),
    });

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.data?.name).toBe('Jersey');
  });
});
```

Key points for the test wrapper:
- `dedupingInterval: 0` prevents SWR from deduplicating requests between tests
- `provider: () => new Map()` gives each test a fresh, isolated cache
- Always mock `fetch` before rendering the hook — SWR fires the request immediately on mount

## Testing reducers directly

Reducers are pure functions — test them without `renderHook`:

```typescript
import { flowReducer } from './use-transfer-flow';

it('transitions from idle to confirm on START', () => {
  const next = flowReducer(
    { step: 'idle' },
    { type: 'START', recipientId: 'user-123' }
  );
  expect(next).toEqual({ step: 'confirm', recipientId: 'user-123' });
});
```
