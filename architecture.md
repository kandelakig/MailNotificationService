# Notification Microservice Architecture Proposal

## Scope

This document proposes a version 1 architecture for the Event Notification Microservice described in `project_requirements.md`.

The design focuses on the required capabilities only:

- Receive notification requests through REST API endpoints.
- Persist notifications in a database.
- Process email delivery asynchronously.
- Support scheduled delivery with `send_at` (optional, if omitted, deliver immediately).
- Render repository-based email templates.
- Retry failed sends using the required retry schedule.
- Expose notification status through read endpoints.
- Run locally with Docker once implementation manifests are added.

Optional features such as bulk notification creation, metrics, health checks, OpenAPI, and rate limiting should be added only after the core acceptance criteria are implemented.

## High-Level Architecture

```text
External Platform
      |
      | REST
      v
+-------------+        +---------------+        +----------------+                   +------------------+
| API Service | -----> |   Database    | <----- | Background     | ----------------> | Email Provider   |
|             |        | notifications |        | Worker         | render template   +------------------+
+-------------+        +---------------+        +----------------+ and send email
      |                                               A
      |                                       consume |
      |                                  notification |
      | enqueue job    +-------------+           jobs |
      +--------------> | Queue       | ---------------+
                       | System      |
                       +-------------+
```

## Components

### API Service

Responsibilities:

- Publish a REST API.
- Validate incoming requests.
- Persist notification records.
  - Only create new records, never update existing records.
  - Return a failure, preferably `409 Conflict`, if a notification with the requested `id` already exists.
- Enqueue background work after successful persistence.
- Return notification status and filtered notification lists.

Required endpoints:

- `PUT /notifications/{id}`
- `GET /notifications/{id}`
- `GET /notifications`

The API service should not send emails directly. It should only create durable work for the worker.

### Background Worker

Responsibilities:

- Consume notification jobs from the queue.
- Load the latest notification state from the database.
- Respect `send_at` before attempting delivery. If `send_at` is empty, deliver immediately.
- Transition notifications through the required lifecycle.
- Render email subject and body from templates.
- Send email through a third-party provider.
- Apply retry delays after failed attempts.

The worker must be safe to run as multiple instances. Any notification claim/update logic should prevent duplicate sends when workers run concurrently.

### Database

The database is the durable source of truth for notification state.

Initial table: `notifications`

Recommended fields based on the requirements:

| Column        | Purpose                                           |
|---------------|---------------------------------------------------|
| `id`          | Stable notification identifier, preferably UUID.  |
| `user_id`     | External platform user identifier.                |
| `email`       | Recipient email address.                          |
| `event_type`  | Template and notification category.               |
| `locale`      | Requested locale, defaulting to `en` when absent. |
| `payload`     | JSON object used for template rendering.          |
| `status`      | `scheduled`, `pending`, `sent`, or `failed`.      |
| `send_at`     | Earliest delivery time.                           |
| `retry_count` | Number of failed send attempts already recorded.  |
| `created_at`  | Creation timestamp.                               |
| `updated_at`  | Last update timestamp.                            |

Recommended indexes:

- `status`
- `send_at`
- `user_id`
- `(status, send_at)` for worker polling or scheduled job lookup
- `(event_type)` if list filtering by event type is expected to grow

### Queue System

The queue decouples API write traffic from email delivery.

Required behavior:

- Accept a job after `PUT /notifications/{id}` persists the notification.
- Deliver jobs asynchronously to workers.
- Support delayed or scheduled execution if available.
- Support retry-safe processing.

If the selected queue supports delayed jobs, schedule jobs using `send_at` and retry delays directly in the queue. If it does not, the worker can requeue delayed jobs or periodically scan the database for due scheduled notifications.

For v1, prefer the simplest queue setup that can run in Docker and meet the required retry and scheduled delivery behavior.

### Email Provider Adapter

Use a small adapter boundary around the third-party email provider.

Responsibilities:

- Accept normalized email data: recipient, subject, body, and metadata.
- Hide provider-specific API details from the worker.
- Return clear success or failure results.
- Preserve provider message IDs if available.

This keeps the worker logic stable if the email provider changes later.

### Template Renderer

Templates are repository files, selected by `event_type` and `locale`.

Example layout:

```text
templates/email/en/WORKSHOP_REMINDER.subject.hbs
templates/email/en/WORKSHOP_REMINDER.body.hbs
```

Rendering rules:

- Try requested locale first.
- Fall back to `en` if the requested locale template is missing.
- Treat missing English fallback templates as a delivery failure.
- Render both subject and body with the stored `payload`.

Template rendering should happen inside the worker so API requests remain fast and notification payloads are persisted before any provider call is attempted.

## Notification Lifecycle

### Create Flow

```text
PUT /notifications/{id}
  -> validate request
  -> insert notification with path id and status = scheduled
  -> fail if a notification with the same id already exists
  -> enqueue notification job
  -> return { id, status }
```

Default status for newly created notifications should be `scheduled`, even when `send_at` is immediate or omitted. The worker then transitions the record to `pending` when it starts a delivery attempt.

Create behavior is insert-only. A repeated request with an existing ID must not overwrite the stored notification.

### Delivery Flow

