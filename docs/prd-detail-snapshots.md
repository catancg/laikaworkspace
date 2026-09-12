# PRD 17 — Detected data: queryable, and kept per cycle

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 12 — Contact lifecycle event log](prd-contact-lifecycle-events.md) — §5.2 excluded arbitrary
  field history on purpose. This PRD is the narrow exception, for one field, at two moments.
- [PRD 16 — Automatic lead disqualification](prd-lead-disqualification.md) — **shipped and
  deployed.** While this PRD was being written, 16's draft announced that its §6.3 would *replace*
  PRD 15's two re-entry booleans. **It does not** — the final §6.3 keeps them and adds a nullable
  `reentryTargetId` override that falls back to the global target. §4.2 below was written to be
  independent of that schema either way, which is still the right call for a different reason than
  the one first given.
- [PRD 15 — Funnel re-entry](prd-funnel-reentry.md) — made the problem in §1.2 reachable in practice.

---

## 1. Why this exists

The requirement, as given:

> Durante la conversación hay una sección "Datos detectados" — Producto de interés: empapelados,
> Ambiente: cocina. Todo eso tiene que quedar atado a la entrada del lead. Necesito el dato presente
> en la base para métricas.

The capture already exists. `Contact.details` is a JSON column filled by the classifier, which
extracts **24 fields** spontaneously from what the customer says — `productInterest`, `room`,
`material`, `city`, and twenty more. The CRM renders them as "Datos detectados" on the Clientes
screen. In the requirements coverage this is the row *"Data collected by Laika — already existed."*

Two things are wrong with it, and they are different problems with different fixes.

### 1.1 It is a blob, so none of it can be counted

`details` is a single `Json` column. Answering *"how many leads wanted wallpaper for a kitchen"*
means scanning JSON across every row. This is also why the coverage matrix marks Email and
ZIP/location as *partial* rather than covered — they are captured, and they are not queryable.

This is the half the requirement is actually about: **métricas**.

### 1.2 It overwrites, so the lead's own history is lost

The merge in [ai.service.ts:574](../soylaika.backend/src/ai/ai.service.ts) is:

```ts
data.details = { ...prev, ...details };
```

Accumulate, last-write-wins per key. One object per contact, forever.

- Cycle 1 — the customer wants wallpaper for the **cocina**. `room: "cocina"`.
- Months later, cycle 2 — the same customer, now the **living**. `room: "living"`.

`"cocina"` is overwritten. `ContactEvent` does not help: PRD 12 §5.2 records lifecycle transitions
only, not arbitrary field changes. So the quote from the first cycle survives in `QuoteCalculation`
while the requirements that produced it do not.

PRD 15 made this reachable on purpose: customers who bought can now run the funnel again.

### 1.3 What is recoverable, stated precisely

An earlier draft of this design said the lost values were "unrecoverable". That was wrong, and the
correction changes the plan:

| | Status |
|---|---|
| **Current detected values**, every contact | **Fully present.** Nothing lost — just unqueryable. |
| **Superseded values** (cycle 1's `"cocina"`) | Gone from `details`, but the **messages that produced them are intact**. Un-extracted, not lost. |
| **Historical cycle boundaries** | Genuinely absent before PRD 12. `Sale.createdAt` is a usable proxy for won cycles. |

That first row is the important one: metrics over the whole existing customer base need **no LLM and
no reconstruction**. They need the blob's current contents copied into columns.

---

## 2. Three parts, and only one is blocked

| | What | Blocked? |
|---|---|---|
| **A** | Promote eight fields to columns, backfilled from `details` | no |
| **B** | Snapshot `details` per cycle, going forward | **yes** — waits for PRD 16 |
| **C** | Re-extract superseded values from historical messages | not in this PRD |

**C is deliberately excluded.** It is a batch job re-running the classifier over every historical
conversation segmented by `Sale.createdAt`, at OpenRouter cost proportional to message volume. It is
possible, and it is a separate decision with a price attached that should be estimated before it is
made, not folded into a schema PRD.

---

## 3. Part A — the eight fields that become columns

### 3.1 Which, and why these

Promoted because each one is either a row of the requirements table currently stuck at *partial*, or
an axis you would group by:

| Column | Requirement row it unblocks |
|---|---|
| `email` | Identity — Email (MVP1) |
| `city`, `province`, `postalCode` | Identity — ZIP code / location (MVP1) |
| `productInterest` | Product — Product inquired about (MVP1), *"Demand"* |
| `room`, `material` | Product — Product category (MVP1), *"Aggregate analysis"* |
| `customerType` | particular / arquitecto / constructora / revendedor — segmentation |

The other sixteen stay in `details`. `unit`, `betweenStreets`, `references`, `wallCondition` and the
rest are transactional details of one job, not things anyone groups by. Promoting all 24 would be a
wide table whose extra width buys nothing.

### 3.2 Dual-write, and the duplication being accepted

`details` keeps **all 24 fields unchanged**. The promoted eight are *also* written to columns.

This is two copies of one fact, which this series has argued against repeatedly — PRD 11 §4.1
refused a `role`-shaped duplicate for exactly this reason. Three things make it the right call here
anyway:

- **They cannot diverge.** One statement writes both, in the same merge in `ai.service.ts`, from the
  same object. There is no second writer to forget.
- **Removing them from `details` breaks the CRM.** The "Datos detectados" panel renders the blob
  directly through `DETAIL_LABELS`, in a repo this PRD does not touch. Dual-write keeps that working.
- **It is reversible in the safe direction.** Once the CRM reads columns, dropping the eight keys
  from `details` is a follow-up. The reverse — discovering you needed the blob after deleting it —
  is not.

The follow-up is named so it does not get lost: when the CRM reads the columns, `details` should stop
carrying those eight.

### 3.3 The backfill is a copy, not a reconstruction

```sql
UPDATE "Contact" SET "productInterest" = "details"->>'productInterest'
  WHERE "details" ? 'productInterest' AND "productInterest" IS NULL;
```

One statement per field. No model call, no inference, no possibility of inventing a value: it reads
a key that is already there and writes it beside itself. Re-running is a no-op because the guard is
`IS NULL` on a column this migration created — and unlike PRD 15 §5.1's mistake, a NULL here really
does mean "never copied", because nothing else writes these columns yet.

After it runs, every contact that has ever had a detected `room` can be grouped by it.

### 3.4 This is the heaviest migration of the series

Eight columns plus an `UPDATE` **over every `Contact` row in every tenant database**. PRDs 11–15
added columns, created tables, or touched a handful of `FunnelStage` rows.

It is still safe — it writes only to columns the same migration creates, from a column that already
exists, with a guard that makes it idempotent. But the row count should be checked on the largest
tenant before it runs, and `premigrate-tenants.js` matters more here than anywhere previous.

---

## 4. Part B — the snapshot, and why it waits

### 4.1 The table

```prisma
model ContactDetailSnapshot {
  id        String   @id @default(uuid())
  contactId String
  contact   Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  eventId   String?  // the ContactEvent that caused it — weak reference, see below
  reason    String   // 'won' | 'reentry'
  details   Json     // Contact.details exactly as it stood
  createdAt DateTime @default(now())

  @@index([contactId, createdAt])
}
```

A **faithful copy**, not a restructure. The snapshot is for reading one lead's history — *"what did
they want when they bought last March"* — and the aggregate question is Part A's job. Splitting the
snapshot into columns as well would be a third copy of the same fact.

`eventId` is a weak reference for the same reason `productId` is in PRD 14 §8.1: it joins while the
event exists and survives if it does not.

Two moments, two reasons: **`won`** captures what they bought on, **`reentry`** captures how a cycle
ended when it closes without a purchase. Taking only one leaves a real case blind — a lead that goes
`perdido` and returns has no win to snapshot, and a customer who buys and never returns has no
re-entry.

### 4.2 The trigger is an event, not a stage flag — and that is the point

This is the design consequence of PRD 16 landing at the same time.

The obvious implementation reads the stage rows: snapshot when `toStage.isWon`, or when
`fromStage.reentersOnReply`.

**Corrected while PRD 16 was in flight.** This section originally said that second half was "exactly
the schema PRD 16 §6.3 replaces". It is not: 16's final §6.3 is titled *"A per-source override, not a
replacement"*, keeps PRD 15's two booleans, and adds a nullable `reentryTargetId` on the source row
that falls back to the global target when NULL. It considers the collapse and rejects it, because
replacing a shipped and migrated schema costs a destructive migration across every tenant database
to benefit one seeded row.

So the re-entry columns are stable after all. Reading them would not have broken.

So this PRD does not read those columns at all. The snapshot hangs off **what
`ContactLifecycleService` already knows it is doing** — it is the chokepoint every transition goes
through, and it decides "this is a re-entry" however the schema of the day expresses it. Whether
re-entry is two booleans, one FK, or something PRD 16 has not written yet, the trigger is unchanged.

The decision stands anyway, for the reason that survives: the snapshot is about *what the lifecycle
service is doing*, not about how the funnel happens to be configured. Reading `reentersOnReply` here
would couple a history feature to a configuration flag it has no business knowing about — and it
would have to change again the day a third mechanism decides what counts as re-entry.

**Part B is no longer blocked.** PRD 16 is merged and deployed; the sequencing note that used to sit
here is spent.

### 4.3 Guarded, not atomic — unlike PRD 12

PRD 12 put the event **inside** the transaction with the contact write: a lead that moves without an
event makes the log lie about whether something happened, and a log with silent holes is worse than
no log.

A snapshot is not that. A missing one loses enrichment; it does not make anything else false. And if
it were atomic, a snapshot failure would fail the **stage change** — the customer-facing path, on the
message that triggered it. That is the failure this codebase has already shipped once, when a
transcription error took the whole audio message down with it.

So the write is guarded: same transaction, wrapped, failure is a `logger.warn`.

**The cost, stated rather than buried:** a cycle can close without its snapshot and nothing will tell
you. That is the trade, and it is the right way round — but it is a trade.

---

## 5. What this PRD does not do

- **No re-extraction of history.** See §2, part C. The superseded values stay in the messages.
- **No frontend.** The "Datos detectados" panel keeps rendering `details` and does not change. A
  screen that shows a lead's snapshots over time is separate work.
- **No change to what the classifier extracts.** The same 24 fields, the same prompt. This PRD moves
  and preserves what it already produces.
- **No change to the re-entry schema.** PRD 16 owns that. See §4.2.

---

## 6. Testing

**Part A** is mostly SQL, and the risk is in the backfill rather than the code:

- the promotion writes both the column and the `details` key from one merge — asserted on the object
  handed to `contact.update`, in the shape `ai.service` specs already use
- a contact whose `details` has no `room` gets `NULL`, not the string `"null"` — the classic
  `->>` trap
- re-running the backfill changes nothing
- a detected value containing quotes or an accent survives the copy intact — the encoding case that
  bit the PRD 13 verification, where a mangled shell made the app look wrong

**Part B**: a won transition writes a snapshot with `reason='won'`; a re-entry writes one with
`reason='reentry'`; a stage change that is neither writes none; and — the one that matters — **a
failing snapshot write does not fail the stage change**.

As with every PRD in this series, `tsc` proves nothing here: `db` is `any`, so a wrong query shape
type-checks clean.

---

## 7. Sequencing

**Part A can ship independently and first.** It touches `Contact`, the classifier merge, and nothing
in the funnel. It is also the half the requirement was actually about — metrics over the existing
customer base.

**Part B is unblocked.** PRD 16 merged and deployed without touching PRD 15's re-entry columns, so
§4 needs no revision — which §4.2 was written to guarantee regardless of which way that went.
