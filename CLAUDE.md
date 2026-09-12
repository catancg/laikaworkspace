# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`f:\docs\Laika` is a workspace holding three **independent git repositories** for
"SoyLaika" — a multi-tenant CRM with an AI bot that handles WhatsApp/Instagram sales
conversations. There is no monorepo tooling (no workspaces, no shared lockfile) tying them
together — the two code repos are siblings on disk, each with its own `.git`,
`node_modules`, and CLAUDE.md:

- **`soylaika.backend/`** — NestJS + Prisma + PostgreSQL + BullMQ (Redis) API. Multi-tenant
  by database: each tenant gets its own Postgres database, resolved per-request via subdomain
  or `X-Tenant-Slug` header. Runs on `:3000`.
- **`soylaika.frontend/`** — Next.js App Router CRM client ("asap-crm"). Talks to the backend
  exclusively through `lib/api.ts`; no server-side data layer of its own. Runs on `:3001`.

The **root itself is the third repo** (`laikaworkspace`) — it tracks this file, `docs/`, and
nothing else. Committing here does not touch either code repo, and vice versa.

**Always read the sub-project's own `CLAUDE.md` before working inside it** —
[soylaika.backend/CLAUDE.md](soylaika.backend/CLAUDE.md) and
[soylaika.frontend/CLAUDE.md](soylaika.frontend/CLAUDE.md) — they contain the real
architecture notes, conventions, and commands. This root file only orients you across the two.

`backend.rar` / `frontend.rar` at the root are point-in-time archive backups, not part of
either working tree — ignore them unless the user specifically asks about a backup.

## How the two repos fit together

A message flow crosses both repos: WhatsApp/Instagram webhook → backend queue
(`MESSAGE_QUEUE`, BullMQ) → `MessageProcessor` → `AiService` (orchestrator picks an agent,
agent replies) → sent back to the customer. The frontend is a separate, purely client-side
surface for the human side of the business (leads, sales, funnel, agent/prompt editing,
metrics) — it never sees message processing directly, only reads/writes via the REST API
defined in `soylaika.frontend/lib/api.ts`.

When a change spans both repos (e.g. a new backend endpoint the CRM needs to call), expect
to touch `soylaika.backend/src/**` and then add the corresponding function to
`soylaika.frontend/lib/api.ts` — see that file's own CLAUDE.md section for the pattern.

## `docs/` — design docs and plans

Product requirement docs and implementation plans live in the **root repo**, deliberately kept
out of the two code repos: they were previously committed alongside the code and grew to ~45%
of the backend PR diff, which made review harder rather than easier. Nothing in `docs/` is read
at runtime.

- **`docs/prd-*.md`** — one file per feature, each titled `PRD N` in its heading. Current set,
  in numbering order: RAG knowledge layer (1), FAQ content ingestion (2), FAQ admin UI (3),
  FAQ bulk import (4), RAG system test (5), agent evaluation suite (6), agent config
  versioning (7), funnel stage criteria (8), manual message line breaks (9), customer
  boundary on agent configuration (10), message origin (11), contact lifecycle events (12),
  acquisition attribution (13), quote capture (14), funnel re-entry (15), lead
  disqualification (16). **Do not trust this line, or a PRD's own `Status:` header — both have
  been wrong.** PRDs 12 and 15 each read "proposed" while their code was merged and their
  migration applied; designing against that turns an edit into a destructive migration across
  every tenant database. Verify with `git log` in `soylaika.backend` and grep `src/` for the
  identifiers the PRD proposes. As of 2026-09-12: 1–5, 8, 9 and 11–16 implemented; 6, 7 and 10
  proposed. PRD 10 overlaps PRD 7's surface — 10 owns *who
  may write* agent config, 7 owns *history and undo* — so changing one means re-reading the
  other's boundary section. PRDs 11–14 come from one ~90-field customer-data dictionary,
  split by what is structurally missing rather than by topic: 11 owns *what sent a message*,
  12 *when things happened to the lead*, 13 *where the lead came from*, 14 *what was
  quoted*. PRD 11 §3 is the boundary between 11 and 12. **The `Opportunity` entity that all
  four of them promise was deliberately NOT built** — PRD 15 §2 is the record of why: PRD 12's
  event log removed its justification, so PRD 15 became funnel re-entry plus a metrics fix
  rather than a refactor across ~55 backend references and twenty frontend files. Their four
  `opportunityId` hooks stay unused and cost nothing; if per-cycle analysis ever proves
  painful to derive, the entity is still available and those PRDs still say where it plugs
  in. A reader who finds four PRDs promising an entity and no entity should land on §2, not
  assume it was forgotten. Four sections carry decisions
  the rest of the work leans on: PRD 12 §5.1 deliberately does not model the
  disqualification taxonomy (and notes that PRD 8's `MAX_OUT_ENABLED` cap leaves only one
  free `out` stage slot), PRD 13 §3.1 establishes that "direct" and "referred" are
  indistinguishable at the webhook so neither is ever inferred, PRD 14 §3 explains why the
  table is not called `Quote`, and PRD 15 §4 is load-bearing for anyone touching the
  resultados metrics: won counts are read from a lead's CURRENT stage, so once leads can
  re-enter the funnel that number drops every time a customer comes back.
