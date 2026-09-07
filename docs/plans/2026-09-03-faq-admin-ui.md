# FAQ admin UI (PRD 3 phase 1) — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** From a tenant's page in the superadmin panel, see that tenant's FAQ knowledge base, fix what is wrong in it, add what is missing, approve what is waiting, and answer "why isn't the bot using this?".

**Architecture:** Six additions to an existing CRM. One API namespace in `lib/api.ts`, one section component on the tenant page, and two new routes under `/superadmin/tenants/[slug]/faq`. Everything is a client component fetching in `useEffect`, matching the rest of this app. The tenant slug travels in the URL and in every request path — never in ambient state.

**Tech Stack:** Next.js 16.2.6 (App Router, client components), React 19, Tailwind v4, shadcn primitives over `@base-ui/react`, sonner for toasts.

**Spec:** `f:\docs\Laika\docs\prd-faq-admin-ui.md` — this plan is that document's phase 1. Phase 0 (the backend routes it calls) is complete and merged on `soylaika.backend`'s `RAG-enablement` branch.

## Global Constraints

- **UI copy and code comments in Spanish (es-AR, voseo).** Match the surrounding files.
- **Everything is `"use client"`.** There is no server-side data layer in this repo; pages fetch via `api.*` in `useEffect` with their own `loading`/error state. Do not introduce server components, server actions, or `export const metadata` on these pages — tab titles come from `lib/page-title.ts`.
- **`lib/api.ts` is the only file that may call `fetch`.** Pages import `api` and types from `@/lib/api`.
- **Styling: hand-written arbitrary hex Tailwind in light/dark pairs**, e.g. `text-[#0a1317] dark:text-[#f0f2f5]`, `border-[#dee3e9] dark:border-white/[0.07]`, `bg-white dark:bg-[#18191c]`. Do **not** use the shadcn CSS-variable token classes on CRM screens — this repo deliberately does not, outside `components/ui/`.
- **There is no test suite in this repo.** The gates are `npm run lint` and `npm run build` (which type-checks). Every task runs both and reports both outputs. Since nothing can be asserted automatically, each task's brief names what to verify by reading instead.

- **The lint baseline is not clean, so the gate is "no NEW problems".** Before any of this work: **10 errors and 3 warnings**, across `app/(crm)/admin/chat-test`, `admin/products`, `admin/users`, `clientes`, `contacts/[id]`, `layout.tsx`, `components/contact-sale-block.tsx` and `components/tenant-agents-section.tsx`. Do not fix them — they are outside this work. Your files must contribute **zero** new ones; if the count rises above 10/13, the increase is yours.

- **Avoid the `setState` in-effect rule that the baseline is full of.** Most of those 10 errors are `Calling setState synchronously within an effect can trigger cascading renders`, and they flag *the house data-fetching pattern* — including `tenant-agents-section.tsx`, which several tasks here use as a template. Copying it verbatim would add new errors. Keep the house shape but initialise the flag instead of setting it inside the effect:

  ```tsx
  // `loading` arranca en true en vez de setearse dentro del effect: llamar
  // setState sincrónicamente ahí dispara un render en cascada, y es la regla
  // que ya tiene 10 errores en este repo. Sólo se vuelve a poner en true en
  // refetches disparados por el usuario, que no ocurren dentro de un effect.
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    api.faq.list(slug)
      .then((rows) => { if (!cancelled) setChunks(rows); })
      .catch((e) => { if (!cancelled) toast.error(errorMessage(e, "No se pudo cargar")); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, [slug]);
  ```

  The `cancelled` guard also stops a state update after unmount, which matters here because the user can navigate away mid-request.
- **Next.js 16 is newer than most training data** (`AGENTS.md`). Params are async in *server* components; these are client components using `useParams()`, which stays synchronous. Do not convert them. Check `node_modules/next/dist/docs/` before relying on any other App Router behaviour from memory.
- Never run `git stash`. Stage only files you edited, by explicit path. **`CLAUDE.md` has uncommitted user changes — do not touch it.**

