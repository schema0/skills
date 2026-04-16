# Integrations

Discover and execute third-party integrations via connected platforms (Gmail, Slack, Stripe, etc.).

## Commands

```bash
schema0 integrations connections                           # List connected platforms
schema0 integrations connections --platform gmail          # Filter by platform
schema0 integrations search <platform> "<query>"           # Search actions (natural language)
schema0 integrations details <systemId>                    # Get action parameters and docs
schema0 integrations execute <connectionKey> <path> [opts] # Execute an action
```

## Typical Workflow

```bash
# 1. See what platforms are connected
schema0 integrations connections

# 2. Search for actions using natural language
schema0 integrations search gmail "send email"
schema0 integrations search slack "send message"

# 3. Get details about a specific action
schema0 integrations details <systemId>

# 4. Execute the action
schema0 integrations execute <connectionKey> /v0/endpoint --method GET --action-id <systemId>
```

## Key Concepts

- **Connection key**: Identifies which authenticated platform account to use (from `connections` output)
- **System ID**: Unique identifier for an action (from `search` output)
- **Search query**: Natural language description of what you want to do
- **Action path**: The API path for the passthrough request (from `search` or `details` output)

## Notes

- The `search` query is natural language — describe the action in plain English
- Always run `details` before `execute` to understand required parameters
- Path parameters use `{{param}}` or `:param` format and are substituted automatically via `--path-params`
