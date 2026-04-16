# Mobile Platform

React Native / Expo app at `apps/native/`. Skip this if `apps/native/` does not exist.

## Worker Architecture

`apps/native/worker.ts`: Single Cloudflare Worker handles both static assets and API. API routes mounted at `/rpc` via Hono + `RPCHandler` (same routers from `packages/api/`). Static assets served by `expo-adapter-workers`. Do NOT delete `worker.ts` or `build-worker.ts`.

## Key Differences from Web

- Calls deployed mobile worker (`EXPO_PUBLIC_WEB_URL/rpc`)
- Session cookies from `expo-secure-store` attached to every ORPC request
- Uses `@tanstack/react-query` directly (NOT TanStack DB)
- Data fetching: `useQuery(orpc.{entity}.selectAll.queryOptions())`
- Auth via `@schema0/auth-mobile` + WorkOS (cookie-based sealed sessions)
- Environment variables use `EXPO_PUBLIC_*` prefix
- API calls go to the deployed backend — deploy with `schema0 sandbox deploy --platform native` first

## Required Env Vars

Validated in `_layout.tsx`:

- `EXPO_PUBLIC_WORKOS_CLIENT_ID`
- `EXPO_PUBLIC_ORGANIZATION_ID`
- `EXPO_PUBLIC_WEB_URL`
- `EXPO_PUBLIC_API_HOSTNAME`
- `EXPO_PUBLIC_APP_ID`

## Deploying Mobile

```bash
schema0 sandbox deploy --platform native
```

This exports the Expo app, runs `expo-adapter-workers`, builds the worker, and deploys everything.

## ORPC Client Setup

Mobile uses `@tanstack/react-query` directly (NOT TanStack DB). The ORPC client connects to the deployed mobile worker at `EXPO_PUBLIC_WEB_URL/rpc`.

```typescript
import { createORPCClient } from "@orpc/client";
import type { AppRouter } from "@template/api";

const client = createORPCClient<AppRouter>({
  url: `${process.env.EXPO_PUBLIC_WEB_URL}/rpc`,
});
```

Session cookies from `expo-secure-store` are attached to every ORPC request.

## Data Fetching with useQuery

Mobile uses `useQuery` with ORPC query options (NOT TanStack DB collections):

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

// Fetch all entities
const { data: entities, isLoading } = useQuery(
  orpc.entities.selectAll.queryOptions(),
);

// Mutations with cache invalidation
const queryClient = useQueryClient();
const createEntity = useMutation({
  ...orpc.entities.create.mutationOptions(),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
});
```

## Navigation Patterns

Mobile uses Expo Router for navigation (file-based routing like Next.js):

- File-based routing under `apps/native/app/`
- Layout routes with `_layout.tsx`
- Auth-protected routes use middleware/guards
- Deep linking supported via Expo Router config

## Mobile-Specific UI Patterns

- Auth via `@schema0/auth-mobile` + WorkOS (cookie-based sealed sessions)
- Environment variables use `EXPO_PUBLIC_*` prefix
- API calls go to the deployed backend — deploy with `schema0 sandbox deploy --platform native` first
- NO TanStack DB on mobile — use React Query directly
- NO `useLiveQuery` — that is web-only (`@tanstack/react-db`)

## Key Differences from Web (Summary)

| Aspect       | Web                                      | Mobile                    |
| ------------ | ---------------------------------------- | ------------------------- |
| Data layer   | TanStack DB collections                  | React Query `useQuery`    |
| Live queries | `useLiveQuery` from `@tanstack/react-db` | Not available             |
| Auth package | `@template/auth` (`@schema0/auth-web`)   | `@schema0/auth-mobile`    |
| API endpoint | Same-origin `/rpc`                       | `EXPO_PUBLIC_WEB_URL/rpc` |
| Env prefix   | Standard                                 | `EXPO_PUBLIC_*`           |
| Routing      | React Router v7                          | Expo Router               |
