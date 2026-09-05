# PRD 3 — FAQ administration UI (CRM)

**Status:** draft
**Repo:** `soylaika.frontend`, with blocking prerequisites in `soylaika.backend`
**Depends on:** [prd-rag-knowledge-layer.md](prd-rag-knowledge-layer.md), [prd-faq-content-ingestion.md](prd-faq-content-ingestion.md)

---

## 1. Problem

The backend knowledge layer is complete: 14 endpoints under `/api/faq`, a review queue, LLM
extraction from documents, versioned re-ingestion. **None of it is reachable from the CRM.**
`soylaika.frontend/lib/api.ts` contains no reference to any FAQ endpoint.

The consequence is not cosmetic. The whole design rests on a human approving content before a
customer can see it — and there is no screen to approve in. Today a tenant's knowledge base can
only be inspected or corrected by calling the API by hand, which means in practice it is not
corrected at all.

Two things follow, and they set this document's priorities:

- **Editing is the point, not a nice-to-have.** Each tenant has its own database and its own
  bot answers. A wrong answer in a tenant's FAQ is a wrong answer given to that tenant's
  customers, repeatedly, until someone fixes it. The person who can fix it needs a list, a
  search, and an edit box.
- **Approval is the gate.** `review_status = 'APPROVED' AND active = true` is what makes a
  chunk visible to the bot. Content that is stuck in `PENDING_REVIEW` is invisible; content
  wrongly approved is live. Both states need to be obvious at a glance.

## 2. Goals

1. From a tenant's page in the superadmin panel, reach that tenant's knowledge base in one click.
2. See every chunk as a question/answer pair, with its status, source and agent targeting.
3. Edit a chunk's question, answer, agent targeting and active flag — and understand what that
   costs and what it changes.
4. See what is waiting for review, with the evidence needed to judge it, and approve or reject.
5. Never leave any doubt about **which tenant** is being edited.

## 3. Non-goals

- Document upload / extraction UI (`POST /api/faq/extract`). Deliberately deferred to phase 2 —
  see §10. The list and the review queue are what make the existing content safe; extraction adds
  more content to a queue nobody can yet work.
- Starter-pack application and CSV import UI. Same reasoning, same phase.
- A tenant-facing (non-superadmin) version of these screens. The tenant-side surface is a
  separate decision about what a client should be trusted to edit unsupervised.
- Any change to retrieval behaviour, thresholds or prompt assembly.

---

## 4. Blocking prerequisites (backend)

**Neither of these is optional, and neither is frontend work.** Both were found by reading the
code while writing this document; without them, the UI described here cannot function at all.

### 4.1 Superadmin cannot currently reach a tenant's FAQ data

`/api/faq/*` resolves its tenant from request context — the `X-Tenant-Slug` header, falling back
to the subdomain (`TenantMiddleware`). The CRM derives that header in `lib/api.ts`:

```ts
function getTenantSlug(): string | null {
  const parts = window.location.hostname.split(".");
  const sub = parts[0];
  if (sub === adminSub || sub === "localhost" || sub === "www") return null;   // ← superadmin
  return sub;
}
```

A superadmin sits on the admin subdomain, so the header is **`null`** and every FAQ call would
resolve to the **master** database rather than the tenant's. There is no per-call override.

Adding one does not fix it either. `JwtStrategy.validate` looks the caller up in whichever
database the request resolved to:

```ts
const db = req.tenantDb ?? this.prisma;
const user = await db.user.findUnique({ where: { id: payload.sub } });
if (!user) throw new UnauthorizedException();
```

A superadmin's user row lives in master, not in the tenant database — so sending
`X-Tenant-Slug: tenant-dev` returns **401**.

**Resolution: follow the pattern this codebase already uses.** Superadmin cross-tenant access is
solved elsewhere by putting the slug in the *path* on a superadmin-only controller, with the
tenant database resolved server-side:

```ts
// tenants.controller.ts — class-level @Roles('superadmin')
@Get(':slug/agents')
getAgents(@Param('slug') slug: string) { return this.tenants.getTenantAgents(slug); }

@Patch(':slug/agents/:id')
updateAgent(@Param('slug') slug: string, @Param('id') id: string, @Body() body) { … }
```

Mirror it for FAQ. The service side is two lines — the same two `getTenantAgents` already uses:

```ts
const tenant = await this.findBySlug(slug);
const db = this.factory.getClient(tenant.database_url);   // TenantPrismaFactory, cached per URL
```

That `db` is exactly the `tenantDb` every `Faq*` service method already accepts as a parameter, so
the new routes are thin delegations with no changes to the FAQ services themselves:

| New route | Delegates to |
|---|---|
| `GET /tenants/:slug/faq` | `FaqIngestionService` list |
| `PATCH /tenants/:slug/faq/:id` | `FaqIngestionService.updateOne` |
| `DELETE /tenants/:slug/faq/:id` | soft delete |
| `GET /tenants/:slug/faq/review` | `FaqReviewService.listQueue` |
| `GET /tenants/:slug/faq/review/counts` | `FaqReviewService.counts` |
| `POST /tenants/:slug/faq/:id/approve` | `FaqReviewService.approve` |
| `POST /tenants/:slug/faq/:id/reject` | `FaqReviewService.reject` |

**Manual create (`POST /api/faq`) is deliberately not in that list, and it is a decision worth
making explicitly rather than by omission.** The endpoint exists and a chunk created through it is
born `APPROVED` under the strict lint — no review step. A list you can edit but not add to is an
odd tool, and the obvious use case is real: a superadmin who spots a missing answer while fixing a
wrong one. But adding it means a screen whose content goes live with no second pair of eyes, which
is the one thing every other intake path in this system avoids. Recommendation: include it in
phase 1, and have the UI create as `PENDING_REVIEW` so it lands in the same queue as everything
else — which requires the endpoint to accept a `reviewStatus`, since it currently does not.

This is preferable to relaxing `JwtStrategy`: it keeps cross-tenant access explicit at the route
level where it can be audited, rather than weakening authentication for every endpoint in the
application. **It is a security-relevant change and should be reviewed as one** — these routes let
one authenticated principal read and write another tenant's data by design, so the superadmin role
check is the only thing standing between tenants.

### 4.2 RAG is off by default and cannot be turned on

```prisma
faq_rag_enabled  Boolean  @default(false)
```

`AiService` gates all retrieval on it. The tenant `PATCH /tenants/:slug` endpoint accepts only
`name`, `openrouter_api_key`, `openrouter_model`, `orchestrator_model` — **there is no way to set
this flag through the API.** Every tenant therefore has retrieval off, and no amount of
well-curated FAQ content will reach a customer.

Add `faq_rag_enabled?: boolean` to that endpoint's body, and surface it as a toggle (§6.1). A
knowledge base that the bot is not consulting is the single most likely way this feature appears
"broken" to someone who has just spent an hour curating it.

---

## 5. Where it lives

```
/superadmin/tenants/[slug]                 existing tenant detail page
  └── Respuestas comunes (card, §6.1)  →
/superadmin/tenants/[slug]/faq             new — list + editor  (§6.2, §6.3)
/superadmin/tenants/[slug]/faq/revision    new — review queue    (§6.4)
```

**The tenant slug stays in the URL.** Not in a context provider, not in a store. The expensive
mistake in a multi-tenant admin panel is editing the right record in the wrong tenant, and a slug
that is visible in the address bar, survives a refresh, and is safe to paste to a colleague is the
cheapest possible defence. It also matches the `:slug`-in-path shape of §4.1.

The existing tenant page is one large `page.tsx` composed of sections, with heavier ones extracted
(`components/tenant-agents-section.tsx`). Follow that: `TenantFaqSection` for the card, and
separate route files for the two screens.

---

## 6. Screens

### 6.1 Entry point — card on the tenant page

A section titled **"Respuestas comunes"**, matching the visual language of the surrounding cards.
It shows, for this tenant:

- total active chunks
- **count awaiting review**, as the prominent number — this is the only number that implies an
  action
