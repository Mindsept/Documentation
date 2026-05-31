# CD API

The CD API adds a first demo-ready Kubernetes deployment path.

V1 CD scope:

```txt
Azure AKS cloud target
-> deterministic Kubernetes manifests
-> generated .github/workflows/cd.yml
-> human approval
-> onboarding Pull Request
```

The generated CD workflow deploys on `main` by configuring kubeconfig or Azure CLI credentials, applying Kubernetes manifests, and updating the deployment image to the image tag selected during preview.

For the validated AKS demo runbook, see [Azure AKS CD Demo Flow](../cd-aks-demo-flow.md).
For the exact generation rules, see [CI/CD Generation Internals](../ci-cd-generation.md).

## Create Or Update Global Cloud Node

```http
POST /api/v1/admin/cloud-nodes
```

Requires a platform admin account.

This is the recommended endpoint for the demo. A global Cloud Node is configured once and can then be selected by repositories from any project.

The endpoint is idempotent by `cloud_target.name` among global nodes.

Request:

```json
{
  "cloud_target": {
    "name": "demo-aks",
    "environment": "demo",
    "provider": "azure",
    "cluster_type": "aks",
    "region": "polandcentral",
    "cluster_name": "aks-cnp-demo",
    "namespace": "cnp-demo",
    "config": {
      "tenant_id": "<azure_tenant_id>",
      "subscription_id": "<azure_subscription_id>",
      "resource_group": "rg-cnp-demo"
    }
  },
  "kubeconfig_strategy": {
    "option_a_secret_name": "KUBECONFIG_CONTENT",
    "option_b_secret_names": {
      "azure_client_id": "AZURE_CLIENT_ID",
      "azure_client_secret": "AZURE_CLIENT_SECRET",
      "azure_tenant_id": "AZURE_TENANT_ID",
      "azure_subscription_id": "AZURE_SUBSCRIPTION_ID"
    },
    "get_credentials_command": "az aks get-credentials --resource-group rg-cnp-demo --name aks-cnp-demo --overwrite-existing"
  },
  "deployment_defaults": {
    "namespace": "cnp-demo",
    "service_type": "LoadBalancer",
    "public_url_format": "http://<external-load-balancer-ip>",
    "manifest_paths": [
      "k8s/namespace.yaml",
      "k8s/secret.yaml",
      "k8s/deployment.yaml",
      "k8s/service.yaml"
    ]
  },
  "secret_values": {
    "AZURE_CLIENT_ID": "<azure_client_id>",
    "AZURE_CLIENT_SECRET": "<azure_client_secret>",
    "AZURE_TENANT_ID": "<azure_tenant_id>",
    "AZURE_SUBSCRIPTION_ID": "<azure_subscription_id>"
  }
}
```

Response:

```json
{
  "cloud_target": {
    "id": "uuid",
    "project_id": null,
    "name": "demo-aks",
    "environment": "demo",
    "provider": "azure",
    "cluster_type": "aks",
    "region": "polandcentral",
    "cluster_name": "aks-cnp-demo",
    "namespace": "cnp-demo",
    "config": {
      "tenant_id": "<azure_tenant_id>",
      "subscription_id": "<azure_subscription_id>",
      "resource_group": "rg-cnp-demo"
    },
    "kubeconfig_strategy": {},
    "deployment_defaults": {},
    "created_at": "2026-05-30T12:00:00Z",
    "updated_at": "2026-05-30T12:00:00Z"
  },
  "created": true,
  "upserted_secret_keys": ["AZURE_CLIENT_ID", "AZURE_CLIENT_SECRET", "AZURE_SUBSCRIPTION_ID", "AZURE_TENANT_ID"]
}
```

## List Global Cloud Nodes

```http
GET /api/v1/admin/cloud-nodes
```

Requires a platform admin account.

Response:

```json
{
  "items": []
}
```

## Create Or Update Project Cloud Target

```http
POST /api/v1/projects/{project_id}/cloud-targets
```

Requires a platform admin account.

This endpoint remains available for project-specific targets, but global Cloud Nodes are preferred for the demo.

The endpoint is idempotent by `(project_id, cloud_target.name)`.

Request:

