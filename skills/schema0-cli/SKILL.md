---
name: schema0-cli
description: CLI commands for sandbox execution, deployment, version management, secrets, and third-party integrations
---

# Sandbox Commands

The codebase lives on a remote sandbox. All file operations and shell commands go through `schema0 sandbox exec`. Deployment uses `schema0 sandbox deploy`.

## Command Summary

| Command                          | Purpose                                    |
| -------------------------------- | ------------------------------------------ |
| `schema0 sandbox exec "<cmd>"`   | Run a shell command on the remote sandbox  |
| `schema0 sandbox deploy`         | Build and deploy the application           |
| `schema0 version list`           | List all versions                          |
| `schema0 version preview <hash>` | Create preview from dev deployment         |
| `schema0 version deploy <hash>`  | Promote preview to production              |
| `schema0 whoami`                 | Verify authentication                      |
| `schema0 secrets list/set/delete`| Manage secrets                             |
| `schema0 doctor`                 | Check deployment readiness                 |
| `schema0 logs`                   | View deployment logs                       |
| `schema0 integrations ...`       | Discover and execute third-party actions   |

## Key Notes

- Always quote the command string in `sandbox exec` to prevent local shell expansion
- `sandbox deploy` handles all builds internally -- do NOT run `bun run build` manually before deploying
- `sandbox exec` max timeout is 300000 (5 minutes), default 30000
- The sandbox filesystem persists between commands
- Integrations use natural language search: `schema0 integrations search gmail "send email"`

## References

- `references/sandbox-exec.md` -- sandbox exec syntax, options, common patterns
- `references/sandbox-deploy.md` -- sandbox deploy options, what it does per platform, notes
- `references/version-management.md` -- version list/preview/deploy/remove/confirm-migration workflow
