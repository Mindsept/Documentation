# CI Onboarding Flow

The V1 product flow is designed to generate a safe GitHub Actions CI workflow and open a Pull Request for human review.

For the implementation-level generation rules, see [CI/CD Generation Internals](ci-cd-generation.md).

## End-to-End Flow

```txt
Create project
-> connect GitHub App
-> list installation repositories
-> import repository
-> analyze repository
-> generate CI preview
-> optionally explain or adapt with AI
-> approve workflow
-> create onboarding Pull Request
-> GitHub Actions runs on the Pull Request
```

## Recommended Default

For most repositories, start with:

```txt
Build Docker image: ON
Push image on main: OFF
```

This validates that tests pass and the Dockerfile builds without requiring registry credentials.

## Registry Push

If `Push image on main` is enabled, the generated workflow pushes the Docker image to a registry on `main`.

For GitHub Container Registry, the workflow expects:

```txt
GHCR_TOKEN
```

For the demo, configure `GHCR_TOKEN` once as the default platform container registry credential. During PR creation with secret sync enabled, the backend pushes it to the repository GitHub Actions secrets automatically.

If this secret is not configured globally or at project level, the platform can still open the Pull Request, but the workflow will fail on the push step after merge to `main`.

For a CI-only validation flow, keep `Push image on main` disabled.

For the AKS CD demo, keep `Push image on main` enabled and use the exact GHCR tag produced by CI as the CD image tag. The generated CI currently pushes:

```txt
ghcr.io/<owner>/<repo>:<commit-sha>
```

The CD preview should use the same `<commit-sha>` value.

## Environment Variables and CI Secrets

During repository analysis, the backend detects variables declared in common example files:

```txt
.env.example
.env.sample
env.example
example.env
```

Detected variables are returned by the latest analysis endpoint and by the CI requirements endpoint.

Before generating CI, call:

```txt
GET /api/v1/repositories/{repository_id}/ci/requirements
```

This returns:

- detected environment variables
- variables that look required or sensitive
- project secrets already configured in CNP
- suggested mappings when a project secret key matches a detected env var
- missing project secrets

CI preview can receive explicit mappings:

```json
{
  "secret_mappings": [
    {
      "env_name": "DATABASE_URL",
      "project_secret_key": "PROD_DATABASE_URL",
      "github_secret_name": "DATABASE_URL"
    }
  ]
}
```

The generated workflow then receives:

```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

When opening the Pull Request, the backend can optionally synchronize mapped project secrets into GitHub Actions repository secrets:

```json
{
  "sync_secrets_to_github": true
}
```

This requires the GitHub App to have permission to manage repository Actions secrets.

## Repository Requirements

The V1 flow works best when the repository contains:

```txt
Dockerfile
dependency manifest
tests
```

Examples:

```txt
FastAPI: Dockerfile, requirements.txt or pyproject.toml, app/main.py, tests/
Node.js: Dockerfile, package.json, test script
Maven:   Dockerfile, pom.xml
Go:      Dockerfile, go.mod
```

If the stack is not recognized but a Dockerfile exists, the platform can generate a generic Docker build workflow.

## Human Approval

The backend never commits AI-adapted YAML directly. The workflow must be approved through:

```txt
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/approve
```

before:

```txt
POST /api/v1/repositories/{repository_id}/ci/{ci_pipeline_id}/create-pr
```

## GitHub Result

The onboarding Pull Request contains:

```txt
.github/workflows/ci.yml
```

After GitHub receives the PR, the Actions check should appear in the Pull Request checks area. If the workflow passes, the check becomes green and the PR can be merged.

When Docker push is enabled, the merge to `main` pushes the image to GHCR using the commit SHA tag. That tag is the recommended CD image tag for the AKS demo.
