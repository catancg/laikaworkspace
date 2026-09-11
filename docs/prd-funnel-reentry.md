# PRD 15 — Funnel re-entry for customers who already bought

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 8 — Per-tenant funnel stage names and criteria](prd-funnel-stage-criteria.md) — §7 and §14 q1
  left "which stage a reactivated lead returns to" **deliberately undesignated**. This PRD designates
  it. §2's frozen slugs and §9's "day one behaves identically" both constrain the design below.
- [PRD 12 — Contact lifecycle event log](prd-contact-lifecycle-events.md) — the reason this PRD is
  small. See §2.

---

## 1. Why this exists

The requirement, as given:

> Quiero que los clientes que ya compraron puedan hacer todo el staging del funnel de nuevo — puede
> volver a "interesado". La definición de estas etapas y cómo categorizan se configura en prompts que
> el superadmin actualiza y refina.

### 1.1 Re-entry already exists, for the wrong half of the funnel

When a lead who is **out of the flow** writes again, they are already moved back to `interesado`:
[message.processor.ts:90](../soylaika.backend/src/queue/message.processor.ts). The condition is
`currentStage.kind === 'out'`, which covers `no-contesta` and `perdido`.

`cliente` is `kind: 'pipeline'`, `isWon: true`
([funnel.service.ts:29](../soylaika.backend/src/funnel/funnel.service.ts)). It is not `out`, so it
never hits that path. **A customer who bought and writes again stays in `cliente` forever**, and the
classifier is separately instructed that *"el estado normalmente solo avanza"*, so it will not move
them either.

That is the whole gap: one condition covers the leads who left, and nothing covers the leads who
bought.

### 1.2 The thing that actually breaks if you only fix that

Won counts are computed from **where the lead is standing right now**
([crm.service.ts:827](../soylaika.backend/src/crm/crm.service.ts)):

```ts
const wonIds = stages.filter((s) => s.isWon).map((s) => s.id);
const clientCount = byStage.filter((s) => s.isWon).reduce((n, s) => n + s.count, 0);
const prevClientCount = await db.contact.count({ where: { stageId: { in: wonIds }, ... } });
const clientsBySource = await db.contact.groupBy({ by: ['source'], where: { stageId: { in: wonIds } } });
```

So the moment a customer re-enters the funnel, **the client count drops by one**. Every returning
customer would decrement the historical win total, and the conversion rate with it.

This is not a new bug introduced by re-entry — it is a latent one that re-entry *detonates*. It is
already mildly wrong today: a customer who bought and was later marked `perdido` also stops being
counted as a client.

**§4 is therefore the load-bearing half of this PRD.** Shipping §3 without it makes the dashboard
lie in proportion to how well the feature works.

---

## 2. What this PRD is not: the `Opportunity` entity

Four PRDs (11, 12, 13, 14) each close with the same note — their table gains a nullable
`opportunityId` when the opportunity entity lands. This PRD was expected to be that entity.

It is not, and the reason is [PRD 12](prd-contact-lifecycle-events.md).

The justification for an entity was that `Contact` *is* the lead, permanently, so a returning
customer overwrites their own funnel history. That was true when the four were written. It stopped
being true when `ContactEvent` shipped: every stage transition is now recorded with its actor and
timestamp, so moving a won customer back to `interesado` destroys nothing.

What an entity would still buy is **per-cycle grouping** — "which purchase cycle did this quote
belong to". That is derivable: a cycle is the span between one re-entry event and the next, and
`QuoteCalculation`, `Sale` and `AcquisitionTouch` all carry `createdAt`. Derived is worse than
direct, but not by enough to justify a refactor across ~55 backend references and twenty frontend
files — three of which keep their own hardcoded copies of the stage enum.

The `opportunityId` hooks cost nothing while unused. If per-cycle analysis later proves genuinely
painful to derive, the entity is still available and the four PRDs still say where it plugs in.

**This section is the record of that decision.** A future reader finding four PRDs that promise an
entity, and no entity, should find this paragraph rather than assume it was forgotten.

---

## 3. Part one — re-entry becomes a designation, not a condition

### 3.1 Two flags on `FunnelStage`, replacing a hardcoded condition and a frozen slug

```prisma
model FunnelStage {
  // ...
  // Un lead parado en esta etapa vuelve al flujo cuando escribe de nuevo.
  reentersOnReply Boolean @default(false)
  // La etapa a la que vuelve. Exactamente una por tenant (§3.3).
  isReentryTarget Boolean @default(false)
}
```

The current code hardcodes both halves: the source is `kind === 'out'`, and the destination is
`findFirst({ where: { slug: 'interesado' } })`. The comment above that lookup already says it is
provisional and only safe because the slug is frozen by PRD 8 §2.

Making both halves data is what the requirement asks for: *"la definición de estas etapas y cómo
categorizan se configura"*. `cliente` opting into re-entry becomes a row the superadmin edits, not a
condition someone widens in code.

It also removes one more load-bearing slug, which PRD 8 §4 lists as a standing hazard.

### 3.2 Day one behaves identically

