# hono-starter

## Vision

A production-grade Node.js + TypeScript Hono API starter template. It is the
sibling to `fastify-starter` (Node/Fastify reference) and `express-starter`
(Bun/Express reference). It keeps the same modular structure and response
envelope while leaning into idiomatic Hono — Web-standard middleware,
contract-first OpenAPI routes with Zod, and an end-to-end typed RPC client.

## Goals

- Idiomatic Hono v4 starter on Node.js 26 LTS + pnpm via `@hono/node-server` v2.
- Modular structure: `modules/<name>/{schemas,routes,handlers,services,index}`.
- Contract-first routes with `@hono/zod-openapi` (Zod 4): one definition drives
  request validation, inferred types, OpenAPI docs, and RPC types.
- Exported `AppType` for `hc<AppType>` clients; integration tests use
  `testClient` from `hono/testing`.
- Scalar API reference (`@scalar/hono-api-reference`) in non-production only.
- Structured stdout-only logging with `hono-pino` + `pino` (pretty in dev,
  JSON in prod, silent in test); request-scoped logger carries the request ID.
- Full built-in middleware stack: requestId, secureHeaders, cors, compress,
  bodyLimit (1MB), timeout, plus `hono-rate-limiter`.
- Fail-fast env validation with a Zod schema; `.env` loaded via Node's
  `--env-file-if-exists` (no dotenv dependency).
- Biome (lint + format), Vitest (tests), tsdown (build), tsc (typecheck only).
- Hardened Docker (non-root, curl-free healthcheck), GitHub Actions CI (incl.
  Docker build), Dependabot, Husky + lint-staged + commitlint, `.editorconfig`.
- `LICENSE` (MIT, ported) and a `README` matching the sibling templates.
- Preserve the `{ success, data?, error?, message?, count?, details? }` envelope.

## Non-Goals

- No database or ORM — the example module uses an in-memory store.
- No authentication / session system.
- Not multi-runtime (no Bun / Workers entrypoints); Node is the target.
- No monorepo; the RPC client is exported, not consumed by a bundled frontend.
- No `/ready` endpoint (no dependencies, so readiness == liveness).

## Architecture

A `createApp()` factory (`src/app.ts`) builds an `OpenAPIHono` instance,
applies global middleware, sets `onError` / `notFound`, mounts docs (non-prod),
and mounts module routers via chained `.route()` calls so `AppType` captures
every route. `src/index.ts` calls `serve()` and wires graceful shutdown.

### Key Components

