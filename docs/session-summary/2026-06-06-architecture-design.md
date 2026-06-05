# 2026-06-06 - Architecture Design

We reviewed the notification microservice requirements and created an initial architecture proposal in `architecture.md`.

After the requirements changed, we updated the architecture draft to reflect the current API contract, especially `PUT /notifications/{id}`, insert-only notification creation, duplicate-ID failure, optional `send_at`, engine-neutral templates, and the removal of `sent_at`.

We then discussed the database transaction and queue publishing boundary. The architecture was updated to use a concise transactional outbox approach: the API writes both the notification and an outbox row in one database transaction, a dispatcher publishes durable queue messages, and outbox rows are deleted only after confirmed publish.

The resulting architecture draft now captures the agreed high-level component boundaries and reliability approach without going into implementation details.
