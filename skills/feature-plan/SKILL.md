---
name: feature-plan
description: "Plans an end-to-end vertical slice for the mouse-springconsult modular monolith — file inventory by layer, zod contracts, error codes, migrations, and the eight registration points that are easy to create a file for and forget to wire. Use when the user starts a new feature or a new endpoint: let's build products, add an endpoint to auth, what files does the media feature need, plan the AI module. Produces a plan in chat only; writing files belongs to the feature-scaffold skill."
allowed-tools: Read, Grep, Glob
---

# Plan a vertical slice

Take the feature name and its purpose from the user's request. If neither is stated,
ask what is being built — do not invent it.

Produce a plan in chat. Do not write files.

`CLAUDE.md` is loaded automatically, so do not restate it. Read what is not:
`ARCHITECTURE.md` (sections "Модулі" and "Dependency rule"), `SPEC.md`, and
`docs/adr/` — in particular ADR 0003, which set the three-layer shape. Read
`apps/api/src/modules/auth/` and `apps/api/src/modules/products/` — the two slices
that already exist end to end, and a more accurate reference than any template.

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

Three layers, marked by the suffix of the class name. One class per file, and the
file is named after it. Name the actual files:

| File | Role | Constraint |
|---|---|---|
| `<Entity>.ts` (singular: `User.ts`, `Product.ts`) | model and ORM mapping in one class | `@Entity`/`@Column`, column type always explicit |
| `<Feature>Errors.ts` | domain errors extending `AppError` | status code here, HTTP mapping in `api.ts` |
| `<Entity>Repository.ts` | persistence | the only place with `typeorm`/`pg`, apart from the entities |
| `<Feature>Service.ts` | business logic | holds repositories; knows nothing about HTTP |
| `<Feature>Controller.ts` | validation, guard, DTO mapping | no business logic, and never a repository |
| `<Name>.ts` for infrastructure (`PasswordHasher`, `SessionTokens`, `ImageStorage`) | one collaborator per class | lives in the module that calls it |
| `index.ts` | module public API | the only entry point for other modules |

A service per feature, not per action: `AuthService` has `login`, `authenticate` and
`logout`. Split it when the file gets uncomfortable, not before.

Frontend: `app/<feature>/` — `<feature>-api.ts` (HTTP), `<feature>-store.ts`
(signals), and the page as `<name>.ts` + `.html` + `.css` side by side. The frontend
keeps kebab-case: it follows the Angular style guide, the backend follows the class.

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
2. `src/api.ts` → `createDataSource({ entities: [...] })` — add the entity class.
3. `src/api.ts` → `new <Entity>Repository(dataSource)` → `new <Feature>Service(...)` →
   `new <Feature>Controller(...)`, then
   `app.register(async (instance) => { controller.register(instance, authController.sessionGuard); }, { prefix: '/<feature>' })`.
4. `db/migrations-list.ts` — import the migration class **and** add it to the
   `migrations` array. `npm run db:migrate:new` does both; check that it did.
5. `contracts/error-codes.ts` — the new codes.
6. `apps/web/src/app/app.routes.ts` — route, Ukrainian `title`, `canActivate`.
7. `apps/web/eslint.config.js` → `FEATURES` — without it the new feature's
   boundaries are not checked at all.
8. `apps/api/.dependency-cruiser.cjs` → the `ENTITIES` constant lists entity classes
   by name, because they carry no suffix. A new entity that imports `typeorm` fails
   `orm-stays-in-repositories-and-entities` until it is added there — the failure is
   loud, and adding the line is the decision about where the ORM may appear.

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
