# PRD 10 — Customer boundary on agent configuration

**Status:** proposed
**Repos:** `soylaika.backend` + `soylaika.frontend` + this doc — three commits
**Related:**
- [PRD 2 — FAQ content ingestion](prd-faq-content-ingestion.md) — the review gate this one copies
- [PRD 7 — Versioning and revert for agent configuration](prd-agent-config-versioning.md) — overlapping surface, different axis (see §1.2)

---

## 1. Why this exists

The requirement, as given:

> The customer must **not** be able to directly edit rules or the **super prompt** in a way that
> could compromise or alter the agent's intended behavior. The customer **may provide Q&A content
> and business information** through the approved workflows, subject to review and approval.
>
> **Acceptance:** the customer-facing panel does not expose controls that directly modify the
> agent's internal logic. SoyLaika administration retains full control over the agent's rules,
> behavior, and underlying configuration.

Three of the four surfaces that feed the system prompt already satisfy this. One does not, and one
satisfies the letter of it while missing the "subject to review and approval" clause.

### 1.1 What the code enforces today

`AiService` assembles the system prompt at [ai.service.ts:426](../soylaika.backend/src/ai/ai.service.ts):

```ts
const stablePrompt = [this.GUARDRAILS, this.CONVERSACION, rulesBlock, businessBlock, imagesBlock, withStages]
  .filter(Boolean).join('\n\n');
```

| Block | Written by | Gate today |
|---|---|---|
| `GUARDRAILS`, `CONVERSACION` | SoyLaika, in code | source control |
| `rulesBlock` (`BotRule`) | **tenant admin** | **none** |
| `businessBlock` (`BusinessProfile`) | tenant admin | none, and none required |
| agent prompt (`withStages`) | superadmin only | enforced |
| FAQ (retrieval) | tenant admin proposes | superadmin approves |

**Agent prompts are compliant.** `AgentsController` throws `ForbiddenException` on POST, PATCH and
DELETE ([agents.controller.ts:25-44](../soylaika.backend/src/agents/agents.controller.ts)); the CRM
page is a lock screen. The only write path is `PATCH /tenants/:slug/agents/:id` on
`TenantsController`, whose class-level `@Roles('superadmin')` is genuinely enforced since
`RolesGuard` switched to `getAllAndOverride` ([roles.guard.ts:21](../soylaika.backend/src/auth/roles.guard.ts)).

**FAQ content is compliant.** Approval sits under `@Roles('superadmin')`
([faq.controller.ts:268-326](../soylaika.backend/src/faq/faq.controller.ts)) and retrieval filters
`AND review_status = 'APPROVED'` ([faq-retrieval.service.ts:148](../soylaika.backend/src/faq/faq-retrieval.service.ts)).

**Rules are not.** `@Roles('admin', 'superadmin')` guards create, update and delete
([rules.controller.ts:21,28,39](../soylaika.backend/src/rules/rules.controller.ts)), and `/settings`
gives any tenant admin a 500-character free-text textarea
([settings/page.tsx:293](../soylaika.frontend/app/(crm)/settings/page.tsx)) with inline edit and
delete. Those rows are injected as ([ai.service.ts:900](../soylaika.backend/src/ai/ai.service.ts)):

> `REGLAS DEL NEGOCIO (definidas por el dueño; cumplilas siempre, tienen prioridad sobre las instrucciones del agente)`

So the customer cannot edit the super prompt, but can prepend up to 50 free-text instructions to it
that the prompt itself declares as outranking the agent. That is the requirement's failure mode
reached through a different door.

Two things bound the damage, and one does not:

- `RulesService` uses a named allowlist and caps length and count
  ([rules.service.ts:51-61](../soylaika.backend/src/rules/rules.service.ts): `MAX_RULES = 50`,
  `MAX_RULE_LENGTH = 500`) — no mass assignment here, unlike the funnel endpoints.
