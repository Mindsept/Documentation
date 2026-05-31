# Application Flows

This document visualizes the main Cloud Native Platform V1 flows.

The V1 product promise is:

```txt
A project owner connects GitHub, imports a repository with a Dockerfile, analyzes the stack, generates CI, reviews it, opens a Pull Request containing .github/workflows/ci.yml, then optionally generates CD for a registered AKS Cloud Node.
```

## High-Level Product Flow

```mermaid
flowchart TD
    A[User logs in] --> B[Create or open project]
    B --> C[Connect GitHub App]
    C --> D[List accessible repositories]
    D --> E[Import repository into project]
    E --> F[Analyze repository stack]
    Admin[Admin configures global GHCR and Azure Cloud Node] --> G
    F --> G[Review detected stack and env requirements]
    G --> H[Generate CI preview]
    H --> I{Need AI changes?}
    I -->|yes| J[Adapt YAML with AI]
    I -->|no| K[Approve YAML]
    J --> K
    K --> L{Sync GitHub Actions secrets?}
    L -->|yes| M[Push mapped secrets to GitHub]
    L -->|no| N[Skip secret sync]
    M --> O[Create onboarding branch]
    N --> O
    O --> P[Commit .github/workflows/ci.yml]
    P --> Q[Open GitHub Pull Request]
    Q --> R[GitHub Actions runs CI checks]
    R --> S[Merge CI PR to main]
    S --> T[CI pushes image to GHCR]
    T --> U[Generate CD preview for selected Cloud Node]
    U --> V[Open CD Pull Request]
    V --> W[Merge CD PR]
    W --> X[GitHub Actions deploys to AKS]
    X --> Y[Refresh deployment status]
    Y --> Z[Open public LoadBalancer URL]
```

## Production Traffic Flow

```mermaid
sequenceDiagram
    participant User
    participant Vercel
    participant Nginx
    participant API
    participant DB

    User->>Vercel: Open https://cnp.mindsept.fr
    Vercel-->>User: React SPA
    User->>Nginx: HTTPS API request to cnp-api.mindsept.fr
    Nginx->>API: Proxy to 127.0.0.1:8000
    API->>DB: Read/write PostgreSQL
    DB-->>API: Result
    API-->>Nginx: JSON response
    Nginx-->>User: HTTPS response
```

Vercel must rewrite all frontend routes to `/` so browser refreshes on `/github/callback`, `/admin/infrastructure`, and `/repositories/{id}/cd` load the React application instead of returning a platform 404.

## Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DB

    User->>Frontend: Submit email and password
    Frontend->>API: POST /api/v1/auth/login
    API->>DB: Load user by email
    DB-->>API: User with password_hash
    API->>API: Verify password
    API->>DB: Store hashed refresh token
    API-->>Frontend: access_token and refresh_token
    Frontend->>API: Protected request with Bearer token
    API->>API: Decode and validate JWT
    API-->>Frontend: Protected resource
```

## GitHub App Connection Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant GitHub
    participant DB

    User->>Frontend: Click connect GitHub
    Frontend->>API: GET /api/v1/github/install-url
    API->>API: Create signed state token with user id
    API-->>Frontend: GitHub install URL
    Frontend->>GitHub: Redirect user to GitHub App installation
    User->>GitHub: Install app on account or organization
    GitHub->>API: GET /api/v1/github/setup-callback?installation_id&state
    API->>API: Validate signed state
    API->>GitHub: Fetch installation metadata
    GitHub-->>API: account and permission metadata
    API->>DB: Upsert github_installations
    API->>DB: Write audit log
    API-->>Frontend: 302 redirect to /github/callback
```

## Repository Import Flow

```mermaid
sequenceDiagram
    participant Frontend
    participant API
    participant GitHub
    participant DB

    Frontend->>API: GET /api/v1/github/installations
    API->>DB: Load linked installations
    DB-->>API: Installations
    API-->>Frontend: Installations

    Frontend->>API: GET /api/v1/github/installations/{id}/repositories
    API->>GitHub: List installation repositories
    GitHub-->>API: Repositories
    API->>DB: Cache github_installation_repositories
    API-->>Frontend: Accessible repositories

    Frontend->>API: POST /api/v1/projects/{project_id}/repositories/import
    API->>DB: Verify project role
    API->>DB: Read cached GitHub repository
    API->>DB: Create repositories row
    API->>DB: Write audit log
    API-->>Frontend: Imported repository
```

## Repository Analysis Flow

```mermaid
sequenceDiagram
    participant Frontend
    participant API
    participant DB
    participant Worker
    participant GitHub

    Frontend->>API: POST /api/v1/repositories/{id}/analyze
    API->>DB: Verify project access
    API->>DB: Create queued analyze_repository job
    API-->>Frontend: job_id

    Worker->>DB: Poll queued jobs
    DB-->>Worker: analyze_repository job
    Worker->>DB: Mark job running
    Worker->>GitHub: Read repository tree
    Worker->>GitHub: Read package files and .env.example
    Worker->>Worker: Detect stack, tests, Dockerfile, env vars
    Worker->>DB: Create repository_analyses row
    Worker->>DB: Update repositories latest analysis fields
    Worker->>DB: Write audit log
    Worker->>DB: Mark job succeeded

    Frontend->>API: GET /api/v1/jobs/{job_id}
    API->>DB: Read job
    API-->>Frontend: succeeded with result
    Frontend->>API: GET /api/v1/repositories/{id}/analysis/latest
    API->>DB: Read latest analysis
    API-->>Frontend: detected stack and env vars
```

