# Development setup

## Prerequisites

- Rust stable (minimal profile) and Cargo.
- Node.js 20.19+ or 22.12+ and npm 10+.
- Docker Compose v2 for PostgreSQL.

Cargo's toolchain and registry cache can grow. Set `CARGO_HOME` and
`RUSTUP_HOME` outside this checkout if desired, and remove the workspace
`target/` directory after checks when disk space is limited.

## Install

From the repository root:

```sh
cargo fetch --locked
cd apps/web && npm ci
```

## Start services

```sh
test -f .env || cp .env.example .env
docker compose -f deploy/compose.yaml up -d postgres
```

In one terminal, from the repository root:

```sh
KERYVIO_BIND_ADDR=127.0.0.1:8080 cargo run -p keryvio-server
```

In another terminal:

```sh
cd apps/web && npm start
```

The API should return `ok` at `http://127.0.0.1:8080/healthz` and the Angular
development server is available at `http://127.0.0.1:4200/`.

## Validation

```sh
cargo check --workspace --locked
cd apps/web && npm run build
```
