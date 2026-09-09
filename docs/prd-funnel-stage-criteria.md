# PRD 8 — Per-tenant funnel stage names and criteria

**Status:** implementado (fases A–F) — ver [el plan](plans/2026-09-08-funnel-stage-criteria.md)
**Related:** [PRD 6 — agent evaluation suite](prd-agent-eval-suite.md) §4.0, which states the same rule this document applies to one specific case: behaviour is driven by tenant data, never by constants.

## 1. The requirement

Different tenants have different commercial stages. One sells and quotes, so its third stage is *Cotizado*. Another books meetings, so its third stage is *Crear reunión*. Today both get "Cotizado", because the stage's meaning lives in a constant in our code.

## 2. The design, in one line

**The slug is frozen forever. `name` and `criteria` are what the tenant gets.**

`slug` stops being a thing anyone can see or set and becomes purely an internal key — a row's permanent identity, like its `id`. Everything a human reads or writes is `name` (the title) and `criteria` (what it means).

This is not a compromise. It is strictly better than making slugs editable, for three reasons: no data migration, no way for a tenant to sever a wire the code depends on (§2.2, §4), and the classifier prompt already works this way.

### 2.1 The prompt is already built for it

`AiService.analyzeConversation` renders the stage list like this ([ai.service.ts:693](../soylaika.backend/src/ai/ai.service.ts)):

```js
const lines = stages.map((s) => {
  const flag = s.isWon ? ' (venta cerrada)' : s.isLost ? ' (perdido)' : '';
  return `- "${s.slug}": ${s.name}${flag}`;      // → - "cotizado": Cotizado
});
```

The slug is already the **key** and the name is already the **gloss**. Rename the stage and that line becomes `- "cotizado": Crear reunión` with no code change. The model returns `cotizado`, `stageMap.get()` finds the row, the contact moves, and every downstream consumer — `Contact.status`, `MessageTemplate.stage_slug`, the funnel board — keeps working.

So the entire requirement reduces to: make `name` editable per tenant (it already is), and make the **criteria** editable per tenant (it is not).

### 2.2 One of those wires crosses a database boundary

`MessageTemplate.stage_slug` ([schema.prisma:339](../soylaika.backend/prisma/schema.prisma)) lives in the **master** database, indexed `[tenant_id, stage_slug]`. `FunnelStage` lives in the **tenant's** database. `getTemplatesForTenant` matches one against the other by string ([crm.service.ts:660](../soylaika.backend/src/crm/crm.service.ts)):

```ts
...(stageSlug ? { OR: [{ stage_slug: stageSlug }, { stage_slug: null }] } : {})
```

Two databases, so no foreign key is possible and none exists. An editable slug would silently unlink a tenant's WhatsApp templates from their stages, with nothing in either schema to catch it and no error at the moment it happened.

**This is the strongest reason to freeze the slug** — stronger than "no data migration", which is merely convenient. It is a referential constraint the database cannot express, so the only place it can be enforced is by never changing the value.

## 3. Why this exists at all

The classifier prompt contains **two lists of stages**, and only one of them is the tenant's. The first is generated from the tenant's rows, above. The second is hardcoded, and it is where the actual classification rules live ([ai.service.ts:713-719](../soylaika.backend/src/ai/ai.service.ts)):

```
- "cotizado": SOLO cuando se le dio un TOTAL/presupuesto concreto (ej: "6 m², te queda en $270.000").
```

The tenant controls **which stages exist**; a constant controls **what they mean**. Those two cannot be kept in agreement, and they already are not.

### 3.1 Two pieces of evidence that this has already gone wrong

**A hand-patched guard.** The hardcoded criteria contain:

> *"cerrando": … Si esa etapa no está en la lista, usa "cotizado".*

Somebody hit the case where the tenant's stages and the hardcoded criteria disagreed, and patched it in prose for that one stage. The other six have no such guard.

**A slug that does not match.** `DEFAULT_STAGES` seeds `slug: 'no-contesta'` — hyphen. The hardcoded criteria block says `- "no contesta":` — space. The model is shown both and told to return the exact slug from the list; if it follows the criteria block instead, `stageMap.has('no contesta')` is false and the classification is silently discarded.

