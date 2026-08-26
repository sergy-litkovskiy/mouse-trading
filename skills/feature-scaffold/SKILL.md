---
name: feature-scaffold
description: "Creates the vertical slice for a feature in the mouse-springconsult modular monolith — contracts, ports, use-cases with tests, an EntitySchema repository, routes, migration, composition-root wiring, and the Angular feature with zoneless tests. Use when the user asks to generate, scaffold or implement a feature or an endpoint — create the products module, implement the upload endpoint, scaffold the media feature — with or without a prior plan. Writes code; quality gates and the commit belong to the feature-ship skill."
---

# Scaffold a vertical slice

Take the feature name from the user's request. If no plan exists yet, draft a short
one and show it before writing files.

Read the reference implementation before generating each file. The repository has
one slice that already exists end to end, and it is more accurate than any template
in this text:

| Creating | Read first |
|---|---|
| `<entity>.ts` | `modules/auth/user.ts` |
| `<feature>.port.ts` | `modules/auth/auth.port.ts` |
| `<feature>.errors.ts` | `modules/auth/auth.errors.ts` |
| `<action>.use-case.ts` + spec | `modules/auth/login.use-case.ts` + `.spec.ts` |
| `<feature>.fixtures.ts` | `modules/auth/auth.fixtures.ts` |
| `<entity>.entity.ts`, `.repository.ts` | `modules/auth/user.entity.ts`, `user.repository.ts` |
| `<feature>.routes.ts` | `modules/auth/auth.routes.ts` |
| Angular api / store / page + specs | `app/auth/auth-api.ts`, `auth-store.ts`, `login-page.ts` + `.spec.ts` |

Run everything in containers. The two commands specific to scaffolding:

```bash
docker compose run --rm api npm run db:migrate:new -- create-products-table
docker compose run --rm --no-deps web npx ng generate component products/product-catalog
```

## Order of creation

Bottom-up, so every file rests on types that already exist:

1. `contracts/<feature>.contract.ts` (+ `<feature>-limits.ts` for runtime constants)
2. `contracts/error-codes.ts` — add the codes
3. `<entity>.ts` → `<feature>.port.ts` → `<feature>.errors.ts`
4. `<action>.use-case.ts` + `.spec.ts` beside it
5. `<feature>.fixtures.ts` — port test doubles
6. `<entity>.entity.ts`, `<entity>.repository.ts`, `*.adapter.ts`
7. `<feature>.routes.ts` → `index.ts`
8. migration + registration in `db/migrate.ts`
9. `src/api.ts` — composition root
10. `apps/web/src/app/<feature>/` + `app.routes.ts`

## Factories, not classes

Types are erased, so parameter properties (`constructor(private x: X)`) are illegal —
which is why this codebase uses factory functions everywhere. A use-case takes ports
and returns a function; a repository takes a `DataSource` and returns the port.

Inject a `Clock` port. Never call `Date.now()` inside a use-case.

Map the ORM with `EntitySchema`, not decorators: snake_case column onto camelCase field.

Domain errors extend `AppError` and state their own status code. Messages are
English; the frontend composes user-facing text from `code`.

## Backend tests

`node --test`, `*.spec.ts` beside the code. No use-case test starts Postgres — that
is what the ports are for. Put the doubles in `<feature>.fixtures.ts`.

Worth testing: the happy path, every domain error, schema edges
(`*.contract.spec.ts`), and each adapter directly.

In an adapter test, do not pin a date in the past: the token is then expired against
the real clock. Round "now" to whole seconds instead — JWT `iat`/`exp` have
second precision, so an unrounded date fails the round-trip comparison.

## Routes

A Fastify plugin: `safeParse` the payload, throw `AppError` with
`code: validation_failed` and `details.fields` from `z.flattenError`, map to the DTO.

A protected endpoint takes `createSessionGuard` from `modules/auth/index.ts` —
through `index.ts`, never a deep import — and mounts it as a `preHandler`.

## Angular

Standalone, signals, zoneless. Material 22: the prebuilt theme is already wired in
`styles.css`, there is no Sass; buttons use the new API (`matButton="filled"`).

Import `@contracts/*.contract` with `import type` only.

### Zoneless tests

One `whenStable()` is not enough for an asynchronous handler: the microtask queue
has to drain before the DOM reflects the result. Await stability, yield to a
`setTimeout(0)`, run change detection, then await stability again.

Fill forms through the DOM rather than the component instance — the test must see
what the user sees. Use `provideHttpClientTesting` with `HttpTestingController`, and
call `http.verify()` in `afterEach`.

## Wire it up before finishing

Creating the files is half the work. All eight registration points:

1. `modules/<feature>/index.ts` — the module's public API
2. `src/api.ts` → `entities` in `createDataSource`
3. `src/api.ts` → build implementations, assemble use-cases, register routes with a prefix
4. `db/migrate.ts` → import **and** the `migrations` array
5. `contracts/error-codes.ts` → the new codes
6. `apps/web/src/app/app.routes.ts` → route, Ukrainian title, guard
7. `apps/web/eslint.config.js` → `FEATURES`
8. `apps/api/.dependency-cruiser.cjs` → the `ports-and-domain-have-no-io` rule
   hardcodes `user\.ts`; a new domain type is unchecked until it is added there

Then hand over to the feature-ship skill.