- `GUARDRAILS` precedes the rules block, so the safety floor outranks it.
- But that ordering is **prompt-text priority, not enforcement**. Nothing validates rule content.
  `ignorá las instrucciones anteriores` is a legal 500-character rule.

**Business info** is tenant-admin writable with no review step
([business.controller.ts:21](../soylaika.backend/src/business/business.controller.ts)). The
requirement permits the customer to provide it, but qualifies that with "subject to review and
approval", which does not exist. It is read in **three** places, not one:
[ai.service.ts:189](../soylaika.backend/src/ai/ai.service.ts) and
[:239](../soylaika.backend/src/ai/ai.service.ts) (orchestrator prompts) and
[:852](../soylaika.backend/src/ai/ai.service.ts) (`buildBusinessBlock`). Any gate that covers only
`buildBusinessBlock` leaves two paths open.

### 1.2 Boundary against PRD 7

PRD 7 §1 names `BotRule` and `BusinessProfile` as having "`updatedAt` and nothing else" and proposes
versioning and revert for them. That is a different axis, and the two compose rather than collide:

- **PRD 10 (this doc): who may write.** Authorization and a review gate.
- **PRD 7: what was written before, and how to undo it.** History and revert.

If PRD 7 ships after this, it versions the approved records this PRD establishes. The draft table
here is not a history mechanism and must not be pressed into service as one.

---

## 2. Decisions

Settled during design; recorded because each closes off an alternative.

1. **`BotRule` becomes superadmin-only** — the same treatment agent prompts already have. Customers
   keep read access.
2. **Business info keeps customer authoring, behind a review gate** — customer edits become
   proposals; nothing reaches the prompt until SoyLaika approves.
3. **Draft/live split, not versioned rows** (§3.1). Rejected: mirroring `FaqChunk`'s versioning would
   force all three `businessService.get()` call sites to become "select the APPROVED one", and each
   is a place where a wrong query returns nothing and the bot silently loses its business context.
   Rejected under YAGNI: a field-level change queue.
4. **Existing content is grandfathered as approved.** Rules stay active, business info stays live.
   The gate applies to edits from cutover onward. See §7 for the risk this accepts.
5. **A superadmin's `PUT /api/business` writes straight to live**, skipping the draft. Making SoyLaika
   staff approve their own edits is ceremony with no control value.

---

## 3. Design

### 3.1 Data model

**`BotRule`: no schema change.** The lockdown is purely authorization; no row moves.

**`BusinessProfile`: no schema change.** It remains the single approved record the bot reads, which
is the entire point of the draft/live split — all three `get()` call sites keep working untouched.

**New `BusinessProfileDraft`** (tenant DB): the twelve nullable content columns of `BusinessProfile`
— `name`, `about`, `hours`, `address`, `branches`, `phone`, `email`, `website`, `paymentMethods`,
`shippingInfo`, `returnPolicy`, `extra` — plus:

```prisma
review_status ReviewStatus @default(PENDING_REVIEW)
submitted_by  String?
submitted_at  DateTime     @default(now())
reviewed_by   String?
reviewed_at   DateTime?
reject_note   String?
```

Reuses the existing `ReviewStatus` enum (`DRAFT | PENDING_REVIEW | APPROVED | ARCHIVED`) rather than
inventing a second vocabulary. At most one live draft row per tenant.

**Migration:** `prisma/migrations/20260909120000_business_profile_draft/`, one `CREATE TABLE`. It adds
a table and touches no rows, so the §2.4 grandfathering is automatic and `TenantMigrationsService`
fanning it out at boot is safe — worst case a tenant gets an empty table. The timestamp sorts after
`20260908150000_aiusage_outcome`. Per the root CLAUDE.md the directory name is both ordering and
identity: once pushed it must never be renamed, or it runs again.

### 3.2 Backend — rules

Follows the `AgentsController` precedent exactly, so the guard, the route shape and the error copy
are all already proven in this codebase.

