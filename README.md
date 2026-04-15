# react-swr-hooks

Hook development conventions for React projects using SWR (stale-while-revalidate). Covers file structure, fetcher patterns, key factories, typing, testing, and anti-pattern prevention.

## Install

```bash
npx skills add amunozdev/react-swr-hooks
```

## What is this?

An AI coding skill that enforces consistent, production-quality hook patterns in any React project using SWR. It guides Claude to create hooks with proper directory structure, centralized key factories, typed returns, and co-located tests.

## Supported Environments

- React Native / Expo
- Vite + React
- Create React App (CRA)
- Remix (client-side)
- Any React 18 or 19 project

## TanStack Query?

If your project uses TanStack Query instead of SWR, use [react-tanstack-hooks](https://github.com/amunozdev/react-tanstack-hooks) instead.

## Next.js?

If your project uses Next.js (Server Components, SSR, `'use client'`), use [nextjs-tanstack-hooks](https://github.com/amunozdev/nextjs-tanstack-hooks) instead. It includes Server Component directives, SSR hydration patterns, and `useActionState` support.

## License

MIT
