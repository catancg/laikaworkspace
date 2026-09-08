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

Let a superadmin decide, per tenant, **which stages exist, what they are called, and what each one means** — and have the classifier prompt be generated entirely from that.

After this, `analyzeConversation` contains **no stage names at all**.

### 2.1 Two kinds of stage

Stages split into two groups, and this distinction is the centre of the design:

| | Meaning | Cap |
|---|---|---|
| **Pipeline** (positive) | The phases of the commercial flow. A lead moving through them is progressing | **up to 5** enabled |
| **Out** (negative) | Where leads go when they leave the flow, or never enter it | **up to 3** enabled |

Today's defaults land exactly on that shape: `nuevo`, `interesado`, `cotizado`, `cerrando`, `cliente` are the five pipeline stages; `no-contesta` and `perdido` are two of the three out stages.

**"Sin etapa" is not a row.** It is the absence of one — `stageId = null` — which already exists and which the classifier already returns when nothing matches. What becomes configurable is the *guidance* for it: when should the model decline to classify at all (§5.3).

### 2.2 This is the concept the code has been missing

`no-contesta` is seeded `isWon: false, isLost: false` — **flag-identical to every pipeline stage**. It is semantically outside the commercial flow and nothing in the data says so. Which is why `message.processor.ts` has to write:

```js
if (currentStage && (currentStage.slug === 'no-contesta' || currentStage.isLost))
```

Half semantic, half hardcoded, because the semantic half does not exist yet. **`kind` is that missing half**, and adding it collapses several of the couplings in §6 rather than merely documenting them.

## 3. Non-goals

- **Not deleting stages.** How many are in use is expressed by enabling and disabling (§4.1); real deletion needs the three designations in §6 first.
- **Not renaming slugs.** Titles change freely; the internal identifier does not (§4).
- **Not changing how stages are applied.** Transitions, notifications, `isWon`/`isLost` handling all stay as they are.
- **Not a prompt editor for the whole analyst prompt.** Only the per-stage criteria and the transition policy (§5.2) become data.

## 4. What becomes editable, and what must not

| | Editable | Why |
|---|---|---|
| **Criteria text** per stage | **Yes — the feature** | Prompt text. Nothing else reads it |
| **`name`** (the title) | **Yes** | Display only. The slug is what code uses |
| **`enabled`** | **Yes, within the caps** | How "how many they use" is expressed (§4.1) |
| `color`, `order`, `notifyOnEnter` | Yes, already | No code branches on them |
| `isWon` / `isLost` | Yes, already | Semantic flags the code reads properly |
| **`kind`** | **No, once set** | Moving a stage between pipeline and out changes what every historical contact in it meant |
| **`slug`** | **No** | **Already immutable** — `FunnelService.update()` accepts only `name`, `color`, `notifyOnEnter`, `isLost`, `isWon`. It stays that way |
| **Deleting a stage** | **No** | `findFirst({ where: { slug } })` returning null is an unhandled path in four files (§6). Disabling is the supported route |

### 4.1 Enabled, not deleted

"How many categories they use" is expressed by an `enabled` flag, not by creating and destroying rows.

A disabled stage does not appear in the prompt, cannot be assigned, and is hidden from the funnel board — **but the row survives**. That matters for three reasons: the nine `findFirst({ slug })` sites still find something; historical contacts keep the stage they were in, so last quarter's numbers do not change retroactively; and it is reversible.

Constraints the API must enforce:

- At most **5 enabled pipeline** stages and **3 enabled out** stages.
- At least **1 enabled pipeline** stage — a funnel with none classifies nothing.
- Disabling a stage that contacts currently sit in is allowed, but the panel says how many will be left there.

**Disabling a stage the system writes to has consequences the panel must state.** Follow-ups move silent contacts to a designated out stage; disable it and follow-ups simply stop marking anyone. That is arguably correct — the business said it does not use that category — but it must be said out loud at the moment of disabling, not discovered later.

**This is the part most likely to be got wrong.** Handing someone a criteria editor makes the funnel *look* fully configurable. The panel has to be explicit that slugs are fixed and stages cannot be removed, or the first person to tidy up their funnel will break revenue reporting and not find out for a month.

