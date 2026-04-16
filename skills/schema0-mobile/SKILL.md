---
name: schema0-mobile
description: Mobile platform patterns — React Native / Expo, worker architecture, ORPC client, and navigation
---

# Mobile Platform

React Native / Expo app at `apps/native/`. Skip this if `apps/native/` does not exist.

## Overview

Mobile apps use a single Cloudflare Worker (`apps/native/worker.ts`) that handles both static assets and API. API routes are mounted at `/rpc` via Hono + `RPCHandler` using the same routers from `packages/api/`.

## Key Differences from Web

- Uses `@tanstack/react-query` directly (NOT TanStack DB / `useLiveQuery`)
- Data fetching: `useQuery(orpc.{entity}.selectAll.queryOptions())`
- Auth via `@schema0/auth-mobile` + WorkOS (NOT `@template/auth`)
- API calls go to deployed backend at `EXPO_PUBLIC_WEB_URL/rpc`
- Environment variables use `EXPO_PUBLIC_*` prefix
- Navigation via Expo Router (file-based)

## Required Env Vars

Validated in `_layout.tsx`:

- `EXPO_PUBLIC_WORKOS_CLIENT_ID`
- `EXPO_PUBLIC_ORGANIZATION_ID`
- `EXPO_PUBLIC_WEB_URL`
- `EXPO_PUBLIC_API_HOSTNAME`
- `EXPO_PUBLIC_APP_ID`

## Deploying

```bash
schema0 sandbox deploy --platform native
```

## References

- `references/patterns.md` -- ORPC client setup, data fetching with useQuery, navigation, web vs mobile comparison table