- **`docs/plans/`** — dated step-by-step implementation plans (`YYYY-MM-DD-<feature>.md`),
  written from a PRD before execution. See `docs/plans/README.md`.

**A PRD is the design record, not a changelog.** When implementation reveals the design was
wrong — a control that doesn't hold, a coupling nobody knew about — correct the PRD in place
and say so in the commit message. Several sections in the current set exist only because a
claim was checked against the code and turned out to be false.

Cross-references between PRDs are by section (`PRD 6 §4.0`), so renumbering a section means
grepping `docs/` for references to it.

## `briefing-contexto-agentes-ES.md`

A Spanish-language onboarding doc for a *separate* kind of session: not for editing backend
code, but for acting as a prompt-engineering assistant that tunes the AI bot's agent prompts
(stored in the backend's `Agent` DB table, edited via the superadmin panel — not files in this
repo). It documents the prompt assembly order, the seven system agents (`_orchestrator`,
`default`, `ventas`, `soporte`, `tracking`, `devolucion`, `humano`), hard invariants (never
invent business data, never expose internal routing, WhatsApp-only formatting, Argentine peso
formatting, voseo Spanish), and known failure modes. Only relevant if the user is explicitly
asking for help revising bot prompt text rather than backend/frontend code — see
[soylaika.backend/doc/agentes-e-info-del-negocio.md](soylaika.backend/doc/agentes-e-info-del-negocio.md)
for the code-level counterpart.

## Working safely in this workspace

Things that have actually gone wrong here, or that would go wrong for a session assuming the
usual conventions. Read before the first write.

### Git

- **Stage explicit paths. Never `git add -A`, `git add .`, or `git stash`.** Both code repos
  carry uncommitted local edits to their own `CLAUDE.md` that are not yours to move — a blanket
  stage or a stash sweeps them up and they are not recoverable from the remote.
- **Check which branch you are on before starting.** The code repos are not always left on
  `main`; a feature branch from earlier work is often still checked out.
- **Three repos, three commits.** A change spanning backend + frontend + a PRD is three separate
  commits in three separate repos. Push nothing without being asked.

### One working tree, one dev server

There are no git worktrees here — every session shares `f:\docs\Laika`. If more than one session
may be active, assume another is editing the same files: prefer narrow, quickly-committed
changes over long-lived working-tree state.

Local dev starts from **one** script, run from the backend directory:

```powershell
powershell -ExecutionPolicy Bypass -File .\start-dev.ps1
```

It brings up docker, the backend (`:3000`) and the frontend (`:3001`) together, and only one
session can hold those ports. **Never run `npm run build` while it is running** — it competes
with `nest start --watch` over `dist/` and takes the running backend down.

### Migrations reach every tenant database on boot

`TenantMigrationsService.onApplicationBootstrap` runs in the background on every backend start:
it reads `prisma/migrations/`, **sorts the directory names lexicographically**, and applies
anything unregistered to **every active tenant's database**. Failures are swallowed into a
`logger.warn`.

- A half-written migration in your branch propagates to every local tenant DB the moment anyone
  starts the dev server. It does not wait to be asked.
- Directory names are both the ordering and the identity. Migrations authored the same day can
  collide or order wrongly, and **renaming a migration directory makes it run again** — the
  `_tenant_migrations` registry is keyed by name.
- Migrations are hand-written SQL, not generated. Master-DB ones use `IF NOT EXISTS` so they can
  be pre-applied before a deploy.

Never run migrations, seeds, or destructive SQL against an existing tenant or production
database. Create a throwaway database, use it, drop it. `psql` and `createdb` are **not on PATH** —
Postgres runs in the container `soylaikabackend-postgres-1` with 5432 published, so reach it as
`docker exec -e PGPASSWORD=<from docker-compose/.env> soylaikabackend-postgres-1 psql -U postgres -d <db>`.

Two more that cost real time:

- **A new migration applies itself the moment the watcher reloads.** Editing any `src/**` file
  restarts `nest start --watch`, and boot runs the tenant migration runner. Author and verify a
  migration somewhere else, then move it into `prisma/migrations/` — a half-written one that gets
  applied is recorded as applied, and fixing the file will not re-run it.