## 5. The prompt becomes generated

### 5.1 Two blocks, built from the two kinds

`FunnelStage` gains `criteria`, `kind` and `enabled`. The prompt presents the two groups **separately**, because they mean different things to the classifier — one is progress through a flow, the other is exit from it:

```js
const enabled = stages.filter((s) => s.enabled);
const block = (kind) => enabled
  .filter((s) => s.kind === kind)
  .sort((a, b) => a.order - b.order)
  .map((s) => `- "${s.slug}" (${s.name}): ${s.criteria?.trim() || s.name}`)
  .join(NEWLINE);
```

rendered under two headings — *ETAPAS DEL FLUJO COMERCIAL* (pipeline, in order) and *FUERA DEL FLUJO* (out).

This also replaces the current duplicate listing: today the stages appear once as a bare list and again inside the hardcoded criteria (§1). One block, one source.

A stage with no criteria falls back to its name — degraded, not broken (§13).

### 5.2 The transition policy

Two rules in the prompt are not per-stage:

> *El estado normalmente solo avanza; no retrocedas salvo evidencia clara. EXCEPCION: "perdido" se puede marcar desde CUALQUIER etapa…*

The first is global policy. The second names a stage — but that stage is identifiable from data: it is the one with `isLost = true`. Generate it:

```
EXCEPCION: "<slug of the isLost stage>" se puede marcar desde CUALQUIER etapa
```

The global sentence becomes a tenant-level setting alongside the criteria, so a business whose funnel legitimately moves backwards can say so.

### 5.3 "None of the above"

The classifier already returns `null` when nothing matches, and `null` is what the business calls *"sin etapa"* (§2.1). But today that happens **by accident** — the model returned something unparseable, or a slug that is not in the map.

Make it deliberate: a short, superadmin-editable line telling the model when to decline to classify, and an explicit way to say so in the output. A conversation that is genuinely not a lead should land in "sin etapa" *because the model said so*, not because parsing failed. Those two outcomes are indistinguishable today, and only one of them is a bug.

### 5.4 The acceptance test

**No stage slug or stage name may appear as a literal in `analyzeConversation` after this.** That is checkable with a grep, and it is the same acceptance test PRD 6 §4.0 sets for the eval suite: replacing every stage definition must require no code change.

## 6. The coupling — what `kind` fixes, and what it does not

Nine sites across four files look a stage up by slug. **`kind` resolves four of them outright**, which is the argument for adding it now rather than later:

| Site | Today | With `kind` |
|---|---|---|
| `message.processor` — is this contact out of the flow? | `slug === 'no-contesta'` OR `isLost` | `kind === 'out'` |
| `crm.service` — where do new contacts start? | `slug: 'nuevo'` | first enabled pipeline stage by `order` |
| `crm.service` — client count | `status === 'cliente'` | `isWon` |
| `ai.service` — `FALLBACK` array of five slugs | hardcoded | generated from enabled stages |

Three need a further designation, because with up to three out stages *"which one"* becomes ambiguous:

| Site | What it still needs |
|---|---|
| `followup.processor` — move a silent contact | **Which out stage means "stopped replying".** A designation among the out stages, the same shape as `isWon`/`isLost` |
| `message.processor` — revive on a new reply | **Which pipeline stage a returning lead re-enters.** Currently `'interesado'`, the second stage. Could be a designation, or `previousStageId`, which already exists on `Contact` |
| `crm.service` — revenue `status IN ('cotizado','cliente')` | Arguably should not be stage-based at all — revenue is `Sale` rows. Out of scope here, and worth its own look |

Two structural notes that survive this PRD:

- **`Contact.status` is a denormalized copy of the stage slug** (`data: { stageId: stage.id, status: 'interesado' }`), and the revenue query reads `status`, not the stage. The coupling is duplicated into a second column, and `kind` does not reach it.
- **A stage still cannot be deleted** (§4.1), only disabled. Real deletion waits for the three designations above.

## 7. Seeding and migration

