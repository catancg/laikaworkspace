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
  versioning (7), funnel stage criteria (8). PRDs 1–5 are implemented and in production;
  6–8 are proposed.
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

## Commands quick reference

Run these **inside the respective sub-project directory**, not from the root.

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
