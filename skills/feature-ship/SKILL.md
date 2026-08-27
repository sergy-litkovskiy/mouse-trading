---
name: feature-ship
description: "Takes a written feature in the mouse-springconsult repository to green and commits it — ten quality gates run in containers in cost order, down/up migration verification, smoke requests against the live stack, documentation updates, and a Conventional Commits commit. Use when the user asks to run the checks, lint, typecheck or tests, to verify a migration, or says the feature is done, ready to commit or ready to ship. Does not write feature code."
---

# Take a feature to green

Run the steps in order; each one is cheaper than the next and filters what would
otherwise be hunted in a longer cycle.

## 1. Format before lint

Prettier rearranges what ESLint would otherwise complain about, so part of the lint
output is phantom until formatting has run.

```bash
docker compose run --rm --no-deps api npm run format
docker compose run --rm --no-deps web npm run format
```

## 2. Ten gates

```bash
docker compose run --rm --no-deps api npm run format:check
docker compose run --rm --no-deps api npm run lint
docker compose run --rm --no-deps api npm run typecheck
docker compose run --rm --no-deps api npm run deps:check
docker compose run --rm --no-deps api npm run build
docker compose run --rm api npm run test
docker compose run --rm --no-deps web npm run format:check
docker compose run --rm --no-deps web npm run lint
docker compose run --rm --no-deps web npm run test
docker compose run --rm --no-deps web npm run build
```

Skip none. `deps:check` is the architecture in executable form, and `web build`
catches what `lint` cannot — an unresolvable `@contracts` path, or a bundle that
grew. The web side has no separate typecheck gate on purpose: `build` covers it.

`api npm run test` is the one command **without** `--no-deps`: the repository and
schema specs run against a real Postgres. It compiles first, so a failure there is a
`tsc` failure, not a test failure — read the top of the output.

### Symptom to cause

| Symptom | Cause |
|---|---|
| `deps:check`: `no-deep-import-between-modules` | an import bypassing a neighbouring module's `index.ts` |
| `deps:check`: `orm-stays-in-repositories-and-entities` | `typeorm`/`pg` in a service or controller (`import type` counts), or a new entity class missing from the `ENTITIES` list |
| `deps:check`: `controller-goes-through-the-service` | a controller reaching for a repository instead of the service |
| `node: cannot find module dist/...` | `dist/` is stale or absent — run `npm run build` (or `docker compose run --rm migrate`, which builds itself) |
| a stack trace pointing at `.js` | the process was started without `--enable-source-maps` |
| `tsc` TS5097 in web | a `.ts` extension in a frontend import specifier |
| bundle suddenly ~55 KB gzip larger | a runtime import from a `*.contract.ts` dragged zod in |
| an Angular test sees stale DOM | the microtask queue was not drained — see the feature-scaffold skill |

## 3. The migration both ways

`down` is verified by running it, not by reading it. `db:migrate:revert` undoes
**one** migration per call, so run it once per migration the feature added — a
table plus a seed needs two — and check `db:migrate:show` in between:

```bash
docker compose run --rm api npm run db:migrate:show
docker compose run --rm api npm run db:migrate:revert
docker compose run --rm api npm run db:migrate
```

The `db:*` scripts run `dist/db/*.js`, so they need a build behind them. Gate 5 above
provides it; from a clean clone, `docker compose run --rm migrate` builds, creates the
database and migrates in one go.

If a revert fails, or leaves an index or table behind, the migration is not ready.

## 4. Smoke against the live stack

```bash
docker compose up -d --build && docker compose ps    # api must be healthy
```

Then issue real requests. The host port comes from `.env` (`API_HOST_PORT`,
3000 by default). Mind the prefix asymmetry: straight to the api container the path
carries no prefix (`/auth/login`), while through Caddy or `ng serve` it is
`/api/auth/login` — `handle_path` strips the prefix. The session lives in an
httpOnly cookie, so curl needs a jar:

```bash
COOKIES=$(mktemp)
curl -s -c "$COOKIES" -X POST localhost:${API_HOST_PORT:-3000}/auth/login \
  -H 'content-type: application/json' -d '{"email":"...","password":"..."}'
curl -s -b "$COOKIES" localhost:${API_HOST_PORT:-3000}/<feature>/...
```

At minimum: the happy path, 401 without a session, 400 on an invalid payload, and
200 from the frontend page — which also exercises the `/api` proxy.

If the feature touched the schema or compose, verify from scratch:
`docker compose down -v && docker compose up -d --build`.

## 5. Documentation

- `ARCHITECTURE.md` — the module's state in the "Модулі" table;
- `CLAUDE.md` — the "Стан" block, if a new skeleton file appeared (`queue.ts`, `worker.ts`);
- `apps/api/.dependency-cruiser.cjs` — the `ENTITIES` list, if the feature added an entity;
- `README.md` and `.env.example` — if a command or an environment variable was added;
- `docs/adr/` — a new ADR **only** when the decision contradicts an existing one or
  closes a fork for a long time. Do not write an ADR as ritual.

## 6. Commit

Before `git add`, look at what is being staged. `.env`,
`environment.development.ts` and `node_modules` never enter a commit:

```bash
git status --short
git diff --cached --name-only | grep -Ei '(^|/)\.env$|node_modules|environment\.development\.ts' || echo "clean"
```

Conventional Commits in English: `feat(products): create and list product cards`.
Run `git push` **only** when the user asks: a push to `main` triggers deployment.

## 7. Retrospective, when asked

`LOG.md` holds feature briefs, not a fixed retrospective format. When the user asks
for one, or wants to compare this feature against the previous, report honestly:

- which gates failed on the first run and why;
- where these skills did not help and the code had to be written by hand;
- what is worth folding into the skills next time.

Do not smooth it over: a step left uncovered is the input for the next iteration.
