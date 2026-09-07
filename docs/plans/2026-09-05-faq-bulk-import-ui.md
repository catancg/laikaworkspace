# FAQ bulk import UI (PRD 4 phase A/B, frontend) — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upload a file of FAQ entries for a tenant, see exactly which rows will go in and which will not **before anything is written**, confirm, and undo the batch if it was wrong.

**Architecture:** One new route under the tenant's FAQ area, plus a multipart upload helper in `lib/api.ts`. The backend already does the work: it can analyse a file without writing it, import it, and withdraw a batch.

**Tech Stack:** Next.js 16.2.6 (App Router, client components), React 19, Tailwind v4, shadcn primitives, sonner.

**Spec:** `f:\docs\Laika\docs\prd-faq-bulk-import.md` §6.1–6.5. Backend is complete and merged on `soylaika.backend`'s `RAG-enablement` branch.

## Global Constraints

- **UI copy and code comments in Spanish (es-AR, voseo).**
- **Everything is `"use client"`**, fetching in `useEffect` with local `loading`/error state. No server components, no server actions.
- **`lib/api.ts` is the only file that may call `fetch`.**
- **Styling: hand-written arbitrary hex Tailwind in light/dark pairs** (`text-[#0a1317] dark:text-[#f0f2f5]`). Not shadcn token classes — this repo deliberately does not use them on CRM screens.
- **No test suite in this repo.** Gates are `npm run lint` and `npm run build` (which type-checks).
- **Lint baseline is 10 errors / 3 warnings**, pre-existing, in 8 files this work does not touch. Your gate is that the count does not rise. Do not fix them.
- **Do not use the house `useEffect(() => { setLoading(true); … })` shape** — it is what most of those errors are. Use `useState(true)` + a `cancelled` guard, as `app/(crm)/superadmin/tenants/[slug]/faq/page.tsx` already does.
- Never run `git stash`. Stage only files you edited. **`CLAUDE.md` has uncommitted user changes — do not touch it.**

---

## The backend contract

All superadmin, slug in the path:

| Method | Path | Notes |
|---|---|---|
| POST | `/tenants/:slug/faq/import?dryRun=true` | multipart `file`. Analyses, **writes nothing**. Returns `{ total, willImport, rejected[], warnings[], chunks[] }` |
| POST | `/tenants/:slug/faq/import` | same upload, writes. Returns `{ batchId, total, imported, rejected[], warnings[] }` |
| POST | `/tenants/:slug/faq/import/:batchId/withdraw` | archives the batch. Returns `{ withdrawn: number }` |
| GET | `/api/faq/import/template` | static CSV. Needs no tenant — works for a superadmin as-is |

`rejected[]` items are `{ row, question, answer, findings[] }`; `findings[]` are `{ rule, severity, message }` with `rule` one of `vacio | largo | precio | imperativo | placeholder`.

---

### Task 1: the upload helper and API calls

**Files:** Modify `lib/api.ts`

**Interfaces:** Produces `api.faq.importPreview/importFile/withdrawImport/downloadTemplate` and the `ImportAnalysis` / `ImportResult` types. Consumed by Task 2.

- [ ] **Step 1: Add a multipart helper**

`req()` sets `Content-Type: application/json`, which breaks multipart — the browser must set its own boundary. There is one existing precedent, `api.products.import`, but **do not copy it wholesale**: it throws `new Error(\`Import error ${res.status}\`)` and discards the backend's message. This feature depends on that message (it names the rejected rows and why), so the helper mirrors `req()`'s error extraction instead.

```ts
// Subida multipart. Aparte de `req()` por dos motivos:
//  - No se setea Content-Type: lo pone el browser con su propio boundary.
//  - Estas rutas llevan el slug en el PATH, asi que NO va X-Tenant-Slug. Un
//    superadmin no tiene contexto de tenant; mandar el header no serviria y
//    pedirlo sugeriria que hace falta.
// El manejo de error copia el de `req()` a proposito: el backend contesta
// diciendo que filas rechazo y por que, y perder ese mensaje vaciaria la
// pantalla de todo lo util.
async function reqUpload<T>(path: string, file: File): Promise<T> {
  const token = getToken();
  const form = new FormData();
  form.append("file", file);

  const res = await fetch(`${BASE}${path}`, {
    method: "POST",
    headers: { ...(token ? { Authorization: `Bearer ${token}` } : {}) },
    body: form,
  });

  if (res.status === 401) {
    if (typeof window !== "undefined" && window.location.pathname !== "/login") {
      localStorage.removeItem("asap_token");
      localStorage.removeItem("asap_user");
      document.cookie = "asap_token=; path=/; max-age=0";
      window.location.href = "/login";
    }
    throw new Error("No autorizado");
  }
  if (!res.ok) {
    let msg = `API error ${res.status}`;
    try {
      const body = await res.json();
      const m = body?.message;
      if (Array.isArray(m)) msg = m.join(", ");
      else if (typeof m === "string") msg = m;
    } catch { /* sin body JSON */ }
    throw new Error(msg);
  }
  return res.json();
}
```