- **`src/index.ts`** — `serve({ fetch: app.fetch, port, hostname })`; on
  SIGINT/SIGTERM: `server.close()` (also closes idle keep-alive sockets), then
  flush logger and exit 0. After 8s (below Docker's 10s kill timeout) it forces
  `closeAllConnections()` and exits 1. Fatal errors flush the logger and exit 1.
- **`src/app.ts`** — app assembly; exports `createApp()` and `type AppType`.
- **`src/client.ts`** — `createClient()` (typed `hc<AppType>`) and `AppType`;
  built with bundled declarations to `dist/client.{js,d.ts}` and exported as
  `hono-starter/client` so consumers do not need this repo's `@/*` aliases.
- **`src/lib/create-app.ts`** — `createRouter()`: `new OpenAPIHono<AppBindings>({
  strict: false, defaultHook })` where `defaultHook` converts Zod failures into
  the 400 envelope (`details` = Zod issues). `AppBindings` types `c.var.logger`
  and `c.var.requestId`.
- **`src/lib/logger.ts`** — pino instance (level from env; `pino-pretty`
  transport in development, a dev dependency; `silent` in test). The hono-pino
  middleware binds `reqId` to the request-scoped logger and logs only method,
  path, status, and response time (no headers).
- **`src/lib/openapi.ts`** — `app.doc31("/openapi.json")` (OpenAPI 3.1), Scalar at
  `/reference`, `/docs` → redirect to `/reference`. Registered only when
  `NODE_ENV !== "production"`.
- **`src/middleware/security.ts`** — `securityHeaders()`: secureHeaders, cors
  (`ALLOWED_ORIGINS` allowlist in every environment, credentials enabled);
  `requestGuards()`: compress, timeout (`REQUEST_TIMEOUT_MS` → 503 envelope),
  bodyLimit (1MB → 413 envelope).
- **`src/middleware/rate-limit.ts`** — `hono-rate-limiter`, memory store,
  `RATE_LIMIT_MAX` per `RATE_LIMIT_WINDOW` ms, `draft-7` headers, skips
  `/health` and docs paths, 429 envelope. Key = LAST `X-Forwarded-For` entry
  (appended by a single trusted proxy) when `TRUST_PROXY=true`, otherwise the
  socket address via `getConnInfo`. Known limitation: IPv6 clients are keyed
  per full address (not per /64).
- **`src/config/env.ts`** — Zod schema parsed at import; on failure prints
  `z.prettifyError` output and exits 1. `ALLOWED_ORIGINS` entries must be bare
  origins (no path/trailing slash); `PORT` is 1–65535. Exports typed `env`. Variables: `NODE_ENV`,
  `PORT`, `HOST`, `APP_NAME`, `LOG_LEVEL`, `ALLOWED_ORIGINS`, `RATE_LIMIT_MAX`,
  `RATE_LIMIT_WINDOW`, `TRUST_PROXY`, `REQUEST_TIMEOUT_MS`. Mirrored in
  `.env.example`.
- **`src/modules/example/`** — `schemas.ts` (Zod + `.openapi()` metadata,
  envelope response schemas; ids are digits-only strings transformed to
  safe-integer numbers; updates must contain at least one field), `routes.ts` (`createRoute`
  definitions declaring success, 400/404, body 413, and shared 429/500/503
  responses),
  `handlers.ts` (typed `RouteHandler`s), `services.ts` (in-memory store +
  `resetExamples()`), `index.ts` (chained `createRouter().openapi(...)`).
- **`src/utils/response.ts`** — `ApiResponse<T>` + `ok` / `created` / `fail`
  helpers over `Context` returning `c.json(...)` so RPC types are preserved.
- **`src/constants/index.ts`** — `HTTP_STATUS`, shared messages.

### Middleware Order

```
requestId → pino logger → secureHeaders → cors → rateLimit
  → compress → timeout → bodyLimit → methodNotAllowed → routes
```

Headers/CORS come first so every response (including 429) is readable by
allowed origins; rate limiting runs before any body is read so rejected clients
cost almost nothing; bodyLimit runs inside the timeout so slow uploads are
bounded.

```
```

### Routes

- `GET /` — API info + endpoint list (docs link only when docs are enabled)
- `GET /health` — `{ status: "ok" }` (not in docs, not rate-limited)
- `GET /openapi.json`, `GET /reference`, `GET /docs` (redirect) — non-production only
- `GET /api/example`, `POST /api/example`, `GET|PATCH|DELETE /api/example/:id`

### Error Handling

- Validation → `defaultHook` → 400 `{ success:false, error:"Validation Error", details }`.
- Malformed JSON → 400 `"Bad Request"`; unsupported content type → 415 (both
  raised by Hono's validator as `HTTPException`).
- `HTTPException` → its status, envelope with its message (`error` = standard
  reason phrase from `node:http` `STATUS_CODES`).
- Anything else → 500; message hidden in production; logged via `c.var.logger`.
- Known path, unsupported method → 405 envelope + `Allow` header (`hono/method-not-allowed`).
- `notFound` → 404 envelope (`error: "Route not found"`, parity with fastify-starter).
- Response logs: 5xx at `error`, 4xx at `warn`, `/health` at `trace`.

## Key Decisions

| Decision          | Choice                                                          | Rationale                                                    |
| ----------------- | --------------------------------------------------------------- | ------------------------------------------------------------ |
| Runtime / PM      | Node 26 LTS + pnpm 12                                           | Active LTS (Oct 2026); matches fastify-starter toolchain     |
| Server adapter    | `@hono/node-server` v2                                          | Official, same API as v1, faster                             |
| Validation / docs | `@hono/zod-openapi` + Zod 4                                     | Most mature; single source for validation, types, docs, RPC  |
| Docs UI           | Scalar at `/reference`                                          | Modern, Hono community default; `/docs` redirect for parity  |
| Response shape    | Keep `ApiResponse` envelope                                     | Parity with sibling templates                                |
| Layout            | `modules/<name>/` with `handlers`                               | Parity; "handlers" is the Hono idiom                         |
| RPC               | Export `AppType`, `testClient` in tests                         | Signature Hono feature; typed tests                          |
| Build             | tsdown (bundle → `dist/index.js`, deps external)                | Resolves `@/*` aliases natively; replaces tsc + tsc-alias    |
| Typecheck         | `tsc` with `noEmit` (TS 7)                                      | Native compiler, fast                                        |
| Dev               | `tsx watch --env-file-if-exists=.env`                           | Instant reload; env loaded before any module runs            |
| Env               | Zod schema + Node `--env-file-if-exists`                        | No dotenv dep; fixes fastify's NODE_ENV-before-dotenv gotcha |
| Logging           | `hono-pino` + `pino`, stdout only                               | 12-factor; parity with fastify's Pino                        |
| Rate limit        | `hono-rate-limiter`, memory store                               | Simple default; swap store for Redis when scaling out        |
| Proxy trust       | `TRUST_PROXY` env (default false), last XFF entry               | Prevents rate-limit bypass via spoofed `X-Forwarded-For`     |
| Health            | `/health` only                                                  | No dependencies to check                                     |
| Shutdown          | Manual `server.close`, forced close + exit after 8s             | Node-server has no built-in; below Docker's 10s SIGKILL      |
| Docker            | `node:26-slim` multi-stage, `USER node`, node-fetch HEALTHCHECK | Least privilege; no curl in image                            |
| CI                | lint/typecheck/test/build + docker build & smoke test; v7       | Catch image breakage; least-privilege permissions            |
| CORS              | `ALLOWED_ORIGINS` allowlist in all environments                 | Reflecting any origin with credentials is unsafe by default  |
| Update verb       | `PATCH` (partial body)                                          | Partial updates are PATCH semantics; PUT implies replacement |
| RPC packaging     | `hono-starter/client` export with bundled `.d.ts`               | Consumers need no access to this repo's path aliases         |
| Deps              | Dependabot weekly, grouped (npm, actions, docker)               | Automated, low-noise updates                                 |
| Commits           | commitlint (conventional) via Husky `commit-msg`                | Enforces existing convention                                 |

## Edge Cases & Constraints

- Routes must be chained (`.openapi().openapi()`, `.route().route()`) or
  `AppType` loses type information.
- Handlers must return `c.json(...)` with explicit status codes matching the
  route's declared `responses`, or `@hono/zod-openapi` handler typing fails.
- `strict: false` on routers so trailing slashes match parity with Fastify.
- `timeout` returns 503 to the client but does not cancel in-flight work.
- `hono/compress` uses Web `CompressionStream` (global on Node 18+); in
  production a reverse proxy may handle compression instead.
- Path alias `@/*` is read from tsconfig by tsdown; Vitest needs its own
  `resolve.alias`.
- tsdown keeps `dependencies` external, so the runtime image needs production
  `node_modules` (Pino transports require real files on disk).
- tsdown prints a deprecation notice for its TS 7.0 declaration generator
  (`tsgo`); it disappears once TypeScript 7.1 is stable (beta as of Nov 2026).
- Node 25+ no longer bundles Corepack: Docker installs a pinned Corepack via npm; CI uses
  `pnpm/action-setup`, which reads `packageManager`.
- Env is parsed at import, so tests set `NODE_ENV=test` via Vitest `env`.
- pnpm 12: `pnpm-workspace.yaml` sets `allowBuilds: { esbuild: false }` and
  denies tsdown's bundled agent skills via `permissions`; `prepare` is
  `husky || true`.
- `hono` ≥ 4.13.18 required (security fixes in validator/bodyLimit).
- Docker: `CMD` exec form so Node is PID 1 and receives SIGTERM directly; app
  handles signals itself.

- Husky hooks call `lint-staged` / `commitlint` directly (Husky 9 puts
  `node_modules/.bin` on `PATH`) rather than via `pnpm exec`, so commits work
  even when the global pnpm/Corepack is outdated. Corepack < 0.34.6 cannot
  launch pnpm 12's native binary.

## Backport to fastify-starter

Separate commit pushed to `justincordova/fastify-starter`: Node 26, Docker
hardening (non-root, curl-free HEALTHCHECK), CI modernization (actions v7,
permissions, concurrency, docker job), Dependabot, commitlint, `.editorconfig`,
dependency refresh, SPEC/README updates. Fastify code/architecture unchanged.

## Open Questions

- None.

## References

- Sibling templates: `../fastify-starter`, `../express-starter`
- Structure inspiration: `w3cj/hono-open-api-starter`
- Hono docs (hono.dev), `@hono/zod-openapi`, Scalar, hono-pino, hono-rate-limiter