Low impact in this instance, because that stage is the one the prompt tells the model never to pick. But it is the disease showing: **two sources of truth for the same list, drifted.**

## 4. What the slugs actually cost

Not all seven slugs are equal. Five are wired into code by their literal string; two are not. This table is the design constraint:

**Load-bearing — rename freely, but never disable, and the slug can never change:**

| slug | site | what breaks if the row stops existing |
|---|---|---|
| `interesado` | [message.processor.ts:80](../soylaika.backend/src/queue/message.processor.ts) | a silent lead who replies is never revived |
| `no-contesta` | [followup.processor.ts:51](../soylaika.backend/src/queue/followup.processor.ts), [crm.service.ts:627](../soylaika.backend/src/crm/crm.service.ts), [message.processor.ts:79](../soylaika.backend/src/queue/message.processor.ts) | follow-ups stop parking anyone |
| `cliente` | [crm.service.ts:738-739](../soylaika.backend/src/crm/crm.service.ts), [765](../soylaika.backend/src/crm/crm.service.ts) | client count and source stats read zero |
| `cotizado` | [crm.service.ts:698](../soylaika.backend/src/crm/crm.service.ts), [775](../soylaika.backend/src/crm/crm.service.ts) | "cotizaciones abiertas" reads zero, and quoted leads drop out of the ticket-estimado average |
| `nuevo` | [crm.service.ts:512](../soylaika.backend/src/crm/crm.service.ts) | the CRM playground creates its demo contact with no stage — and nothing else, which is not what it looks like (§4.2) |

**Free — code only reaches these through `isWon` / `isLost`:** `cerrando`, `perdido`.

Every one of those failures is silent. The pattern is `findFirst({ where: { slug } })` followed by `if (stage)` — a missing row means the branch is skipped, with no error and no log. The `cotizado` and `cliente` sites are quieter still: they read the `Contact.status` string with no lookup at all, so there is no branch to skip — the count simply comes back zero.

**"Cotizado" → "Crear reunión" still costs nothing** — the motivating example is a *rename*, and the slug is frozen. But not because `cotizado` is free; it is not. Disabling it is what would break things (§9.1).

### 4.1 The `cliente` case is worse than the others

`cliente` is load-bearing through `Contact.status`, not through the stage table:

```js
db.contact.count({ where: { status: 'cliente', ... } })
```

`Contact.status` is a **denormalized copy of the stage slug**, written at [ai.service.ts:559](../soylaika.backend/src/ai/ai.service.ts) as `data.status = statusChange`. So the revenue-adjacent queries read a string column, not a join. Freezing slugs keeps this working; it does not make it good. §7 covers it.

It is a copy only *after* the first classification, though. Before that the two disagree, which is §4.2.

### 4.2 `nuevo` is the weakest row in that table, and the reason matters elsewhere

The only code that looks up `slug: 'nuevo'` is [crm.service.ts:512](../soylaika.backend/src/crm/crm.service.ts), inside the CRM playground — it creates the demo contact, `name: 'Cliente de prueba'`, `phone: demo-<uuid>`. **Real leads never touch it.** Both production intake paths create a contact with no stage at all:

```ts
// whatsapp.service.ts:96 — and instagram.service.ts:111, identically
create: { phone, name: name ?? null, source: 'WhatsApp' }
```

No `stageId`. `status` comes from the Prisma column default, `@default("nuevo")`. So every real lead sits at `stageId = null, status = 'nuevo'` until the classifier first runs.

Three things follow, and each one lands somewhere else in this document:

- **`stageId = null` already means two things, and §6.3 wants to add a third.** §5 says "sin etapa" is the absence of a row. §6.3 wants that absence to mean *the model deliberately declined*. But today it also means *brand new, never classified* — every lead's first state, and by far the most common occupant of the bucket. §6.3 separates two of the three; the other two stay indistinguishable.
- **§4.1's "denormalized copy" is not a copy at creation.** `status = 'nuevo'` and `stageId = null` disagree by construction until the first classification. So a never-classified lead counts as `nuevo` in the stats queries and as "Sin etapa" on the funnel board ([crm.service.ts:733](../soylaika.backend/src/crm/crm.service.ts)) at the same time.
- **§7's proposed fix does not reach the real path.** "First enabled pipeline stage by `order`" changes the playground and nothing else, unless the two upserts change too — and that is a behaviour change (leads would start on the board instead of in "Sin etapa"), not a refactor. §7 names it rather than assuming it; §14 q6 asks it.

## 5. Two kinds of stage

Stages split into two groups, and this distinction carries the caps in the requirement:

| | Meaning | Cap |
|---|---|---|
| **Pipeline** (positive) | The phases of the commercial flow. A lead moving through them is progressing | **up to 5** enabled |
| **Out** (negative) | Where leads go when they leave the flow, or never enter it | **up to 3** enabled |

Today's defaults land exactly on that shape: `nuevo`, `interesado`, `cotizado`, `cerrando`, `cliente` are the five pipeline stages; `no-contesta` and `perdido` are two of the three out slots. **The third out slot is the only genuinely new row this PRD adds.**

**How much of that is actually free.** Four of the five pipeline slots are load-bearing (§4) and cannot be disabled: `nuevo`, `interesado`, `cotizado`, `cliente`. Of the out slots, `no-contesta` is load-bearing by slug, and `perdido` is the only `isLost` row — §6.2's escape-hatch rule is generated from it, so disabling it leaves no way out of the funnel. A tenant therefore gets **one free pipeline slot, one free out slot, and seven renames.**

That covers §1 and every variation of it, because the requirement is that stages *mean* different things, not that there be different numbers of them. It does not let a tenant design a funnel from scratch, and it should not be sold as though it did.

**"Sin etapa" is not a row.** It is the absence of one — `stageId = null` — which already exists and which the classifier already returns when nothing matches. What becomes configurable is the *guidance* for it: when should the model decline to classify at all (§6.3). It is also where every lead starts (§4.2), which is what makes §6.3 harder than it sounds.

### 5.1 `kind` is the concept the code has been missing

`no-contesta` is seeded `isWon: false, isLost: false` — **flag-identical to every pipeline stage**. It is semantically outside the commercial flow and nothing in the data says so. Which is why `message.processor.ts` has to write:

```js
if (currentStage && (currentStage.slug === 'no-contesta' || currentStage.isLost))
```

Half semantic, half hardcoded, because the semantic half does not exist. **`kind` is that missing half**, and adding it collapses several couplings in §7 rather than merely documenting them.

## 6. The prompt becomes generated

### 6.1 Two blocks, built from the two kinds

`FunnelStage` gains `criteria`, `kind` and `enabled`. The prompt presents the groups **separately**, because they mean different things to the classifier — one is progress through a flow, the other is exit from it:

```js
const enabled = stages.filter((s) => s.enabled);
const block = (kind) => enabled
  .filter((s) => s.kind === kind)
  .sort((a, b) => a.order - b.order)
  .map((s) => `- "${s.slug}" (${s.name}): ${s.criteria?.trim() || s.name}`)
  .join('\n');
```

rendered under two headings — *ETAPAS DEL FLUJO COMERCIAL* (pipeline, in order) and *FUERA DEL FLUJO* (out).

This also replaces the current duplicate listing: today the stages appear once as a bare list and again inside the hardcoded criteria (§3). One block, one source.

A stage with no criteria falls back to its name — degraded, not broken (§12).

### 6.2 The transition policy

Two rules in the prompt are not per-stage:

> *El estado normalmente solo avanza; no retrocedas salvo evidencia clara. EXCEPCION: "perdido" se puede marcar desde CUALQUIER etapa…*

The first is global policy. The second names a stage — but that stage is identifiable from data: it is the one with `isLost = true`. Generate it. The global sentence becomes a tenant-level setting alongside the criteria, so a business whose funnel legitimately moves backwards can say so.

### 6.3 "None of the above"