```text
worker receives job
  -> load notification
  -> skip if status is sent or failed
  -> wait/requeue if send_at is in the future
  -> atomically claim notification as pending
  -> render templates
  -> call email provider
  -> on success: status = sent
  -> on failure: retry or mark failed
```

The `pending` state should represent an active send attempt, not a long-term queue state.

### Retry Flow

Required retry schedule:

| Failed Attempt | Next Delay |
|----------------|------------|
| 1              | 1 minute   |
| 2              | 5 minutes  |
| 3              | 15 minutes |

After the third retry is exhausted, set status to `failed`.

The architecture assumes this means three retries after the initial attempt:

- Initial attempt fails: increment `retry_count` to `1`, schedule next attempt after 1 minute.
- Second failed attempt: increment `retry_count` to `2`, schedule next attempt after 5 minutes.
- Third failed attempt: increment `retry_count` to `3`, schedule next attempt after 15 minutes.
- Fourth failed attempt: mark `failed`.

## Consistency And Reliability

### Persist Before Enqueue

The API service should insert the notification before creating the queue job. The queue job should contain only the notification ID, not the full email payload.

This ensures the database remains the durable source of truth and avoids queue/database payload drift.

### Transaction Boundary

Preferred v1 approach:

- Insert notification in a database transaction.
- Commit the transaction.
- Enqueue a job with the notification ID.

Risk: if enqueue fails after commit, the notification remains stored but may not be processed.

Mitigation options:

- For the smallest v1, return an error if enqueue fails and leave the record visible for diagnosis.
- Better production option: add an outbox table so notification creation and work scheduling are committed atomically, then have a dispatcher publish queue jobs.

The outbox pattern is more reliable, but it adds moving parts. It should be considered if reliability is prioritized over v1 simplicity.

### Idempotency

- Workers should ignore notifications already in `sent` or `failed` status.
- Worker claim should be atomic, for example updating status only when current status allows processing.
- Provider calls should include a stable idempotency key if the provider supports it.

### Concurrency

Workers should use one of these strategies to avoid duplicate delivery:

- Atomic conditional update from `scheduled` to `pending`.
- Row-level locking with skip-locked semantics if using database polling.
- Queue-level single-consumer guarantees combined with database status checks.

The database status check should still exist even if the queue claims to deliver each job once.

## Suggested Directory Boundaries

The exact stack is not defined yet, so these are conceptual boundaries rather than language-specific package names.

```text
src/
  api/              REST routes, request validation, response mapping
  worker/           queue consumers and delivery orchestration
  queue/            queue publisher and consumer integration
  notifications/    notification domain logic and status transitions
  email/            provider adapter and email sending contracts
  templates/        template discovery and rendering logic
  db/               database access and migrations integration
templates/
  email/
    en/
      WORKSHOP_REMINDER.subject.hbs
      WORKSHOP_REMINDER.body.hbs
migrations/
docker-compose.yml
README.md
```

Keep API service, worker, queue integration, templates, and migrations separated as the repo instructions require.

## API Design Notes

### `PUT /notifications/{id}`

Validation should require:

- `id` path parameter
- `user_id`
- `email`
- `event_type`
- `payload`

Validation should normalize or default:

- `locale`: default to `en` when absent.
- `send_at`: default to immediate delivery when absent.

If a notification with the requested `id` already exists, return a failure and do not update the existing record. The recommended HTTP response is `409 Conflict`.

Response status should match the specification:

```json
{
  "id": "uuid",
  "status": "scheduled"
}
```

### `GET /notifications/{id}`

Return only public status fields required by the spec:

```json
{
  "id": "uuid",
  "status": "scheduled|pending|sent|failed",
  "updated_at": "timestamp|null",
  "retry_count": 0
}
```

### `GET /notifications`

Support filters:

- `status`
- `event_type`
- `user_id`

Add pagination in v1 even if not explicitly stated. Without pagination, this endpoint can become unsafe as notification volume grows. A minimal approach is `limit` plus `cursor` or `offset`.

## Local Development

Once a stack is chosen, local Docker should include at minimum:

- API service container.
- Worker container.
- Database container.
- Queue container, unless using the database as the queue for v1.

The documented startup command should come from the actual manifests added to the repo. Do not assume package manager, build command, or runtime until those files exist.

## Recommended V1 Build Order

1. Define database migration for `notifications`.
2. Implement notification domain model and status transitions.
3. Implement `PUT /notifications/{id}` with insert-only persistence and duplicate-ID failure.
4. Add queue publisher and worker consumer.
5. Add template loading and rendering with English fallback.
6. Add email provider adapter with a local/dev fake provider.
7. Implement retry scheduling and final failure state.
8. Implement `GET /notifications/{id}` and filtered `GET /notifications`.
9. Add Docker-based local runtime.
10. Add README setup and API examples.

## Open Decisions Before Implementation

These should be decided before code is written:

- Backend language and framework.
  - Java / Spring Boot
- Database engine.
  - ?
- Queue technology.
  - ?
- Email provider.
  - ?
- Whether API callers need idempotency keys for duplicate prevention.

## Minimal Recommended Architecture

For the smallest production-shaped v1:

- One API service process.
- One worker process type, horizontally scalable.
- One relational database for durable notification state.
- One queue for async and delayed work.
- Repository-based email templates.
- Email provider isolated behind an adapter.

This satisfies the current acceptance criteria while keeping optional endpoints and operational features out of the initial implementation path.