Read `req()` before writing this and mirror its 401 handling exactly — if it has diverged from the sketch above, follow the real one.

- [ ] **Step 2: Add the types**

```ts
export interface FaqLintFinding {
  rule: "vacio" | "largo" | "precio" | "imperativo" | "placeholder";
  severity: "error" | "warning";
  message: string;
}

export interface FaqImportRejectedRow {
  row: number;
  question: string;
  answer: string;
  findings: FaqLintFinding[];
}

/** Vista previa: lo que PASARIA. No escribe nada. */
export interface FaqImportAnalysis {
  total: number;
  willImport: number;
  rejected: FaqImportRejectedRow[];
  warnings: { row: number; findings: FaqLintFinding[] }[];
}

/** Resultado real de una importacion confirmada. */
export interface FaqImportResult {
  batchId: string;
  total: number;
  imported: number;
  rejected: FaqImportRejectedRow[];
  warnings: { row: number; findings: FaqLintFinding[] }[];
}
```

`FaqImportAnalysis` deliberately omits the backend's `chunks[]`: the screen never needs the normalised rows, and typing a field nothing reads invites someone to render it.

- [ ] **Step 3: Add the calls to the `faq` namespace**

```ts
    importPreview: (slug: string, file: File) =>
      reqUpload<FaqImportAnalysis>(`/tenants/${slug}/faq/import?dryRun=true`, file),
    importFile: (slug: string, file: File) =>
      reqUpload<FaqImportResult>(`/tenants/${slug}/faq/import`, file),
    withdrawImport: (slug: string, batchId: string) =>
      req<{ withdrawn: number }>(`/tenants/${slug}/faq/import/${encodeURIComponent(batchId)}/withdraw`, { method: "POST" }),
```

`batchId` is `import:<uuid>` — it contains a colon, so it **must** be `encodeURIComponent`'d or the path breaks.

- [ ] **Step 4: Template download**

The template endpoint sits behind `JwtAuthGuard`, so a plain `<a href>` will not authenticate. Fetch it with the token and hand the browser a blob:

```ts
    // El endpoint pide Authorization, asi que un <a href> pelado da 401. Se
    // baja con token y se entrega como blob.
    downloadTemplate: async () => {
      const token = getToken();
      const res = await fetch(`${BASE}/api/faq/import/template`, {
        headers: { ...(token ? { Authorization: `Bearer ${token}` } : {}) },
      });
      if (!res.ok) throw new Error("No se pudo descargar la plantilla");
      return res.blob();
    },
```

- [ ] **Step 5: Both gates, then commit**

```bash
npm run lint      # must stay at 10 errors / 3 warnings
npm run build
git add lib/api.ts
git commit -m "feat(api): add the FAQ import preview, upload, withdraw and template calls"
```

---

### Task 2: the import screen

**Files:** Create `app/(crm)/superadmin/tenants/[slug]/faq/importar/page.tsx`

**Interfaces:** Consumes Task 1's calls, `FaqTenantBanner`.

- [ ] **Step 1: Shell**

`"use client"`, `useParams()` for the slug, `FaqTenantBanner` at the top, tenant name via `api.tenants.list().then(l => l.find(t => t.slug === slug) ?? null)` — there is no single-tenant getter. Model the shell, styling and states on the sibling `faq/page.tsx`.

- [ ] **Step 2: The template, shown before anything is chosen**

PRD §6.1. Before a file is selected the page shows **what the file should look like**, so nobody has to download it to find out:

- The four columns rendered as a small table — `pregunta`, `respuesta`, `agentes`, `tags` — with one filled example row.
- Which are required (`pregunta`, `respuesta`) and which are optional.
- A **download** button calling `api.faq.downloadTemplate()` and saving the blob as `plantilla-faq.csv`.
- **The price rule, stated as an instruction, not as a future error:**

```tsx
<p>
  Poné la política, no el número: <b>&quot;hay descuento por transferencia&quot;</b> sí,
  <b>&quot;20% de descuento&quot;</b> no. Los precios se desactualizan y terminan contradiciendo
  al bot, así que las filas con importes se rechazan.
</p>
```

That paragraph is the highest-value copy on the page. The lint rejecting prices is the safety net; saying it here is what stops twenty rows bouncing.

Also state plainly that everything imported lands **pending review** and is not visible to the bot until approved.

- [ ] **Step 3: Choose file → preview**

