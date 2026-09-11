# PRD 10 Phase 3 — Business info review (UI) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the customer a form that submits a *proposal* and tells them where it stands, and give SoyLaika a screen to approve or reject it.

**Architecture:** Pure client work against Phase 2's endpoints. The customer's Ajustes card keeps its form but gains pending/rejected banners. Superadmins get a `TenantBusinessSection` on the tenant detail page linking to a `negocio/` screen that diffs draft against live, mirroring how `tenant-faq-section.tsx` links to `faq/revision`.

**Tech Stack:** Next.js 16.2.6 App Router, all `"use client"`, Tailwind with inline hex light/dark pairs, `sonner` for toasts.

**Spec:** [docs/prd-customer-agent-config-boundary.md](../prd-customer-agent-config-boundary.md) — §3.3, §3.5, §4.6, §5

**Depends on:** [Phase 2](2026-09-09-prd10-business-review-backend.md) must be merged and deployed first. Every endpoint here 404s without it.

## Global Constraints

- **Stage explicit paths.** Never `git add -A`, `git add .`, or `git stash` — this repo carries an uncommitted local `CLAUDE.md` edit that is not recoverable from the remote, and an untracked, non-git-ignored `.superpowers/` scratch directory.
- **`npm run lint` baseline is 13 problems (10 errors, 3 warnings) and must not grow.** Record it before the first edit and report it after every task. There is no test suite in this repo — the type-check, the lint count and diff reading are the whole gate.
- **Never `npm run build`** — dev servers are live on :3000 and :3001. Type-check with `npx tsc --noEmit`.
- **`package.json` pins `next@16.2.6`**, well past most training data. Check `node_modules/next/dist/docs/` before relying on remembered App Router or caching semantics.
- **UI copy is Spanish (es-AR, voseo)** — "podés", "querés", "enviá".
- **Styling outside `components/ui/` uses inline arbitrary hex Tailwind classes in light/dark pairs** (`text-[#0a1317] dark:text-[#f0f2f5]`), never shadcn CSS-variable tokens.
- **Do not set state synchronously inside a `useEffect` body** — `react-hooks/set-state-in-effect` is the rule the baseline is made of, and Phase 1 already paid for this. Fetch in the effect and set state in the async callbacks, guarded by an `alive` flag; reset per-identity state with a `key` prop rather than an effect.
- **Never call a destructive action without `confirm()`.** Every destructive action on the superadmin tenant page confirms first; Phase 1 shipped a regression here and had to fix it.

## File Structure

| File | Responsibility |
|---|---|
| `lib/api.ts` | `BusinessDraft` type; `api.business.draft`; `api.tenants.business.*` |
| `app/(crm)/settings/page.tsx` | customer card: proposal semantics + pending/rejected banners |
| `components/tenant-business-section.tsx` | create — superadmin entry point on the tenant detail page |
| `app/(crm)/superadmin/tenants/[slug]/page.tsx` | mount the section |
| `app/(crm)/superadmin/tenants/[slug]/negocio/page.tsx` | create — the review screen |
| `app/(crm)/superadmin/tenants/page.tsx` | pending badge on the tenant list |

---

### Task 1: API client

**Files:**
- Modify: `soylaika.frontend/lib/api.ts`

**Interfaces:**
- Consumes: Phase 2's endpoints.
- Produces, for every later task: `BusinessDraft`, `api.business.draft()`, `api.tenants.business.{get,approve,reject,pendingCount}`.

- [ ] **Step 1: Record the lint baseline**

Run: `npm run lint` and write the number down. It must not grow across this plan.

- [ ] **Step 2: Add the type**

In `lib/api.ts`, next to the existing `Business` interface:

```ts
// Propuesta del cliente esperando revision de SoyLaika (PRD 10 §3.1). Mismos
// campos que Business, mas el estado de revision. `ARCHIVED` = rechazada.
export interface BusinessDraft extends Business {
  id: string;
  review_status: "DRAFT" | "PENDING_REVIEW" | "APPROVED" | "ARCHIVED";
  submitted_by: string | null;
  submitted_at: string;
  reviewed_by: string | null;
  reviewed_at: string | null;
  reject_note: string | null;
}
```

- [ ] **Step 3: Extend the `business` namespace**

Replace the existing `business:` block:

```ts
  business: {
    // Lo aprobado — lo que usa el bot.
    get: () => req<Business | null>("/api/business"),
    // Para un admin del tenant esto NO escribe lo que usa el bot: crea o
    // actualiza un borrador que SoyLaika tiene que aprobar (PRD 10 §3.3). Para un
    // superadmin escribe directo. La respuesta es el borrador o el perfil segun
    // el rol de quien llama.
    upsert: (data: Partial<Business>) =>
      req<Business | BusinessDraft>("/api/business", { method: "PUT", body: JSON.stringify(data) }),
    // El borrador propio del tenant, con la nota si se lo rechazaron.
    draft: () => req<BusinessDraft | null>("/api/business/draft"),
  },
```

- [ ] **Step 4: Add the superadmin namespace**

Inside `tenants:`, after the Phase 1 `rules` block:

```ts
    // Revision de la info del negocio de un tenant (PRD 10 §3.3).
    business: {
      get: (slug: string) =>
        req<{ live: Business | null; draft: BusinessDraft | null }>(`/tenants/${slug}/business`),
      approve: (slug: string) =>
        req<Business>(`/tenants/${slug}/business/approve`, { method: "POST" }),
      reject: (slug: string, note?: string) =>
        req<BusinessDraft>(`/tenants/${slug}/business/reject`, {
          method: "POST",
          body: JSON.stringify({ note: note ?? "" }),
        }),
      // slug -> cuantos cambios esperan. Solo trae los que tienen pendientes.
      pendingCount: () => req<Record<string, number>>("/tenants/business/pending-count"),
    },
```

- [ ] **Step 5: Type-check, lint and commit**

Run: `npx tsc --noEmit` → clean.
Run: `npm run lint` → at or below baseline.

```bash
git add lib/api.ts
git commit -m "feat(negocio): cliente de API para la revision de info del negocio (PRD 10 fase 3)"
```

---

### Task 2: Customer card submits a proposal

**Files:**
- Modify: `soylaika.frontend/app/(crm)/settings/page.tsx`

**Interfaces:**
- Consumes: `api.business.get`, `api.business.upsert`, `api.business.draft`, `BusinessDraft`.

The card keeps its `BUSINESS_FIELDS`-driven form. What changes is what saving *means* and what the customer is told afterwards. Without the banner, a customer whose edit silently doesn't apply retypes it and opens a support ticket.

- [ ] **Step 1: Load the draft alongside the profile**

Add state beside the existing `business` / `loadingBiz` / `savingBiz`:

```ts
  const [draft, setDraft] = useState<BusinessDraft | null>(null);
```

Import `type BusinessDraft` from `@/lib/api`.

In the existing effect that calls `api.business.get()`, fetch both and prefill the form from the draft when one exists — the customer should see what they last submitted, not a stale copy of live. Set state only in async callbacks:

```ts
  useEffect(() => {
    let alive = true;
    Promise.all([
      api.business.get(),
      api.business.draft().catch(() => null),
    ])
      .then(([live, d]) => {
        if (!alive) return;
        if (d) setDraft(d);
        const source = d ?? live;
        if (source) setBusiness({ ...EMPTY_BUSINESS, ...source });
      })
      .catch(() => { /* la tarjeta ya muestra vacio */ })
      .finally(() => { if (alive) setLoadingBiz(false); });
    return () => { alive = false; };
  }, []);
```

Replace the existing `api.business.get()` effect entirely — do not leave both.

- [ ] **Step 2: Make saving submit a proposal**

Replace `saveBusiness`:

```ts
  async function saveBusiness() {
    setSavingBiz(true);
    try {
      const result = await api.business.upsert(business);
      // Un admin del tenant recibe el borrador; un superadmin, el perfil vivo.
      if (result && "review_status" in result) {
        setDraft(result as BusinessDraft);
        toast.success("Cambios enviados. SoyLaika los va a revisar.");
      } else {
        setDraft(null);
        setBusiness({ ...EMPTY_BUSINESS, ...(result as Business) });
        toast.success("Información del negocio guardada");
      }
    } catch (e) {
      toast.error(e instanceof Error ? e.message : "No se pudieron guardar los cambios");
    } finally {
      setSavingBiz(false);
    }
  }
```

If the file does not already import `toast` from `sonner`, add it and match how other handlers on this page report errors.

- [ ] **Step 3: Add the banners**

Directly under the card's heading and description, before the form fields:

```tsx
          {draft?.review_status === "PENDING_REVIEW" && (
            <div className="mb-4 rounded-2xl border border-[#f2a918]/30 bg-[#fef3e2] p-3 text-xs text-[#7a5a12] dark:border-[#f2a918]/25 dark:bg-[#f2a918]/[0.12] dark:text-[#f2a918]">
              Tus cambios están esperando la aprobación de SoyLaika. Hasta que se aprueben, el bot
              sigue usando la información anterior.
            </div>
          )}

          {draft?.review_status === "ARCHIVED" && (
            <div className="mb-4 rounded-2xl border border-[#e41e3f]/25 bg-[#fde8ec] p-3 text-xs text-[#a11226] dark:border-[#e41e3f]/25 dark:bg-[#e41e3f]/[0.12] dark:text-[#f87171]">
              <p className="font-bold">SoyLaika no aplicó tus últimos cambios.</p>
              {draft.reject_note && <p className="mt-1">Motivo: {draft.reject_note}</p>}
              <p className="mt-1">Podés corregirlos y enviarlos de nuevo.</p>
            </div>
          )}
```

- [ ] **Step 4: Make the button say what it does**

The save button currently reads "Guardar". For a tenant admin it now sends a proposal, so label it accordingly — reuse the page's existing `isAdmin` to distinguish:

```tsx
                  {savingBiz ? <Loader2 className="w-4 h-4 animate-spin" /> : <Save className="w-4 h-4" />}
                  {isAdmin ? "Enviar cambios" : "Guardar"}
```

Update the card's description line to say the same thing in one sentence: that the information is reviewed by SoyLaika before the bot uses it.

- [ ] **Step 5: Type-check, lint, self-review**

Run: `npx tsc --noEmit` → clean.
Run: `npm run lint` → at or below baseline. If a new `set-state-in-effect` appears, fix the effect rather than adding an `eslint-disable`.

Read your own diff: is there exactly one effect loading business data, does the form still render every `BUSINESS_FIELDS` entry, and does a failed request leave `savingBiz` stuck on?

- [ ] **Step 6: Commit**

```bash
git add "app/(crm)/settings/page.tsx"
git commit -m "feat(negocio): el cliente propone cambios y ve en que estado quedaron (PRD 10 fase 3)"
```

---

### Task 3: Superadmin review screen

**Files:**
- Create: `soylaika.frontend/components/tenant-business-section.tsx`
- Modify: `soylaika.frontend/app/(crm)/superadmin/tenants/[slug]/page.tsx`
- Create: `soylaika.frontend/app/(crm)/superadmin/tenants/[slug]/negocio/page.tsx`

**Interfaces:**
- Consumes: `api.tenants.business.{get,approve,reject}`, `Business`, `BusinessDraft`.
- Produces: `<TenantBusinessSection slug={string} />`, and the route `/superadmin/tenants/[slug]/negocio`.

**Read first:** `components/tenant-faq-section.tsx`. It is the exact pattern — a compact section on the tenant detail page that links out to a dedicated review screen (`faq/revision`). Match its shell: a `<section>` with `overflow-hidden rounded-2xl border … bg-white`, a header row with `border-b … px-5 py-4`, and a `size-10 rounded-xl bg-[#e8f0fe] text-[#0064e0]` icon badge. Phase 1 shipped a section using the wrong shell and had to fix it — do not repeat that.

- [ ] **Step 1: Create the entry-point section**

Create `components/tenant-business-section.tsx`: a small client component that fetches `api.tenants.business.get(slug)`, shows whether a change is pending (and who submitted it, when), and renders a `Link` to `/superadmin/tenants/${slug}/negocio` labelled "Revisar cambios" when a draft is pending or "Ver info del negocio" when it is not.

Follow `tenant-faq-section.tsx` for the shell, the `errorMessage` helper, and the link styling. Fetch with the `alive`-guard pattern:

```tsx
  useEffect(() => {
    let alive = true;
    api.tenants.business
      .get(slug)
      .then((r) => { if (alive) setData(r); })
      .catch((e) => { if (alive) toast.error(errorMessage(e, "No se pudo cargar la info del negocio")); })
      .finally(() => { if (alive) setLoading(false); });
    return () => { alive = false; };
  }, [slug]);
```

- [ ] **Step 2: Mount it**

In `app/(crm)/superadmin/tenants/[slug]/page.tsx`, import it beside the other section imports and render it as a bare sibling with a `key`, next to `<TenantRulesSection key={slug} slug={slug} />`:

```tsx
            <TenantBusinessSection key={slug} slug={slug} />
```