## CI Requirements and Secret Mapping Flow

```mermaid
sequenceDiagram
    participant Frontend
    participant API
    participant DB

    Frontend->>API: GET /api/v1/repositories/{id}/ci/requirements
    API->>DB: Load latest repository analysis
    API->>DB: Load project secret metadata
    API->>API: Build suggested mappings
    API-->>Frontend: detected env vars, suggested mappings, missing secrets

    Frontend->>Frontend: User reviews or edits mappings
    Frontend->>API: POST /api/v1/repositories/{id}/ci/preview
    API->>DB: Verify maintainer role
    API->>DB: Load latest analysis and project secrets
    API->>API: Resolve env_name -> project_secret_key -> github_secret_name
    API->>API: Render GitHub Actions template
    API->>DB: Store ci_pipelines row
    API->>DB: Write audit log
    API-->>Frontend: generated YAML, required secrets, missing secrets
```

## CI Preview, Approval, and Pull Request Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DB
    participant GitHub

    User->>Frontend: Generate CI preview
    Frontend->>API: POST /api/v1/repositories/{id}/ci/preview
    API->>DB: Store generated pipeline
    API-->>Frontend: YAML preview

    opt AI adaptation
        User->>Frontend: Ask AI to adapt YAML
        Frontend->>API: POST /api/v1/repositories/{id}/ci/{pipeline_id}/adapt
        API->>API: Call AI service
        API->>DB: Store adapted_yaml and AI interaction
        API-->>Frontend: Adapted YAML and diff summary
    end

    User->>Frontend: Approve YAML
    Frontend->>API: POST /api/v1/repositories/{id}/ci/{pipeline_id}/approve
    API->>DB: Mark pipeline approved
    API->>DB: Write audit log
    API-->>Frontend: Approved pipeline

    User->>Frontend: Open Pull Request
    Frontend->>API: POST /api/v1/repositories/{id}/ci/{pipeline_id}/create-pr

    opt sync_secrets_to_github is true
        API->>DB: Load encrypted project secrets
        API->>API: Decrypt project secrets
        API->>GitHub: Get Actions public key
        API->>API: Encrypt values with GitHub public key
        API->>GitHub: Upsert repository Actions secrets
        API->>DB: Mark github_secrets_synced_at
    end

    API->>GitHub: Create onboarding branch
    API->>GitHub: Upsert .github/workflows/ci.yml
    API->>GitHub: Open Pull Request
    API->>DB: Mark pipeline pr_created
    API->>DB: Write audit log
    API-->>Frontend: Pull Request URL
```

## Platform Registry Setup Flow

```mermaid
sequenceDiagram
    participant Admin
    participant Frontend
    participant API
    participant DB

    Admin->>Frontend: Open Admin / Container Registry
    Admin->>Frontend: Enter GHCR registry metadata and token
    Frontend->>API: POST /api/v1/admin/container-registries
    API->>API: Require platform admin role
    API->>DB: Upsert container_registries
    API->>DB: Encrypt token in platform_secrets
    API->>DB: Write audit log
    API-->>Frontend: Registry metadata and stored secret key
```

The default registry token is shared platform infrastructure. Project teams do not need to duplicate `GHCR_TOKEN` for the demo unless they want a project-specific override.

## Global Cloud Node Setup Flow

```mermaid
sequenceDiagram
    participant Admin
    participant Frontend
    participant API
    participant DB

    Admin->>Frontend: Open Admin / Cloud Nodes
    Frontend->>API: GET /api/v1/admin/cloud-nodes
    API->>DB: Load global cloud_targets
    API-->>Frontend: Global Cloud Node list

    Admin->>Frontend: Paste cnp_cd_contract and Azure secrets
    Frontend->>API: POST /api/v1/admin/cloud-nodes
    API->>API: Require platform admin role
    API->>DB: Upsert cloud_targets with project_id null
    API->>DB: Encrypt Azure secrets in platform_secrets
    API->>DB: Write audit log
    API-->>Frontend: Global Cloud Node metadata and stored secret keys