The classifier already returns `null` when nothing matches, and `null` is what the business calls *"sin etapa"* (§5). But today that happens **by accident** — the model returned something unparseable, or a slug not in the map.

Make it deliberate: a short, superadmin-editable line telling the model when to decline to classify. A conversation that is genuinely not a lead should land in "sin etapa" *because the model said so*, not because parsing failed. Those two outcomes are indistinguishable today, and only one of them is a bug.

### 6.4 The acceptance test

**No stage slug or stage name may appear as a literal in `ai.service.ts` after this**, with one named exemption: the `FALLBACK` array at [ai.service.ts:918](../soylaika.backend/src/ai/ai.service.ts), which belongs to a different function and is phase E (§7).

Scoped to `analyzeConversation` alone the grep would pass in phase A while five slugs sat two hundred lines below it. Scoped to the file, it is checkable with a grep, and it is the same acceptance test PRD 6 §4.0 sets for the eval suite: replacing every stage definition must require no code change.

### 6.5 The semantic friction, and why to measure it rather than pre-solve it

After a rename, the model sees `- "cotizado" (Crear reunión): cuando el cliente acepta agendar una reunión`. The key says one thing in Spanish and the criteria say another.

The model should key off the criteria text — that is what it is told to do, and the name is right there in parentheses. But `cotizado` carries real meaning, and a contradicting key is a plausible source of misclassification.

**Ship it as written and measure.** PRD 6's eval suite is exactly the instrument: run the corpus before and after a rename. If drift shows up, the fix is contained — render a neutral key in the prompt (`- "etapa-3"`) and map it back to the slug when parsing the response. That is a change to two functions, and it is not worth making speculatively.

## 7. The coupling — what `kind` fixes, and what it does not

Eleven sites across four files reach a stage by its literal slug, or by the `Contact.status` string that mirrors it. **`kind` resolves three of them outright**, which is the argument for adding it now rather than later:

| Site | Today | With `kind` |
|---|---|---|
| `message.processor` — is this contact out of the flow? | `slug === 'no-contesta'` OR `isLost` | `kind === 'out'` |
| `crm.service` — client count | `status === 'cliente'` | `isWon` |
| `ai.service` — `FALLBACK` array of five slugs | hardcoded | generated from enabled stages |

The rest need a further designation — because with up to three out stages *"which one"* becomes ambiguous, and because two of them ask a question `kind` does not answer:

| Site | What it still needs |
|---|---|
| `followup.processor` — move a silent contact | **Which out stage means "stopped replying".** A designation among the out stages, the same shape as `isWon`/`isLost` |
| `message.processor` — revive on a new reply | **Which pipeline stage a returning lead re-enters.** Currently `'interesado'`, the second stage. Could be a designation, or `previousStageId`, which already exists on `Contact` |
| `crm.service` — where do new contacts start? | **A decision, before a refactor.** The `slug: 'nuevo'` lookup is the playground's (§4.2); real leads start at `stageId = null`. "First enabled pipeline stage by `order`" is the right *mechanism*, but applying it to the two intake upserts changes what every tenant's board looks like on the day it ships — §14 q6 |
| `crm.service` — open quotes and ticket average | **A way to say "quoted, not yet won".** [crm.service.ts:698](../soylaika.backend/src/crm/crm.service.ts) and [775](../soylaika.backend/src/crm/crm.service.ts) name `cotizado` directly, and that concept is not `kind`, `isWon` or `isLost`. Either a third designation, or accept that it means "the last enabled pipeline stage before the won one" |
| `crm.service` — revenue by `status` | Arguably should not be stage-based at all — revenue is `Sale` rows (§4.1). Out of scope here, and worth its own look |

## 8. The frontend has its own hardcoded copy

**This section is why the feature would otherwise ship visibly broken.** The CRM does not read stage names from the API everywhere — three screens carry their own copy of the list.

**`contacts/page.tsx`** declares the stages as a TypeScript union and a style map:

```ts
type Status = "nuevo" | "interesado" | "cotizado" | "cliente" | "perdido";
const STATUS_CONFIG: Record<Status, { label; className; dot }> = { ... }
```

