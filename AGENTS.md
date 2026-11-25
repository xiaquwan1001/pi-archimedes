# AGENTS.md

Rules for AI agents working on this monorepo. **Only rules every change runs into live here.**
The long-form procedures — adding a package, publishing, CI, dependency pins — live in
`CONTRIBUTING.md`; open the one section you need instead of reciting the flow from memory.

## Monorepo Structure

- `packages/core` — event-channel vocabulary, bridge, chrome, text/color utils, overlay, profiler, settings-io, tool-render
- `packages/ui` — TUI enhancements: bash tool styling, custom editor, border spinner, thinking collapse, splash animation (depends on core)
- `packages/ask` — structured question tool with tabbed flow and inline notes (depends on core)
- `packages/footer` — status bar (depends on core)
- `packages/diff` — Shiki-powered diff rendering (standalone)
- `packages/image-paste` — clipboard image paste (depends on core)
- `packages/subagent` — subagent dispatch with live TUI streaming and cost tracking (depends on core, todo)
- `packages/todo` — todo list tool with auto-clear and subagent visibility (depends on core)
- `packages/notify` — delayed desktop notifications with circuit breaker (depends on core)
- `packages/session-name` — auto session naming (depends on core)
- `packages/web` — web search and content fetching with multi-provider routing (depends on core)
- `packages/sudo` — safe privileged execution (sudo_exec tool, masked password prompt, interactive-sudo bash guard) (depends on core)
- `meta` — orchestrator + composed settings (depends on all twelve)

## Conventions

### Package Manager
- This is a **pnpm workspace** — always use `pnpm install`, never `npm install`
- `packageManager` field in root `package.json` pins pnpm via Corepack
- `workspace:*` protocol in dependencies is pnpm-specific; npm cannot parse it
- `.npmrc` sets `package-manager-strict=true` and `manage-package-manager-versions=true`

### Imports
- **Within a package:** always relative paths (e.g., `from "../chrome.js"`)
- **Cross-package:** use package subpath exports (e.g., `from "@pi-archimedes/core/events"`)
- **Never** import `../utils/...` — those barrels were dissolved into `text.ts` and `color.ts`
- **Never take schema symbols (`Type`, `Static`, `TSchema`) from `@earendil-works/pi-ai`'s re-exports — import them from `typebox` at their source.** pi 1.5.0 deleted pi-ai's typebox re-exports in a patch-level train release; the packages that imported `typebox` directly were unaffected and the one that imported through the host broke (`packages/todo`). A host re-export is not a stable surface (ADR 0026).

### Config
- Each package reads its own namespace in `~/.pi/agent/settings.json`
- Core: `archimedes.core`, Footer: `archimedes.footer`, Diff: `archimedes.diff`, Session-name: `archimedes.sessionName`
- Plugin on/off: a non-core package's namespace may carry a suite-managed `enabled` boolean (default on when absent). **Only meta reads/writes it**, via `/plugins` (`isPluginEnabled` / `setPluginEnabled` in `meta/src/plugins.ts`, ADR 0012). Package code must never read it as a runtime guard, and it never appears in a package's own settings items — except the first-run keybinding offer (image-paste gate 1, plan-approved 2026-09-11, see ADR 0012 Exception), which runs from the meta `session_start` handler before plugin registration and must read the flag directly to honour a plugin-off toggle
- No migration from old `hephaestus` keys

### No Build Step
- All packages use `"type": "module"` with `.ts` entry points
- Pi's jiti loader handles TypeScript at runtime
- Verification is `tsc --noEmit`, not a build or test command

### Two Never-Do Rules (suite-wide, mechanically guarded)
1. **Never decide a colour by parsing an escape string — ask the theme for a token.** A theme token may be `""` (terminal default, rendered `\x1b[49m` — truthy, so `?? FALLBACK` never falls back); read `theme.colors[token]` / `theme.style(...)`, treat a terminal-default token as *no colour* (`isTerminalDefault()` / `NO_PANEL_COLOR` in `packages/core/src/chrome.ts`). Emitting codes and `packages/diff/src/ansi/` are exempt. Full wording: `docs/features/ui.md`.
2. **Never `console.*` from a render or event path — use `core/log`'s `diag`.** A stray write smears a fullscreen differential frame, and a subagent child discards every non-JSON stdout line. Use `diag(scope, msg)` / `describeError(err)`; user-facing text goes to `ctx.ui.notify`. Guard: `packages/core/src/log-discipline.test.ts` (allow-list is `packages/core/src/log.ts` alone; `packages/diff` keeps a sibling copy).

