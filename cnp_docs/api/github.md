# GitHub App API

The GitHub integration uses a GitHub App installation. The backend generates temporary installation access tokens server-side.

## Get Install URL

```http
GET /api/v1/github/install-url
```

Requires authentication.

Returns a GitHub App installation URL with a signed state token.

Response `200`:

```json
{
  "install_url": "https://github.com/apps/my-app/installations/new?state=..."
}
```

## Installation Callback

```http
GET /api/v1/github/setup-callback?installation_id=123&setup_action=install&state=...
```

GitHub redirects to this endpoint after App installation. The endpoint validates the signed state, links the installation to the encoded platform user, and redirects the browser to the frontend callback page.

Response `302`:

```txt
Location: http://localhost:5173/github/callback?installation_id=123456&installation_db_id=uuid&account_login=example-org&account_type=Organization&setup_action=install
```

`/api/v1/github/callback` is kept as a backward-compatible alias and has the same redirect behavior.

## List Installations

```http
GET /api/v1/github/installations
```

Requires authentication.

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "installation_id": 123456,
      "github_account_login": "example-org",
      "github_account_type": "Organization",
      "repository_selection": "selected",
      "created_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

## List Installation Repositories

```http
GET /api/v1/github/installations/{installation_id}/repositories
```

`installation_id` is the platform UUID for the linked installation, not the GitHub numeric installation id.

The backend fetches repositories from GitHub and caches them locally.

Response `200`:

```json
{
  "items": [
    {
      "github_repo_id": 987654,
      "full_name": "example-org/api-orders",
      "name": "api-orders",
      "owner_login": "example-org",
      "private": true,
      "default_branch": "main",
      "html_url": "https://github.com/example-org/api-orders"
    }
  ]
}
```

## Local GitHub App URLs

For local development through ngrok:

```txt
Setup URL:   https://your-ngrok-domain.ngrok-free.dev/api/v1/github/setup-callback
Webhook URL: https://your-ngrok-domain.ngrok-free.dev/api/v1/webhooks/github
```