| Endpoint | Change |
|---|---|
| `GET /api/rules` | unchanged — any authenticated tenant user reads |
| `POST` / `PATCH` / `DELETE /api/rules` | throw `ForbiddenException` naming the superadmin panel |
| `GET` / `POST` / `PATCH` / `DELETE /tenants/:slug/rules[/:id]` | new, on `TenantsController` under its class-level `@Roles('superadmin')` |

`RulesService` does not change. Its methods already take a `tenantDb` and already use a named
allowlist. `TenantsService` resolves the slug to a DB and calls them, the same way
`updateTenantAgent` does ([tenants.service.ts:451](../soylaika.backend/src/tenants/tenants.service.ts)).

### 3.3 Backend — business

| Endpoint | Change |
|---|---|
| `GET /api/business` | unchanged — returns the approved live record |
| `PUT /api/business` | same `@Roles('admin','superadmin')`; upserts the **draft** for a tenant admin, writes **live** for a superadmin (§2.5) |
| `GET /api/business/draft` | new — the customer sees their pending submission and any `reject_note` |
| `GET /tenants/:slug/business` | new, superadmin — returns live and draft together for diffing |
| `POST /tenants/:slug/business/approve` | copies draft fields onto live, deletes the draft, invalidates the cache |
| `POST /tenants/:slug/business/reject` | marks the draft rejected with a note; live untouched |
| `GET /tenants/business/pending-count` | new — badges the superadmin tenant list |

### 3.4 Prompt read path

**Unchanged, deliberately.** `buildBusinessBlock` and the two orchestrator call sites keep reading
`BusinessProfile`. Nothing in the bot's path learns what a draft is. This is what makes the change
incapable of blanking the business block in a live conversation.

**Cache invalidation is new.** `businessCaches` and `rulesCaches` are 10-second TTL WeakMaps
([ai.service.ts:26-29](../soylaika.backend/src/ai/ai.service.ts)) and only `invalidateAgentCache` is
exposed ([:91](../soylaika.backend/src/ai/ai.service.ts)). Add `invalidateBusinessCache(db)` and
`invalidateRulesCache(db)` beside it, called from the approve handler and from the superadmin rule
writes, so an approval takes effect immediately rather than up to 10 seconds later.

### 3.5 Frontend

**Tenant CRM, `/settings` — "Reglas del bot".** Keeps the list, loses the controls: the textarea
([settings/page.tsx:293](../soylaika.frontend/app/(crm)/settings/page.tsx)), the Agregar button, the
inline-edit textarea and the delete action. The `isAdmin` conditional at `:291` goes with them, and
the helper copy at `:288` — *"Solo un admin puede editarlas"* — becomes false and is replaced with a
pointer to SoyLaika. `admin/agents/page.tsx` is already this lock-screen treatment; borrow its icon
and phrasing rather than inventing copy.

**Stale copy that falls out of this:**
[admin/agents/page.tsx:28](../soylaika.frontend/app/(crm)/admin/agents/page.tsx) tells the customer
*"podés configurar la info del negocio y las reglas del bot en Ajustes"*. Both halves stop being
true. Rewrite it in the same change, or the lock screen contradicts the page it links to.

**Tenant CRM, business info.** The form still accepts edits; Guardar now submits a proposal. After
saving, the card shows a pending banner with the submitted values and, when the last draft was
rejected, the `reject_note`. The customer must be able to see why their edit is not live, or they
retype it and open a support ticket.

**Superadmin panel.** Rules get a `TenantRulesSection` component on the tenant detail page, beside
the existing `TenantAgentsSection` and `TenantFunnelSection` — not a sub-route. Corrected during
planning: the detail page composes sections for exactly this kind of short per-tenant list, and it
puts rules where a superadmin is already editing prompts. Business review does get its own screen,
`/superadmin/tenants/[slug]/negocio/`, beside `faq/` and `funnel/`, since a field-by-field diff with
Aprobar and Rechazar is too large to inline. The tenant list gets a pending badge from
`GET /tenants/business/pending-count`.

