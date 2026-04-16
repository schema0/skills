# Version Management

Manage preview and production deployments. A **preview** runs your new application code against a database branched from production, letting you test changes with the production schema before going live.

## Commands

```bash
schema0 version list                                    # List all versions
schema0 version preview <commitHash>                    # Create preview from dev deployment
schema0 version deploy <commitHash>                     # Promote preview to production
schema0 version remove <commitHash>                     # Remove a preview
schema0 version confirm-migration <commitHash> --statements "SQL..."  # Apply migration
```

## Typical Flow

```bash
# 1. Check current state
schema0 version list

# 2. Create preview from a dev deployment
schema0 version preview abc1234

# 3. If migration required, review the schema diff and apply SQL
schema0 version confirm-migration abc1234 --statements "ALTER TABLE ...;"

# 4. Deploy preview to production
schema0 version deploy abc1234

# 5. If production migration required
schema0 version confirm-migration abc1234 --production --statements "ALTER TABLE ...;"
```

## Notes

- Schema diffs are returned when migrations are needed — you write the SQL statements
- Preview must exist before deploying to production
- Migration must be completed before a preview can be promoted
- Use `--production` flag on `confirm-migration` when targeting production
