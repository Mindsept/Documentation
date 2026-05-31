# Webhooks API

The webhook API receives GitHub App webhook events.

## GitHub Webhook

```http
POST /api/v1/webhooks/github
```

This endpoint is called by GitHub, not by the frontend.

Required headers:

```http
X-Hub-Signature-256: sha256=...
X-GitHub-Event: push
X-GitHub-Delivery: uuid
```

Response `200`:

```json
{
  "received": true
}
```

## Signature Verification

The backend verifies `X-Hub-Signature-256` with:

```env
GITHUB_WEBHOOK_SECRET="..."
```

The same secret must be configured in the GitHub App webhook settings.

## Local Development URL

With ngrok forwarding to local Nginx:

```txt
https://your-ngrok-domain.ngrok-free.dev/api/v1/webhooks/github
```

## Supported Events

V1 stores and audits useful events, with emphasis on:

```txt
push
pull_request
workflow_run
check_suite
```

Push events on `main` can be used to update repository state and audit activity.
