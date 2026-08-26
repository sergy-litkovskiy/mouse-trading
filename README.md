# mouse-trading

A Claude Code plugin for the **mouse-springconsult** admin panel — three skills that
carry a feature from plan to commit through the architecture that codebase enforces.

The plugin lives in its own repository but is not generic: every skill references
that project's layout (`apps/api/src/modules/`, `apps/web/src/app/`, its nine quality
gates, its dependency-cruiser rules). Installing it elsewhere would load three skills
that describe files which do not exist there.

```
.claude-plugin/
  marketplace.json     single-plugin marketplace; the plugin is the repository root
  plugin.json
skills/
  feature-plan/        file inventory by layer suffix + eight registration points
  feature-scaffold/    the slice itself, pointed at the reference implementation
  feature-ship/        nine gates, migration down/up, smoke, docs, commit
```

## Installation

From the target project (`mouse-springconsult`), pointing at a local clone:

```bash
claude plugin marketplace add /path/to/mouse-trading
claude plugin install mouse-trading@mouse-trading --scope project
```

Or straight from GitHub, once this repository is pushed:

```bash
claude plugin marketplace add sergy-litkovskiy/mouse-trading
claude plugin install mouse-trading@mouse-trading --scope project
```

Restart the session afterwards. `--scope project` writes `enabledPlugins` into the
target project's `.claude/settings.json`, so the plugin switches on by itself for
anyone who already has the marketplace registered.

The `marketplace add` step cannot be skipped. Claude Code's settings schema documents
`extraKnownMarketplaces` for exactly this, but declaring it in a project's
`.claude/settings.json` was measured not to work: with the marketplace removed from
the user-level registry and declared only in project settings, a fresh session
reported no skills from this plugin at all. A project file registering a plugin source
on its own would let any cloned repository inject components, so treat the manual step
as deliberate.

## Invocation

Plugin skills are namespaced, so the canonical form is
`/mouse-trading:feature-plan`, `/mouse-trading:feature-scaffold`,
`/mouse-trading:feature-ship`. The bare `/feature-plan` works only while no other
skill or command claims that name — and `feature-plan` is generic enough to collide.

Claude also invokes them on its own when a request matches their description, so the
three descriptions carry explicit trigger phrases and an explicit statement of what
each skill does *not* do.

## Releasing a change

Installing from a `directory` or git source **copies** the plugin into a version-keyed
cache (`~/.claude/plugins/cache/mouse-trading/mouse-trading/<version>/`). Editing a
file here therefore changes nothing in a running session, and `claude plugin update`
reports "already at the latest version" as long as the version string is unchanged.

The loop that works:

```bash
# bump "version" in .claude-plugin/plugin.json AND in .claude-plugin/marketplace.json
claude plugin marketplace update mouse-trading
claude plugin uninstall mouse-trading@mouse-trading --scope project
claude plugin install  mouse-trading@mouse-trading --scope project
```

Keep the two version strings equal: at install time `plugin.json` wins and the
marketplace entry is ignored, so a mismatch is silent drift. `claude plugin validate .
--strict` checks both manifests, but not skill frontmatter — it accepts invented
frontmatter keys, so a malformed `description` will not be reported.

## What the skills add beyond the project's CLAUDE.md

`CLAUDE.md` is in context in every session of the target project, so the skills do not
restate it. They hold what it does not:

- **eight registration points** — the places a new file has to be wired into, and
  which are easy to forget: `FEATURES` in `apps/web/eslint.config.js`, the
  `migrations` array in `db/migrate.ts`, `entities` in `createDataSource`, the route
  in `app.routes.ts`, and the `ports-and-domain-have-no-io` rule in
  `.dependency-cruiser.cjs`, which hardcodes a single domain-type file name;
- **an exemplar map** — which existing file to read before creating each new one;
- **order of creation**, bottom-up, so each file rests on types that already exist;
- **a symptom-to-cause table** for the nine gates;
- **two failure modes that are invisible until they bite**: a single `whenStable()`
  is not enough in a zoneless Angular test, and `db:migrate:revert` undoes exactly one
  migration per call.

## Limits

The skills know the shape of the code, not the domain. They do not decide what fields
a product card has, and they do not replace a conversation where the choice is
genuinely open.

Architectural boundaries are enforced by the target project's gates, not by the
editor: `deps:check` (dependency-cruiser), `lint` (ESLint), `typecheck` (`tsc` with
`erasableSyntaxOnly`). Three things no gate catches, measured: a runtime import from a
`*.contract.ts` (drags zod into the browser bundle — both `web lint` and `web build`
pass), file names such as `.service.ts`, and creating a `shared/` directory as long as
nothing imports from it.