Seeds: `reentersOnReply = true` for `no-contesta` and `perdido`; `isReentryTarget = true` for
`interesado`. Everything else false, **including `cliente`**.

So the migration changes nothing on its own, and the feature the requirement asks for is switched on
per tenant, deliberately, by a person. This follows PRD 8 §9: a change to the funnel's meaning should
not alter every tenant's bot behaviour the moment it deploys.

### 3.3 Exactly one re-entry target

`isReentryTarget` must be true for exactly one enabled stage. Zero means a lead standing in a
re-entering stage has nowhere to go — and the current code fails *silently* there (`findFirst`
returns null, the `if` is skipped, nothing happens). Two means the destination depends on row order.

Validated on write, next to PRD 8's existing stage-count limits in
[funnel-criteria.ts](../soylaika.backend/src/funnel/funnel-criteria.ts), with the same shape: reject
the write, name the reason.

The read path keeps the existing degrade — if no target is configured, do nothing — but now logs a
warning instead of silently skipping, because after this PRD an unconfigured target is a
misconfiguration rather than a normal state.

### 3.4 What re-entry does not do

It does not clear `lostReason`, `value`, `details`, or the acquisition touches. Those belong to the
contact, not to a cycle. A returning customer keeps everything known about them; only their funnel
position resets.

---

## 4. Part two — a win stops being a function of current position

### 4.1 `Sale` rows are the tempting source, and the wrong one

The obvious fix is to count clients as contacts with at least one `Sale`. It is wrong here: `Sale`
rows are created **by hand** by a salesperson (`createSale`), while `cliente` is set by the
**classifier** when the customer says they paid. The two diverge routinely, and a contact sitting in
`cliente` with no `Sale` row is normal.

Switching to `Sale` would silently restate every historical number in the panel. That is a different
decision — arguably a good one — and it is not this PRD's to make.

### 4.2 The rule: currently won, or ever won

A contact counts as a client if they are **in a won stage now, or ever entered one**, the second
read from `ContactEvent`:

```sql
stageId IN (won)  OR  EXISTS (SELECT 1 FROM "ContactEvent"
                              WHERE "contactId" = "Contact".id
                                AND "toStageId" IN (won))
```

Properties that make this the right rule here:

- **It is never smaller than today's number.** The first half is exactly the current query, so no
  existing client disappears from the panel the day this ships.
- **It survives re-entry**, which is the point.
- **It fixes the latent bug in §1.2** — a customer who bought and was later marked `perdido` starts
  being counted again.

The cost is that `ContactEvent` has no history before PRD 12 shipped and was deliberately not
backfilled. So a contact who was a client and left *before* that date is still missed. The union
means this degrades to exactly today's behaviour rather than to something worse, and the gap closes
on its own as events accumulate.

### 4.3 Three call sites

`clientCount`, `prevClientCount` and `clientsBySource` in
[crm.service.ts](../soylaika.backend/src/crm/crm.service.ts) all derive from `wonIds` against current
`stageId`. All three take the union rule. `conversionRate` follows from `clientCount` and needs no
separate change.

**`clientsBySource` deserves a note:** it groups by `Contact.source`, which PRD 13 §1.2 established
holds the *channel* and never the acquisition source. This PRD does not fix that — it only stops the
row set from shrinking when customers return.

---

## 5. Migration

One migration: two boolean columns with `DEFAULT false`, plus the `UPDATE`s that set the seeds in
§3.2 — scoped by slug, and only where the column is still at its default, so a tenant who has already
configured something is never overwritten.

This is the first migration in the series with an `UPDATE` in it. It is safe because it writes only
to columns this same migration created, and because the values it writes reproduce the behaviour that
already exists in code.

Same `DO`-block idiom as PRDs 11–14 for idempotency, and the directory name must sort after
`20260915120000_contact_event` and be settled before it is pushed.

---

## 6. Testing

The seeds and the validation are pure logic and go in
[funnel-criteria.spec.ts](../soylaika.backend/src/funnel/funnel-criteria.spec.ts) next to PRD 8's
limits, with adversarial inputs: zero targets, two targets, a target that is disabled.

The re-entry rule itself needs a test that a **won** stage with `reentersOnReply` moves the lead, and
one that a won stage **without** it does not — the second is what guarantees §3.2's "day one behaves
identically."

The metrics rule needs a test that the count **does not drop** when a contact leaves a won stage,
which is the entire point of §4 and the one that fails today.

As with PRDs 11–14: `tsc` proves nothing here, because `db` is `any` and a wrong query shape
type-checks clean.

---

## 7. What this PRD does not do

- **No `Opportunity` entity.** See §2.
- **No frontend.** The three CRM screens that keep hardcoded copies of the stage enum are untouched;
  the panel control for the two new flags is a separate, small piece of work. Until it exists the
  flags are set the way funnel criteria were before PRD 8's panel — directly, by a superadmin.
- **No change to how `cliente` is reached.** The classifier's criteria for a won stage, and the
  "el estado normalmente solo avanza" instruction, are untouched. Re-entry is a code rule about a lead
  who writes again, not a model decision.
- **No change to `Contact.source`.** See §4.3.