The `key` is not decoration: without it, switching tenants leaves the previous tenant's data on screen until the refetch resolves. Phase 1 hit exactly this.

- [ ] **Step 3: Create the review screen**

Create `app/(crm)/superadmin/tenants/[slug]/negocio/page.tsx`:

```tsx
"use client";

import { useEffect, useState } from "react";
import { useParams } from "next/navigation";
import Link from "next/link";
import { ArrowLeft, Check, Loader2, X } from "lucide-react";
import { toast } from "sonner";
import { api, type Business, type BusinessDraft } from "@/lib/api";

// Mismos doce campos y las mismas etiquetas que arma el prompt del bot
// (ai.service.ts buildBusinessBlock). Si agregás uno alla, agregalo aca.
const CAMPOS: { key: keyof Business; label: string }[] = [
  { key: "name", label: "Nombre del negocio" },
  { key: "about", label: "Sobre el negocio" },
  { key: "hours", label: "Horarios de atención" },
  { key: "address", label: "Dirección" },
  { key: "branches", label: "Sucursales" },
  { key: "phone", label: "Teléfono de contacto" },
  { key: "email", label: "Email" },
  { key: "website", label: "Sitio web" },
  { key: "paymentMethods", label: "Medios de pago" },
  { key: "shippingInfo", label: "Envíos" },
  { key: "returnPolicy", label: "Cambios y devoluciones" },
  { key: "extra", label: "Otros datos" },
];

function errorMessage(error: unknown, fallback: string) {
  return error instanceof Error ? error.message : fallback;
}

function valor(v: string | null | undefined) {
  const t = (v ?? "").trim();
  return t.length ? t : null;
}

export default function TenantBusinessReviewPage() {
  const params = useParams<{ slug: string }>();
  const slug = params.slug;

  const [live, setLive] = useState<Business | null>(null);
  const [draft, setDraft] = useState<BusinessDraft | null>(null);
  const [loading, setLoading] = useState(true);
  const [busy, setBusy] = useState(false);
  const [note, setNote] = useState("");

  useEffect(() => {
    let alive = true;
    api.tenants.business
      .get(slug)
      .then((r) => { if (alive) { setLive(r.live); setDraft(r.draft); } })
      .catch((e) => { if (alive) toast.error(errorMessage(e, "No se pudo cargar la info del negocio")); })
      .finally(() => { if (alive) setLoading(false); });
    return () => { alive = false; };
  }, [slug]);

  async function approve() {
    if (busy) return;
    if (!confirm("¿Aprobar estos cambios? El bot los va a usar a partir de ahora.")) return;
    setBusy(true);
    try {
      const updated = await api.tenants.business.approve(slug);
      setLive(updated);
      setDraft(null);
      setNote("");
      toast.success("Cambios aprobados");
    } catch (e) {
      toast.error(errorMessage(e, "No se pudieron aprobar los cambios"));
    } finally {
      setBusy(false);
    }
  }

  async function reject() {
    if (busy) return;
    setBusy(true);
    try {
      const updated = await api.tenants.business.reject(slug, note.trim() || undefined);
      setDraft(updated);
      toast.success("Cambios rechazados");
    } catch (e) {
      toast.error(errorMessage(e, "No se pudieron rechazar los cambios"));
    } finally {
      setBusy(false);
    }
  }

  const pendiente = draft?.review_status === "PENDING_REVIEW";

  return (
    <div className="p-6">
      <Link
        href={`/superadmin/tenants/${slug}`}
        className="mb-4 inline-flex items-center gap-2 text-sm text-[#5d6c7b] hover:text-[#0a1317] dark:text-[#8d9199] dark:hover:text-[#f0f2f5]"
      >
        <ArrowLeft className="size-3.5" />
        Volver al cliente
      </Link>

      <h1 className="mb-1 text-lg font-bold text-[#0a1317] dark:text-[#f0f2f5]">Info del negocio</h1>
      <p className="mb-5 text-xs text-[#5d6c7b] dark:text-[#8d9199]">
        Lo que el bot dice como verdad sobre este negocio. La columna de la izquierda es lo que usa
        hoy; la de la derecha, lo que propuso el cliente.
      </p>

      {loading ? (
        <div className="flex justify-center py-12">
          <Loader2 className="size-5 animate-spin text-[#8595a4]" />
        </div>
      ) : (
        <>
          {!pendiente && (
            <div className="mb-4 rounded-2xl border border-[#dee3e9] bg-[#f1f4f7] p-3 text-xs text-[#5d6c7b] dark:border-white/[0.07] dark:bg-white/[0.04] dark:text-[#8d9199]">
              {draft?.review_status === "ARCHIVED"
                ? "El último cambio propuesto fue rechazado. No hay nada pendiente."
                : "No hay cambios esperando revisión."}
            </div>
          )}

          <section className="overflow-hidden rounded-2xl border border-[#dee3e9] bg-white dark:border-white/[0.07] dark:bg-[#18191c]">
            <div className="grid grid-cols-[1fr_1fr] gap-4 border-b border-[#dee3e9] px-5 py-3 text-xs font-bold text-[#5d6c7b] dark:border-white/[0.07] dark:text-[#8d9199]">
              <div>En uso</div>
              <div>Propuesto</div>
            </div>
            {CAMPOS.map(({ key, label }) => {
              const antes = valor(live?.[key] as string | null | undefined);
              const despues = pendiente ? valor(draft?.[key] as string | null | undefined) : antes;
              const cambio = pendiente && antes !== despues;
              return (
                <div
                  key={key}
                  className={`border-b border-[#dee3e9] px-5 py-3 last:border-b-0 dark:border-white/[0.07] ${cambio ? "bg-[#fef3e2] dark:bg-[#f2a918]/[0.08]" : ""}`}
                >
                  <div className="mb-1 text-xs font-bold text-[#0a1317] dark:text-[#f0f2f5]">{label}</div>
                  <div className="grid grid-cols-[1fr_1fr] gap-4 text-sm">
                    <div className="min-w-0 break-words whitespace-pre-wrap text-[#5d6c7b] dark:text-[#8d9199]">
                      {antes ?? <span className="italic text-[#8595a4]">vacío</span>}
                    </div>
                    <div className="min-w-0 break-words whitespace-pre-wrap text-[#1c1e21] dark:text-[#e4e6eb]">
                      {despues ?? <span className="italic text-[#8595a4]">vacío</span>}
                    </div>
                  </div>
                </div>
              );
            })}
          </section>

          {pendiente && (
            <div className="mt-4 rounded-2xl border border-[#dee3e9] bg-white p-4 dark:border-white/[0.07] dark:bg-[#18191c]">
              <label className="mb-2 block text-xs font-bold text-[#0a1317] dark:text-[#f0f2f5]">
                Motivo del rechazo (opcional, lo ve el cliente)
              </label>
              <textarea
                value={note}
                onChange={(e) => setNote(e.target.value)}
                rows={2}
                maxLength={500}
                placeholder="Ej: faltan los horarios de sábado"
                className="w-full resize-none rounded-2xl border border-[#dee3e9] bg-white px-3 py-2 text-sm text-[#0a1317] outline-none focus:border-[#0064e0] dark:border-white/[0.07] dark:bg-white/[0.04] dark:text-[#f0f2f5]"
              />
              <div className="mt-3 flex gap-2">
                <button
                  onClick={approve}
                  disabled={busy}
                  className="inline-flex items-center gap-1.5 rounded-full bg-[#31a24c] px-4 py-2 text-sm font-bold text-white transition-colors hover:bg-[#2b8f43] disabled:opacity-50"
                >
                  {busy ? <Loader2 className="size-4 animate-spin" /> : <Check className="size-4" />}
                  Aprobar
                </button>
                <button
                  onClick={reject}
                  disabled={busy}
                  className="inline-flex items-center gap-1.5 rounded-full border border-[#e41e3f]/30 px-4 py-2 text-sm font-bold text-[#e41e3f] transition-colors hover:bg-[#fde8ec] disabled:opacity-50 dark:hover:bg-[#e41e3f]/[0.12]"
                >
                  <X className="size-4" />
                  Rechazar
                </button>
              </div>
            </div>
          )}
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Add the page title**

