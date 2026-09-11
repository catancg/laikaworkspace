# PRD 14 — Quote capture

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 13 — Acquisition attribution](prd-acquisition-attribution.md) — §5.1 there is the failure-isolation
  rule this PRD reuses: a nice-to-have write must never cost a customer message.
- **PRD 15 — Opportunity entity** (not yet written) — the other half of what was originally one project.
  See §7.

---

## 1. Why this exists

Fourth project from the ~90-field customer-data dictionary (PRD 11 §1), covering the Quote group and the
price-related rows of Product/Interest.

Quoting in this product is conversational. The bot is told to give a concrete total — the `cotizado`
stage criteria is *"SOLO cuando se le dio un TOTAL/presupuesto concreto (ej: 6 m2, te queda en
$270.000)"* — and that number then exists only as prose inside a chat message. Nothing records what was
quoted, for which product, at what price, or when.

### 1.1 The number is already structured, and already correct

It does not have to be parsed back out of prose. `calcularPrecioM2` at
[ai.service.ts:643](../soylaika.backend/src/ai/ai.service.ts) computes it server-side:

```ts
const m2 = ancho * alto;                 // sin redondeo (m² exactos)
const total = Math.round(m2 * precioM2);  // total en pesos enteros
return { title, ancho, alto, m2, precioM2, total };
```

When the call carries a SKU the unit price comes from the catalogue row, not from the model — a
deliberate choice already recorded in the code: *"si viene SKU, se usa el precio real del catálogo (más
confiable que el que pueda pasar el modelo)"*. The tool exists precisely because the model's own
arithmetic is not trusted, and the agent prompt says to use it always for m²-priced products rather than
calculating by hand.

That object is serialised into the tool-result message, handed back to the model, and discarded.

So this PRD needs **no prose parsing and no new AI work**. It persists a result that is already computed
and already right.

### 1.2 What is lost by not storing it

- The quoted total, and which product it was for
- **The unit price at the moment of quoting.** `Product.price` is mutable and the catalogue is
  re-imported; once it changes, what a lead was quoted last month is unrecoverable from anywhere.
- The measurements the customer gave, which are the closest thing to a specification of the job

---

## 2. The design, in one line

One table, `QuoteCalculation`, written from inside the pricing tool.

---

## 3. The name is the design

The table is **not** called `Quote`.

A row records *the bot calculated this total*. That is close to, but not the same as, *the customer was
quoted this*. The model can call the tool and then not use the number, or call it again with different
measurements, or compute a price for an option the customer immediately rejects. The tool result is an
input to the reply, not a record of the reply.

Calling the table `Quote` would invite a report that treats a calculation as a commitment — the same
class of error as PRD 13 §3.1, where two labels backed by one absence of evidence would have produced a
number that looked like measurement. Naming it for what it is keeps the gap visible to whoever writes
that report.

The gap is narrow in practice: the tool is called *in order to* quote, and the agent is instructed to use
its exact output. It is narrow, not zero, and the name is where that is recorded.

---

## 4. The schema

```prisma
model QuoteCalculation {
  id           String   @id @default(uuid())
  contactId    String
  contact      Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  productId    String?  // resolved when the tool was called with a sku
  sku          String?
  title        String?
  widthM       Float?
  heightM      Float?
  squareMeters Float?
  unitPrice    Float    // price per m² at the moment of calculation
  total        Float
  createdAt    DateTime @default(now())

  @@index([contactId, createdAt])
  @@index([productId])
}
```

`productId`, `sku` and `title` are nullable because the tool accepts a bare `precioM2` with no SKU —
the model may price a wall against a number the customer supplied. Those rows are still worth keeping:
the total was still quoted.

`unitPrice` is stored **even when a SKU was passed**. It is `product.price` as it was at that moment,
and the catalogue is mutable and re-imported from a feed. This is the dictionary's "current price at the
time" row, and it is the one field here that cannot be reconstructed later by joining to `Product`.

### 4.1 No deduplication

Each calculation is its own event. Three walls produce three rows. The same wall priced in two materials
produces two rows — that is the customer comparing options, which is information rather than noise.

Unlike PRD 13 §4.2, there is no retry concern to design around: the tool loop runs inside a single reply,
and a repeated calculation is a real repeated calculation.

---

## 5. Capture point

Inside the `calcularPrecioM2` case, after `total` is computed and immediately before the result is
returned. Only the success path writes — the three `error` returns (invalid measurements, product not
found, no unit price available) write nothing, because nothing was quoted.

`executeTool` does not currently know which contact it is serving. Its signature is