**`lib/api.ts`** — the usual single edit. `api.rules.create/update/remove` are **removed**, not left
to 403: a dead client function is a trap for the next feature. Added: `api.business.saveDraft`,
`api.business.getDraft`, `api.tenants.rules.*`, `api.tenants.business.*`. Spanish UI copy and the
inline hex light/dark pairs, per the frontend CLAUDE.md.

---

## 4. Acceptance criteria

Restated as assertions, because per the root CLAUDE.md a decorator is not a control until a test
asserts on it. `roles.guard.spec.ts` has the harness pattern to copy.

1. A tenant `admin` receives 403 from `POST`, `PATCH` and `DELETE /api/rules`.
2. A tenant `admin` calling `PUT /api/business` leaves the `BusinessProfile` row **byte-identical**
   and creates a `BusinessProfileDraft` instead. This is the core control; everything else is plumbing.
3. `buildBusinessBlock` returns live content while a draft is pending — an unapproved edit reaches
   the prompt through none of the three `businessService.get()` call sites.
4. `approve` copies draft fields onto live and clears the draft; `reject` leaves live untouched.
5. A `superadmin` can still create, edit and delete rules via `/tenants/:slug/rules`, and their
   `PUT /api/business` writes live directly.
6. The customer-facing panel exposes no control that writes to `BotRule` or to a live
   `BusinessProfile`.

**Verification beyond the suite.** A green suite is not the finish line here: drive the real API as a
tenant admin and attempt to reach the endpoints the UI no longer exposes — removing a button proves
nothing about the route behind it. Frontend verification goes through the browser at
`tenant-dev.localhost:3001`. Record the `npm run lint` count in the frontend before any edit; the
known-nonzero baseline must not grow.

---

## 5. Edge cases

- **No `BusinessProfile` row yet.** `approve` must create it, not update a missing row.
- **Draft table missing** on a tenant DB that has not taken the migration. The bot is safe by
  construction (it never reads drafts), but the CRM would throw, so the draft read degrades to
  "no draft" the way `listActive` already does with `.catch(() => [])`.
- **Two admins editing concurrently.** One draft row per tenant, last write wins; the pending banner
  shows whose submission is current.
- **Editing after a rejection** resets the draft to `PENDING_REVIEW` and clears `reject_note`.
- **Approval racing an edit.** Approve copies the draft as read; a submission landing afterwards
  becomes a fresh pending draft rather than being silently absorbed.

---

## 6. Deployment

The migration must be **pre-applied** to tenant databases before the backend deploy, not left to
`TenantMigrationsService` at boot. Boot-time application is background work whose failures are
swallowed into a `logger.warn`, and a tenant whose CRM reads a table that does not exist yet
surfaces as a broken settings screen for the customer.

Backend and frontend deploy independently, and the order matters: **backend first**. A frontend that
has already removed the rules controls against a backend that still accepts them is merely
over-restrictive; the reverse — a frontend still offering Agregar against a backend returning 403 —
is a visibly broken screen.

---

## 7. Accepted risks and out of scope

**Decided, not merely accepted:** by §2.4, every tenant's existing `BotRule` rows stay live and
unreviewed after cutover. Those rows were authored by customers with no gate, so the requirement is
satisfied going forward rather than retroactively — and that is the intended outcome, not a gap
awaiting a fix. There is **no** backfill, no one-time review pass and no migration over existing
rules; the product owner reviewed this and chose to leave them alone. Do not add remediation for
them during implementation.

**Accepted:** `GUARDRAILS` still outranks the rules block by prompt-text ordering alone. This PRD
changes *who writes rules*, not *how strongly the model honours block precedence*.

**Out of scope:** rule content validation (a superadmin can still write a self-defeating rule; that
is a trusted-author problem, not an authorization one); versioning and revert (PRD 7); the FAQ
workflow, which already complies; products, branches and funnel criteria, which do not feed the
system prompt through these blocks.
