# Keryvio

On-premise, multi-user AI platform. The product requirements and architecture
are documented in [`docs/specification.md`](docs/specification.md).

## Current foundation

- Rust workspace with an Axum API health endpoint.
- Angular web application scaffold in `apps/web`.
- PostgreSQL and reverse-proxy deployment foundations in `deploy/`.
- Local toolchain/setup notes in [`docs/development.md`](docs/development.md).

This repository is at implementation step 1 (foundation). Authentication,
inference and OpenAI-compatible endpoints are not implemented yet.

## Run locally

Start PostgreSQL with `docker compose -f deploy/compose.yaml up -d postgres`.
Then start the API and web app in separate terminals; see
[`docs/development.md`](docs/development.md) for exact commands and health checks.