- whether retrieval is on (`faq_rag_enabled`), as a toggle, not a read-only badge (§4.2)
- a link into the list

When retrieval is off, say so plainly and near the toggle: *"El bot no está consultando estas
respuestas."* A silent boolean is how a curated knowledge base sits unused for a month.

When the pending count is greater than zero, it links to the review queue rather than the list —
send people where the work is.

### 6.2 The list

A table of every chunk in the tenant, `GET /tenants/:slug/faq`.

| Column | Notes |
|---|---|
| Pregunta | truncated to one line, full text on the row expand |
| Respuesta | truncated; the pair is the unit people recognise |
| Estado | `APPROVED` / `PENDING_REVIEW` / `ARCHIVED` as a coloured pill |
| Agentes | `Todos` when the array is empty (see §7.2) — never blank |
| Origen | `source_type` + a hint of `source_ref` (imported CSV, starter pack, document, manual) |
| Activo | toggle |
| v | `version`, shown only when > 1 |

**The list endpoint's `select` does not currently return `source_ref` or `version`**, only
`source_type`. The two right-hand columns therefore need the projection widened when the
superadmin route of §4.1 is added — it is a new route, so this costs nothing beyond remembering
to do it. Compare `QUEUE_SELECT` in `faq-review.service.ts`, which already returns `source_ref`
and `source_ordinal` and is the better template.

Requirements:

- **Filters** on status, agent, source type, and active — these map exactly to the query params
  `GET /api/faq` already accepts, so no backend work.
- **Text search over question and answer is client-side.** There is no server-side search, and
  this document does not ask for one: the endpoint returns the tenant's full list unpaginated
  (§6.5), so filtering in the browser is both possible and consistent with what the user sees.
  This is the tool for *"the bot said something wrong — find where that came from"*, and that
  search is how the fixer starts.
- Row expands in place to show the full pair rather than navigating away; scanning is the
  dominant activity.
- An `ARCHIVED` chunk is visible but visually recessed, and **never deleted from the view** —
  nothing in this system is hard-deleted, and the UI should not imply otherwise.

### 6.3 Editing

Editing opens a drawer over the list, not a separate page: the surrounding rows are context.

Editable: `question`, `answer`, `agents[]`, `tags[]`, `active`. Everything else — status, version,
provenance, embedding metadata — is displayed read-only. `PATCH /tenants/:slug/faq/:id`.

Three behaviours the UI has to get right, all consequences of §7.1:

1. **Saving is not instant and must not be optimistic.** Changing question or answer triggers a
   synchronous embedding call inside the request. Show a pending state on the save button, keep
   the drawer open until it resolves, and do not update the row until the response arrives.
2. **A failed save saved nothing.** If the embedding provider errors, the whole update throws and
   the row is untouched — there is no half-written state where the text changed and the vector
   did not. Say so: *"No se guardó ningún cambio. Reintentá."* A retry is safe.
3. **Editing an approved chunk publishes immediately.** `updateOne` does not touch
   `review_status`, so an approved chunk stays approved and the new text is live on the next
   customer message. This is correct for a superadmin, but it must not be a surprise — a short
   line under the save button (*"Este cambio queda publicado al guardar"*) is enough, and only
   when the chunk is currently `APPROVED`.

Show the cost honestly where it exists: an edit that changes only agents/tags/active performs no
embedding call at all. That distinction is worth surfacing as a small note, because it is the
difference between a free edit and a billed one.

**A manual edit overwrites; it does not create a version.** This is the one place where the UI
contradicts a reasonable expectation, so it has to be stated. PRD 2 phase 4 added versioning, but
it lives in `upsertBatch`: re-ingesting a *document* whose text changed inserts a new row at
`version + 1` and leaves the approved one serving until the new one is approved. `updateOne` — the
path behind this drawer — does none of that. It is an in-place `UPDATE WHERE id = …`, so the
previous question and answer are gone, with no superseded row and nothing to roll back to.

Two consequences:

- Do not show version history or a "revert" affordance in the editor. There is nothing to revert
  to, and offering it would be a lie. Open question 3 (§13) covers whether superseded rows from
  *re-ingestion* deserve a separate history view; that is a different data set.
- Because the edit is destructive and immediately live on an approved chunk, the drawer is the
  right place for a plain confirmation on save, not a silent autosave. Weigh this against §12 if
  edit history is later judged necessary — adding it means routing manual edits through the
  versioning path, which is a backend change, not a UI one.

### 6.4 Review queue

`GET /tenants/:slug/faq/review`. One row per pending chunk. This screen has one job that the list
does not: **let a human judge whether an answer is true.**

Therefore the non-negotiable requirement: **show `source_span` beside the answer.** Every chunk
produced by document extraction carries the verbatim excerpt it was derived from, and the queue
endpoint already returns it. Approving means reading the answer against that excerpt — not against
how plausible the answer sounds. An LLM's fabrications are, by construction, plausible. A queue
that shows only the question and answer is a queue that approves hallucinations.

**Show the lint findings the queue already returns.** `listQueue` does not return bare rows — it
appends `findings` to each one, computed with the strict `'approval'` lint, so the reviewer sees
exactly what `approve()` will refuse *before* pressing the button. The backend comment calls this
"el backstop contra el rubber-stamping". Render blocking findings inline on the row, and disable
approve while any is present, pointing at edit instead. This costs no extra request — the findings
arrive with the queue — and it converts an error-after-the-fact into a visible precondition.

**Order and filters are already right; do not fight them.** The queue is sorted `created_at` ascending
— oldest first, because it is a queue and not a listing — so the UI should not default-sort by
anything else. It accepts `source_type` and `source_ref` filters, which matter more than they look:
one document extraction can deposit up to 60 candidates at once, and reviewing them as a batch
scoped to that `source_ref` is the natural unit of work.

- Approve: `POST /tenants/:slug/faq/:id/approve`, optionally with edits in the same call (the
  endpoint accepts `question`, `answer`, `agents`, `tags` — and **only** those; extra fields are
  stripped at the controller).
- Reject: `POST /tenants/:slug/faq/:id/reject` — archives and deactivates, keeps the row.
- Approval is linted strictly on the backend and can fail with a 400 explaining why. Render that
  message; it names the rule that was broken.
- Batch approve is **out of scope**, deliberately. The entire extraction pipeline drops
  unsupported candidates rather than flagging them precisely because flagged-but-present content
  gets bulk-approved by a tired reviewer. A "select all" button reintroduces the failure the
  backend design spent effort avoiding.

---

### 6.5 Volume, empty states and failure states

Applies to both the list (§6.2) and the queue (§6.4).

**Neither endpoint paginates.** Both are `findMany` with no `take`/`skip`, returning everything.
That is workable at the volumes this feature produces — a starter pack is 20 entries, a CSV import
is capped at 500 rows, document extraction at 60 candidates per document — so a mature tenant lands
in the high hundreds to low thousands. Accept it for phase 1, and treat it as a deliberate ceiling
rather than an oversight: **client-side search and filtering depend on the whole list being
present**, so adding pagination later means adding server-side search in the same change. Revisit
when any single tenant passes roughly 2,000 chunks.

Three states the screens must distinguish, because they look identical if handled carelessly:

- **No content yet** — the tenant has never imported or extracted anything. Offer the phase-2
  entry points rather than an empty table.
- **Nothing matching the current filters** — say which filter is excluding everything, with a
  reset. Distinct from the above, and the more common cause of a confused "it's empty".
- **The tenant has not run the `FaqChunk` migration.** `FaqReviewService.counts` explicitly
  catches this and logs it, precisely so a real database error does not read as a reassuring zero.
  The UI owes the same distinction: a broken tenant must not render as a calm "0 pendientes".

---

## 7. Answers to the two questions this document was asked

### 7.1 When a chunk is edited, how is the vector table updated?

Synchronously, inside the same request, and only when the text actually changed.

`FaqIngestionService.updateOne` computes `content_hash = sha256(question + "\n" + answer)` and
compares it against the stored hash:

- **Hash differs** (question or answer edited) → it calls the embedding provider with
  `` `${question}\n${answer}` ``, then writes text, `agents`, `tags`, `active`, `content_hash`,
  `embedding`, `embedded_at` and `updated_at` in **one** `UPDATE`. Text and vector can never
  disagree, because they are written together.
- **Hash matches** (only `agents`, `tags` or `active` changed) → a plain update of those three
  columns. No embedding call, no provider cost, vector untouched.

There is no background job and no queue. Implications, in order of how much they affect the UI:

- The request is as slow as the embedding call, so the save must be non-optimistic (§6.3).
- If embedding fails, `updateOne` throws before any write — the row keeps its old text *and* its
  old vector. Retrying is safe and idempotent.
- Both question and answer feed the embedding, so editing either re-embeds. The question carries
  most of the retrieval signal, but they are embedded as one text.
- Each re-embed is billed and recorded in `AiUsage` with `kind = 'faq_index'`, which is how FAQ
  indexing cost stays separable from chat cost.
- `review_status` is untouched by an edit (§6.3, point 3).

The dimension is fixed at 1536 by the `vector(1536)` column; changing the embedding model requires
a migration and a full reindex, which is documented in `.env.example` and is not something this UI
should ever offer.

### 7.2 Which agents call the RAG tool?

**All of them, and none of them — because it is not a tool.**

Retrieval is not exposed to the model as a callable function. It runs in `AiService` *before* the
completion request, and injects a knowledge block into the **dynamic** region of the prompt
(alongside the contact block, ahead of history, deliberately never inside `stablePrompt` so the
provider's cached prefix is not disturbed). The agent never decides to consult it.

It fires for the **answering agent on every message**, gated only by the tenant flag:

```ts
if (tenant?.faq_rag_enabled) {
  const outcome = await this.faqRetrieval.retrieve(lastUserMessage, agentType, resolvedDb, tenant);
}
```

The orchestrator does not retrieve — it is a separate, earlier classification call. Targeting is a
property of the **chunk**, not the agent, applied in the retrieval query:

```sql
WHERE active = true
  AND review_status = 'APPROVED'
  AND embedding IS NOT NULL
  AND (cardinality(agents) = 0 OR $agentType = ANY(agents))
ORDER BY embedding <=> $vector
LIMIT $topK
```

So:

- `agents = []` (empty, the default) → **available to every agent**. This is why the list must
  render an empty array as `Todos` and never as blank (§6.2).
- `agents = ['ventas']` → only the `ventas` agent can retrieve it.

The UI should therefore offer an agent multi-select on each chunk, defaulting to empty, and
populate its options from the tenant's own agents (`GET /tenants/:slug/agents`, which already
exists) rather than a hardcoded list — agents are per-tenant rows, not a fixed enum. Exclude
`_orchestrator`, which never retrieves.

Two further conditions are invisible in the UI unless it shows them, and both cause "my FAQ isn't
working" reports: a chunk with `embedding IS NULL` is unreachable, and results below
`FAQ_RETRIEVAL_THRESHOLD` (default `0.78`) are discarded even when they are the closest match.
`POST /api/faq/test` runs the real retrieval path and answers this, which is why the preview is in
phase 1 (§10) rather than a later nicety. Be precise about what it returns, though, because it is
less than "the results with their scores":

```ts
{ block, fired, topScore, chunkIds, prefilterHit, cacheHit, embedMs }
```

One `topScore`, not a score per chunk, and `chunkIds` rather than the question/answer text. The
diagnostic value is nonetheless exactly where it is needed: **when nothing fires, `topScore` is the
best score that *failed* the threshold** — so the screen can say "el mejor match dio 0.74, el
umbral es 0.78", which is the actual answer to "I approved it and the bot ignores it". Resolve
`chunkIds` against the list already loaded (§6.5) to name the chunks. A ranked table with a score
per row would need the endpoint extended; do not promise one in phase 1.