---

## The backend contract this builds on

All superadmin-scoped, all with the slug in the path, all live on `soylaika.backend`:

| Method | Path | Returns |
|---|---|---|
| GET | `/tenants/:slug/faq?active=&agent=&review_status=&source_type=` | `FaqChunk[]` — unpaginated, newest first |
| PATCH | `/tenants/:slug/faq/:id` | the updated chunk |
| DELETE | `/tenants/:slug/faq/:id` | soft delete |
| GET | `/tenants/:slug/faq/review?review_status=&source_type=&source_ref=` | queue items, oldest first, each with `source_span` **and `findings`** |
| GET | `/tenants/:slug/faq/review/counts` | `Record<string, number>` keyed by review status |
| POST | `/tenants/:slug/faq/:id/approve` | accepts `{question?, answer?, agents?, tags?}` only |
| POST | `/tenants/:slug/faq/:id/reject` | archives and deactivates |
| POST | `/tenants/:slug/faq/test` | `{ block, fired, topScore, chunkIds, prefilterHit, cacheHit, embedMs }` |
| PATCH | `/tenants/:slug` | now also accepts `faq_rag_enabled?: boolean` |
| GET | `/tenants/:slug/agents` | the tenant's agents, for the targeting selector |

Manual creation uses the tenant-context endpoint `POST /api/faq` with `reviewStatus: 'PENDING_REVIEW'` — **but that route is not superadmin-reachable**, which Task 4 addresses explicitly; read that task before assuming otherwise.

---

## File structure

| File | Responsibility |
|---|---|
| `lib/api.ts` | modify — `FaqChunk`/`FaqQueueItem` types, `api.faq.*`, `faq_rag_enabled` on `Tenant` and on `tenants.update` |
| `components/tenant-faq-section.tsx` | new — the card on the tenant page |
| `app/(crm)/superadmin/tenants/[slug]/page.tsx` | modify — render the card |
| `app/(crm)/superadmin/tenants/[slug]/faq/page.tsx` | new — list, filters, search, edit dialog, create |
| `app/(crm)/superadmin/tenants/[slug]/faq/revision/page.tsx` | new — review queue |
| `components/faq-tenant-banner.tsx` | new — the "which tenant am I editing" band, shared by both routes |

---

### Task 1: the API client

**Files:**
- Modify: `lib/api.ts`

**Interfaces:**
- Produces: `FaqChunk`, `FaqQueueItem`, `FaqRetrievalOutcome`, `api.faq.*`. Consumed by every later task.

- [ ] **Step 1: Add the types**

Near the other interfaces in `lib/api.ts`:

```ts
export interface FaqChunk {
  id: string;
  question: string;
  answer: string;
  agents: string[];
  tags: string[];
  active: boolean;
  review_status: "PENDING_REVIEW" | "APPROVED" | "ARCHIVED";
  source_type: "MANUAL" | "IMPORT" | "TEMPLATE" | "DOCUMENT";
  source_ref: string | null;
  source_ordinal: number | null;
  source_span: string | null;
  version: number;
  reviewed_by: string | null;
  reviewed_at: string | null;
  updated_by: string | null;
  embedded_at: string | null;
  created_at: string;
  updated_at: string;
}

/** Lo que devuelve la cola de revisión: el chunk más los hallazgos del lint en
 *  modo estricto, o sea exactamente lo que `approve()` va a rechazar. Vienen
 *  calculados en la misma respuesta, sin pedido extra. */
export interface FaqQueueItem extends FaqChunk {
  findings: { code: string; message: string; severity: "error" | "warning" }[];
}

/** Preview de retrieval. Ojo: `topScore` es UNO solo, no un score por chunk, y
 *  cuando `fired` es false ese número es el mejor puntaje que NO llegó al
 *  umbral — que es justo el dato que explica "lo aprobé y el bot lo ignora". */
export interface FaqRetrievalOutcome {
  block: string;
  fired: boolean;
  topScore: number | null;
  chunkIds: string[];
  prefilterHit: boolean;
  cacheHit: boolean;
  embedMs: number;
}
```

