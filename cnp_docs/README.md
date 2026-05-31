# Cloud Native Platform Documentation

This directory contains the human-facing documentation for the Cloud Native Platform backend and demo operating model.

The generated OpenAPI schema remains the API contract source of truth during development:

```txt
Local Swagger UI:      http://localhost:8080/docs
Local OpenAPI JSON:    http://localhost:8080/openapi.json
Production Swagger UI: https://cnp-api.mindsept.fr/docs
```

These documents explain how the API is structured, how the GitHub App integration works, how CI/CD files are generated, and how the production demo is operated.

## Documentation Map

### Platform and Operations

- [Team Onboarding Runbook](team-onboarding.md)
- [Architecture](architecture.md)
- [Database Model](database.md)
- [Containers and Local Network](containers.md)
- [Application Flows](flows.md)
- [Local Development](local-development.md)
- [Production Deployment](production-deployment.md)
- [GitHub App Setup](github-app-setup.md)
- [CI/CD Generation Internals](ci-cd-generation.md)
- [CI Onboarding Flow](ci-onboarding-flow.md)
- [Azure AKS CD Demo Flow](cd-aks-demo-flow.md)

### API Documentation

- [API Overview](api/README.md)
- [Authentication API](api/authentication.md)
- [Projects API](api/projects.md)
- [GitHub App API](api/github.md)
- [Repositories and Analysis API](api/repositories.md)
- [CI API](api/ci.md)
- [CD API](api/cd.md)
- [Container Registries API](api/registries.md)
- [Secrets API](api/secrets.md)
- [Webhooks API](api/webhooks.md)
- [Jobs, Audit, Dashboard, and Health APIs](api/jobs-audit-dashboard.md)

## Scope of V1

Cloud Native Platform V1 proves the repository onboarding path up to CI, with a validated Azure AKS demo CD extension:

```txt
Authenticate
-> create project
-> connect GitHub App
-> import repository
-> analyze stack
-> generate CI
-> review and approve
-> open Pull Request with .github/workflows/ci.yml
-> optionally push image to GHCR
-> optionally generate CD manifests for an Azure AKS demo target
-> deploy to AKS through generated GitHub Actions CD
-> refresh deployment status and public LoadBalancer URL
```

V1 does not implement cloud provisioning, ArgoCD, Terraform/OpenTofu inside the app, rollback automation, or full multi-cloud CD. The CD extension assumes an AKS cluster already exists and generates Kubernetes/GitHub Actions deployment files for a first demo.

## Recommended Reading Order For New Team Members

1. [Team Onboarding Runbook](team-onboarding.md)
2. [Application Flows](flows.md)
3. [CI/CD Generation Internals](ci-cd-generation.md)
4. [Azure AKS CD Demo Flow](cd-aks-demo-flow.md)
5. [Production Deployment](production-deployment.md)
6. [API Overview](api/README.md)

## Documentation Standards

- API examples use `/api/v1` paths.
- Protected endpoints assume `Authorization: Bearer <access_token>`.
- Secret values are never shown in examples after creation.
- Local development examples use Docker Compose and Nginx on `localhost:8080`.
- Production-specific values are represented as placeholders.
