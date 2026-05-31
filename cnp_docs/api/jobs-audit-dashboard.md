# Jobs, Audit, Dashboard, and Health APIs

These endpoints support asynchronous workflows, operational visibility, and health checks.

## Jobs

```http
GET /api/v1/jobs/{job_id}
```

Requires authentication.

Returns status and result for a queued backend job.

Response `200`:

```json
{
  "id": "uuid",
  "type": "analyze_repository",
  "status": "succeeded",
  "payload": {},
  "result": {},
  "error": null,
  "created_at": "2026-05-20T12:00:00Z",
  "started_at": "2026-05-20T12:00:01Z",
  "finished_at": "2026-05-20T12:00:08Z"
}
```

Job statuses:

```txt
queued
running
succeeded
failed
```

Job types:

```txt
analyze_repository
generate_ci
commit_ci_pr
```

## Audit

```http
GET /api/v1/audit?project_id={project_id}&limit=20
```

Requires authentication.

Returns recent audit events. `project_id` is optional. `limit` defaults to `20` and is capped at `100`.

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "actor_id": "uuid",
      "project_id": "uuid",
      "repository_id": "uuid",
      "action": "repository.analyzed",
      "entity_type": "repository",
      "entity_id": "uuid",
      "payload": {},
      "created_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

Audited actions include:

```txt
user.login
project.created
github.installation.linked
repository.imported
repository.analyzed
ci.generated
ci.adapted_by_ai
ci.approved
ci.pr_created
secret.created
secret.updated
webhook.received
```

## Dashboard

```http
GET /api/v1/dashboard/summary
```

Requires authentication.

Response `200`:

```json
{
  "projects_count": 2,
  "repositories_count": 5,
  "repositories_analyzed_count": 4,
  "ci_generated_count": 3,
  "ci_pr_created_count": 2,
  "secrets_count": 6,
  "recent_events": [
    {
      "type": "repository.analyzed",
      "message": "Repository example-org/api-orders analyzed as python_fastapi",
      "created_at": "2026-05-20T12:00:00Z"
    }
  ]
}
```

## Health

```http
GET /api/v1/health
```

Does not require authentication.

Response `200`:

```json
{
  "status": "ok",
  "service": "Cloud Native Platform API",
  "environment": "development"
}
```
