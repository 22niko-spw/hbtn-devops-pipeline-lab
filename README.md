# hbtn-devops-pipeline-lab

[![CI](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/workflows/ci.yml?query=branch%3Amain)

This repository contains the application used in the **CI/CD Pipeline Essentials** lab. It is a small Express API backed by PostgreSQL. The application and tests are already implemented; your work is to diagnose and extend its delivery pipeline.

The original workflow faults were repaired in three separate `fix(ci)` commits. The pipeline now lints and tests the app, retains JUnit reports for 7 days, and publishes a production image on pushes to `main`. See [DEPLOY.md](DEPLOY.md) for staging setup and [PIPELINE_EVIDENCE.md](PIPELINE_EVIDENCE.md) for observed results and remaining work.

Repository: [22niko-spw/hbtn-devops-pipeline-lab](https://github.com/22niko-spw/hbtn-devops-pipeline-lab), default branch `main`. The application, Dockerfile, lock file, and original tests are preserved from the supplied starter.

## Repository contents

```text
src/
  server.js                       Express app, routes, and error handling
  routes/health.js                GET /health
  routes/items.js                 GET /items, POST /items, and input validation
  db/connection.js                lazy PostgreSQL pool built from DATABASE_URL
  db/migrate.js                   idempotent migration runner
  db/migrations/*.sql             schema and seed data
tests/
  unit/health.test.js             3 unit tests
  unit/items.unit.test.js         5 unit tests
  integration/items.int.test.js  3 integration tests that require PostgreSQL
Dockerfile                        builder and production runtime stages
docker-compose.yml                local app and database services
jest.config.js                    Jest configuration; JUnit is installed but disabled
.eslintrc.json                    lint rules used by npm run lint
.github/workflows/ci.yml          test → build → staging pipeline
```

The test suite contains 11 deterministic tests: 8 unit tests and 3 integration tests. Unit tests need only Node.js. Integration tests need a reachable PostgreSQL database.

## API

| Method | Path | Behavior |
|---|---|---|
| `GET` | `/health` | Returns `200` with `{"status":"ok"}`. This is a liveness check and does not query the database. |
| `GET` | `/items` | Returns `200` with stored items ordered by creation. This route requires the database. |
| `POST` | `/items` | Creates an item and returns `201`, or returns `400` when `name` is invalid. |

The application listens on port `3000`. Database operations read the connection string from `DATABASE_URL`; there is no built-in fallback.

## Run the application in containers

You do not need Node.js or npm on the host. Run the supplied commands inside the containers:

```bash
docker compose up -d
docker compose ps

docker compose run --rm app npm run test:unit
docker compose run --rm app npm run test:integration
docker compose run --rm app npm test
docker compose run --rm app npm run lint

curl -s http://localhost:3000/health
curl -s http://localhost:3000/items
curl -s -X POST http://localhost:3000/items \
  -H 'Content-Type: application/json' \
  -d '{"name":"Delta Item"}'

docker compose down -v
```

The Compose `app` service targets the `builder` stage, which includes Jest, Supertest, and ESLint. The pipeline builds the smaller `runtime` stage for deployment.

## Pipeline

- Pull requests targeting `main`: npm download cache, `npm ci`, lint, all 11 tests,
  and JUnit artifact upload even after a test failure.
- Pushes to `main`: the same tests, then `build` with `needs: test`, then
  `deploy` with `needs: build`. Failed upstream jobs block release jobs.
- Cache: `~/.npm`, keyed by runner OS, architecture, Node version and lock-file
  hash, with a compatible restore prefix. `npm ci` always runs.
- Images: `ghcr.io/22niko-spw/hbtn-devops-pipeline-lab:<full-commit-SHA>` and
  `:latest`; Buildx uses the container driver and GitHub Actions layer caching.
- Only `build` receives `packages: write`; repository contents remain read-only.
  Render credentials are available only to `deploy` through GitHub Secrets.
- Staging targets Render using the exact SHA image. The job waits for that
  deployment, then requires HTTP 200 from both `/health` and `/items`.

A green run establishes that its configured checks passed. It does not prove
absence of bugs, vulnerabilities, or failures outside the tested scenarios.
A commit tag provides traceability as long as it is not overwritten; a digest
identifies the exact image content. `latest` is a moving pointer.

## Local baseline

[local_verify.txt](local_verify.txt) records 8 passing unit tests, 3 passing
integration tests, HTTP 200 from `/health`, and removal of the lab containers
and volumes. The verification used host port 13000 through a temporary Compose
override because port 3000 was already occupied. npm ran only in containers.

## Workflow audit

Run `actionlint .github/workflows/ci.yml`. The CI database uses disposable
`pipeline` test values from the starter; these are not staging credentials.
Third-party secrets must remain in the platform secret stores. A credential
pattern scan helps detect mistakes but is not proof that no secret was exposed.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `DATABASE_URL is not set` | Run through Docker Compose locally, or provide the connection string in the environment that executes integration tests or the deployed API. |
| Integration tests cannot connect | Confirm that the database is healthy with `docker compose ps`. |
| Changes to `package.json` are ignored | Recreate the development volume with `docker compose down -v`, then rebuild. |
| `npm ci` reports a lock-file mismatch | Regenerate dependencies inside the container, review the lock-file change, and commit both manifests. |
| Port `3000` is already in use | Stop the process using the port or change the published host port in `docker-compose.yml`. |

## License

This lab is distributed under the MIT License. See `LICENSE`.