```ts
private async executeTool(name, args, db, { tenant, attachments, mediaAssets })
```

and `contactId` is in scope at the only call site,
[ai.service.ts:509](../soylaika.backend/src/ai/ai.service.ts) — it is used in the logger two lines below.
So the change is one field added to that context object. The same shape as PRD 11 §5.1: the caller knows
and the callee does not, and the fix is an argument rather than a plumbing project.

### 5.1 A failed write must not cost the customer their price

The write is guarded; a failure is a `logger.warn` and nothing else. The tool result returns and the
reply goes out regardless.

This is the same rule as PRD 13 §5.1, and it is written down in both places because the failure it
prevents is the one this codebase has actually shipped: an exception in a secondary step escaping and
taking the primary flow with it, which is what made WhatsApp audios disappear entirely when their
transcription failed.

---

## 6. Deliberately not modelled

Nothing in the product produces these, so no columns are added for them:

- **Quote status** (pending / accepted / rejected). Nothing sets it. It could be inferred from the funnel
  — a lead reaching `cliente` accepted *something* — but which calculation they accepted is not knowable,
  and a status column filled by inference is worse than no column.
- **Discount** and **payment terms offered.** The bot does not offer either; discounts do not exist in the
  catalogue model.
- **A quote as a document.** There is no grouping of lines into a quotation with one identity. Grouping
  requires a rule about which calculations belong together, and any such rule is a guess: a customer
  pricing two materials for one wall would get a "quote" summing options they would never buy together.

**"Total quoted amount" stays a query** — the latest row, or the sum across rows, depending on the
question. Choosing one now would encode an answer that changes, exactly as PRD 13 §4.1 declined to choose
between first-touch and last-touch attribution.

Grouping can be added later without losing anything, because the rows underneath it are the primitive.
The reverse is not true.

---

## 7. What this PRD is not

This was originally scoped together with an **opportunity entity** — a real `lead_id` / `opportunity_id`
so that a returning customer starts a new cycle instead of overwriting their last one. That half is
PRD 15 and is deliberately separated, for the same reason PRD 11 went first: the two halves are
independent, and one of them is small and loses data every day while the other is a refactor.

The size difference is not close. `Contact.stageId` and `Contact.status` are read across ~55 references
in five backend files, and twenty frontend files touch stage or status — three of which keep their own
hardcoded copies of the stage enum, which the root CLAUDE.md already flags as a shipping hazard. Quote
capture touches one file.

A quote arguably belongs to an opportunity rather than to a contact. When PRD 15 lands,
`QuoteCalculation` gains a nullable `opportunityId` — additive, and the same closing note PRDs 11, 12
and 13 all carry.

---

## 8. Migration

One migration, `prisma/migrations/20260914120000_quote_calculation/migration.sql`: `CREATE TABLE IF NOT
EXISTS "QuoteCalculation"`, two indexes, two foreign keys. Purely additive, no backfill possible — the
totals were never stored and prose in old messages is not a source to reconstruct them from.

The directory name must sort after the other three PRDs' migrations if they land first, and must be
settled **before it is pushed**: `TenantMigrationsService` keys its registry on the directory name, so
renaming it later makes it run again on every tenant database.

---

## 9. Testing

Unit tests against the tool handler with a fake `db`, in the shape
[audio-transcription.spec.ts](../soylaika.backend/src/whatsapp/audio-transcription.spec.ts) uses:

- a call with a SKU writes one row with `productId`, `sku`, `title` and `unitPrice` from the **catalogue
  row**, not from the model's `precioM2` argument — the check that the trust decision in §1.1 survives
  persistence
- a call with a bare `precioM2` and no SKU writes a row with null product fields and the supplied price
- each of the three error branches writes **nothing**
- two calls in one reply write two rows (§4.1)
- **a failing write does not change the tool's return value** — the §5.1 guarantee, and the test that
  matters most, because its failure mode is a customer not getting a price they asked for

As in PRD 11 §8.1, `tsc` proves nothing here: `db` is `any`, so a write with every field misspelled
type-checks clean.

---

## 10. What this PRD does not do

- **No frontend.** Nothing displays quotes; `lib/api.ts` is untouched.
- **No opportunity entity.** See §7.
- **No change to the agent prompts or the tool's contract.** The tool returns exactly what it returns
  today; this PRD only stops throwing the result away. Changing what the bot quotes is a different kind
  of change with a different kind of risk.
- **No stock or search capture.** `checkStock` and `searchProducts` results are not persisted. They were
  considered and cut: there is no ecommerce requirement for now, and the classifier's free-text
  `productInterest` in `Contact.details` already carries a weaker version of the same signal.