The seven current criteria strings become the seeded default for `criteria` on the seven `DEFAULT_STAGES`. **The behaviour on day one is identical to today** — the text simply moves from a constant into rows.

Two details:

- **Fix the `no contesta` / `no-contesta` mismatch while seeding** (§1.1). It is a one-character correction that can only be made once, at the moment the text becomes data.
- **`kind` is backfilled from what the stages already mean**: the five defaults become pipeline, `no-contesta` and `perdido` become out. For a tenant that added its own stages, anything not matching a default slug defaults to **pipeline** and is flagged in the panel for confirmation — guessing "out" would silently take contacts off the board.
- **`enabled` defaults to true** for everything that exists, so day one is unchanged.
- **Existing tenants get the criteria backfilled by the same migration**, matched on slug. A tenant that has already renamed or added stages gets criteria for the ones that match and blanks for the rest — visible in the panel as something to fill in, which is the correct outcome.

## 8. Panel

On the existing funnel screen, each stage gains a criteria field: a textarea, with the seeded default visible and a **"restore the default"** action, because someone will edit it, make it worse, and want the original back — and there is no history (the same problem [PRD 7](prd-agent-config-versioning.md) solves for agent prompts; the same solution applies here and is cheaper to add now than later).

Slug shown but not editable, with one line saying why. Delete disabled, with the same.

**Writing guidance beside the field**, because the quality of this text decides the quality of the classification:

- Say what *the customer* did, not what the seller should do.
- Give the phrases customers actually use — the current `perdido` criteria list six of them, and that is why it works.
- Say what does **not** count. Half the value in the current `cotizado` text is *"Mencionar precios de lista o por m² NO alcanza"*.

## 9. Making it safe

The criteria are free text written by a person and injected into a system prompt that runs on every conversation. That deserves more thought than "add a column".

### 9.1 The role trap, and the main recommendation

**`criteria` must not be editable through the existing funnel endpoints.**

`PATCH /api/funnel/stages/:id` and `DELETE /api/funnel/stages/:id` are `@Roles('admin', 'superadmin')` — **tenant admins**. Adding `criteria` to `FunnelStage` and surfacing it on the existing funnel screen would hand a tenant admin write access to a system prompt, silently, on the day the column ships. The requirement says superadmin.

So: **a separate superadmin-only route** — `PATCH /tenants/:slug/funnel/stages/:id/criteria`, on `TenantsController`, which is superadmin by class — and the existing admin-facing update explicitly **strips** `criteria` from its payload, the same named-allowlist pattern already used on `PATCH /tenants/:slug` after the mass-assignment fix.

Without that strip, the admin endpoint takes a partial body and would happily write the field.

### 9.2 Keep the format instruction last

The prompt ends by demanding a specific JSON shape:

```
{"stage":"<slug exacto de la lista>","info":{ ...}}
```

**That instruction must stay after the criteria block, always.** Criteria text sits in the middle of the prompt; the closing format rule is what stops a careless — or hostile — criterion from redefining the output. It happens to be ordered correctly today. Make it a rule rather than an accident, and put the criteria inside a delimited region so it is visibly a data block rather than more instructions.

### 9.3 Validate on write

Criteria are prose about what a customer did. They never legitimately need braces, JSON, or instructions to the model.

- **Reject `{` and `}`** — the only reason for them here is to interfere with the output shape.
- **Cap the length** (§12 q3). The whole block ships on every classification call.
- **Flag imperative-to-the-model phrasing** — *"respondé"*, *"ignorá"*, *"formato"*, *"en vez de"*. A warning, not a block: legitimate criteria describe the customer, so a criterion addressing the model is usually a mistake and occasionally an attack.

### 9.4 Fail safe, not empty

If every stage has blank criteria — a bad migration, a bulk delete — the prompt must fall back to the seeded defaults, not ship an empty `Criterio:` block. Classification silently degrading to name-guessing is worse than a stale default.

Keep the seeded default stored **alongside** the current value rather than only in code, so "restore the default" (§8) still works after the constant is eventually deleted.

### 9.5 Proportionality: what the blast radius actually is

Worth stating plainly, because it should govern how much machinery to build: `analyzeConversation` **never produces text sent to a customer.** It returns a stage and extracted CRM fields.

