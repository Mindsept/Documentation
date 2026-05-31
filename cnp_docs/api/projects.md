# Projects API

Projects group imported repositories, members, and secrets.

All project endpoints require platform authentication.

## Roles

```txt
owner       full project control, including members
maintainer  repository onboarding, CI generation, secrets
viewer      read access
```

Platform admins are not project roles. They can list all projects, manage legacy project Cloud Nodes, and manage global Cloud Nodes/container registries, even when they are not explicit members of each project.

## Create Project

```http
POST /api/v1/projects
```

Creates a project and assigns the authenticated user as owner.

Request:

```json
{
  "name": "CNP Demo Project",
  "slug": "cnp-demo",
  "description": "Demo project for Cloud Native Platform"
}
```

Response `201`:

```json
{
  "id": "uuid",
  "name": "CNP Demo Project",
  "slug": "cnp-demo",
  "description": "Demo project for Cloud Native Platform",
  "owner_id": "uuid",
  "created_at": "2026-05-20T12:00:00Z"
}
```

## List Projects

```http
GET /api/v1/projects
```

Lists projects where the current user is a member. Platform admins receive all projects.

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "name": "CNP Demo Project",
      "slug": "cnp-demo",
      "role": "owner",
      "repositories_count": 2,
      "created_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

When an admin sees a project without explicit membership, `role` is returned as:

```txt
admin
```

## Get Project

```http
GET /api/v1/projects/{project_id}
```

Returns project details and members.

Response `200`:

```json
{
  "id": "uuid",
  "name": "CNP Demo Project",
  "slug": "cnp-demo",
  "description": "Demo project",
  "owner_id": "uuid",
  "members": [
    {
      "project_id": "uuid",
      "user_id": "uuid",
      "email": "demo@cnp.dev",
      "role": "owner"
    }
  ]
}
```

## Add or Update Member

```http
POST /api/v1/projects/{project_id}/members
```

Requires project owner role.

Request:

```json
{
  "email": "member@example.com",
  "role": "maintainer"
}
```

Response `200`:

```json
{
  "project_id": "uuid",
  "user_id": "uuid",
  "email": "member@example.com",
  "role": "maintainer"
}
```