- **`prisma generate` runs only inside `npm run build`** — no postinstall hook, and `start:dev`
  does not do it. Since you must not build while the dev server is running, the generated client
  goes stale after any schema change, and a new column then reads back as `undefined` rather than
  `null` — so a `!== null` guard is true for every row. That has already caused one real defect.
  Run `npx prisma generate` by hand after touching the schema.

### Notifications are written tenant-side and read master-side

Everything that actually reaches a person — `handoff.service.ts`, `message.processor.ts` — writes
`db.notification` on the **tenant** handle, where that tenant's users live. `NotificationsService`
writes and reads through `PrismaService`, which is the **master** database; its `createForBranch`
has no callers, and the CRM bell returns nothing for a tenant user because the read path queries
master. Write notifications on the tenant handle. The read path is a known, unfixed gap — see
[docs/prd-lead-disqualification.md](docs/prd-lead-disqualification.md) §4.3.

### There is no global `ValidationPipe`

Request bodies are plain objects, not DTO classes, and the `@Body()` type annotations are
TypeScript only — erased at runtime. Nothing validates them.

**So mass assignment is the default here**, and it has bitten this codebase repeatedly:
`PATCH /tenants/:slug`, `PATCH /tenants/:slug/templates/:id`, `PATCH /crm/contacts/:id`, and
— still live — `POST` and `PATCH /api/funnel/stages`, which forward the raw body into
`db.funnelStage.create` / `.update`.

The fix is always a **named allowlist** in the controller or service. Do **not** reach for
`app.useGlobalPipes(new ValidationPipe({ whitelist: true }))`: almost no endpoint here has a DTO
to validate against, so a global pipe would reject most of the API.

`CrmService.updateContact` is the worked example — `CONTACT_PATCH_FIELDS` plus
`src/crm/crm.service.update-contact.spec.ts`. Two details from it that generalise: use `in` and
not `!== undefined`, or a legitimate `notes: null` (clearing a field) becomes indistinguishable
from an absent one; and write the tests that pin the *surrounding* behaviour at the same time,
because an allowlist silently drops anything you forget to list. The worst field there was not
the obvious one — `claimedById` let any user bypass `claimContact`'s conflict check entirely, so
when auditing one of these, ask which columns are themselves controls.

The same shape applies to guards. `@Roles` on a controller class only works because `RolesGuard`
reads `reflector.getAllAndOverride([handler, class])` — with `reflector.get(handler)` it silently
ignored every class-level `@Roles`, leaving three controllers unguarded. **A decorator is not a
control until a test asserts on it.**

### Frontend

- `lib/api.ts` is the entire backend contract in one file. Every API-touching feature edits it,
  which makes it the most likely merge conflict in the repo.
- **Stage names and colours are no longer duplicated in the CRM** — PRD 8 §8 removed those maps.
  `lib/stage-style.ts` derives the badge from `stage.color`, and screens fetch the list through
  `api.funnel.stages()`, so a new backend stage renders and filters with no frontend change. This
  bullet used to say the opposite and cite the very PRD that fixed it, which sent a session hunting
  maps that no longer exist. Grep before believing any claim of a hardcoded copy.
- `npm run lint` starts from a **non-zero baseline** (`react-hooks/set-state-in-effect`). Record
  the count before you change anything and do not grow it — there is no test suite to catch
  regressions otherwise.

## Commands quick reference

Run these **inside the respective sub-project directory**, not from the root.

For normal local development use `start-dev.ps1` above, not these directly.

**Backend** (`soylaika.backend/`):
```bash
npm run start:dev            # dev server w/ watch, :3000; seeds superadmin + master agents
npx tsc --noEmit -p tsconfig.json   # type-check
npm run build
npm run lint
npm test                     # jest; single file: npm test -- src/ai/ai.service.spec.ts
npx prisma migrate deploy    # master DB migrations (dev)
```

**Frontend** (`soylaika.frontend/`):
```bash
npm run dev     # next dev -p 3001 (expects backend on :3000)
npm run build
npm run lint
```
No test suite is configured in the frontend repo.

## Cross-cutting conventions

- **User-facing text, code comments, and commit-adjacent prose are in Spanish** in the
  backend; the frontend mixes Spanish UI copy with English code/comments — match whatever
  convention the file you're editing already uses.
- **`soylaika.frontend/package.json` pins `next@16.2.6`** — well past most training data.
  Check `node_modules/next/dist/docs/` in that repo before assuming familiar App
  Router/caching semantics; see `soylaika.frontend/AGENTS.md`.
- Don't assume a change in one repo is deployed with the other — they have separate
  Dockerfiles/deploy configs (`soylaika.backend/railway.toml`,
  `soylaika.backend/docker-compose.yml`) and are versioned independently.
