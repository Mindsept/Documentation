# CI API

The CI API generates, adapts, approves, and opens Pull Requests for GitHub Actions workflows.

All endpoints require authentication.

## Principles

- CI generation is deterministic by default.
- AI can explain or adapt YAML, but cannot bypass human approval.
- The backend does not commit directly to `main`.
- Pull Requests contain `.github/workflows/ci.yml`.
- Registry push is optional and requires either project secrets or the default platform registry token.

See [CI/CD Generation Internals](../ci-cd-generation.md) for the exact template and secret mapping rules.

## Preview CI

Before generating a preview, the frontend can fetch detected env requirements:

```http
GET /api/v1/repositories/{repository_id}/ci/requirements
```

Response `200`:

```json
{
  "repository_id": "uuid",
  "detected_env_vars": [
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
  "configured_project_secrets": [
    "DATABASE_URL"
  ],
  "suggested_mappings": [
    {
      "env_name": "DATABASE_URL",
      "project_secret_key": "DATABASE_URL",
      "github_secret_name": "DATABASE_URL",
      "project_secret_configured": true
    }
  ],
  "missing_project_secrets": []
}
```

```http
POST /api/v1/repositories/{repository_id}/ci/preview
```

Generates a workflow preview without committing it.

Request:

```json
{
  "mode": "deterministic",
  "provider": "github_actions",
  "include_docker_build": true,
  "include_docker_push_on_main": false,
  "registry": "ghcr.io",
  "explain_with_ai": false,
  "auto_map_project_secrets": true,
  "secret_mappings": [
    {
      "env_name": "DATABASE_URL",
      "project_secret_key": "DATABASE_URL",
      "github_secret_name": "DATABASE_URL"
    }
  ]
}
```

Response `200`:

```json
{
  "ci_pipeline_id": "uuid",
  "repository_id": "uuid",
  "status": "generated",
  "generation_mode": "deterministic",
  "workflow_path": ".github/workflows/ci.yml",
  "generated_yaml": "name: CI\n...",
  "explanation": null,
  "required_secrets": [
    "DATABASE_URL"
  ],
  "detected_env_vars": [
    {
      "name": "DATABASE_URL",
      "source": ".env.example",
      "required": true,
      "sensitive": true,
      "default_value_present": false
    }
  ],
  "secret_mappings": [
    {
      "env_name": "DATABASE_URL",
      "project_secret_key": "DATABASE_URL",
      "github_secret_name": "DATABASE_URL",
      "project_secret_configured": true
    }
  ],
  "missing_project_secrets": [],
  "github_secrets_to_sync": [
    "DATABASE_URL"
  ]
}
```

If `include_docker_push_on_main` is `true` and the registry is GHCR, the workflow requires:

```txt
GHCR_TOKEN
```

During PR creation with `sync_secrets_to_github = true`, the backend first looks for project `ci / GHCR_TOKEN`, then falls back to the default platform container registry token configured through `POST /api/v1/admin/container-registries`.

## Adapt CI with AI

```http
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/adapt
```

Adapts the current YAML draft using an AI instruction.

Request:

```json
{
  "instruction": "Add a ruff lint step before tests, but keep the Docker build unchanged."
}
```

Response `200`:

```json
{
  "ci_pipeline_id": "uuid",
  "status": "adapted",
  "adapted_yaml": "name: CI\n...",
  "explanation": "A ruff lint step was added before pytest.",
  "diff_summary": [
    "Added ruff installation",
    "Added ruff check before tests"
  ]
}
```

The adapted YAML still requires approval before PR creation.

## Explain CI

```http
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/explain
```

Explains the generated or adapted workflow.

Request:

```json
{
  "detail_level": "medium"
}
```

Response `200`:

```json
{
  "ci_pipeline_id": "uuid",
  "explanation": "The workflow runs tests and builds the Docker image on pull requests and pushes to main."
}
```

## Approve CI

```http
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/approve
```

Marks the YAML as human-approved.

Request:

```json
{
  "use_adapted_yaml": true
}
```

Response `200`:

```json
{
  "ci_pipeline_id": "uuid",
  "status": "approved",
  "approved_by": "uuid",
  "approved_at": "2026-05-20T12:00:00Z"
}
```

## Create Pull Request

```http
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/create-pr
```

Requires the pipeline to be approved.

Request:

```json
{
  "base_branch": "main",
  "branch_name": "cnp/onboarding-ci",
  "commit_message": "chore(cnp): add generated CI workflow",
  "pull_request_title": "chore(cnp): add CI workflow",
  "pull_request_body": "This PR adds a generated GitHub Actions CI workflow from Cloud Native Platform.",
  "sync_secrets_to_github": true
}
```

Response `200`:

```json
{
  "ci_pipeline_id": "uuid",
  "status": "pr_created",
  "pull_request_url": "https://github.com/example-org/api-orders/pull/12",
  "pull_request_number": 12,
  "branch_name": "cnp/onboarding-ci",
  "synced_secrets": [
    "DATABASE_URL"
  ],
  "missing_project_secrets": []
}
```

When `sync_secrets_to_github` is `true`, CNP decrypts mapped project secrets server-side, encrypts them with the GitHub Actions public key, and upserts them as repository Actions secrets. Secret values are never returned by the API.

## List CI Pipelines

```http
GET /api/v1/repositories/{repository_id}/ci
```

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "status": "pr_created",
      "generation_mode": "deterministic",
      "workflow_path": ".github/workflows/ci.yml",
      "pull_request_url": "https://github.com/example-org/api-orders/pull/12",
      "created_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

## Recommended Default

For a CI that validates tests and Docker builds without secrets:

```json
{
  "include_docker_build": true,
  "include_docker_push_on_main": false
}
```