Confirm the `findings` element shape against `soylaika.backend/src/faq/faq-lint.ts`'s return type before finalising — if it differs, use the real shape and say so in your report. Do not guess.

- [ ] **Step 2: Extend `Tenant` and `tenants.update`**

Add to the `Tenant` interface:

```ts
  orchestrator_model: string | null;
  faq_rag_enabled: boolean;
```

(`orchestrator_model` is already accepted by `tenants.update` but missing from the type — an existing gap, fix it while here.)

And widen `update`'s data type:

```ts
    update: (slug: string, data: { name?: string; openrouter_api_key?: string; openrouter_model?: string; orchestrator_model?: string; faq_rag_enabled?: boolean }) =>
      req<Tenant>(`/tenants/${slug}`, { method: "PATCH", body: JSON.stringify(data) }),
```

- [ ] **Step 3: Add the `faq` namespace**

Inside the `api` object, after `tenants`:

```ts
  // Todo va con el slug en el PATH, no por header: un superadmin no tiene
  // contexto de tenant (getTenantSlug() devuelve null en el subdominio admin),
  // así que `/api/faq/*` resolvería contra la base master. Estas rutas son el
  // único camino soportado — ver PRD 3 §4.1.
  faq: {
    list: (slug: string, q?: { review_status?: string; source_type?: string; agent?: string; active?: boolean }) => {
      const p = new URLSearchParams();
      if (q?.review_status) p.set("review_status", q.review_status);
      if (q?.source_type) p.set("source_type", q.source_type);
      if (q?.agent) p.set("agent", q.agent);
      if (q?.active !== undefined) p.set("active", String(q.active));
      const qs = p.toString();
      return req<FaqChunk[]>(`/tenants/${slug}/faq${qs ? `?${qs}` : ""}`);
    },
    update: (slug: string, id: string, data: Partial<Pick<FaqChunk, "question" | "answer" | "agents" | "tags" | "active">>) =>
      req<FaqChunk>(`/tenants/${slug}/faq/${id}`, { method: "PATCH", body: JSON.stringify(data) }),
    remove: (slug: string, id: string) =>
      req<FaqChunk>(`/tenants/${slug}/faq/${id}`, { method: "DELETE" }),
    queue: (slug: string, q?: { review_status?: string; source_type?: string; source_ref?: string }) => {
      const p = new URLSearchParams();
      if (q?.review_status) p.set("review_status", q.review_status);
      if (q?.source_type) p.set("source_type", q.source_type);
      if (q?.source_ref) p.set("source_ref", q.source_ref);
      const qs = p.toString();
      return req<FaqQueueItem[]>(`/tenants/${slug}/faq/review${qs ? `?${qs}` : ""}`);
    },
    counts: (slug: string) => req<Record<string, number>>(`/tenants/${slug}/faq/review/counts`),
    approve: (slug: string, id: string, patch?: { question?: string; answer?: string; agents?: string[]; tags?: string[] }) =>
      req<FaqChunk>(`/tenants/${slug}/faq/${id}/approve`, { method: "POST", body: JSON.stringify(patch ?? {}) }),
    reject: (slug: string, id: string) =>
      req<FaqChunk>(`/tenants/${slug}/faq/${id}/reject`, { method: "POST" }),
    // El panel manda siempre PENDING_REVIEW: el alta manual es el único camino
    // de ingesta que históricamente se salteaba la revisión, y no hay razón
    // para que lo cargado desde acá sea la excepción.
    create: (slug: string, data: { question: string; answer: string; agents?: string[]; tags?: string[]; reviewStatus?: string }) =>
      req<FaqChunk>(`/tenants/${slug}/faq`, { method: "POST", body: JSON.stringify(data) }),
    test: (slug: string, message: string, agentType?: string) =>
      req<FaqRetrievalOutcome>(`/tenants/${slug}/faq/test`, { method: "POST", body: JSON.stringify({ message, agentType }) }),
  },
```

- [ ] **Step 4: Both gates**

```bash
npm run lint
npm run build
```

- [ ] **Step 5: Commit**

```bash
git add lib/api.ts
git commit -m "feat(api): add the tenant FAQ endpoints to the client"
```

---

### Task 2: the tenant banner

Small, shared, and worth its own task because both routes depend on it and it is the mitigation for the most expensive mistake in this feature.

**Files:**
- Create: `components/faq-tenant-banner.tsx`

**Interfaces:**
- Produces: `<FaqTenantBanner slug={string} tenantName={string | null} />`. Consumed by Tasks 3 and 5.

- [ ] **Step 1: Implement**

```tsx
"use client";

import Link from "next/link";
import { ArrowLeft, Database } from "lucide-react";

/**
 * Qué tenant estoy editando. Permanente y visible en las dos pantallas de FAQ.
 *
 * El error caro de un panel multi-tenant no es romper algo: es editar el
 * registro correcto en el tenant equivocado. Es silencioso, plausible, y puede
 * durar semanas — el bot del cliente equivocado empieza a contestar distinto y
 * nadie lo relaciona con esto. El slug ya viaja en la URL; esto lo pone también
 * donde el ojo está mirando (PRD 3 §9).
 */
