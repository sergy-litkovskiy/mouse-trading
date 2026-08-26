---
name: feature-plan
description: "Plans an end-to-end vertical slice for the mouse-springconsult modular monolith — file inventory by layer suffix, zod contracts, error codes, migrations, and the eight registration points that are easy to create a file for and forget to wire. Use when the user starts a new feature or a new endpoint: let's build products, add an endpoint to auth, what files does the media feature need, plan the AI module. Produces a plan in chat only; writing files belongs to the feature-scaffold skill."
allowed-tools: Read, Grep, Glob
---

# Plan a vertical slice

Take the feature name and its purpose from the user's request. If neither is stated,
ask what is being built — do not invent it.

Produce a plan in chat. Do not write files.

`CLAUDE.md` is loaded automatically, so do not restate it. Read what is not:
`ARCHITECTURE.md` (sections "Модулі" and "Dependency rule"), `SPEC.md`, and
`docs/adr/`. Read `apps/api/src/modules/auth/` and `apps/web/src/app/auth/` — the
one slice that already exists end to end, and a more accurate reference than any
template.

## 1. Module boundary

Decide first whether this is a new module or a new file in an existing one. The
`modules/{auth,products,media,ai}` skeleton is fixed in git; do not add top-level
directories. If it is an endpoint in an existing module, keep only steps 2, 5 and 6.

## 2. Contracts and error codes

Name concretely:

- the zod schemas in `contracts/<feature>.contract.ts` and the types derived from
  them — `z.input` is what the client sends, `z.output` what the backend sees after
  validation;
- whether `<feature>-limits.ts` is needed: constants the frontend imports **at
  runtime** live there, never in the schema file;
- the new codes added to `contracts/error-codes.ts` and the domain error each
  one belongs to.

## 3. File inventory

Layers are suffixes, not directories. Name the actual files:

| File | Role | Constraint |
|---|---|---|
| `<entity>.ts` (singular: `user.ts`, `product.ts`) | domain type | no I/O |
| `<feature>.port.ts` | dependency interfaces | no I/O |
| `<feature>.errors.ts` | domain errors extending `AppError` | status code here, HTTP mapping in `api.ts` |
| `<action>.use-case.ts` | one action per file | depends on ports, never on implementations |
| `<entity>.entity.ts` | ORM mapping via `EntitySchema` | decorators are not erasable |
| `<entity>.repository.ts`, `<name>.adapter.ts` | port implementations | instantiated only by the composition root |
| `<feature>.routes.ts` | validation, DTO mapping | no business logic |
| `<feature>.fixtures.ts` | port test doubles | imported only from `*.spec.ts` |
| `index.ts` | module public API | the only entry point for other modules |

Frontend: `app/<feature>/` — `<feature>-api.ts` (HTTP), `<feature>-store.ts`
(signals), and the page as `<name>.ts` + `.html` + `.css` side by side.

## 4. Migrations

Name the table, its indexes, constraints and any seed. Write a real `down` — the
feature-ship skill executes it rather than reading it.

## 5. Asynchronous work

If the feature has anything slow — sharp, a Claude call, an outbound HTTP request —
it is a queued job, not the HTTP process's work. **Stop and ask the user before
planning it**: `src/queue.ts` and `src/worker.ts` do not exist yet, adding them
creates a second composition root, and it likely needs its own ADR.

A plain CRUD feature skips this step.

## 6. Registration points

The expensive half of the plan: creating a file is easy, wiring it is easy to
forget. List every point the feature touches.

1. `modules/<feature>/index.ts` — export everything other modules may see.
2. `src/api.ts` → `createDataSource({ entities: [...] })` — add the entity.
3. `src/api.ts` → build repository and adapters, assemble use-cases,
   `app.register(create<Feature>Routes({...}), { prefix: '/<feature>' })`.
4. `db/migrate.ts` — import the migration class **and** add it to the `migrations` array.
5. `contracts/error-codes.ts` — the new codes.
6. `apps/web/src/app/app.routes.ts` — route, Ukrainian `title`, `canActivate`.
7. `apps/web/eslint.config.js` → `FEATURES` — without it the new feature's
   boundaries are not checked at all.
8. `apps/api/.dependency-cruiser.cjs` → the `ports-and-domain-have-no-io` rule
   hardcodes `user\.ts` as the domain-type file name. A new domain type is not
   matched, so `deps:check` stays green while `product.ts` imports `typeorm`.
   Either add the file name to that regex or generalise it once and say so.

## 7. Conflict with an ADR

Check the plan against `docs/adr/`. When the task needs a decision that contradicts
one already taken, do not pick a side silently: name the conflict, propose an
option, and flag that a new ADR is required.

## What to hand back

Module boundary → contracts and error codes → file list → migrations →
registration points → ADR conflicts and open questions.

Ask only what changes the work: entity name, field set, whether the endpoint needs
authentication, whether there is asynchronous work. Decide the rest from the
conventions.