Two details for the screen: it takes `agentType` and defaults it to `'default'`, so it needs an
agent picker — targeting changes the result set (§7.2 above). And `fired: false` with a `topScore`
of `null` means nothing was even retrieved before scoring, which is a different failure from a low
score: an empty knowledge base, a null embedding, or agent targeting excluding everything.

---

## 8. API client changes

Add a `faq` domain to `lib/api.ts`, following the existing `api.tenants.*` shape — slug in the
path, no header dependency:

```ts
faq: {
  list:    (slug: string, q?: { review_status?: string; source_type?: string; agent?: string; active?: boolean }) =>
             req<FaqChunk[]>(`/tenants/${slug}/faq${qs(q)}`),
  update:  (slug: string, id: string, data: Partial<Pick<FaqChunk, "question" | "answer" | "agents" | "tags" | "active">>) =>
             req<FaqChunk>(`/tenants/${slug}/faq/${id}`, { method: "PATCH", body: JSON.stringify(data) }),
  remove:  (slug: string, id: string) => req<{ ok: boolean }>(`/tenants/${slug}/faq/${id}`, { method: "DELETE" }),
  queue:   (slug: string) => req<FaqQueueItem[]>(`/tenants/${slug}/faq/review`),
  counts:  (slug: string) => req<Record<string, number>>(`/tenants/${slug}/faq/review/counts`),
  approve: (slug: string, id: string, patch?: { question?: string; answer?: string; agents?: string[]; tags?: string[] }) =>
             req<FaqChunk>(`/tenants/${slug}/faq/${id}/approve`, { method: "POST", body: JSON.stringify(patch ?? {}) }),
  reject:  (slug: string, id: string) => req<FaqChunk>(`/tenants/${slug}/faq/${id}/reject`, { method: "POST" }),
},
```

Three notes on the sketch above, so it is not mistaken for existing code:

- `FaqChunk` and `FaqQueueItem` **do not exist** in `lib/api.ts` and must be added alongside the
  other interfaces at the top of that file. `FaqQueueItem` is `FaqChunk` plus `source_span`,
  `source_ref`, `source_ordinal`, `created_at`, and `findings` — without `source_span` and
  `findings`, §6.4 cannot be built. The queue endpoint already returns all of them.
- `qs(...)` is shorthand here, not an existing helper. Either add a small query-string builder or
  interpolate the two or three params directly; the file currently has no convention for query
  strings because nothing needed one.
- Every call takes `slug` as its first argument by design (§9). Resist adding a
  "current tenant" wrapper that hides it.

---

## 9. Multi-tenancy and safety

The requirement that motivated this document — *"every superadmin/customer will have their own
data, that is why it is crucial that we can edit the information"* — is also the main hazard. Four
rules:

1. **The slug is in the URL and in every request path.** No ambient tenant state (§5).
2. **The tenant is named on screen, permanently.** A header band on both FAQ screens showing the
   tenant's name and slug, styled distinctly enough that "which tenant am I in" is never a
   question that requires scrolling. Editing tenant A's knowledge base believing it is tenant B is
   silent, plausible, and can persist for weeks.
3. **The superadmin routes are the only cross-tenant surface**, and they are superadmin-gated at
   the controller class (§4.1). Nothing in these screens should ever call `/api/faq/*` directly —
   that path is for a tenant acting on itself.
4. **Destructive actions name their target.** A reject or deactivate confirmation should include
   the tenant name, not just the question text.

---

## 10. Phasing

| Phase | Scope | Rationale |
|---|---|---|
| **0** | Backend prerequisites §4.1 and §4.2 | Nothing else can be built or even manually tested until a superadmin can reach tenant FAQ data and turn retrieval on |
| **1** | Entry card, list, edit drawer, manual create, review queue, **retrieval preview** | The whole point: see, fix, add, approve — and be able to answer "why didn't the bot use this?" |
| **2** | Document extraction UI, CSV import, starter packs | Adds content — worth doing only once there is a working queue to receive it |
| **3** | Duplicate flags, `missing` handling from re-uploads, history of superseded versions | Re-ingestion concerns, valuable once volume exists |