The label itself degrades correctly — [line 107](../soylaika.frontend/app/(crm)/contacts/page.tsx) is `contact.stage?.name ?? STATUS_CONFIG[status]?.label ?? …`, so it prefers the real stage name. But the **badge colour and dot** come only from `STATUS_CONFIG`, and the lead filter at line 359 is built from `Object.keys(STATUS_CONFIG)`. So after a rename the contacts list shows the new name with no styling, and the filter still offers the old five.

**And the fallback is the common path, not the rare one.** A lead with `stageId = null` — every lead before its first classification (§4.2) — has no `contact.stage`, so it renders `STATUS_CONFIG["nuevo"].label`: the hardcoded string "Nuevo". A tenant that renames `nuevo` sees the new name only on leads the classifier has already touched.

**That map is already wrong today,** independent of this PRD: it has five keys, and `cerrando` and `no-contesta` are missing — even though `followup.processor.ts:55` writes `status: 'no-contesta'` on its own. Contacts in those two stages already render unstyled and are already absent from the filter.

**`resultados/page.tsx`** reads `stats.byStatus.cliente`, `.perdido` and `.cotizado` directly ([lines 109-111](../soylaika.frontend/app/(crm)/resultados/page.tsx)), two of them behind a substring fallback over stage names — `includes("cliente")`, `includes("perdid")` — which itself breaks on a rename, and breaks *silently*, by falling through to zero. `inicio/page.tsx` reads `stats.byStatus.cliente` for its headline number ([line 225](../soylaika.frontend/app/(crm)/inicio/page.tsx)). Both pages also hardcode `href="/funnel?stage=cotizado"` with a fixed label.

**Scope this adds:** those three screens must read names, colours and the filter list from the stages endpoint. It is not large, but it is not optional — a rename that updates the funnel board and nothing else is worse than no feature.

## 9. Seeding and migration

The seven current criteria strings become the seeded default for `criteria` on the seven `DEFAULT_STAGES`. **Behaviour on day one is identical to today** — the text moves from a constant into rows.

- **Fix the `no contesta` / `no-contesta` mismatch while seeding** (§3.1). A one-character correction that can only be made once, at the moment the text becomes data.
- **`kind` is backfilled from what the stages already mean**: the five defaults become pipeline, `no-contesta` and `perdido` become out. Anything not matching a default slug defaults to **pipeline** and is flagged in the panel for confirmation — guessing "out" would silently take contacts off the board.
- **`enabled` defaults to true** for everything that exists.
- **Existing tenants get criteria backfilled by the same migration**, matched on slug. A tenant that added its own stages gets criteria for the ones that match and blanks for the rest — visible in the panel as something to fill in, which is the correct outcome.

**How the backfill actually reaches tenant databases.** Not through `seedDefaults`: it upserts with `update: {}` ([funnel.service.ts:30](../soylaika.backend/src/funnel/funnel.service.ts)), so adding `criteria` to `DEFAULT_STAGES` changes nothing for a row that already exists. The mechanism that works is `TenantMigrationsService` ([tenant-migrations.service.ts](../soylaika.backend/src/tenants/tenant-migrations.service.ts)), which replays every `prisma/migrations/*/migration.sql` against every registered tenant database at boot and records what it applied in `_tenant_migrations`.

So the backfill is **raw SQL**, not a Prisma seed — the seven criteria strings get embedded in a `.sql` file complete with accents, apostrophes and `$` (`"6 m², te queda en $270.000"`). Mechanical, but it has to be written rather than assumed.

### 9.1 Enabled, not deleted

"How many categories they use" is expressed by an `enabled` flag, not by creating and destroying rows.

A disabled stage does not appear in the prompt, cannot be assigned, and is hidden from the funnel board — **but the row survives**. That matters for three reasons: the load-bearing lookups in §4 still find something; historical contacts keep their stage, so last quarter's numbers do not change retroactively; and it is reversible.

Constraints the API must enforce:

- At most **5 enabled pipeline** and **3 enabled out** stages.
- At least **1 enabled pipeline** stage — a funnel with none classifies nothing.
- **The five load-bearing stages in §4 cannot be disabled at all** — `interesado`, `no-contesta`, `cliente`, `cotizado`, and `nuevo` (weakly, for the playground only: §4.2). They can be renamed and redefined; they cannot be switched off, because the code reads and writes them unconditionally. `perdido` is effectively a sixth: it is the only `isLost` row, and §6.2 generates the escape-hatch rule from it.
- Disabling a stage that contacts currently sit in is allowed, but the panel says how many will be left there.

## 10. Panel

On the funnel screen, each stage gains a criteria textarea, with the seeded default visible and a **"restore the default"** action — because someone will edit it, make it worse, and want the original back, and there is no history (the same problem [PRD 7](prd-agent-config-versioning.md) solves for agent prompts; cheaper to add now than later).

**The slug disappears from the UI entirely.** Not shown read-only — removed. It is an internal key, and showing it invites the question of why it cannot be changed. Two places, not one: the create input at [admin/funnel/page.tsx:130](../soylaika.frontend/app/(crm)/admin/funnel/page.tsx), and the monospace slug rendered beside every row of the list at [line 200](../soylaika.frontend/app/(crm)/admin/funnel/page.tsx).

**Writing guidance beside the field**, because the quality of this text decides the quality of the classification:

- Say what *the customer* did, not what the seller should do.
- Give the phrases customers actually use — the current `perdido` criteria list six, and that is why it works.
- Say what does **not** count. Half the value in the current `cotizado` text is *"Mencionar precios de lista o por m² NO alcanza"*.

## 11. Making it safe

The criteria are free text written by a person and injected into a system prompt that runs on every conversation. That deserves more thought than "add a column".

### 11.1 The mass-assignment hole, which is live today

`POST /api/funnel/stages` and `PATCH /api/funnel/stages/:id` are `@Roles('admin', 'superadmin')` — **tenant admins** — and both forward the request body into Prisma unfiltered:

```ts
// controller — the @Body() type is TypeScript, erased at runtime
update(@Param('id') id, @Body() body: { name?; color?; notifyOnEnter?; isLost?; isWon? }, @Req() req) {
  return this.funnel.update(id, body, req.tenantDb);
}
// service
return db.funnelStage.update({ where: { id }, data });      // data IS the raw body
return db.funnelStage.create({ data: { ...data, order } }); // create spreads it too
```

There is no global `ValidationPipe` in this project, so nothing enforces those signatures. Prisma accepts any real column it finds.

**This is already exploitable, before this PRD.** A tenant admin's token can `PATCH` `{"slug": "otra-cosa"}` and rename a stage, breaking the §4 lookups; or `POST` a second stage with `isWon: true`, double-counting clients.

**One qualifier, because it changes the severity.** The browser cannot do the rename: the slug input is `disabled={!!editingId}` ([admin/funnel/page.tsx:130](../soylaika.frontend/app/(crm)/admin/funnel/page.tsx)), so the UI only sets a slug on create. This is an API-level hole reachable with a token, not a button anyone can click by accident. Real, but not urgent in the way a clickable version would be.

Adding `criteria`, `kind` and `enabled` makes it worse: each new column becomes tenant-admin writable with no code change and no decision. `criteria` in particular is write access to a system prompt.

So phase B is:

- **Named-field allowlists** on both `create` and `update`, forwarding only the fields each is meant to accept — never the raw body. This closes `slug` permanently, which §2 requires anyway.
- **A superadmin-only route** for the new fields — `PATCH /tenants/:slug/funnel/stages/:id`, on `TenantsController`, which is superadmin by class.
- **Creation moves to superadmin.** Creating a stage mints a permanent slug and picks a `kind`; both are irreversible decisions, and the caps in §9.1 have to be enforced somewhere.

**This is why phase A cannot ship alone.** The surface already exists and already forwards whatever it is given; A simply puts something worth taking behind it.

### 11.2 Keep the format instruction last