`lib/page-title.ts` maps route prefixes to tab titles. Add an entry for `/superadmin/tenants` scoped screens if the existing `["/superadmin/tenants", "Clientes"]` entry does not already produce a sensible title for this route. Check the file's matching rule before adding — do not add a duplicate prefix.

- [ ] **Step 5: Type-check, lint, self-review**

Run: `npx tsc --noEmit` → clean.
Run: `npm run lint` → at or below baseline.

Read your own diff: does every handler clear `busy` in a `finally`? Does the diff highlight only fields that actually changed? Does a 500-character unbroken value force horizontal scroll (`break-words` and `min-w-0` are present to prevent it)?

- [ ] **Step 6: Commit**

```bash
git add components/tenant-business-section.tsx "app/(crm)/superadmin/tenants/[slug]/page.tsx" "app/(crm)/superadmin/tenants/[slug]/negocio/page.tsx" lib/page-title.ts
git commit -m "feat(negocio): pantalla de revision de info del negocio (PRD 10 fase 3)"
```

---

### Task 4: Pending badge on the tenant list

**Files:**
- Modify: `soylaika.frontend/app/(crm)/superadmin/tenants/page.tsx`

A review queue nobody can see is a review queue nobody works. The list is where a superadmin starts.

**Interfaces:**
- Consumes: `api.tenants.business.pendingCount()` → `Record<slug, number>`.

