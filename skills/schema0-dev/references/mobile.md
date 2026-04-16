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