The prompt ends by demanding a specific JSON shape. **That instruction must stay after the criteria block, always.** Criteria sit in the middle of the prompt; the closing format rule is what stops a careless — or hostile — criterion from redefining the output. It happens to be ordered correctly today. Make it a rule rather than an accident, and put the criteria inside a delimited region so it is visibly a data block rather than more instructions.

### 11.3 Validate on write

Criteria are prose about what a customer did. They never legitimately need braces, JSON, or instructions to the model.

- **Reject `{` and `}`** — the only reason for them here is to interfere with the output shape.
- **Cap the length** (§13 q3). The whole block ships on every classification call.
- **Flag imperative-to-the-model phrasing** — *"respondé"*, *"ignorá"*, *"formato"*, *"en vez de"*. A warning, not a block: legitimate criteria describe the customer, so a criterion addressing the model is usually a mistake and occasionally an attack.

### 11.4 Fail safe, not empty

If every stage has blank criteria — a bad migration, a bulk delete — the prompt must fall back to the seeded defaults, not ship an empty `Criterio:` block. Classification silently degrading to name-guessing is worse than a stale default.

Keep the seeded default stored **alongside** the current value rather than only in code, so "restore the default" still works after the constant is eventually deleted.

### 11.5 Proportionality: what the blast radius actually is

Worth stating plainly, because it should govern how much machinery to build: `analyzeConversation` **never produces text sent to a customer.** It returns a stage and extracted CRM fields.

The worst case of a bad criterion is misfiled leads and corrupted `Contact.details` — bad, and invisible for a while, but not "the bot told a customer something false". This is a lower-severity surface than agent prompts (PRD 7), and the controls should be lighter in proportion: a role boundary, a validation, a restore button, and detection via PRD 6 — not an approval queue.

### 11.6 A pre-existing hole this PRD does not open

`DELETE /api/funnel/stages/:id` is available to tenant admins **today**, and §4 shows what losing a load-bearing row costs: follow-ups stop parking anyone, silent leads are never revived, the client count reads zero. That is true before this PRD and independent of it.

**One qualifier, the same kind §11.1 needs.** `remove` is not unguarded ([funnel.service.ts:73](../soylaika.backend/src/funnel/funnel.service.ts)):

```ts
const count = await db.contact.count({ where: { stageId: id } });
if (count > 0) throw new BadRequestException(`No se puede eliminar: hay ${count} contactos en esta etapa`);
```

So a load-bearing stage can only be deleted while it is **empty** — a fresh tenant, or `no-contesta` before the first follow-up fires. Real, but not the "one request and the funnel breaks" it would otherwise read as.

The sharper problem is *what* the guard protects. It keys on contacts currently sitting in the stage, not on whether any code depends on the row. Those are different properties, and only the second one matters here: an empty `no-contesta` is exactly as load-bearing as a full one, and it is the only one of the two that can be deleted.

Worth fixing in the same pass, because §9.1 already requires the panel to disable deletion — doing it in the UI while the endpoint stays open would be a control that is not a control.

## 12. Interaction with the eval suite

PRD 6's corpus contains scenarios whose `expects` include stage transitions. Once criteria are tenant data, they become part of the configuration fingerprint (PRD 6 §4.0) — a criteria edit is a configuration change, and a classification that shifts afterwards is expected rather than a regression.

This is also the cheapest way to tell whether an edited criterion is better, and the instrument for §6.5: run the suite before and after.

## 13. Phasing

| Phase | Scope |
|---|---|
| **A** | `criteria`, `kind`, `enabled` columns, seeded and backfilled (§9); prompt generated as two blocks (§6.1); the `no-contesta` fix |
| **B** | Allowlists on create and update; superadmin-only route; creation and deletion closed at the API (§11.1, §11.6); validation (§11.3) |
| **C** | Panel: edit title and criteria, enable/disable within the caps, restore default, slug removed from the UI |
| **D** | Frontend: the three screens in §8 read names, colours and filters from the API |
| **E** | Replace the three slug lookups `kind` covers outright (§7); decide the designations the rest need |
| **F** | Transition policy (§6.2) and the "none of the above" line (§6.3) as tenant settings |