```json
{
  "cloud_target": {
    "name": "demo-aks",
    "environment": "demo",
    "provider": "azure",
    "cluster_type": "aks",
    "region": "polandcentral",
    "cluster_name": "aks-cnp-demo",
    "namespace": "cnp-demo",
    "config": {
      "tenant_id": "<azure_tenant_id>",
      "subscription_id": "<azure_subscription_id>",
      "resource_group": "rg-cnp-demo"
    }
  },
  "kubeconfig_strategy": {
    "option_a_secret_name": "KUBECONFIG_CONTENT",
    "option_b_secret_names": {
      "azure_client_id": "AZURE_CLIENT_ID",
      "azure_client_secret": "AZURE_CLIENT_SECRET",
      "azure_tenant_id": "AZURE_TENANT_ID",
      "azure_subscription_id": "AZURE_SUBSCRIPTION_ID"
    },
    "get_credentials_command": "az aks get-credentials --resource-group rg-cnp-demo --name aks-cnp-demo --overwrite-existing"
  },
  "deployment_defaults": {
    "namespace": "cnp-demo",
    "service_type": "LoadBalancer",
    "public_url_format": "http://<external-load-balancer-ip>",
    "manifest_paths": [
      "k8s/namespace.yaml",
      "k8s/secret.yaml",
      "k8s/deployment.yaml",
      "k8s/service.yaml"
    ]
  },
  "secret_values": {
    "AZURE_CLIENT_ID": "<azure_client_id>",
    "AZURE_CLIENT_SECRET": "<azure_client_secret>",
    "AZURE_TENANT_ID": "<azure_tenant_id>",
    "AZURE_SUBSCRIPTION_ID": "<azure_subscription_id>"
  }
}
```

`secret_values` is optional. For project-scoped targets, values are encrypted into project secrets with environment `cd` and are never returned.

For the recommended Azure CLI strategy, store:

```json
{
  "secret_values": {
    "AZURE_CLIENT_ID": "<azure_client_id>",
    "AZURE_CLIENT_SECRET": "<azure_client_secret>",
    "AZURE_TENANT_ID": "<azure_tenant_id>",
    "AZURE_SUBSCRIPTION_ID": "<azure_subscription_id>"
  }
}
```

`KUBECONFIG_CONTENT` is only required when the repository CD preview uses `kubeconfig_strategy = "secret"`.

Response:

```json
{
  "cloud_target": {
    "id": "uuid",
    "project_id": "uuid",
    "name": "demo-aks",
    "environment": "demo",
    "provider": "azure",
    "cluster_type": "aks",
    "region": "polandcentral",
    "cluster_name": "aks-cnp-demo",
    "namespace": "cnp-demo",
    "config": {
      "tenant_id": "<azure_tenant_id>",
      "subscription_id": "<azure_subscription_id>",
      "resource_group": "rg-cnp-demo"
    },
    "kubeconfig_strategy": {},
    "deployment_defaults": {},
    "created_at": "2026-05-30T12:00:00Z",
    "updated_at": "2026-05-30T12:00:00Z"
  },
  "created": true,
  "upserted_secret_keys": ["AZURE_CLIENT_ID", "AZURE_CLIENT_SECRET", "AZURE_SUBSCRIPTION_ID", "AZURE_TENANT_ID"]
}
```

## List Cloud Targets

```http
GET /api/v1/projects/{project_id}/cloud-targets
```

Requires project viewer access. Platform admins can list cloud targets for any project.

The response includes both global Cloud Nodes (`project_id = null`) and project-specific Cloud Targets.

Response:

```json
{
  "items": []
}
```

## Get CD Requirements

```http
GET /api/v1/repositories/{repository_id}/cd/requirements
```

Requires project viewer access.

Returns:

- available cloud targets;
- detected env vars from repository analysis;
- configured effective CD secret keys from project secrets, global Cloud Node secrets, and the default platform registry secret;
- required cloud secret names;
- suggested env secret mappings;
- default GHCR image name.

The requirements response currently includes the possible cloud secret names for both supported kubeconfig strategies. A missing `KUBECONFIG_CONTENT` is not blocking when the CD preview uses `kubeconfig_strategy = "azure_cli"`.

Example response excerpt:

```json
{
  "cloud_targets": [
    {
      "id": "uuid",
      "project_id": null,
      "name": "demo-aks",
      "provider": "azure",
      "cluster_type": "aks",
      "namespace": "cnp-demo"
    }
  ],
  "configured_cd_secret_keys": [
    "AZURE_CLIENT_ID",
    "AZURE_CLIENT_SECRET",
    "AZURE_SUBSCRIPTION_ID",
    "AZURE_TENANT_ID",
    "GHCR_TOKEN"
  ],
  "missing_project_secrets": []
}
```

## Preview CD

```http
POST /api/v1/repositories/{repository_id}/cd/preview
```

Requires maintainer access.

Request:

```json
{
  "cloud_target_id": "uuid",
  "app_name": "dummy-fastapi",
  "image": "ghcr.io/hamza-hnt/dummy-fastapi",
  "image_tag": "02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0",
  "container_port": 8000,
  "replicas": 1,
  "service_type": "LoadBalancer",
  "kubeconfig_strategy": "azure_cli",
  "include_image_pull_secret": true,
  "image_pull_secret_name": "ghcr-pull-secret",
  "auto_map_project_secrets": true,
  "env_secret_mappings": [
    {
      "env_name": "API_TOKEN",
      "project_secret_key": "API_TOKEN",
      "github_secret_name": "API_TOKEN",
      "kubernetes_secret_name": "app-secrets"
    }
  ]
}
```

`kubeconfig_strategy` can be:

```txt
secret
azure_cli
```

For the first Azure AKS demo, `azure_cli` is recommended when the selected Cloud Node stores `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID`.

Use `secret` only when storing a raw kubeconfig as `KUBECONFIG_CONTENT`.

When `include_image_pull_secret` is true, the generated workflow creates a Kubernetes `docker-registry` secret from `GHCR_TOKEN`. The backend uses the project `cd / GHCR_TOKEN` if present, otherwise it falls back to the default platform container registry token.

Response contains generated files:

```json
{
  "cd_pipeline_id": "uuid",
  "repository_id": "uuid",
  "cloud_target_id": "uuid",
  "status": "generated",
  "app_name": "dummy-fastapi",
  "namespace": "cnp-demo",
  "image": "ghcr.io/hamza-hnt/dummy-fastapi",
  "image_tag": "02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0",
  "required_secrets": ["AZURE_CLIENT_ID", "AZURE_CLIENT_SECRET", "AZURE_SUBSCRIPTION_ID", "AZURE_TENANT_ID", "GHCR_TOKEN"],
  "missing_project_secrets": [],
  "files": [
    {
      "path": ".github/workflows/cd.yml",
      "content": "name: CD\n..."
    },
    {
      "path": "k8s/deployment.yaml",
      "content": "apiVersion: apps/v1\n..."
    }
  ]
}
```

## Approve CD

```http
POST /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/approve
```

Requires maintainer access.

Marks the generated CD files as approved before PR creation.

## Create CD Pull Request

```http
POST /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/create-pr
```

Requires maintainer access and an approved CD pipeline.

Request:

```json
{
  "base_branch": "main",
  "branch_name": "cnp/onboarding-cd",
  "commit_message": "chore(cnp): add generated CD workflow",
  "pull_request_title": "chore(cnp): add Kubernetes CD workflow",
  "pull_request_body": "This PR adds generated CD from CNP.",
  "sync_secrets_to_github": true
}
```

If `sync_secrets_to_github` is true, required secrets are pushed to GitHub Actions secrets before opening the PR. The backend resolves them in this order:

```txt
project cd secrets
-> global Cloud Node secrets, for AZURE_* or KUBECONFIG_CONTENT
-> default platform container registry token, for GHCR_TOKEN
```

The PR contains:

```txt
.github/workflows/cd.yml
k8s/namespace.yaml
k8s/secret.yaml
k8s/deployment.yaml
k8s/service.yaml
```

## List CD Pipelines

```http
GET /api/v1/repositories/{repository_id}/cd
```

Requires project viewer access.

Each item includes stored deployment metadata:

```json
{
  "id": "uuid",
  "cloud_target_id": "uuid",
  "status": "pr_created",
  "deployment_status": "deployed",
  "app_name": "dumdumappcnp",
  "namespace": "cnp-demo",
  "image": "ghcr.io/hamza-hnt/dumdumappcnp",
  "image_tag": "02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0",
  "external_ip": "134.112.165.194",
  "public_url": "http://134.112.165.194",
  "last_deployment_checked_at": "2026-05-30T12:00:00Z",
  "pull_request_url": "https://github.com/hamza-hnt/dumdumappcnp/pull/2",
  "created_at": "2026-05-30T11:58:00Z"
}
```

## Get Stored Deployment Status

```http
GET /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/deployment
```

Requires project viewer access.

This endpoint returns the last deployment status stored in CNP. It does not call Kubernetes.

Response:

```json
{
  "cd_pipeline_id": "uuid",
  "repository_id": "uuid",
  "cloud_target_id": "uuid",
  "status": "pr_created",
  "deployment_status": "deployed",
  "app_name": "dumdumappcnp",
  "namespace": "cnp-demo",
  "image": "ghcr.io/hamza-hnt/dumdumappcnp",
  "image_tag": "02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0",
  "external_ip": "134.112.165.194",
  "public_url": "http://134.112.165.194",
  "last_deployment_checked_at": "2026-05-30T12:00:00Z",
  "deployment_details": {
    "service_type": "LoadBalancer",
    "cluster_ip": "10.0.36.245"
  }
}
```

## Refresh Deployment Status

```http
POST /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/refresh-deployment
```

Requires project maintainer access.

This endpoint queries the target Kubernetes Service, stores the latest deployment status, and returns the refreshed metadata.

For Azure AKS with `azure_cli`, the backend uses encrypted global Cloud Node secrets or project `cd` secrets:

```txt
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
```

For AKS status refresh, CNP first tries AKS user credentials and falls back to AKS admin credentials when the user credential is unauthorized. The Azure service principal must therefore have the AKS Cluster Admin role in addition to the user role.

For raw kubeconfig mode, the backend uses:

```txt
KUBECONFIG_CONTENT
```

Deployment statuses:

```txt
unknown       no refresh has succeeded yet
pending       Kubernetes Service exists but LoadBalancer external IP is not assigned yet
deployed      external IP or hostname is available
failed        refresh reached the target provider or cluster but failed with an external error
not_deployed  project-level virtual status for repositories without a CD pipeline
```

Response body is the same as `GET /deployment`.

## List Project Deployments

```http
GET /api/v1/projects/{project_id}/deployments
```

Requires project viewer access. Platform admins can read all projects.

This endpoint returns one row per imported repository in the project and is intended for the project overview UI.

Repositories without CD return:

```json
{
  "repository_id": "uuid",
  "repository_full_name": "hamza-hnt/dumdumappcnp",
  "repository_name": "dumdumappcnp",
  "latest_cd_pipeline_id": null,
  "deployment_status": "not_deployed",
  "public_url": null
}
```

Repositories with refreshed CD status return:

```json
{
  "repository_id": "uuid",
  "repository_full_name": "hamza-hnt/dumdumappcnp",
  "repository_name": "dumdumappcnp",
  "latest_cd_pipeline_id": "uuid",
  "cloud_target_id": "uuid",
  "cloud_target_name": "demo-aks",
  "cloud_target_environment": "demo",
  "app_name": "dumdumappcnp",
  "pipeline_status": "pr_created",
  "deployment_status": "deployed",
  "namespace": "cnp-demo",
  "image": "ghcr.io/hamza-hnt/dumdumappcnp",
  "image_tag": "02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0",
  "external_ip": "134.112.165.194",
  "public_url": "http://134.112.165.194",
  "last_deployment_checked_at": "2026-05-30T12:00:00Z",
  "updated_at": "2026-05-30T12:00:00Z"
}
```

## Deployment URL

The backend can now refresh and store the Kubernetes Service external IP through:

```txt
POST /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/refresh-deployment
```

Manual fallback:

```bash
az aks get-credentials --resource-group rg-cnp-demo --name aks-cnp-demo --overwrite-existing
kubectl -n cnp-demo get svc <app-name>
```

When `EXTERNAL-IP` is available, the public URL is:

```txt
http://<external-ip>
```
