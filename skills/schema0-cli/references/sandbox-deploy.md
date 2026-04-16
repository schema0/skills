# sandbox deploy

Build and deploy the application from the remote sandbox.

```bash
schema0 sandbox deploy [options]
```

| Option                 | Description                                 |
| ---------------------- | ------------------------------------------- |
| `--skip-db`            | Skip database migrations                    |
| `--platform`           | Deploy specific platform: `web` or `native` |
| `--include-source-map` | Include source maps in the deployment       |

## What It Does

**Web** (`--platform web`):

1. Generates and runs database migrations (unless `--skip-db`)
2. Builds the web application on the sandbox
3. Deploys the build output

**Native** (`--platform native`):

1. Generates and runs database migrations (unless `--skip-db`)
2. Exports the Expo app with `EXPO_PUBLIC_*` env vars injected
3. Runs `expo-adapter-workers` to prepare for deployment
4. Builds the worker (`build-worker.ts`)
5. Deploys the build output

## Notes

- **Do NOT run build commands manually** -- `schema0 sandbox deploy` handles all builds internally
- **Do NOT run `schema0 sandbox exec "bun run build"` before deploying**
- Use `--skip-db` when only frontend code changed
- Use `--platform` to deploy a specific platform