### Verification Order (run independently, wait for each)
1. `npx tsc --noEmit` in the package directory
2. If it fails: read the error → read the relevant file → fix → re-run
3. Loop-break: if a check fails twice without edits between, stop and report BLOCKED

### Commits
- `chore:` for infra (workflows, config, README)
- `feat:` for new features or packages
- `fix:` for bug fixes
- Always commit per logical unit — don't batch unrelated changes

### Documentation Structure
Docs follow the skill-suite layout (`specify`/`finish` conventions):
- **Plans** live in `docs/roadmap/<feature>.md` (kebab-case, named for the outcome) with front-matter `status` (`considering` / `committed`) + `done-when`. Only *unfinished* plans live on disk — the doc is **deleted when the feature ships** (history lives in git; no plan index, no `done/` graveyard). Supersede: note it in the old doc's body, then delete it.
- **Persistent knowledge** goes to `docs/features/<feature>.md` (present-tense, front-matter `status: live` + `last-verified`) or `docs/decisions/NNNN-slug.md` — never left in a plan.
- `docs/research/` holds research reports (front-matter, kebab-cased `docs/research/<topic>.md`); `docs/reviews/` holds code-review write-ups.
- Use the `specify` skill to generate plans (spec input from the `discuss` conversation), then dispatch the `reviewer` subagent to review the plan against the codebase. **Commit the plan file to git** — it is untracked by default and would be lost.

### Event Handlers
- Register `session_shutdown` handlers at the top level of `register()`, NOT inside `session_start`
- Nested registration causes handler accumulation on `/reload`
- Never keep per-session state at module scope. Pi caches an extension module per process and re-invokes its factory with a fresh `pi` per session, so a module-scope `let`/`var` belongs to the process — and a `session_shutdown` belonging to ANOTHER session (a subordinate in-process session, e.g. a third-party subagent child) will clear the live session's state, silently (#156). Keep it in the `register()` closure; if a top-level handler must reach it, key a `const` Map by `ctx.sessionManager.getSessionId()`. Guard: `packages/core/src/session-state-discipline.test.ts`; rule: `docs/features/session-state.md`

### Source Files
- Pi loads `.ts` files directly — publish `src/` in `"files"` field
- ANSI escape sequences use `\x1b` with semicolons (e.g., `\x1b[38;2;255;215;0m`)
- Common pitfall: commas instead of semicolons in truecolor ANSI codes

## Adding a package, publishing, releasing

- **New-package checklist (8 steps)** — manifest fields, `pi.extensions`, meta deps+register, the `PLUGINS` manifest, npm trusted publisher, release `ORDER`, this file's structure list, README → `CONTRIBUTING.md` §Adding a New Package. Missing any one of them is **silent**: nothing publishes and nothing fails.
- **Releasing** — all packages share one version; `git tag -a v0.x.y` then push to trigger the release workflow. Five-step flow, publish order, release-notes wording → `CONTRIBUTING.md` §Release steps.
- **Version/dependency pins** — every manifest pins the pi family to the exact `1.4.0` (compile-time only) and `@types/node` to `22.19.19`; peer floors are documentation, not protection → `CONTRIBUTING.md` §Type-check target, §Peer floors.
- **CI's two jobs** — floor `typecheck` and floating `pi-latest` (SKIPPED on a PR is the expected result) → `CONTRIBUTING.md` §CI.

## Testing Locally

```bash
ln -s $(pwd) ~/.pi/agent/extensions/pi-archimedes
```

Root `package.json`'s `pi.extensions` points at `meta/src/index.ts` (detail: `CONTRIBUTING.md` §Testing Locally).

- Hardware stability (random SEGV/ICE, corrupted build products, page cache/bit flips): tools and criteria all live in `~/hw-stability` (entry `~/hw-stability/AGENTS.md`, skill `hw-stability-tools`). This repo only follows global `AGENTS.md` hard rule 12 — file evidence under `~/hw-stability/evidence/inbox/` — and does not restate it here.
