# API Overview

The Cloud Native Platform API is exposed under:

```txt
/api/v1
```

In local development:

```txt
Nginx base URL: http://localhost:8080/api/v1
Direct API URL: http://localhost:8000/api/v1
```

In the validated production demo:

```txt
API base URL: https://cnp-api.mindsept.fr/api/v1
Swagger UI:   https://cnp-api.mindsept.fr/docs
```

OpenAPI documentation:

```txt
Swagger UI:   http://localhost:8080/docs
OpenAPI JSON: http://localhost:8080/openapi.json
```

## Authentication

Most endpoints require:

```http
Authorization: Bearer <access_token>
```

Public or externally authenticated endpoints:

```txt
GET  /api/v1/health
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
GET  /api/v1/github/setup-callback
POST /api/v1/webhooks/github
```

The GitHub setup callback is validated through a signed state token and redirects to the frontend callback page. The webhook endpoint is validated through `X-Hub-Signature-256`.

## Error Format

All application errors use:

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {}
  }
}
```

Common status codes:

```txt
400  invalid request or failed external validation
401  missing or invalid authentication
403  authenticated but insufficient project permissions
404  resource not found
409  valid request but invalid current state
422  request validation error
500  unexpected server error
```

## List Responses

List endpoints return:

```json
{
  "items": []
}
```

## Resource Identifiers

Internal resources use UUIDs:

```txt
project_id
repository_id
ci_pipeline_id
cd_pipeline_id
cloud_target_id
job_id
secret_id
```

GitHub resources keep GitHub numeric identifiers when relevant:

```txt
installation_id
github_repo_id
pull_request_number
```

## Endpoint Groups

- [Authentication](authentication.md)
- [Projects](projects.md)
- [GitHub App](github.md)
- [Repositories and Analysis](repositories.md)
- [CI](ci.md)
- [CD](cd.md)
- [Container Registries](registries.md)
- [Secrets](secrets.md)
- [Webhooks](webhooks.md)
- [Jobs, Audit, Dashboard, and Health](jobs-audit-dashboard.md)