export function FaqTenantBanner({ slug, tenantName }: { slug: string; tenantName: string | null }) {
  return (
    <div className="flex items-center gap-3 rounded-2xl border border-[#dee3e9] bg-[#f1f4f7] px-4 py-3 dark:border-white/[0.07] dark:bg-[#1f2024]">
      <Link
        href={`/superadmin/tenants/${slug}`}
        className="flex size-8 shrink-0 items-center justify-center rounded-lg bg-white text-[#5d6c7b] transition-colors hover:text-[#0a1317] dark:bg-white/[0.08] dark:text-[#8d9199] dark:hover:text-[#f0f2f5]"
        aria-label="Volver al tenant"
      >
        <ArrowLeft className="size-4" />
      </Link>
      <Database className="size-4 shrink-0 text-[#5d6c7b] dark:text-[#8d9199]" />
      <div className="min-w-0">
        <p className="truncate text-sm font-bold text-[#0a1317] dark:text-[#f0f2f5]">
          {tenantName ?? slug}
        </p>
        <p className="truncate font-mono text-xs text-[#8595a4]">
          Estás editando la base de conocimiento de <span className="font-bold">{slug}</span>
        </p>
      </div>
    </div>
  );
}
```

Check that `lucide-react` is the icon library this repo uses and that `ArrowLeft`/`Database` exist in it — read another component's imports rather than assuming.

- [ ] **Step 2: Both gates, then commit**

```bash
npm run lint && npm run build
git add components/faq-tenant-banner.tsx
git commit -m "feat(faq): add the banner naming the tenant being edited"
```

---

### Task 3: the entry card on the tenant page

**Files:**
- Create: `components/tenant-faq-section.tsx`
- Modify: `app/(crm)/superadmin/tenants/[slug]/page.tsx`

**Interfaces:**
- Consumes: `api.faq.counts`, `api.tenants.update` (Task 1).
- Produces: `<TenantFaqSection slug tenant onTenantChange />`.

- [ ] **Step 1: Implement the section**

Model it closely on `components/tenant-agents-section.tsx` — same card markup, same `useCallback` + `useEffect` load, same `toast.error(errorMessage(e, "…"))` handling. It shows:

- **Pendientes de revisión** as the prominent number. It is the only figure that implies an action.
- Total aprobadas.
- A **toggle** for `faq_rag_enabled` — not a read-only badge.
- A link to the list, and, when the pending count is above zero, a link straight to `/faq/revision` instead: send people where the work is.

The copy when retrieval is off matters and is not decoration:

```tsx
{!tenant?.faq_rag_enabled && (
  <p className="mt-2 text-xs leading-relaxed text-[#b4552d] dark:text-[#e0a06a]">
    El bot no está consultando estas respuestas. Activá la búsqueda para que las use.
  </p>
)}
```

A knowledge base the bot is not reading is the single most likely way this feature looks broken to someone who has just spent an hour curating it, and the flag defaults to `false` for every tenant.

The toggle calls `api.tenants.update(slug, { faq_rag_enabled: next })` and lifts the returned tenant up via `onTenantChange` so the parent page stays in sync. **Send only that one field** — the backend now allowlists this endpoint, but sending a whole tenant object back is the exact pattern that made the allowlist necessary.

- [ ] **Step 2: Render it on the tenant page**

In `page.tsx`, beside `<TenantAgentsSection slug={slug} />` (around line 919), add `<TenantFaqSection slug={slug} tenant={tenant} onTenantChange={setTenant} />`, using whatever the page's existing tenant state and setter are actually called — read them, do not assume.

- [ ] **Step 3: Both gates, then commit**

```bash
npm run lint && npm run build
git add components/tenant-faq-section.tsx "app/(crm)/superadmin/tenants/[slug]/page.tsx"
git commit -m "feat(faq): surface the knowledge base on the tenant page"
```

---

### Task 4: the list, the editor, and manual create

The largest task. Read the whole task before starting.

**Files:**
- Create: `app/(crm)/superadmin/tenants/[slug]/faq/page.tsx`

**Interfaces:**
- Consumes: `api.faq.list/update/remove`, `api.tenants.agents`, `FaqTenantBanner`.

- [ ] **Step 1: The page shell**

`"use client"`, `useParams()` for the slug, `useState` + `useEffect` for data, `FaqTenantBanner` at the top. Load in parallel: `api.faq.list(slug)`, `api.tenants.get?.(slug)` — check whether a single-tenant getter exists; if not, use `api.tenants.list()` and find by slug, and note it — and `api.tenants.agents(slug)` for the targeting selector.

- [ ] **Step 2: The table**

Columns: Pregunta · Respuesta · Estado · Agentes · Origen · Activo. Truncate question and answer to one line; the row expands in place to show the full pair. Scanning is the dominant activity.

Two renderings that are load-bearing, not cosmetic:

```tsx
// Un array vacío significa "todos los agentes", no "ninguno". La consulta de
// retrieval es `cardinality(agents) = 0 OR $agentType = ANY(agents)`, así que
// renderizar esto en blanco haría pensar que el chunk no lo usa nadie cuando
// en realidad lo usan todos.
{chunk.agents.length === 0 ? "Todos" : chunk.agents.join(", ")}
```

```tsx
// ARCHIVED se muestra atenuado pero NUNCA se saca de la lista: acá no se borra
// nada en duro, y la UI no debería sugerir lo contrario.
```

- [ ] **Step 3: Filters and search**

Filters on estado, agente, origen and activo — these map to the query params `api.faq.list` already accepts, so they refetch. **Search is client-side** over question and answer: there is no server-side search, and the endpoint returns the tenant's whole list unpaginated, so filtering in the browser is both possible and consistent with what the user sees. Normalise accents and case on both sides before comparing.

- [ ] **Step 4: The three empty states**

They look identical if handled carelessly and mean completely different things:

- **No hay contenido todavía** — offer the phase-2 entry points (import, packs, documento) as text; they do not exist yet, so do not link them.
- **Nada coincide con los filtros** — name the filter that is excluding everything, with a reset.
- **La petición falló.** Do not render a calm empty table. The backend deliberately distinguishes a real database error from "no hay nada" (`FaqReviewService.counts` catches and logs the missing-table case for exactly this reason); the UI owes the same distinction. A tenant that has not run the `FaqChunk` migration must not look like a tenant with no content.

- [ ] **Step 5: The edit dialog**

Use `Dialog`/`DialogContent`/`DialogTitle` from `@/components/ui/dialog` exactly as `tenant-agents-section.tsx` does — **there is no Drawer primitive in this repo**, and Dialog over the list serves the same purpose the PRD's "drawer" describes.

Editable: `question`, `answer`, `agents[]`, `tags[]`, `active`. Read-only and displayed: estado, versión, origen, `updated_by`, `reviewed_by`.

Three behaviours, all consequences of how the backend re-embeds:

1. **The save is not optimistic.** Changing question or answer triggers a synchronous embedding call inside the request. Disable the button, show a pending state, keep the dialog open, and only update the row from the response.
2. **A failed save saved nothing.** The backend embeds before it writes, so a provider error throws before any write — the row keeps its old text *and* its old vector. Say so and let them retry: *"No se guardó ningún cambio. Reintentá."*
3. **Editing an approved chunk publishes immediately**, because `updateOne` does not touch `review_status`. Show a line under the save button **only when the chunk is `APPROVED`**: *"Este cambio queda publicado al guardar."*

And state the cost honestly where it exists: an edit touching only agents/tags/active performs no embedding call at all. That is the difference between a free edit and a billed one.

Do **not** offer version history or a revert control. A manual edit overwrites in place with no superseded row — there is nothing to revert to, and offering it would be a lie.

- [ ] **Step 6: Manual create**

A "Nueva respuesta" button opening the same dialog in create mode, calling
`api.faq.create(slug, { question, answer, agents, tags, reviewStatus: "PENDING_REVIEW" })`.

**Always send `reviewStatus: "PENDING_REVIEW"`.** The endpoint defaults to `APPROVED` when the field is absent, preserving the historic behaviour of manual entry — but manual entry is the only ingest path that ever skipped review, and there is no reason for content typed into this panel to be the exception. It lands in the queue like everything else, and the person who typed it can approve it there.

On success, navigate to or link the review queue rather than silently returning to the list: something was created and it is *not* live yet, which the user needs to know.

**Do not** call `/api/faq` from this screen, and do not set `X-Tenant-Slug` by hand. `/api/faq` resolves the tenant from request context, which a superadmin does not have, so it would write to the **master** database. `POST /tenants/:slug/faq` exists precisely so that cannot happen.

- [ ] **Step 7: Both gates, then commit**

```bash
npm run lint && npm run build
git add "app/(crm)/superadmin/tenants/[slug]/faq/page.tsx"
git commit -m "feat(faq): list and edit a tenant's knowledge base"
```

---

### Task 5: the review queue

**Files:**
- Create: `app/(crm)/superadmin/tenants/[slug]/faq/revision/page.tsx`

**Interfaces:**
- Consumes: `api.faq.queue/approve/reject`, `FaqTenantBanner`.

- [ ] **Step 1: The screen**

Same shell as Task 4. One row per pending item, in the order the backend returns them — **do not re-sort.** It is sorted oldest-first deliberately, because it is a queue and not a listing.

- [ ] **Step 2: The two things that make this screen work**

**Show `source_span` beside the answer.** This is the whole job. Every chunk extracted from a document carries the verbatim excerpt it came from, and approving means reading the answer *against that excerpt* — not against how plausible it sounds. An LLM's fabrications are, by construction, plausible. Lay them out side by side, and make the approve control unreachable before both are visible.

**Render `findings` inline and block approval while any is an error.** The queue already computes them with the strict approval lint, so this costs no extra request. They are exactly what `approve()` will refuse. Showing them turns a confusing after-the-fact 400 into a visible precondition, and points at edit instead.

```tsx
{item.findings.length > 0 && (
  <ul className="mt-2 space-y-1">
    {item.findings.map((f) => (
      <li key={f.code} className="text-xs text-[#b4322c] dark:text-[#e08b85]">{f.message}</li>
    ))}
  </ul>
)}
```

- [ ] **Step 3: Approve and reject**

Approve posts to `api.faq.approve`, optionally with edits made in place. Reject archives and deactivates — confirm first, and **name the tenant in the confirmation**, not just the question.

Render the backend's 400 message verbatim when approval is refused; it names the rule that was broken.

**No batch approve, deliberately.** The extraction pipeline drops unsupported candidates rather than flagging them precisely because flagged-but-present content gets bulk-approved by a tired reviewer. A "select all" hands that failure straight back.

Filters by `source_type` and `source_ref` are worth having: one document extraction can deposit up to 60 candidates, and reviewing them scoped to that `source_ref` is the natural unit of work.

- [ ] **Step 4: Both gates, then commit**

```bash
npm run lint && npm run build
git add "app/(crm)/superadmin/tenants/[slug]/faq/revision/page.tsx"
git commit -m "feat(faq): review queue with the source excerpt and blocking findings"
```

---

### Task 6: retrieval preview

**Files:**
- Modify: `app/(crm)/superadmin/tenants/[slug]/faq/page.tsx`

- [ ] **Step 1: Implement**

A panel on the list screen: a message input, an agent selector (defaulting to `default`, since the endpoint does), and a "Probar" button calling `api.faq.test(slug, message, agentType)`.

Render, in this order of prominence:

1. **Did it fire.** `fired: true` → it would inject; `false` → it would not.
2. **The score against the threshold.** When `fired` is false, `topScore` is *the best score that failed the threshold* — that is the actual answer to "lo aprobé y el bot lo ignora". Show it as e.g. *"el mejor match dio 0.74; el umbral es 0.78"*. The threshold is `FAQ_RETRIEVAL_THRESHOLD`, default `0.78`, and is not exposed by the API — state it as the default and label it as such.
3. **Which chunks**, by resolving `chunkIds` against the list already loaded on this page.
4. The raw `block`, collapsed.

Be careful not to overpromise: the endpoint returns **one** `topScore`, not a score per chunk, and ids rather than text. A ranked table with a score per row is not available without a backend change — do not fake one by guessing.

`fired: false` with `topScore: null` is a **different** failure from a low score: nothing was retrieved at all before scoring. Say so — it means an empty knowledge base, a null embedding, or agent targeting excluding everything.

- [ ] **Step 2: Both gates, then commit**

```bash
npm run lint && npm run build
git add "app/(crm)/superadmin/tenants/[slug]/faq/page.tsx"
git commit -m "feat(faq): preview what the bot would retrieve for a message"
```

---

## Self-review notes

**Spec coverage of PRD 3 phase 1:** §6.1 is Task 3; §6.2 and §6.3 are Task 4; §6.4 is Task 5; §6.5's empty/error states are Task 4 Step 4; §7.2's agent targeting is Task 4 Step 2; §8 is Task 1; §9 is Task 2 plus the slug-in-path shape of every call; the retrieval preview promoted into phase 1 by §10 is Task 6.

**Known gaps, stated rather than hidden:**
- **Manual create needed a backend route that phase 0 had missed**, found while writing this plan: `POST /api/faq` is tenant-context and a superadmin has none, so the panel had nothing to call. The PRD listed create in phase 1 assuming the endpoint was reachable; it was not. `POST /tenants/:slug/faq` was added to `soylaika.backend` (commit `c79dcea`) before this plan was executed, so Task 4 Step 6 is unblocked.
- No pagination, by design — PRD 3 §6.5 explains why, and client-side search depends on it.
- Document extraction, CSV import and starter-pack UI are phase 2.
- There is no test suite here, so none of this is covered by automated assertions. The gates are lint and a type-checking build; correctness of behaviour rests on review.
