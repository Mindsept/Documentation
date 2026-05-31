# Azure AKS CD Demo Flow

This runbook documents the demo path that was validated end to end:

```txt
repository import
-> analysis
-> CI generation
-> CI Pull Request
-> GHCR image push
-> CD generation
-> CD Pull Request
-> Azure AKS deployment
-> public LoadBalancer endpoint
```

The platform does not provision AKS in this flow. AKS is created outside CNP, for example with Terraform/OpenTofu, then registered once as a global reusable Cloud Node.

The generation internals for `.github/workflows/cd.yml` and `k8s/*.yaml` are described in [CI/CD Generation Internals](ci-cd-generation.md).

## Preconditions

- A GitHub App installation is linked to CNP.
- The application repository contains a valid `Dockerfile`.
- The repository can be analyzed by CNP.
- An Azure AKS cluster already exists.
- The AKS cluster has a namespace prepared for the demo, for example `cnp-demo`.
- The GitHub package image can be pulled by Kubernetes.

## 1. Configure The Shared GHCR Registry

Use a platform admin account.

Go to:

```txt
Admin -> Container Registry
```

Register GHCR once:

```txt
name: ghcr
provider: ghcr
registry_url: ghcr.io
auth_secret_name: GHCR_TOKEN
token: <GitHub token with packages write/read permissions>
default: true
```

The token is encrypted as a platform secret. CI/CD PR creation can then sync `GHCR_TOKEN` to repository GitHub Actions secrets without requiring each project to store the same token.

## 2. Register The Global Cloud Node

Use a platform admin account.

Go to:

```txt
Admin -> Cloud Nodes
```

Register the AKS target from:

```bash
tofu output -json cnp_cd_contract
```

The admin form stores the Cloud Node globally and encrypts the Azure CD secrets as platform secrets.

For the Azure CLI strategy, provide:

```txt
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
```

The Azure service principal used by those values must have enough access to get AKS credentials and allow CNP to refresh deployment status:

```txt
Reader
Azure Kubernetes Service Cluster User Role
Azure Kubernetes Service Cluster Admin Role
```

The default demo values are:

```txt
Kubeconfig strategy: azure_cli
Service type: LoadBalancer
Public URL format: http://<external-load-balancer-ip>
```

`KUBECONFIG_CONTENT` is only required when the CD preview uses `kubeconfig_strategy = secret`.

## 3. Configure Project Runtime Secrets

If the application has runtime variables from `.env.example`, store those runtime values at project level under:

```txt
environment: cd
```

Example:

```txt
environment: cd
key: GREETING
```

`GHCR_TOKEN` and `AZURE_*` do not need to be duplicated per project when the global registry and global Cloud Node are configured.

## 4. Generate And Merge CI

For a full CI/CD demo, generate CI with:

```txt
Build Docker image: ON
Push image on main: ON
Registry: ghcr.io
```

After the CI PR is merged to `main`, GitHub Actions pushes an image like:

```txt
ghcr.io/<owner>/<repo>:<commit-sha>
```

Use the exact pushed image tag for the CD preview.

Example:

```txt
Image: ghcr.io/hamza-hnt/dumdumappcnp
Image tag: 02cdbd7884b3e95e2a5900f19e99eab04b1f2ed0
```

## 5. Generate And Merge CD

On the repository CD page, select the AKS Cloud Node.

Recommended demo options:

```txt
Kubeconfig strategy: azure_cli
Include image pull secret: ON, if GHCR package is private
Image pull secret name: ghcr-pull-secret
Auto-map project secrets: ON
Container port: application container port, usually 8000 for FastAPI
Replicas: 1
Service type: LoadBalancer
```

Approve the preview and open the CD PR.

The PR contains:

```txt
.github/workflows/cd.yml
k8s/namespace.yaml
k8s/secret.yaml
k8s/deployment.yaml
k8s/service.yaml
```

After the PR is merged to `main`, GitHub Actions deploys to AKS.

## 6. Refresh The Public URL In CNP

After the CD workflow succeeds, refresh the deployment status through CNP:

```txt
POST /api/v1/repositories/{repository_id}/cd/{cd_pipeline_id}/refresh-deployment
```

The backend reads the Kubernetes Service, stores the external IP, and returns:

```json
{
  "deployment_status": "deployed",
  "external_ip": "<external-load-balancer-ip>",
  "public_url": "http://<external-load-balancer-ip>"
}
```

The LoadBalancer IP is discovered from Kubernetes. In this demo setup it is not guaranteed to be stable across infrastructure recreation. If the AKS cluster, Azure public IP, or Kubernetes `Service` is destroyed and recreated, refresh the deployment status again and use the new URL returned by CNP.

Project-level deployment cards can then read:

```txt
GET /api/v1/projects/{project_id}/deployments
```

## Manual Fallback

If the refresh endpoint cannot reach Azure or Kubernetes, use:

```bash
az aks get-credentials --resource-group rg-cnp-demo --name aks-cnp-demo --overwrite-existing
kubectl -n cnp-demo get svc <app-name>
```

When the service receives an external IP:

```txt
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)
dumdumappcnp   LoadBalancer   10.0.36.245   <external-ip>     80:31959/TCP
```

The public URL is:

```txt
http://<external-ip>
```

FastAPI demo endpoints can be tested with:

```bash
curl -s http://<external-ip>/
curl -s http://<external-ip>/health
```

## Known Current Limitation

CNP can refresh deployment status on demand, but it does not yet poll automatically after a CD PR is merged.

```txt
Current: user clicks Refresh deployment status
Later: background job or GitHub webhook triggers refresh automatically
```
