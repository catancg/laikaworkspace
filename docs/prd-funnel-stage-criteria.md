# PRD 8 — Per-tenant funnel stage criteria

**Status:** proposed
**Related:** [PRD 6 — agent evaluation suite](prd-agent-eval-suite.md) §4.0, which states the same rule this document applies to one specific case: behaviour is driven by tenant data, never by constants.

## 1. Why this exists

`AiService.analyzeConversation` classifies every conversation into a funnel stage. Its prompt contains **two lists of stages**, and only one of them is the tenant's.

The first is generated from the tenant's real rows:

```js
const lines = stages.map((s) => {
  const flag = s.isWon ? ' (venta cerrada)' : s.isLost ? ' (perdido)' : '';
  return `- "${s.slug}": ${s.name}${flag}`;
});
```

The second is hardcoded, and it is where the actual classification rules live — seven stages with criteria, examples and edge cases (`nuevo`, `interesado`, `cotizado`, `cerrando`, `cliente`, `perdido`, `no contesta`).

So the tenant controls **which stages exist**; a constant in our code controls **what they mean**. Those two cannot be kept in agreement, and they already are not.

### 1.1 Two pieces of evidence that this has already gone wrong

**A hand-patched guard.** The hardcoded criteria contain:

> *"cerrando": … Si esa etapa no está en la lista, usa "cotizado".*

Somebody hit the case where the tenant's stages and the hardcoded criteria disagreed, and patched it in prose for that one stage. The other six have no such guard.

**A slug that does not match.** `DEFAULT_STAGES` seeds `slug: 'no-contesta'` — hyphen. The hardcoded criteria block says `- "no contesta":` — space. The model is shown both and told to return *"el slug exacto de la lista"*; if it follows the criteria block instead of the list, `stageMap.has('no contesta')` is false and the classification is silently discarded.

Low impact in this instance, because that stage is the one the prompt tells the model never to pick. But it is the disease showing: **two sources of truth for the same list, drifted.**

## 2. Goal

Let a superadmin write the classification criteria for each funnel stage, per tenant, and have the classifier prompt be generated entirely from that.

After this, `analyzeConversation` contains **no stage names at all**.

## 3. Non-goals

- **Not making the funnel fully configurable.** Adding, renaming and deleting stages have consequences outside this prompt (§6) that this document does not fix. §4 says what the panel must therefore refuse.
- **Not changing how stages are applied.** Transitions, notifications, `isWon`/`isLost` handling all stay as they are.
- **Not a prompt editor for the whole analyst prompt.** Only the per-stage criteria and the transition policy (§5.2) become data.

## 4. What becomes editable, and what must not

| | Editable | Why |
|---|---|---|
| **Criteria text** per stage | **Yes — this is the feature** | It is prompt text. Nothing else reads it |
| `name`, `color`, `order`, `notifyOnEnter` | Yes, already | Display and notification behaviour; no code branches on them |
| `isWon` / `isLost` | Yes, already | Semantic flags the code already reads properly |
| **`slug`** | **No — must be locked** | ~9 sites across 4 files look stages up by slug (§6). Renaming one silently breaks followups, revenue metrics, or the revive-on-reply logic |
| **Deleting a stage** | **No — must be blocked** | Same reason: `findFirst({ where: { slug: 'no-contesta' } })` returning null is an unhandled path |
| **Adding a stage** | Yes, with a warning | Safe for the classifier — it appears in the list and carries its own criteria. But code that assumes a fixed set will not know about it |

**This is the part most likely to be got wrong.** Handing someone a criteria editor makes the funnel *look* fully configurable. The panel has to be explicit that slugs are fixed and stages cannot be removed, or the first person to tidy up their funnel will break revenue reporting and not find out for a month.

## 5. The prompt becomes generated

### 5.1 Per-stage criteria

`FunnelStage` gains a `criteria` text field. The prompt's `Criterio:` block is built from it exactly as the stage list already is:

```js
const criterios = stages
  .filter((s) => s.criteria?.trim())
  .map((s) => `- "${s.slug}": ${s.criteria.trim()}`)
  .join('\n');
```

A stage with no criteria still appears in the list with its name — degraded, not broken.

### 5.2 The transition policy

Two rules in the prompt are not per-stage:

> *El estado normalmente solo avanza; no retrocedas salvo evidencia clara. EXCEPCION: "perdido" se puede marcar desde CUALQUIER etapa…*

The first is global policy. The second names a stage — but that stage is identifiable from data: it is the one with `isLost = true`. Generate it:

```
EXCEPCION: "<slug of the isLost stage>" se puede marcar desde CUALQUIER etapa
```

The global sentence becomes a tenant-level setting alongside the criteria, so a business whose funnel legitimately moves backwards can say so.

### 5.3 The acceptance test

**No stage slug or stage name may appear as a literal in `analyzeConversation` after this.** That is checkable with a grep, and it is the same acceptance test PRD 6 §4.0 sets for the eval suite: replacing every stage definition must require no code change.

## 6. The coupling this exposes but does not fix

Making the criteria editable does not make the funnel configurable, because the slugs are load-bearing elsewhere. Roughly **nine sites across four files** look a stage up by slug:

| Where | What it does | What breaks if the slug changes |
|---|---|---|
| `followup.processor.ts` | Moves a silent contact to `'no-contesta'` | Follow-ups stop moving anyone |
| `message.processor.ts` | Revives a `'no-contesta'`/lost contact to `'interesado'` on a new message | Contacts stay dead after replying |
| `crm.service.ts` | Revenue counts `status IN ('cotizado','cliente')`; client count on `'cliente'`; new contacts start at `'nuevo'` | **Revenue and conversion silently wrong** |
| `ai.service.ts` | A `FALLBACK` array of five slugs | Fallback classification degrades |

Two structural notes:

- **The codebase already has the right pattern and does not use it consistently.** `isWon`/`isLost` exist and are read properly in several places. One line does both at once: `if (currentStage.slug === 'no-contesta' || currentStage.isLost)`. Half semantic, half hardcoded, same condition.
- **`Contact.status` is a denormalized copy of the stage slug** (`data: { stageId: stage.id, status: 'interesado' }`), and the revenue query reads `status`, not the stage. So the coupling is duplicated into a second column.

Fixing this properly means semantic flags for the two remaining concepts the code needs — "the stage meaning no reply" and "the stage meaning actively engaged" — plus moving the metrics off `Contact.status`. It is a separate piece of work and §11 q1 asks whether it should happen before or after this one.

## 7. Seeding and migration

The seven current criteria strings become the seeded default for `criteria` on the seven `DEFAULT_STAGES`. **The behaviour on day one is identical to today** — the text simply moves from a constant into rows.

Two details:

- **Fix the `no contesta` / `no-contesta` mismatch while seeding** (§1.1). It is a one-character correction that can only be made once, at the moment the text becomes data.
- **Existing tenants get the criteria backfilled by the same migration**, matched on slug. A tenant that has already renamed or added stages gets criteria for the ones that match and blanks for the rest — visible in the panel as something to fill in, which is the correct outcome.

## 8. Panel

On the existing funnel screen, each stage gains a criteria field: a textarea, with the seeded default visible and a **"restore the default"** action, because someone will edit it, make it worse, and want the original back — and there is no history (the same problem [PRD 7](prd-agent-config-versioning.md) solves for agent prompts; the same solution applies here and is cheaper to add now than later).

Slug shown but not editable, with one line saying why. Delete disabled, with the same.

**Writing guidance beside the field**, because the quality of this text decides the quality of the classification:

- Say what *the customer* did, not what the seller should do.
- Give the phrases customers actually use — the current `perdido` criteria list six of them, and that is why it works.
- Say what does **not** count. Half the value in the current `cotizado` text is *"Mencionar precios de lista o por m² NO alcanza"*.

## 9. Interaction with the eval suite

PRD 6's corpus contains scenarios whose `expects` include stage transitions. Once criteria are tenant data, they become part of the configuration fingerprint (PRD 6 §4.0) — a criteria edit is a configuration change, and a classification that shifts afterwards is expected rather than a regression.

This is also the cheapest way to tell whether an edited criterion is better: run the suite before and after.

## 10. Phasing

| Phase | Scope |
|---|---|
| **A** | `criteria` column, seeded from the current constants, backfilled for existing tenants; prompt generated from it; the `no-contesta` fix |
| **B** | Panel: edit, restore default, slug locked, delete disabled |
| **C** | The transition policy (§5.2) as a tenant setting |

**A alone is worth shipping.** It changes no behaviour and removes the second source of truth, which is the actual defect.

## 11. Open questions

1. **Fix the slug coupling (§6) before or after this?** After is defensible: this PRD does not make the coupling worse, and locking slugs in the panel contains it. But every day it stays, "the funnel is configurable" is half-true in a way that will eventually cost a tenant their revenue numbers.
2. **Should criteria be per-agent as well as per-tenant?** `analyzeConversation` runs once per conversation regardless of which agent replied. Probably not, but worth naming before someone assumes it.
3. **How long can a criteria field be?** The whole block goes into every classification call, so it is a per-conversation cost. The current seven total ~1,200 characters. A cap — or at least a visible character count — stops one enthusiastic edit from doubling the prompt.
4. **Does the eval suite need a scenario per stage transition?** It would be the natural regression test for this feature, and it is the same corpus. Depends on PRD 6 phase A landing.

## 12. Risks

- **Someone edits a criterion and the classifier gets worse, silently.** Conversations are misfiled for weeks before anyone notices. Mitigated by "restore the default" (§8) and properly by PRD 6 — this feature makes prompt-quality mistakes easier to introduce and PRD 6 is what detects them.
- **The panel implies the funnel is fully configurable** (§4). The single most likely misunderstanding, and the one that breaks revenue reporting.
- **Criteria drift from the business.** The text says "cotizado is when you gave a total" long after the business changed how it quotes. Nothing detects this; it is the same class of staleness as PRD 6 §6.5.1.
- **Empty criteria degrade classification quietly.** A stage with a blank field still appears in the list, so the model guesses from the name. Correct behaviour, but the panel should show which stages have no criteria rather than leaving it to be discovered.
