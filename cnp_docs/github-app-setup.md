# GitHub App Setup

The platform integrates with GitHub through a GitHub App. It does not require permanent personal access tokens.

## Required GitHub App Permissions

Configure the GitHub App with the minimum permissions needed for V1:

```txt
Metadata: read
Contents: read/write
Pull requests: read/write
Actions: read
Workflows: write, when available
Actions secrets: write, if CI secret synchronization is enabled
Checks: read
Commit statuses: read
```

## Webhook Events

Subscribe to:

```txt
push
pull_request
workflow_run
check_suite, optional
```

## Local Development URLs

When using ngrok, point GitHub to the public ngrok domain.

Example:

```txt
Setup URL:   https://your-ngrok-domain.ngrok-free.dev/api/v1/github/setup-callback
Webhook URL: https://your-ngrok-domain.ngrok-free.dev/api/v1/webhooks/github
```

ngrok should forward to the local Nginx service:

```txt
https://your-ngrok-domain.ngrok-free.dev -> http://localhost:8080
```

## Environment Variables

Set these values in `.env`:

```env
PUBLIC_API_URL="https://your-ngrok-domain.ngrok-free.dev"
FRONTEND_URL="http://localhost:5173"
GITHUB_APP_ID="..."
GITHUB_APP_SLUG="..."
GITHUB_APP_CLIENT_ID="..."
GITHUB_APP_CLIENT_SECRET="..."
GITHUB_APP_PRIVATE_KEY_BASE64="..."
GITHUB_WEBHOOK_SECRET="..."
GITHUB_CALLBACK_URL="https://your-ngrok-domain.ngrok-free.dev/api/v1/github/setup-callback"
```

## Private Key Encoding

Download the GitHub App private key from GitHub and encode it as one base64 line.

Linux:

```bash
base64 -w 0 your-github-app.private-key.pem
```

macOS:

```bash
base64 < your-github-app.private-key.pem | tr -d '\n'
```

Put the output in:

```env
GITHUB_APP_PRIVATE_KEY_BASE64="..."
```

Never commit the `.pem` file.

## Webhook Secret

Generate a webhook secret:

```bash
openssl rand -hex 32
```

Use the same value in:

```env
GITHUB_WEBHOOK_SECRET="..."
```

and in the GitHub App webhook settings.

## Production Notes

For the validated production deployment, use:

```txt
Setup URL:   https://cnp-api.mindsept.fr/api/v1/github/setup-callback
Webhook URL: https://cnp-api.mindsept.fr/api/v1/webhooks/github
```

Backend `.env`:

```env
PUBLIC_API_URL="https://cnp-api.mindsept.fr"
FRONTEND_URL="https://cnp.mindsept.fr"
GITHUB_CALLBACK_URL="https://cnp-api.mindsept.fr/api/v1/github/setup-callback"
```

After installation, the backend redirects to:

```txt
https://cnp.mindsept.fr/github/callback
```

Because the frontend is a React SPA deployed on Vercel, the frontend repository must include a rewrite rule to serve `index.html` for `/github/callback` and all other deep routes.

The callback and webhook URLs can be changed later in the GitHub App settings.
