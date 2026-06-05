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
+--------------+        +---------------+        +----------------+                   +------------------+
| API Service  | -----> |   Database    | <----- | Background     | ----------------> | Email Provider   |
| & Dispatcher |        | notifications |        | Worker         | render template   +------------------+
+--------------+        +---------------+        +----------------+ and send email
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
- Create a durable outbox entry in the same database transaction as the notification.
- Return notification status and filtered notification lists.

Required endpoints:

- `PUT /notifications/{id}`
- `GET /notifications/{id}`
- `GET /notifications`

The API service should not send emails directly or publish queue messages directly. It should only create durable database state.

### Outbox Dispatcher

Responsibilities:

- Read notification IDs from the transactional outbox table.
- Publish durable queue messages containing only the notification ID.
- Delete outbox rows only after the queue confirms the message was published.

The dispatcher is intentionally small. The outbox table is the handoff between API writes and queue publishing.

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
- `(event_type)` if list filtering by event type is expected to grow

Outbox table: `notification_outbox`

| Column            | Purpose                                       |
|-------------------|-----------------------------------------------|
| `notification_id` | Primary key and foreign key to notifications. |

The outbox table does not need a separate ID or audit fields for v1. Its only purpose is to durably record that a notification still needs to be published to the queue.

### Queue System

The queue decouples outbox dispatch from email delivery.

Required behavior:

- Accept durable jobs from the outbox dispatcher.
- Deliver jobs asynchronously to workers.
- Support reliable acknowledgements so messages are not lost if a worker crashes before finishing processing.
- Support delayed or scheduled execution if available.
- Support retry-safe processing.

If the selected queue supports delayed jobs, the worker can use delayed messages for `send_at` and retry delays. If it does not, the worker can requeue delayed jobs or periodically scan the database for due scheduled notifications.

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
  -> in one database transaction:
       insert notification with path id and status = scheduled
       insert notification_outbox row with the same id
  -> fail if a notification with the same id already exists
  -> return 202 - Accepted
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

### Transactional Outbox

The API service should insert the notification and its outbox row in the same database transaction. The queue job should contain only the notification ID, not the full email payload.

The dispatcher publishes outbox rows to the queue and deletes each outbox row only after the queue confirms durable publish. This avoids the failure mode where the notification is stored but no queue job is ever created.

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

Successful response status: 202 - Accepted. No response body.

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

- API + dispatcher service container.
- Worker container.
- Database container.
- Queue container.

The documented startup command should come from the actual manifests added to the repo. Do not assume package manager, build command, or runtime until those files exist.

## Recommended V1 Build Order

1. Define database migrations for `notifications` and `notification_outbox`.
2. Implement notification domain model and status transitions.
3. Implement `PUT /notifications/{id}` with insert-only persistence, outbox insert, and duplicate-ID failure.
4. Add outbox dispatcher and queue consumer.
5. Add template loading and rendering with English fallback.
6. Add email provider adapter with a local/dev fake provider.
7. Implement retry scheduling and final failure state.
8. Implement `GET /notifications/{id}` and filtered `GET /notifications`.
9. Add Docker-based local runtime.
10. Add README setup and API examples.

## Tech Stack

These should be decided before code is written:

- Backend language and framework.
  - Java / Spring Boot
- Database engine.
  - ?
- Queue technology.
  - ?
- Email provider.
  - ?

## Minimal Recommended Architecture

For the smallest production-shaped v1:

- One API + dispatcher service process.
- One worker process, horizontally scalable.
- One relational database for durable notification state.
- One queue for async and delayed work.
- Repository-based email templates.
- Email provider isolated behind an adapter.
