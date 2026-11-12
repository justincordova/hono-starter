# Hono Starter

A production-grade Node.js + TypeScript [Hono](https://hono.dev) API starter.
Contract-first OpenAPI routes with Zod, an end-to-end typed RPC client,
structured logging, a full security middleware stack, and graceful shutdown.

## Stack

- **Runtime:** Node.js 26 LTS (ESM) via `@hono/node-server`
- **Framework:** Hono v4
- **Validation + docs:** `@hono/zod-openapi` (Zod 4): one route definition drives validation, types, OpenAPI, and RPC
- **API reference:** Scalar (non-production only)
- **Logging:** Pino via `hono-pino` (pretty in development, JSON in production)
- **Middleware:** request ID, secure headers, CORS, compression, body limit, timeout, rate limiting
- **Env:** Zod schema validated at boot, `.env` loaded natively by Node
- **Build:** tsdown (bundled ESM), `tsc` for type checking (TypeScript 7)
- **Lint/format:** Biome
- **Tests:** Vitest (service unit tests + typed `testClient` integration tests)
- **Tooling:** Husky + lint-staged + commitlint, GitHub Actions CI, Dependabot

## Prerequisites

- Node.js 26 or newer
- [pnpm](https://pnpm.io) 12 or newer. Corepack is no longer bundled with Node
  25+, so either install pnpm directly or run `npm i -g corepack && corepack enable`.
  Corepack must be 0.34.6 or newer; older versions cannot launch pnpm 12's native binary.

## Setup

```bash
pnpm install
cp .env.example .env
pnpm dev
```

The server starts on `http://localhost:3000`. API reference: `http://localhost:3000/reference`.

## Scripts

| Script            | Description                                     |
| ----------------- | ----------------------------------------------- |
| `pnpm dev`        | Start in watch mode (tsx), loading `.env`       |
| `pnpm build`      | Bundle to `dist/` (tsdown)                      |
| `pnpm start`      | Run the built server, loading `.env` if present |
| `pnpm test`       | Run the test suite once                         |
| `pnpm test:watch` | Run tests in watch mode                         |
| `pnpm coverage`   | Run tests and generate a coverage report        |
| `pnpm lint`       | Lint and check formatting with Biome            |
| `pnpm format`     | Format with Biome                               |
| `pnpm typecheck`  | Type-check without emitting                     |

## Project Structure

```
src/
├── index.ts              # entry point: serve() + graceful shutdown
├── client.ts             # typed RPC client export (createClient, AppType)
├── app.ts                # createApp(): middleware, error handlers, routes; exports AppType
├── config/env.ts         # Zod env schema, validated at boot
├── constants/            # HTTP status codes, limits, messages
├── lib/
│   ├── create-app.ts     # createRouter() with validation-error envelope + AppBindings
│   ├── logger.ts         # root Pino logger
│   └── openapi.ts        # /openapi.json + Scalar reference
├── middleware/           # security stack, rate limiter
├── modules/example/      # schemas, routes, handlers, services, router
├── utils/response.ts     # standard response envelope helpers
└── tests/                # unit + integration tests
```

### Adding a module

1. Create `src/modules/<name>/` with `schemas.ts`, `routes.ts` (`createRoute`),
   `handlers.ts` (`RouteHandler<typeof route, AppBindings>`), `services.ts`, and
   `index.ts` (chained `createRouter().openapi(...)`).
2. Mount it in `src/app.ts` with `.route("/api/<name>", router)`. Keep the calls
   chained so `AppType` picks up the new routes.

## Endpoints

| Method | Path               | Description                                                        |
| ------ | ------------------ | ------------------------------------------------------------------ |
| GET    | `/`                | API info                                                           |
| GET    | `/health`          | Health check                                                       |
| GET    | `/reference`       | Scalar API reference (non-production only; `/docs` redirects here) |
| GET    | `/openapi.json`    | OpenAPI 3.1 document (non-production only)                         |
| GET    | `/api/example`     | List examples                                                      |
| POST   | `/api/example`     | Create an example                                                  |
| GET    | `/api/example/:id` | Get an example                                                     |
| PATCH  | `/api/example/:id` | Update an example                                                  |
| DELETE | `/api/example/:id` | Delete an example                                                  |

All API responses use a standard envelope: `{ success, data?, error?, message?, count?, details? }`.

## Typed RPC client

`pnpm build` also emits `dist/client.js` + `dist/client.d.ts`, exported as
`hono-starter/client`. The declarations are self-contained (no `@/*` aliases),
so a frontend in the same workspace (or one that installs this package) gets
end-to-end types for every route, including error responses. The consumer needs
`hono` installed (the client is a thin wrapper over `hono/client`):

```ts
import { createClient } from "hono-starter/client";

const client = createClient("http://localhost:3000");
const res = await client.api.example.$post({ json: { name: "Alice", email: "alice@example.com" } });
if (res.ok) {
  const { data } = await res.json(); // typed as Example
}
```

## Environment

| Variable             | Default                                       | Description                                                          |
| -------------------- | --------------------------------------------- | -------------------------------------------------------------------- |
| `NODE_ENV`           | `development`                                 | `development`, `production`, or `test`                               |
| `PORT`               | `3000`                                        | Port to listen on                                                    |
| `HOST`               | `0.0.0.0`                                     | Interface to bind                                                    |
| `APP_NAME`           | `hono-starter`                                | Shown in `/` and the API reference                                   |
| `LOG_LEVEL`          | `info`                                        | Pino log level                                                       |
| `ALLOWED_ORIGINS`    | `http://localhost:3000,http://localhost:5173` | CORS allowlist (all environments)                                    |
| `RATE_LIMIT_MAX`     | `100`                                         | Requests per window per client                                       |
| `RATE_LIMIT_WINDOW`  | `60000`                                       | Rate limit window (ms)                                               |
| `TRUST_PROXY`        | `false`                                       | Use `X-Forwarded-For` for client IPs (only behind one trusted proxy) |
| `REQUEST_TIMEOUT_MS` | `30000`                                       | Request timeout; responds 503 when exceeded                          |

Invalid values stop the process at startup with a readable error.

## Docker

```bash
docker compose up --build
```

The image is a multi-stage `node:26-slim` build with production dependencies only.
It runs as the unprivileged `node` user, has a built-in `HEALTHCHECK` on
`/health`, and runs Node as PID 1 so `SIGTERM` triggers a graceful shutdown.

## Notes

- **Rate limiting** uses an in-memory store, so each replica keeps its own
  counters. Use a shared store (e.g. Redis) when running more than one instance.
- **Timeouts** respond with 503 but do not cancel the handler, which keeps running.
- **Client IPs** for rate limiting come from the socket, or from the last
  `X-Forwarded-For` entry when `TRUST_PROXY=true` (exactly one trusted proxy).
  IPv6 clients are keyed per full address.
- **`pino-pretty`** is a dev dependency used when `NODE_ENV=development`. Run
  production installs (`pnpm install --prod`) with `NODE_ENV=production`.
- **Commits** must follow [Conventional Commits](https://www.conventionalcommits.org)
  (enforced by commitlint).

## License

MIT
