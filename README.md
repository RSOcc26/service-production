# service-production

Production service: execution of released work orders: operation status, produced quantities, scrap and rejected quantities, and completion.

Part of **CompanyCentral**, a multi-tenant, cloud-native platform built for
the *Cloud Computing Services* course (RSO 2026/27, FRI UL).
See the [architecture](https://github.com/RSOcc26/.github/blob/main/docs/architecture.md) and the
[RSOcc26 organization](https://github.com/RSOcc26) for the other services.

> **Status:** scaffold — no implementation yet. API resources and events below
> are the planned design, not working features.

## Responsibilities

- Initializing execution when a work order is released
- Operation status (pending, ready, running, paused, done)
- Recording produced, scrapped and rejected quantities per operation
- Requesting material issue from Inventory when an operation starts
- Completing work orders when their last operation is done

## Data ownership

This service owns its own PostgreSQL database; no other service reads or
writes it directly ([ADR-0003](https://github.com/RSOcc26/.github/blob/main/docs/adr/0003-database-per-service.md)).

Owns: executions, operation runs, quantity records (good, scrap, rejected).

Multi-tenancy: tenants share tables. Every tenant-owned row has a `tenant_id`,
and PostgreSQL row-level security restricts each transaction to the tenant
from the verified tenant context ([ADR-0004](https://github.com/RSOcc26/.github/blob/main/docs/adr/0004-multi-tenancy-shared-tables-rls.md),
[ADR-0008](https://github.com/RSOcc26/.github/blob/main/docs/adr/0008-tenant-context-validation.md)). All data is tenant-scoped.

## API

The OpenAPI 3.1 contract is [`src/api/openapi.yaml`](src/api/openapi.yaml) and
will be served at `/api-docs`. It currently defines only the operational
endpoints; resources are added contract-first, together with their
implementation.

| Interface | Description |
|---|---|
| Health | `/health/live` and `/health/ready` for Kubernetes probes |
| Metrics | `/metrics` in Prometheus format |
| API docs | `/api-docs`, the OpenAPI document |

Planned resources (through the gateway as `/api/production/v1/...`):

| Resource | Path | Purpose |
|---|---|---|
| Executions | `/v1/executions` | Execution state of released work orders |
| Operations | `/v1/executions/{id}/operations` | Operations of an execution and their status |
| Start / complete | `/v1/executions/{id}/operations/{opId}/start`, `…/complete` | Worker actions |
| Quantities | `/v1/executions/{id}/operations/{opId}/quantities` | Record good, scrap and rejected quantities |

## Events

Published through the transactional outbox to the `companycentral.events`
exchange; consumers are idempotent
([ADR-0006](https://github.com/RSOcc26/.github/blob/main/docs/adr/0006-rabbitmq-transactional-outbox.md)). Planned:

| Direction | Event | Description |
|---|---|---|
| Publishes | `production.operation.completed` | An operation finished, with its quantities |
| Publishes | `production.work-order.completed` | The last operation of a work order finished |
| Consumes | `workflow.work-order.released` | Initialize execution from the graph snapshot |
| Consumes | `workflow.work-order.cancelled` | Stop the execution |
| Consumes | `inventory.reservation.confirmed` | Mark material as ready |
| Consumes | `inventory.reservation.failed` | Mark the execution as blocked |
| Consumes | `inventory.material.issued` | Record the issued material |

Synchronous calls to other services:

| Service | Why |
|---|---|
| `service-inventory` | Issue reserved material when an operation starts |

## Configuration

All configuration comes from environment variables; see
[`.env.example`](.env.example). Secrets are never committed; in Kubernetes
they come from Secrets, everything else from ConfigMaps.

| Variable | Required | Description |
|---|---|---|
| `PORT` | no | HTTP port (default `3000`) |
| `LOG_LEVEL` | no | `debug`, `info`, `warn` or `error` (default `info`) |
| `DATABASE_URL` | yes | PostgreSQL URL of this service's database, runtime role |
| `DATABASE_MIGRATION_URL` | yes | Same database, migration role (used only by migrations) |
| `RABBITMQ_URL` | yes | RabbitMQ connection URL |
| `RABBITMQ_EXCHANGE` | no | Event exchange (default `companycentral.events`) |
| `TENANT_CONTEXT_JWKS_URL` | yes | Where to get the gateway's public keys for verifying the tenant context |
| `TENANT_CONTEXT_ISSUER` | yes | Expected issuer of the tenant context |
| `TENANT_CONTEXT_AUDIENCE` | yes | Expected audience (`service-production`) |
| `INVENTORY_URL` | yes | Issue reserved material when an operation starts |

## Getting started

Prerequisites: Node.js 24 (see `.nvmrc`), Docker, and the local stack from the
[deploy](https://github.com/RSOcc26/deploy) repository (PostgreSQL, RabbitMQ).

_Commands will be added with the first implementation._

## Project structure

```
src/
  api/          HTTP routes, request validation, OpenAPI document
  domain/       business rules, independent of HTTP and storage
  data/         database access (repositories, queries)
  events/       outbox publisher and event consumers
  config/       configuration loading and validation
migrations/     versioned database migrations
test/
  unit/         fast tests with no external dependencies
  integration/  tests against a real database
docs/
  adr/          architecture decision records
```

## Deployment

Built as a container image and published to GitHub Container Registry.
Kubernetes manifests and Helm charts live in the deployment repository.

## Contributing

- Branch from `main` (`feat/…`, `fix/…`, `chore/…`) and open a pull request.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/).
- Pull requests need a review from a code owner and passing checks; they are squash-merged.
- Record notable changes in `CHANGELOG.md`.

## License

[MIT](LICENSE)
