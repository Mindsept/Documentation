# Secrets API

Project secrets store values required by generated workflows and future deployment flows.

All endpoints require authentication.

## Security Rules

- Secret values are encrypted before storage.
- Secret values are never returned by the API.
- Responses include metadata only.
- Secret operations require maintainer or owner permissions.

## Create or Update Secret

```http
POST /api/v1/projects/{project_id}/secrets
```

Creates or updates a secret by `(project_id, environment, key)`.

Recommended environments:

```txt
ci          values used by generated CI workflows
cd          values used by generated CD workflows and Kubernetes runtime
dev         application-specific development values
staging     application-specific staging values
production  application-specific production values
```

Request:

```json
{
  "environment": "ci",
  "key": "GHCR_TOKEN",
  "value": "secret-value",
  "description": "Token used by generated GitHub Actions workflows to push images to GHCR"
}
```

Response `201`:

```json
{
  "id": "uuid",
  "project_id": "uuid",
  "environment": "ci",
  "key": "GHCR_TOKEN",
  "description": "Token used by generated GitHub Actions workflows to push images to GHCR",
  "created_at": "2026-05-20T12:00:00Z",
  "updated_at": "2026-05-20T12:00:00Z"
}
```

## List Secrets

```http
GET /api/v1/projects/{project_id}/secrets
```

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "project_id": "uuid",
      "environment": "ci",
      "key": "GHCR_TOKEN",
      "description": "Token used by generated GitHub Actions workflows to push images to GHCR",
      "created_at": "2026-05-20T12:00:00Z",
      "updated_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

## Delete Secret

```http
DELETE /api/v1/projects/{project_id}/secrets/{secret_id}
```

Response `200`:

```json
{
  "deleted": true
}
```

## Project Secrets Versus Platform Registry Secrets

Application runtime secrets remain project-owned.

Examples:

```txt
ci / DATABASE_URL
cd / API_TOKEN
cd / GREETING
```

For the demo, `GHCR_TOKEN` should normally be configured once as a platform container registry credential through:

```txt
POST /api/v1/admin/container-registries
```

When CI or CD PR creation uses `sync_secrets_to_github = true`, the backend falls back to the default platform registry token if a project-level `GHCR_TOKEN` is missing.

For CI-only validation, disable Docker image push on `main` and no registry secret is required.

Project-level `ci / GHCR_TOKEN` or `cd / GHCR_TOKEN` entries still work as overrides, but they are no longer required for the shared demo registry.
