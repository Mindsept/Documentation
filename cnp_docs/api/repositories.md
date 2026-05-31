# Repositories and Analysis API

Repositories are imported into projects from cached GitHub App installation repositories.

All endpoints require authentication.

## Import Repository

```http
POST /api/v1/projects/{project_id}/repositories/import
```

Requires project maintainer or owner permissions.

Request:

```json
{
  "github_installation_id": "uuid",
  "github_repo_id": 987654
}
```

Response `201`:

```json
{
  "id": "uuid",
  "project_id": "uuid",
  "provider": "github",
  "full_name": "example-org/api-orders",
  "name": "api-orders",
  "default_branch": "main",
  "onboarding_status": "registered",
  "detected_stack": null,
  "dockerfile_present": false,
  "ci_present": false,
  "html_url": "https://github.com/example-org/api-orders"
}
```

## List Project Repositories

```http
GET /api/v1/projects/{project_id}/repositories
```

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "full_name": "example-org/api-orders",
      "detected_stack": "python_fastapi",
      "dockerfile_present": true,
      "ci_present": false,
      "onboarding_status": "analyzed",
      "last_analyzed_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

## Get Repository

```http
GET /api/v1/repositories/{repository_id}
```

Response `200`:

```json
{
  "id": "uuid",
  "project_id": "uuid",
  "provider": "github",
  "full_name": "example-org/api-orders",
  "name": "api-orders",
  "default_branch": "main",
  "onboarding_status": "registered",
  "detected_stack": null,
  "dockerfile_present": false,
  "ci_present": false,
  "html_url": "https://github.com/example-org/api-orders"
}
```

## Analyze Repository

```http
POST /api/v1/repositories/{repository_id}/analyze
```

Runs deterministic stack analysis. Async mode queues a job.

Request:

```json
{
  "ref": "main",
  "async": true
}
```

Async response `200`:

```json
{
  "job_id": "uuid",
  "status": "queued",
  "type": "analyze_repository"
}
```

Sync response `200`:

```json
{
  "id": "uuid",
  "repository_id": "uuid",
  "detected_stack": "python_fastapi",
  "confidence": 0.92,
  "dockerfile_present": true,
  "dockerfile_path": "Dockerfile",
  "ci_present": false,
  "ci_paths": [],
  "test_detected": true,
  "test_command": "pytest",
  "package_manager": "pip",
  "language_version": null,
  "files_detected": {
    "requirements_txt": true,
    "dockerfile": true,
    "env_example": true,
    "tests": true
  },
  "env_vars": [
    {
      "name": "DATABASE_URL",
      "source": ".env.example",
      "required": true,
      "sensitive": true,
      "default_value_present": false
    }
  ],
  "required_env_vars": [
    "DATABASE_URL"
  ],
  "recommendations": [
    "Generate GitHub Actions CI from python_fastapi template"
  ],
  "created_at": "2026-05-20T12:00:00Z"
}
```

## Latest Analysis

```http
GET /api/v1/repositories/{repository_id}/analysis/latest
```

Returns the most recent analysis.

Response `200`:

```json
{
  "id": "uuid",
  "repository_id": "uuid",
  "detected_stack": "python_fastapi",
  "confidence": 0.92,
  "dockerfile_present": true,
  "dockerfile_path": "Dockerfile",
  "ci_present": false,
  "ci_paths": [],
  "test_detected": true,
  "test_command": "pytest",
  "package_manager": "pip",
  "language_version": null,
  "files_detected": {
    "requirements_txt": true,
    "dockerfile": true,
    "env_example": true,
    "tests": true
  },
  "env_vars": [
    {
      "name": "DATABASE_URL",
      "source": ".env.example",
      "required": true,
      "sensitive": true,
      "default_value_present": false
    }
  ],
  "required_env_vars": [
    "DATABASE_URL"
  ],
  "recommendations": [],
  "created_at": "2026-05-20T12:00:00Z"
}
```

If no analysis exists yet, the endpoint returns `404`. The frontend should treat this as "no analysis yet".

## Supported Stacks

```txt
python_fastapi
node
java_maven
go
generic_docker
unknown
```

The V1 CI flow expects a Dockerfile for Docker build validation.
