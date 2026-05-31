# Container Registries API

Container registries are platform-level settings managed by admin users.

For the demo, the main use case is a shared GHCR token configured once by an admin, then reused when generated CI/CD Pull Requests sync GitHub Actions secrets.

Secret values are stored encrypted as platform secrets and are never returned by the API.

## Create Or Update Registry

```http
POST /api/v1/admin/container-registries
```

Requires a platform admin account.

The endpoint is idempotent by `name`.

Request:

```json
{
  "name": "ghcr",
  "provider": "ghcr",
  "registry_url": "ghcr.io",
  "namespace": "hamza-hnt",
  "auth_secret_name": "GHCR_TOKEN",
  "token": "<github_pat_with_packages_write>",
  "is_default": true,
  "description": "Shared GHCR registry for the demo"
}
```

Response:

```json
{
  "registry": {
    "id": "uuid",
    "name": "ghcr",
    "provider": "ghcr",
    "registry_url": "ghcr.io",
    "namespace": "hamza-hnt",
    "auth_secret_name": "GHCR_TOKEN",
    "is_default": true,
    "description": "Shared GHCR registry for the demo",
    "created_at": "2026-05-30T12:00:00Z",
    "updated_at": "2026-05-30T12:00:00Z"
  },
  "created": true,
  "stored_secret_key": "GHCR_TOKEN"
}
```

If `is_default` is true, other registries are marked non-default.

## List Registries

```http
GET /api/v1/container-registries
```

Requires authentication.

Response:

```json
{
  "items": [
    {
      "id": "uuid",
      "name": "ghcr",
      "provider": "ghcr",
      "registry_url": "ghcr.io",
      "namespace": "hamza-hnt",
      "auth_secret_name": "GHCR_TOKEN",
      "is_default": true,
      "description": "Shared GHCR registry for the demo",
      "created_at": "2026-05-30T12:00:00Z",
      "updated_at": "2026-05-30T12:00:00Z"
    }
  ]
}
```

## Runtime Behavior

When CI or CD PR creation uses `sync_secrets_to_github = true`, the backend first looks for a project secret. If `GHCR_TOKEN` is missing at project level, it falls back to the default platform registry token.

This means projects no longer need to duplicate the same GHCR token for the demo. Application runtime secrets such as `DATABASE_URL`, `API_TOKEN`, or `GREETING` still belong to the project.