**A alone is worth shipping** — it changes no behaviour and removes the second source of truth, which is the actual defect. But **A must not ship without B**: the moment `criteria` exists as a column it is writable through the tenant-admin endpoint, so the role boundary lands in the same release.

**C must not ship without D.** C is what makes renaming possible; D is what makes a rename look right everywhere. Shipping C alone produces a tenant whose funnel says "Crear reunión" and whose contacts list says "Cotizado".

## 14. Open questions

> Cuatro de estas se decidieron al implementar; quedan anotadas con su respuesta
> en lugar de borrarse, porque el *por qué* es lo que no se puede reconstruir
> después. La q2 sigue abierta.

1. **The designations §7 still needs.** ~~Which out stage means "stopped replying", where a revived lead re-enters, how "quoted but not yet won" gets expressed once `cotizado` is renameable, and whether revenue should move off `Contact.status` entirely.~~ **Deferred, deliberately.** Phase E replaced only the three lookups `kind` covers outright; the other three still resolve by slug, which is safe precisely because §2 freezes the slug. Inventing an `isNoAnswer` flag to satisfy a lookup that already works would have been speculative. Still what would make real *deletion* safe — but deletion is closed now (§11.6), so nothing depends on it.
2. **Should `Contact.status` exist at all?** *(still open)* It is a denormalized copy of the stage slug (§4.1) that the stats queries read instead of joining. Freezing slugs keeps it correct, so this is not urgent — but it is the reason `cliente` is load-bearing.
3. **How long can a criteria field be?** **Decided: 500 characters per stage** (`MAX_CRITERIA_CHARS`), enforced on write and shown as a live counter in the panel. The seven seeded criteria average ~170 characters and the longest is ~300, so 500 is generous headroom while keeping the whole block under ~4 KB even with all eight slots full.
4. **Should criteria be per-agent as well as per-tenant?** `analyzeConversation` runs once per conversation regardless of which agent replied. Probably not, but worth naming before someone assumes it.
5. **Does the third out slot have a use yet?** **Decided: not seeded at all.** The cap allows three and superadmin can create one when a tenant needs it. Seeding a blank third row would put a chore in every tenant's panel; seeding it disabled would put a mystery there. Not seeding is cheaper than both and equally reversible.
6. **Should a new lead start on the board, or in "sin etapa"?** **Decided: unchanged — leads keep starting at `stageId = null`.** Phase E removed the `'nuevo'` literal from the CRM playground (§4.2), which is the only code that ever read it, and left the two intake upserts alone. Moving new leads onto the board is a product decision that would change every tenant's funnel on the day it ships, not a refactor — and §6.3's "sin etapa" still means three things until it is made.

## 15. Risks

- **Someone edits a criterion and the classifier gets worse, silently.** Conversations are misfiled for weeks before anyone notices. Mitigated by "restore the default" (§10) and properly by PRD 6 — this feature makes prompt-quality mistakes easier to introduce, and PRD 6 is what detects them.
- **C ships without D** (§13) and a rename shows up on one screen out of four. The most likely way this feature looks broken to the person who just used it.
- **`criteria` ships as a plain column and is immediately tenant-admin writable** (§11.1). Happens by omission rather than by decision — nobody has to do anything wrong beyond adding the column and reusing the existing screen.
- **The renamed key confuses the classifier** (§6.5). Unquantified by design; PRD 6 is how it gets quantified, and the fix is known and small if it materialises.
- **Criteria drift from the business.** The text says "cotizado is when you gave a total" long after the business changed how it quotes. Nothing detects this; the same class of staleness as PRD 6 §6.5.1.
- **Empty criteria degrade classification quietly.** A stage with a blank field still appears in the list, so the model guesses from the name. Correct behaviour, but the panel should show which stages have no criteria rather than leaving it to be discovered.
- **"Sin etapa" still means three things** (§4.2, §6.3). The deliberate decline §6.3 introduces lands in the same bucket as every never-classified lead and every parse failure, so nothing can show a tenant which conversations the model actually rejected. §6.3 is worth doing regardless, but it does not deliver the visibility it sounds like it does.