So the worst case of a bad criterion is misfiled leads and corrupted `Contact.details` — bad, and invisible for a while, but not "the bot told a customer something false". This is a lower-severity surface than agent prompts (PRD 7), and the controls should be lighter in proportion: a role boundary, a validation, a restore button, and detection via PRD 6 — not an approval queue.

### 9.6 A pre-existing hole this PRD does not open

`DELETE /api/funnel/stages/:id` is available to tenant admins **today**, and §6 shows deleting a stage breaks follow-ups, the revive-on-reply path, and revenue reporting. That is true before this PRD and independent of it.

It is worth fixing in the same pass, because §4 already requires the panel to disable deletion — doing it in the UI while the endpoint stays open would be a control that is not a control.

## 10. Interaction with the eval suite

PRD 6's corpus contains scenarios whose `expects` include stage transitions. Once criteria are tenant data, they become part of the configuration fingerprint (PRD 6 §4.0) — a criteria edit is a configuration change, and a classification that shifts afterwards is expected rather than a regression.

This is also the cheapest way to tell whether an edited criterion is better: run the suite before and after.

## 11. Phasing

| Phase | Scope |
|---|---|
| **A** | `criteria`, `kind` and `enabled` columns, seeded and backfilled (§7); prompt generated as two blocks (§5.1); the `no-contesta` fix |
| **B** | Superadmin-only route (§9.1) + `criteria` stripped from the admin endpoint; delete closed at the API (§9.6); validation (§9.3) |
| **C** | Panel: edit title and criteria, enable/disable within the caps, restore default, slug shown but locked, delete absent |
| **D** | Replace the four slug lookups `kind` now covers (§6) |
| **E** | The transition policy (§5.2) and the "none of the above" line (§5.3) as tenant settings |

**A alone is worth shipping** — it changes no behaviour and removes the second source of truth, which is the actual defect. But **A must not ship without B**: the moment `criteria` exists as a column it is writable through the tenant-admin endpoint, so the role boundary has to land in the same release, not the next one.

## 12. Open questions

1. **Fix the slug coupling (§6) before or after this?** After is defensible: this PRD does not make the coupling worse, and locking slugs in the panel contains it. But every day it stays, "the funnel is configurable" is half-true in a way that will eventually cost a tenant their revenue numbers.
2. **The three remaining designations (§6).** Which out stage means "stopped replying", where a revived lead re-enters, and whether revenue should move off stages entirely. Each is small; together they are what makes real deletion safe.
3. **Should criteria be per-agent as well as per-tenant?** `analyzeConversation` runs once per conversation regardless of which agent replied. Probably not, but worth naming before someone assumes it.
4. **How long can a criteria field be?** The whole block goes into every classification call, so it is a per-conversation cost. The current seven total ~1,200 characters. A cap — or at least a visible character count — stops one enthusiastic edit from doubling the prompt.
5. **Does the eval suite need a scenario per stage transition?** It would be the natural regression test for this feature, and it is the same corpus. Depends on PRD 6 phase A landing.

## 13. Risks

- **Someone edits a criterion and the classifier gets worse, silently.** Conversations are misfiled for weeks before anyone notices. Mitigated by "restore the default" (§8) and properly by PRD 6 — this feature makes prompt-quality mistakes easier to introduce and PRD 6 is what detects them.
- **`criteria` ships as a plain column and becomes tenant-admin editable** (§9.1). The specific way this feature turns into a privilege escalation, and it happens by omission rather than by decision — nobody has to do anything wrong beyond adding the column and reusing the existing screen.
- **The panel implies the funnel is fully configurable** (§4). The single most likely misunderstanding, and the one that breaks revenue reporting.
- **Criteria drift from the business.** The text says "cotizado is when you gave a total" long after the business changed how it quotes. Nothing detects this; it is the same class of staleness as PRD 6 §6.5.1.
- **Empty criteria degrade classification quietly.** A stage with a blank field still appears in the list, so the model guesses from the name. Correct behaviour, but the panel should show which stages have no criteria rather than leaving it to be discovered.
