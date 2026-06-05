
# Event Notification Microservice — Project Specification

## Overview
We are building a SaaS platform for managing online workshops and need a **Notification Microservice** responsible for sending automated emails to users when certain events occur.

The service will integrate with our existing platform via **REST API**. It must be designed to be **reliable, scalable, and easy to extend**.

---

# Objectives

Build a backend service that:

1. Receives notification requests from external systems via REST API.
2. Stores notifications in a database.
3. Processes notifications asynchronously.
4. Sends email notifications using a third‑party provider.
5. Supports scheduled delivery and retry logic.

---

# Core Features

## 1. REST API

### Create Notification

`PUT /notifications/{id}`

Example request:

```json
{
  "user_id": "12345",
  "email": "user@example.com",
  "event_type": "WORKSHOP_REMINDER",
  "locale": "en",
  "payload": {
    "workshop_name": "Intro to Rust",
    "start_time": "2026-03-05T15:00:00Z"
  },
  "send_at": "2026-03-05T14:30:00Z"
}
```

`send_at` is optional. If omitted send immediately.

Behavior:

- Validates request
- Stores notification in database
  - If record with given `id` already exists, return failure (do NOT overwrite)
- Schedules background processing

Response:

```json
{
  "id": "uuid",
  "status": "scheduled"
}
```

---

### Get Notification

`GET /notifications/{id}`

Example response:

```json
{
  "id": "uuid",
  "status": "scheduled|pending|sent|failed",
  "updated_at": "timestamp|null",
  "retry_count": 0
}
```

---

### List Notifications

`GET /notifications`

Supported filters:

- `status`
- `event_type`
- `user_id`

Example:

```
GET /notifications?status=sent&event_type=WORKSHOP_REMINDER
```

---

# Email Templates

## Template Selection

Templates are selected using:

- `event_type`
- optional `locale`

Example:

```
template_key = event_type
```

If a template for the requested locale does not exist, the service should **fall back to English (`en`)**.

---

## Template Storage

For version 1 of the system, templates will be stored **in the repository as files**.

Example structure:

```
/templates/email/en/WORKSHOP_REMINDER.subject.hbs
/templates/email/en/WORKSHOP_REMINDER.body.hbs
```

Template engine can be chosen by the developer (Handlebars / Mustache / Jinja / etc.).

Example template:

Subject:
```
Reminder: {{workshop_name}}
```

Body:

```
Hello,

This is a reminder that your workshop "{{workshop_name}}" starts at {{start_time}}.

Thank you.
```

---

# Background Processing

Email sending must be **asynchronous**.

Workflow:

1. API receives notification
2. Notification stored in DB
3. Job placed in queue
4. Worker processes jobs
5. Worker sends email when `send_at` time is reached

---

# Retry Logic

If email sending fails:

Retry schedule:

| Attempt | Delay      |
|---------|------------|
| 1       | 1 minute   |
| 2       | 5 minutes  |
| 3       | 15 minutes |

After the third retry the notification status becomes **failed**.

---

# Notification Status Lifecycle

Possible statuses:

```
scheduled
pending
sent
failed
```

Example lifecycle:

```
scheduled → pending → sent
scheduled → pending → failed
```

---

# Expected Traffic

Approximate load expectations:

### Write Requests

- ~20,000 notification requests per day

### Read Requests

- ~5,000 requests per day

### Typical Request Rate

- Average: **5–20 requests per minute**
- Peak bursts: **200–500 requests per minute**

Traffic is **bursty**, especially around workshop start times.

---

# Fan-out Notifications

Sometimes a single event may require notifying many users.

Example:

- workshop canceled
- schedule changed

Typical sizes:

- 50–500 users
- up to ~10,000 users in rare cases

For **version 1**, external systems will send:

**one API request per user**.

Optional (nice-to-have):

```
POST /notifications/bulk
```

Which accepts a list of recipients and expands them internally.

This endpoint is **not required for the first version**.

---

# System Architecture

Expected components:

- API Service
- Background Worker
- Database
- Queue System

---

# Database Schema (Suggested)

Table: `notifications`

| column      | type      |
|-------------|-----------|
| id          | uuid      |
| user_id     | string    |
| email       | string    |
| event_type  | string    |
| locale      | string    |
| payload     | jsonb     |
| status      | string    |
| send_at     | timestamp |
| retry_count | integer   |
| created_at  | timestamp |
| updated_at  | timestamp |

Indexes:

- `(status)`
- `(send_at)`
- `(user_id)`

---

# Local Development

The service must run locally using Docker.

Example:

```
docker-compose up
```

---

# Deliverables

The repository should include:

### Source Code

- API service
- worker
- queue integration
- email templates
- database migrations

### Documentation

README including:

- architecture overview
- setup instructions
- API examples

---

# Acceptance Criteria

The project will be considered complete when:

1. API endpoints work as specified.
2. Notifications are persisted correctly.
3. Emails are rendered from templates.
4. Scheduled sending respects `send_at`.
5. Retry logic works.
6. Status transitions are correct.
7. System runs locally using Docker.
8. Code is readable and production-quality.

---

# Nice-to-Have (Optional)

If time permits:

- OpenAPI / Swagger documentation
- `/health` endpoint
- `/metrics` endpoint
- basic rate limiting
- bulk notification endpoint