```

Global Cloud Nodes are reusable deployment targets. They are not tied to one repository or one project. Any project can list them through `GET /api/v1/projects/{project_id}/cloud-targets` and select one during CD generation.

## CD Preview, Approval, and Pull Request Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API
    participant DB
    participant GitHub

    User->>Frontend: Open repository CD page
    Frontend->>API: GET /api/v1/repositories/{id}/cd/requirements
    API->>DB: Load repository analysis
    API->>DB: Load global and project Cloud Nodes
    API->>DB: Load project and platform secret metadata
    API-->>Frontend: Cloud targets, env vars, required secrets

    User->>Frontend: Select Azure Cloud Node and image tag pushed by CI
    Frontend->>API: POST /api/v1/repositories/{id}/cd/preview
    API->>DB: Verify maintainer role
    API->>API: Render Kubernetes manifests and cd.yml
    API->>DB: Store cd_pipelines row
    API->>DB: Write audit log
    API-->>Frontend: Generated files and missing secrets

    User->>Frontend: Approve CD
    Frontend->>API: POST /api/v1/repositories/{id}/cd/{pipeline_id}/approve
    API->>DB: Mark pipeline approved
    API-->>Frontend: Approved pipeline

    User->>Frontend: Open CD Pull Request
    Frontend->>API: POST /api/v1/repositories/{id}/cd/{pipeline_id}/create-pr

    opt sync_secrets_to_github is true
        API->>DB: Load encrypted project and platform secrets
        API->>GitHub: Upsert GitHub Actions secrets
    end

    API->>GitHub: Create onboarding branch
    API->>GitHub: Commit cd.yml and k8s manifests
    API->>GitHub: Open Pull Request
    API->>DB: Mark pipeline pr_created
    API-->>Frontend: Pull Request URL
```

## AKS Deployment Result Flow

```mermaid
sequenceDiagram
    participant GitHub
    participant Actions
    participant GHCR
    participant AKS
    participant User

    GitHub->>Actions: CD workflow runs on main
    Actions->>Actions: Azure login from synced AZURE_* secrets
    Actions->>AKS: az aks get-credentials
    Actions->>AKS: Apply namespace, deployment, service
    opt private GHCR image
        Actions->>AKS: Create imagePullSecret from synced GHCR_TOKEN
    end
    AKS->>GHCR: Pull selected image tag
    Actions->>AKS: kubectl rollout status deployment
    Actions->>AKS: kubectl get service
    AKS-->>Actions: LoadBalancer EXTERNAL-IP or pending
    User->>API: POST refresh-deployment
    API->>AKS: Read Kubernetes Service
    API-->>User: deployment_status and public_url
    User->>AKS: curl http://<external-ip>
```

The current platform refreshes deployment status on demand. Automatic refresh from a GitHub webhook or background job is still future work.

## GitHub Actions Result Flow

```mermaid
sequenceDiagram
    participant GitHub
    participant Actions
    participant API
    participant DB

    GitHub->>Actions: Pull Request created with ci.yml
    Actions->>Actions: Run CI workflow
    Actions->>GitHub: Publish check result

    GitHub->>API: POST /api/v1/webhooks/github
    API->>API: Verify X-Hub-Signature-256
    API->>DB: Store github_webhook_events
    API->>DB: Write webhook.received audit log
    API-->>GitHub: received true
```

## Environment Secret Flow

```mermaid
flowchart TD
    A[Repository contains .env.example] --> B[Analyzer reads env example file]
    B --> C[Analyzer extracts env var names]
    C --> D[Analyzer marks required and sensitive variables]
    D --> E[CI requirements endpoint returns detected variables]
    E --> F[Frontend maps env var to project secret and GitHub secret]
    F --> G[Preview request sends secret_mappings]
    G --> H[Template renders env block]
    H --> I[Generated workflow references GitHub Actions secrets]
    I --> J{sync_secrets_to_github}
    J -->|true| K[Backend pushes mapped secrets to GitHub Actions]
    J -->|false| L[User must configure GitHub secrets manually]
```

Generated YAML example:

```yaml
env:
  IMAGE_NAME: ghcr.io/example-org/api-orders
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

## Permission Flow

```mermaid
flowchart LR
    viewer[viewer] --> read[Read project, repositories, analysis, CI list]
    maintainer[maintainer] --> read
    maintainer --> write[Import repository, analyze, generate CI/CD, manage secrets]
    owner[owner] --> write
    owner --> members[Manage project members]
    admin[platform admin] --> all_projects[List all projects]
    admin --> cloud_nodes[Manage global Cloud Nodes]
    admin --> registries[Manage shared container registries]
```

## CI/CD File Generation Flow

```mermaid
flowchart TD
    A[Latest repository analysis] --> B[CI template context]
    B --> C[Render deterministic ci.yml]
    C --> D[Human approval]
    D --> E[Commit CI PR]
    E --> F[GitHub Actions pushes image tag]
    F --> G[CD render context]
    G --> H[Render namespace, secret, deployment, service]
    H --> I[Render cd.yml]
    I --> J[Human approval]
    J --> K[Commit CD PR]
```

See [CI/CD Generation Internals](ci-cd-generation.md) for the exact generation rules.

## Failure Points to Watch

```mermaid
flowchart TD
    A[CI onboarding request] --> B{Repository analyzed?}
    B -->|no| B1[409 repository_analysis_required]
    B -->|yes| C{Dockerfile required and present?}
    C -->|no| C1[409 dockerfile_required]
    C -->|yes| D{Mapped project secrets configured?}
    D -->|no| D1[Show missing_project_secrets]
    D -->|yes| E{GitHub permissions valid?}
    E -->|no| E1[GitHub API error]
    E -->|yes| F[Create PR]
```