Retrieval preview (`POST /api/faq/test`) moved into phase 1 on reflection. It is one endpoint and
one screen, and without it the first support question — *"I approved it and the bot still doesn't
use it"* — has no diagnostic answer. The reasons a chunk is silently unreachable are all invisible
in the list: `faq_rag_enabled` off, `embedding IS NULL`, an `agents` array that excludes the
answering agent, or a similarity below `FAQ_RETRIEVAL_THRESHOLD` (0.78) which no screen displays.
The preview reports whether retrieval fired and, when it did not, the best score that missed the
threshold — which settles the question in one request (§7.2 has the exact response shape and its
limits). Shipping the curation UI without it means every such question becomes a database session.

Phase 0 is small and unglamorous and must not be skipped: without §4.2 the bot ignores everything
the UI produces, and the feature will look broken in a way that has nothing to do with the UI.

## 11. Metrics

- Time from a chunk entering `PENDING_REVIEW` to a decision on it. If this grows, the queue is not
  being worked and content is invisible.
- Share of approvals that included an edit — high means extraction quality is poor; zero means
  nobody is reading the source excerpt. **Not directly recorded today:** nothing marks an approval
  as having carried a patch, so it can only be inferred from `updated_at` landing within the same
  request as `reviewed_at`, which is fragile. If this metric is wanted, add the field rather than
  mining the timestamps.
- Number of tenants with `faq_rag_enabled = true` and at least one approved chunk. This is the
  real adoption number; everything else is activity.

## 12. Risks

- **The reviewer approves without reading the excerpt.** The mitigation is layout, not copy: the
  answer and its `source_span` shown side by side, with no approve control reachable before both
  are visible, plus the `findings` the queue already returns rendered inline (§6.4). Batch
  approval stays out of scope.
- **A manual edit destroys the previous answer.** `updateOne` writes in place — no version, no
  superseded row, nothing to roll back to (§6.3). Document re-ingestion *does* version, so the
  system behaves inconsistently depending on how the content arrived, and the inconsistency
  favours the path a human is least likely to take carefully. Acceptable for a superadmin acting
  deliberately; it would not be acceptable for a tenant admin, which is another reason §3 holds.
- **A superadmin edits the wrong tenant, and nothing records that they did.** `updateOne` writes
  no actor — `reviewed_by` is set only by approve and reject, so an edit to a tenant's answer is
  unattributed. Combined with the previous risk (the edit also overwrites the old text), a wrong
  edit is both untraceable and unrecoverable, and is caught only by somebody noticing the bot
  saying something new. §9 reduces the chance of making the mistake; nothing currently reduces the
  cost of having made it. Adding an `updated_by` alongside the existing `reviewed_by` is a small
  backend change and is the single cheapest mitigation available — worth folding into phase 0
  while that controller is already being touched.
- **Editing an approved chunk publishes with no second pair of eyes.** Correct for a superadmin,
  but if this UI is ever extended to tenant admins (§3), that decision must be revisited — for a
  tenant admin an edit should plausibly re-enter review.
- **Embedding provider outage makes editing impossible**, since re-embedding is synchronous. Edits
  to agents/tags/active still work. Acceptable, and the failure is loud rather than silent.

## 13. Open questions

1. Should a tenant admin (not superadmin) get a read-only view of their own knowledge base? It
   would reduce "why did the bot say that" support load, and it is a much smaller surface than
   editing.
2. When a re-upload reports `missing` chunks (§6 of PRD 2), where should that surface? It is
   currently returned by the extraction endpoint and consumed by nobody. It probably belongs in
   the review queue as a distinct "no longer in the document" section — but that is a phase-2/3
   design question.
3. Should the list expose `version` history — i.e. read the superseded rows? Nothing is
   hard-deleted, so the data is there. Useful for "what did the bot say last month", which is the
   scenario the retention design was built for, but it is a separate screen.
4. Does the CSV import UI belong in the superadmin panel at all, or on the tenant side? It is the
   channel most likely to be used by the client's own staff.