A file input (`accept=".csv,text/csv"`), then a **Analizar** button calling `api.faq.importPreview(slug, file)`.

Nothing is written by this call. Say so on the button's surrounding copy — the user is about to upload a file to a system that will not change anything yet, and that is the whole point of the screen.

- [ ] **Step 4: The preview result**

- **`willImport` of `total`** as the headline figure.
- **Every rejected row**: its row number, a truncated question, and each finding's `message` verbatim. The backend messages are already written for a person; do not paraphrase them.
- **Warnings**, visually secondary — they do not block.
- **The promise, stated:** if a row passes here it will pass review, with one exception. Worth saying because it is what makes the preview worth reading:

```tsx
<p>
  Lo que pasa esta revisión pasa también la aprobación. La única excepción es un
  <code>{"{{"}texto{"}}"}</code> sin resolver: acá avisa, y al aprobar bloquea.
</p>
```

- A **Confirmar e importar** button, disabled when `willImport === 0`.

- [ ] **Step 5: Confirm → result**

Calls `api.faq.importFile(slug, file)` with the **same `File` object** still held in state.

**Two things this step must get right, both from PRD §6.3:**

1. **The file is uploaded a second time.** Preview and confirm are separate requests and nothing carries the parsed file between them. That is the accepted design — it keeps the file as the source of truth — but the user should not be surprised by a second upload of a large file. No special handling needed; just do not disable the input in a way that loses the `File`.

2. **Report what actually landed, not what was promised.** The import is not transactional: `upsertBatch` writes row by row, and a mid-write failure leaves part of the batch in place. So render `imported` from the *response*, never the `willImport` from the preview. If the call throws, the backend's message names the `batchId` — surface it, and offer the withdraw action, because that message is currently the only way to find the partial rows.

On success: the count, the `batchId`, a way into the review queue, and a **Deshacer esta carga** button.

**About that link — I checked, so you do not have to.** The review queue keeps `sourceRef` in local state fed by an input; it does **not** read `source_ref` from the URL (no `useSearchParams` anywhere in that file). So `/faq/revision?source_ref=<batchId>` would be silently ignored — the user would land on the unfiltered queue believing it was scoped.

**Do not add URL-param support to the queue in this task.** In Next 16 `useSearchParams` forces client-side rendering up to the nearest `Suspense` boundary on a prerendered route, which is a real consideration and a third file to touch for a convenience.

Instead: show the `batchId` with a **copy** button, link to the queue plainly, and say in one line that pasting it into the queue's origin filter scopes the view to this batch. It uses the filter that already exists, it is honest about what the link does, and it costs nothing.

- [ ] **Step 6: Withdraw**

`api.faq.withdrawImport(slug, batchId)`, behind a confirmation that names the **tenant** and how many rows will be archived. Say that it archives rather than deletes — nothing here is hard-deleted, and a batch that was approved and answering customers is exactly the history the retention design keeps.

- [ ] **Step 7: Both gates, then commit**

```bash
npm run lint && npm run build
git add "app/(crm)/superadmin/tenants/[slug]/faq/importar/page.tsx"
git commit -m "feat(faq): upload a batch of answers, preview it, and undo it"
```

---

### Task 3: reach it from the list

**Files:** Modify `app/(crm)/superadmin/tenants/[slug]/faq/page.tsx`

- [ ] **Step 1: Add the entry point**

Beside the existing **Nueva respuesta** button, a secondary **Importar archivo** linking to `/superadmin/tenants/${slug}/faq/importar`.

**Add to the existing header block; do not restructure it.** That file already carries the list, filters, search, the edit dialog, create, and the retrieval preview — a rewrite risks silently dropping one.

- [ ] **Step 2: Both gates, then commit**

```bash
npm run lint && npm run build
git add "app/(crm)/superadmin/tenants/[slug]/faq/page.tsx"
git commit -m "feat(faq): link the bulk import from the knowledge base list"
```

---

## Self-review notes

**Spec coverage:** PRD 4 §6.1 (template in the product) is Task 2 step 2; §6.3 (preview) is steps 3–4; §6.4 (after confirming) is step 5; §6.5 (withdraw) is step 6. Phase A's route and phase B's `dryRun` are both already built server-side.

**Deliberately not here:**
- **No XLSX.** Blocked on PRD 4 §13 question 1. The input accepts `.csv` only, and the copy should say so rather than accept a file the backend will reject.
- **No tenant-facing surface.** Phases D/F/G, two of them blocked on product decisions.
- **No in-screen fixing of a rejected row.** PRD §13 question 5 leaves it open and prefers correcting the file — which keeps the file as the source of truth.

**Known risk:** Task 3 edits the largest file in this feature. The instruction is to add to the header, not restructure — the same hazard that applied when the retrieval preview was added to that page.
