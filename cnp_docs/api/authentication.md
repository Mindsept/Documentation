# Authentication API

The platform has its own email/password authentication. GitHub App authentication is used only for repository integration.

## Register

```http
POST /api/v1/auth/register
```

Creates a platform user account.

Request:

```json
{
  "email": "user@example.com",
  "password": "StrongPassword123!",
  "full_name": "Example User"
}
```

Response `201`:

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "full_name": "Example User",
  "role": "user",
  "is_active": true,
  "created_at": "2026-05-20T12:00:00Z"
}
```

## Login

```http
POST /api/v1/auth/login
```

Authenticates a user and returns tokens.

Request:

```json
{
  "email": "demo@cnp.dev",
  "password": "demo12345"
}
```

Response `200`:

```json
{
  "access_token": "jwt",
  "refresh_token": "opaque-refresh-token",
  "token_type": "bearer",
  "expires_in": 1800
}
```

Use the access token on protected requests:

```http
Authorization: Bearer <access_token>
```

## Refresh

```http
POST /api/v1/auth/refresh
```

Issues a new short-lived access token.

Request:

```json
{
  "refresh_token": "opaque-refresh-token"
}
```

Response `200`:

```json
{
  "access_token": "jwt",
  "refresh_token": "opaque-refresh-token",
  "token_type": "bearer",
  "expires_in": 1800
}
```

## Current User

```http
GET /api/v1/auth/me
GET /api/v1/users/me
```

Returns the authenticated user profile.

Response `200`:

```json
{
  "id": "uuid",
  "email": "demo@cnp.dev",
  "full_name": "Demo User",
  "role": "user"
}
```

## Development Users

After running the seed script:

```txt
admin@cnp.dev / admin12345
demo@cnp.dev  / demo12345
```