- [ ] **Step 1: Fetch the counts**

Beside the existing `tenants` state, add:

```ts
  const [pendingBusiness, setPendingBusiness] = useState<Record<string, number>>({});
```

The page already loads tenants in an effect (`api.tenants.list().then(setTenants)`). Add the counts to that same load rather than introducing a second effect, keeping state-setting inside the async callbacks:

```ts
    api.tenants.business
      .pendingCount()
      .then((c) => { if (alive) setPendingBusiness(c); })
      .catch(() => { /* el badge es informativo: si falla, la lista sigue */ });
```

If the existing effect has no `alive` guard, add one — do not introduce a synchronous `setState` in an effect body.

- [ ] **Step 2: Render the badge**

In the row rendered per tenant, next to the tenant's name, add:

```tsx
                {pendingBusiness[tenant.slug] ? (
                  <Link
                    href={`/superadmin/tenants/${tenant.slug}/negocio`}
                    title="Cambios en la info del negocio esperando revisión"
                    className="inline-flex items-center gap-1 rounded-full bg-[#fef3e2] px-2 py-0.5 text-xs font-bold text-[#7a5a12] dark:bg-[#f2a918]/[0.18] dark:text-[#f2a918]"
                  >
                    {pendingBusiness[tenant.slug]} pendiente
                    {pendingBusiness[tenant.slug] > 1 ? "s" : ""}
                  </Link>
                ) : null}
```

Match the surrounding row markup — read the existing name cell before inserting, and add the `Link` import if the file does not already have it.

- [ ] **Step 3: Type-check, lint and commit**

Run: `npx tsc --noEmit` → clean.
Run: `npm run lint` → at or below baseline.

```bash
git add "app/(crm)/superadmin/tenants/page.tsx"
git commit -m "feat(negocio): badge de cambios pendientes en el listado de clientes (PRD 10 fase 3)"
```

---

### Task 5: Verify in a browser

**Files:** none — this task changes no code.

This repo has no test suite, so this is the only functional gate. Phase 1's browser check went unperformed because no automation was connected; do not let that repeat silently — if no browser automation is available, say so and hand this list to a human rather than marking it done.

- [ ] **Step 1: Confirm both dev servers are up**

`:3000` and `:3001` must be listening. From the backend directory:

```powershell
powershell -ExecutionPolicy Bypass -File .\start-dev.ps1
```

The backend must be running Phase 2's code, or every endpoint here 404s.

- [ ] **Step 2: Customer side**

At `http://tenant-dev.localhost:3001/settings` as a tenant admin:
- The business card's button reads "Enviar cambios".
- Editing a field and saving shows the "esperando la aprobación" banner, and the values persist on reload.
- The bot's data is unchanged — confirm with `GET /api/business`, which must still return the old values.

- [ ] **Step 3: Superadmin side**

In the superadmin panel, on the tenant detail page: the Negocio section appears beside Agentes and Reglas, in the same card shell, and links to the review screen. On the review screen, only the changed fields are highlighted, and both columns render long values without forcing horizontal scroll.

- [ ] **Step 4: Approve, then reject**

Approve and confirm the customer's `GET /api/business` now returns the new values and the banner is gone. Then submit a second change, reject it with a note, and confirm the customer sees the rejection banner with that note while live stays unchanged.

- [ ] **Step 5: Badge**

The tenant list shows the pending badge while a change waits, and it disappears after approval.

- [ ] **Step 6: Final gates**

Run: `npx tsc --noEmit` → clean.
Run: `npm run lint` → at or below the number recorded in Task 1 Step 1.

Report the actual output and the actual lint number. Do not claim this phase is done on the strength of the type-check alone — Steps 2 and 4 are the acceptance criteria (§4.6, §5).
