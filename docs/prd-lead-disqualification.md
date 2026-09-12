# PRD 16 — Automatic lead disqualification

**Status:** proposed
**Repos:** `soylaika.backend` + `soylaika.frontend` + this doc — three commits
**Related:**
- [PRD 8 — Per-tenant funnel stage names and criteria](prd-funnel-stage-criteria.md) — §5's stage
  caps, §4's frozen slugs and §11.4's criteria editing are all load-bearing here. `MAX_OUT_ENABLED`
  moves; §2.3 explains what that costs.
- [PRD 12 — Contact lifecycle event log](prd-contact-lifecycle-events.md) — §5.1 deliberately did
  not model the disqualification taxonomy and said defining it later "adds rows and at most one
  nullable reference." This PRD is that definition, and §5 reports what it actually cost.
- [PRD 15 — Funnel re-entry](prd-funnel-reentry.md) — **shipped** (`11c19d4`, migration
  `20260916120000_funnel_reentry`). Its `reentersOnReply` / `isReentryTarget` pair is the mechanism
  this PRD's reactivation rides on; §6.3 adds a per-source override rather than replacing it.

---

## 1. Why this exists

The requirement, as given:

> Cerrar automáticamente las conversaciones que no se convierten en oportunidad comercial y evitar
> que queden activas indefinidamente. (…) Diferenciar DESCALIFICADO por Laika de PERDIDO por el
> equipo de ventas.

Priority is the highest in the current set: first feature of MVP 1.

### 1.1 Roughly half of it already runs

This matters for sizing, and it is easy to miss because nothing in the code is named
"disqualification":

- **Automatic no-response disqualification already runs**, at 24 hours, in
  [followup.processor.ts](../soylaika.backend/src/queue/followup.processor.ts). Two nudges (6h, 23h)
  and then a move to `no-contesta`.
- **Reactivation already runs, and is already configurable.**
  [message.processor.ts:93](../soylaika.backend/src/queue/message.processor.ts) moves a lead standing
  in a stage flagged `reentersOnReply` back to the stage flagged `isReentryTarget`, the moment they
  write again, before the AI runs, recorded as a `system` actor. That is PRD 15, shipped.
- **Per-tenant editable criteria**, the 500-character cap and per-stage enable/disable are PRD 8's,
  already shipped.
- **Every stage move already writes a `ContactEvent`** with its actor — PRD 12 is built.

So this PRD is mostly re-pointing existing machinery at a configurable rule, plus one new sweep.

### 1.2 What is genuinely missing

Three things, and only three:

1. The 24-hour timer is not a business rule. It is WhatsApp's free-form messaging window, which is
   why it is 24 hours and not a number a person chose.
2. There is one disqualification category where the requirement asks for three — and the distinction
   the requirement actually cares about, *where in the funnel the lead was lost*, is recorded for
   none of them (§5).
3. Nothing closes a lead who was quoted and then went silent. That is the most valuable lead in the
   system and the only one with no expiry at all.

---

## 2. The taxonomy: four stages outside the flow

| slug | means | trigger |
| --- | --- | --- |
| `no-calificado` | never became commercially qualified: one message and gone, or spam / promo / misfire / off-topic | timer **or** classifier |
| `no-contesta` | real commercial conversation, went silent before a quote | timer only |
| `no-interesado` | says no **during** the funnel, before commitment | classifier only |
| `perdido` | late-funnel loss: they wanted it and it fell through for a business reason (delivery, product fit) | human, mostly |

Two of these exist. `no-contesta` **keeps its slug** — it is in `LOAD_BEARING_SLUGS`
([funnel-criteria.ts](../soylaika.backend/src/funnel/funnel-criteria.ts)) and renaming a frozen slug
is exactly the hazard PRD 8 §4 lists. Its display name becomes "No responde", which PRD 8 already
supports per tenant.

### 2.1 `perdido` changes meaning, and its seeded criteria is now wrong

Today `perdido` is seeded with *"el cliente desiste o no va a comprar aca. Ejemplos claros: compre en
otro lado, ya lo consegui en otra parte, no me interesa, lo dejo, esta muy caro, no gracias"*
([funnel.service.ts:33](../soylaika.backend/src/funnel/funnel.service.ts)). Under this PRD every one
of those examples is `no-interesado`.

`perdido` narrows to the late-funnel case: the lead went the distance, wanted to buy, and it fell
through for a reason on our side. Its criteria has to be rewritten, and that is **a live prompt
change for every existing tenant**, not a new-tenant seed. It is the one part of this PRD that alters
bot behaviour on deploy day.

### 2.2 Only `perdido` keeps `isLost`

`isLost` is not a synonym for "out of the flow" — `kind` is, and PRD 8 §5.1 added `kind` precisely
because `no-contesta` was `isLost: false, isWon: false` and therefore indistinguishable from a
pipeline stage by flags alone.

`isLost` is also assumed **unique** by code: [stage-prompt.ts:78](../soylaika.backend/src/ai/stage-prompt.ts)
does `enabled.find((s) => s.isLost)` to emit the "markable from any stage" exception. Marking three
new stages `isLost` would silently drop two of them from that line — a `find` where the author meant
"the lost stage", in a file where there would now be four.

So: `perdido` stays `isLost: true`; the other three are `kind: 'out'`, `isLost: false`. That keeps
the uniqueness assumption true instead of quietly breaking it.

`stage-prompt.ts` still needs one change: `.find` → `.filter`, so `no-interesado` gets its own
"markable from any stage" exception. A customer can say no at any point in the funnel; that is the
whole definition of the stage.

### 2.3 What the fourth out-stage costs

`MAX_OUT_ENABLED` goes from 3 to 4 ([funnel-criteria.ts](../soylaika.backend/src/funnel/funnel-criteria.ts)).

That cap is not arbitrary, and PRD 8 §14 q3 says why: every enabled stage's criteria text ships in
the classifier prompt **on every message**, so the cap is a per-conversation cost control. Raising it
is a real cost the business is choosing — roughly one line plus up to 500 characters per
classification.

`checkCaps` already forbids only *worsening*, so tenants above the cap are not trapped; the constant
moving to 4 is the whole change.

**One offset is available and this PRD does not take it.** `no-contesta` is now purely
timer-driven — its own seeded criteria already says *"la maneja el sistema por tiempo, casi nunca la
marques vos"* — so it could be excluded from the classifier prompt entirely and pay for
`no-interesado`. That changes how `buildStagePrompt` selects rows, it affects a tenant who has
repurposed the stage by hand, and it belongs in PRD 8's territory rather than here.

---

## 3. The timer belongs on the stage the lead is standing in

This is the design's load-bearing move. The rules as written look like properties of the destination,
but they are not. They are all the same sentence: *a lead standing **here**, silent for **this long**,
goes **there***.

```prisma
model FunnelStage {
  // ...
  // Días HÁBILES de silencio antes de actuar sobre un lead parado en esta etapa.
  // NULL = esta etapa no tiene regla de tiempo.
  silenceDays     Int?
  // A dónde se lo manda. NULL con silenceDays seteado = sólo avisa, no mueve (§4.3).
  silenceTargetId String?
}
```

Seeds, which reproduce the requirement exactly:

| stage | `silenceDays` | `silenceTargetId` |
| --- | --- | --- |
| `nuevo` | 5 | `no-calificado` |
| `interesado` | 5 | `no-contesta` |
| `cerrando` | 5 | `no-contesta` |
| `cotizado` | 10 | **NULL** — notify, do not close (§4.3) |
| everything else | NULL | NULL |

Three requirements fall out of the shape rather than needing code:

- *"No aplica si el lead ya recibió una cotización"* stops being a special case in a condition and
  becomes a different row with a different number.
- The discriminator between "never qualified" and "went silent" — which the requirement describes as
  whether there was *una conversación comercial genuina* — becomes **did the lead ever leave
  `nuevo`**. No new data, and no judgment call about what "genuine" means.
- "Business-configurable timers and on/off" is one nullable integer per row, edited in the panel that
  already edits criteria. `NULL` is the off switch, which matters because `no-contesta` cannot be
  turned off through `enabled` — it is a load-bearing slug.

### 3.1 Validation

Next to PRD 8's caps in `funnel-criteria.ts`, same shape — reject the write, name the reason:

- `silenceTargetId` must be an **enabled** stage, must be `kind: 'out'`, and must not be the row
  itself.
- `silenceDays`, when set, must be ≥ 1. Zero would disqualify a lead in the same sweep that first
  sees them.
- A stage with `silenceTargetId` set and `silenceDays` NULL is rejected: it reads as a rule and does
  nothing.

---

## 4. The sweep

A BullMQ repeatable job walks active tenants. Per tenant it issues one query per stage that has
`silenceDays` set — at most five — asking for contacts standing in that stage with no inbound message
since the computed cutoff.

The decision is a pure function of `(stage config, last inbound timestamp, now)`. That is
deliberate: it is the only part of this feature where the defects will be, and it is the shape this
repo already tests adversarially.

Idempotency is free — `ContactLifecycleService.moveStage` writes no event when the stage does not
actually change, so a sweep that runs twice produces one transition.

### 4.1 No denormalised `lastInboundAt` column

The tempting version adds `Contact.lastInboundAt` and writes it from every inbound path. It is
rejected: the failure mode of missing one write site is a lead who **silently never disqualifies**,
and nothing reports it. That is worse here than elsewhere because the feature still looks like it is
working.

Instead: one composite index `@@index([contactId, role, createdAt])` on `Message`, and a `NOT EXISTS`
predicate against it. `Message` today has `@@index([contactId])` and `@@index([role])`, neither of
which serves this query. Nothing new has to be kept in sync, because the messages are the truth.

This also leaves `lastMessageAt` derived in the CRM list
([crm.service.ts:211](../soylaika.backend/src/crm/crm.service.ts)) — that query needs the message
*content* and *role* too, so denormalising the timestamp would not have removed the include anyway.

### 4.2 Business days are new code

[business-hours.ts](../soylaika.backend/src/queue/business-hours.ts) is the wrong function. It
answers *"when may the bot send"* — it shifts an instant into the 09:00–20:00 ART window — and it is
weekend-blind, because a follow-up sent on a Saturday afternoon is fine.

This PRD needs the other question: *how much business time has elapsed*, which must skip weekends.
New pure module, tested adversarially: a Friday-evening last message, a Sunday cutoff, a window that
spans two weekends.

**Argentine public holidays are out of scope.** A holiday makes the bot wait slightly less than the
configured number of working days. Named here so nobody later assumes it was handled.

The requirement says both *"120 horas hábiles"* and *"5 días hábiles"*. Those are not the same number
under any reading of "hábil" this codebase uses — at the existing 09:00–20:00 window, 120 business
hours is about eleven working days. This PRD implements **5 business days, weekends excluded**, and
treats the two phrasings as one intent.

### 4.3 The quoted lead: notify, do not close

`cotizado` gets a longer window and a NULL target. When it expires the bot does **not** disqualify —
it notifies the branch and leaves the lead where it is. A human decides between `perdido` and another
attempt.

This is the one rule where automatic closure was rejected on purpose: a quoted lead is the most
commercially valuable state in the funnel, and the bot should not be the thing that writes it off.

**Corrected during implementation.** This section originally said the alert goes through
`NotificationsService.createForBranch`, and that the sweep de-duplicates by recording a
`ContactEvent`. Both were wrong.

`NotificationsService` writes through `PrismaService`, which is the **master** database, and resolves
its recipients there too. A tenant's users and contacts live in that tenant's own database, so the
call would have written a row addressed to a `userId` that exists nowhere in that database —
a notification received by nobody, with nothing reporting the failure. It also has no callers
anywhere in `src/`. Every notification that actually reaches a person in this codebase is written
against the tenant `db`: [handoff.service.ts:63](../soylaika.backend/src/notifications/handoff.service.ts)
and [message.processor.ts:374](../soylaika.backend/src/queue/message.processor.ts). The sweep follows
that precedent — tenant `db`, recipients resolved there, and the whole sales team when the contact
has no branch, because skipping those would drop the alert for exactly the leads nobody owns.

The de-duplication does **not** add a `ContactEvent` type. PRD 12 §3 keeps that set closed and small,
and "a human was told" is not a fact about the lead's lifecycle. The `Notification` rows are their own
marker: the sweep skips a contact that already has one, titled the same, created after the customer's
last inbound message — so it re-arms if the customer writes and goes quiet again.

`notifyOnEnter` exists on `FunnelStage`, is editable through the funnel controller, and **is read by
nothing in `src/`** — it is a dead flag today, so it was never the hook.

**Known limitation, not fixed here.** The notification *read* path is master-scoped while every write
path is tenant-scoped: `NotificationsController` never touches `req.tenantDb`, so the CRM bell returns
nothing for a tenant user. This is pre-existing — handoff alerts have never reached the bell either —
and fixing it changes a shipped feature's behaviour, so it belongs to its own change. Until then the
alert is written, durable and correct, but invisible; the lead simply stays in `cotizado`, which is
the safe failure.

---

## 5. The structured reason, and the dimension the requirement actually wants

The requirement asks to record a structured disqualification reason, then explains what it is for:
crossing *"Etapa previa = Interesado"* with *"Descalificación = No responde"* to see where in the
process each kind of loss concentrates.

That is two fields, and both already exist in some form.

**The stage is the category.** Four stages, four categories. No enum column is added. `lostReason`
stays what it is — free text, written by the classifier or a salesperson — because two categorical
fields describing one concept is how they drift apart. Which *rule* fired goes in
`ContactEvent.reason`, alongside the actor that already distinguishes Laika's decision from the sales
team's.

**The previous stage is `previousStageId`**, which exists, is indexed, and already has a consumer:
the bulk-send screen filters by it ([crm.service.ts:533](../soylaika.backend/src/crm/crm.service.ts)).

It just never fires for these stages. Both write sites gate on `targetStage.isLost`
([ai.service.ts:583](../soylaika.backend/src/ai/ai.service.ts),
[crm.service.ts:408](../soylaika.backend/src/crm/crm.service.ts)), and per §2.2 three of the four
out-stages are not `isLost`. So today a lead that times out into `no-contesta` loses exactly the
dimension the requirement asks for — and has been losing it for as long as the 24-hour rule has
existed.

**Both gates change from `isLost` to `kind === 'out'`.** The schema comment —
*"etapa en la que estaba justo antes de pasar a Perdido"* — is corrected in the same migration.

This is the "at most one nullable reference" PRD 12 §5.1 predicted. It turned out to be zero: the
column was already there, pointed at the wrong condition.

---

## 6. Reactivation

The acceptance criterion — the contact can be reactivated if they write again — is met by code that
already runs, **but not for free.**

PRD 15 replaced the old `kind === 'out'` condition with an explicit `reentersOnReply` flag, seeded
true for exactly the two out-stages that existed then. A new out-stage is `false` by default and
therefore **does not re-enter at all**. So all three new stages must be seeded `reentersOnReply =
true`, and a test has to pin it (§9) — the failure is a lead permanently stuck outside the funnel,
with no error and nothing in the logs.

### 6.1 Disqualification must not pause the bot

Re-entry is gated on `contact.botActive` one line above it
([message.processor.ts:74](../soylaika.backend/src/queue/message.processor.ts)). Setting
`botActive = false` on disqualification would silently disable reactivation for every disqualified
lead — the feature would look like it worked, and the acceptance criterion would fail in production
with no error anywhere.

No disqualification path touches `botActive`. This is a constraint, not a preference, and it is
written here because it is invisible at the call site.

### 6.2 The mechanical move is a floor, not a placement

The re-entry move runs at line 93; `ai.chat` runs at
[line 159](../soylaika.backend/src/queue/message.processor.ts) — same message. So the classifier
reads what the customer actually said and places them properly a moment later. The existing comment
says as much: *"La IA puede reclasificarlo después si corresponde."*

That makes the destination a **prior**, not a decision: where the lead is left if the classifier
abstains — a real case, since PRD 8 §6.3 deliberately gave the model a `none` sentinel to abstain
with.

Read that way, the single global target — `interesado` as seeded by PRD 15 — is right for three
sources and wrong for the one this PRD adds. A lead coming back from `no-contesta` was engaged
before, so `interesado` is a fair assumption. A lead coming back from `no-calificado` may be the same promo blast that put them there;
asserting they are "interested" is a claim with nothing behind it.

### 6.3 A per-source override, not a replacement

```prisma
model FunnelStage {
  // ...
  // Override: a dónde vuelve un lead parado EN ESTA etapa cuando escribe de nuevo.
  // NULL = al destino global de PRD 15 (`isReentryTarget`), que es el caso normal.
  reentryTargetId String?
}
```

PRD 15 §3.3 allows **exactly one** enabled re-entry target per tenant, kept single by a swap on
write rather than a validator. That is the right shape for the question it was answering — "where do
returning leads go" has one answer — and it cannot express what §6.2 needs: `no-calificado` returning
to `nuevo` while everything else returns to `interesado`.

**The obvious move is to collapse both PRD 15 booleans into this one FK, and it is rejected.** It
would read better: a single nullable FK on the source says both halves, and makes §3.3's zero-and-two
failure modes unsayable rather than guarded. But PRD 15 is shipped and migrated, its swap logic has
tests, and its §3.3 already carries one design correction made during implementation. Replacing it
means a destructive migration across every tenant database and a rewrite of working code, to benefit
exactly one seeded row. Elegance does not pay for that here.

So `reentryTargetId` is an **override with a fallback**, and the read path becomes one `??`:

```
destino = origen.reentryTargetId ?? (la etapa con isReentryTarget)
```

The cost is honest and worth naming: a global default plus per-row overrides is two mechanisms for
one concept. It is a documented fallback chain rather than a contradiction, and if a third or fourth
stage ever needs an override, that is the signal to reconsider the collapse.

Validation, next to `checkReentryTarget`: when set, the override must point at an **enabled** stage
of `kind: 'pipeline'` (re-entering into another out-stage is a loop), and never at the row itself.

Seeds, all three new stages included:

| source stage | `reentersOnReply` | `reentryTargetId` | result |
| --- | --- | --- | --- |
| `no-contesta` | true *(already)* | NULL | → `interesado`, unchanged |
| `perdido` | true *(already)* | NULL | → `interesado`, unchanged |
| `no-interesado` | **true** *(new)* | NULL | → `interesado` — coming back means reconsidering |
| `no-calificado` | **true** *(new)* | **`nuevo`** | the fix: an honest floor for a contact we know nothing good about |
| `cliente` | false | NULL | PRD 15 §3.2's opt-in, untouched |

No existing row changes. PRD 15's "day one behaves identically" still holds, and now so does this
PRD's: the only rows written are the two stages this PRD creates.

The read path keeps PRD 15's degrade — an unconfigured or disabled destination means the lead is not
moved — and the override inherits it, since a NULL override simply falls through to the same lookup.

### 6.4 The asymmetry is deliberate

`FunnelStage` ends up with two nullable target FKs that look symmetric and are not:

- `silenceTargetId` — where silence takes you **out**. Per-source, no global default, because each
  pipeline stage genuinely loses leads to a different place (§3).
- `reentryTargetId` — where writing brings you **back**. Per-source *override* on a global default,
  because almost every stage returns leads to the same place and only one does not.

Reading these as a matched pair is the mistake this section exists to prevent. They answer different
questions: "out" is many-to-many and belongs entirely on the row; "back" is many-to-one with a single
exception.

### 6.5 What this does not fix

A spam contact that keeps writing keeps triggering AI calls, because `botActive` stays true and the
bot answers every inbound. This is **pre-existing** — a lead sitting in `no-contesta` today gets the
same treatment — and `Tenant.ai_monthly_budget` is the existing backstop.

It cannot be fixed by pausing the bot, for the reason in §6.1. Fixing it properly means a "quiet but
recoverable" state that does not exist yet, and that is its own change.

---

## 7. What this changes that is live today

Three behaviours change on deploy. Each is small and each is easy to miss.

1. **`no-answer-24h` stops moving the stage.** The handler in
   [followup.processor.ts](../soylaika.backend/src/queue/followup.processor.ts) becomes a no-op and
   the job stops being enqueued. The 6h and 23h nudges are untouched: they live inside WhatsApp's
   free-form window and are the two recontact attempts the requirement asks for. Leads now go silent
   for five business days instead of one before anything closes them.
2. **`crm.service.testFollowup`** does the same 24-hour move for the demo chat and has to follow, or
   the test chat demonstrates behaviour the product no longer has.
3. **`perdido`'s criteria is rewritten** (§2.1), which changes the classifier prompt for every
   existing tenant.

### 7.1 Frontend

The CRM keeps its own hardcoded copies of the stage enum — `lib/stage-style.ts`,
`app/(crm)/contacts/page.tsx`, and per the workspace notes `resultados/page.tsx` and
`inicio/page.tsx`. Two slugs are added to each, or the new stages render unstyled.

Those files already carry comments about having missed `no-contesta` once, when `followup.processor`
started writing a status the frontend did not know about. This is the same failure, so those are the
first place to look.

No new screen. The three `FunnelStage` fields are set by a superadmin directly, the way funnel
criteria were before PRD 8's panel existed.

---

## 8. Migration

One tenant migration:

- `FunnelStage.silenceDays`, `silenceTargetId`, `reentryTargetId` — all nullable, `DEFAULT NULL`.
- `Message` composite index `(contactId, role, createdAt)`.
- Two new seeded stage rows: `no-calificado` and `no-interesado`, **both with `reentersOnReply =
  true`** (§6), and `no-calificado` with `reentryTargetId` pointing at `nuevo` (§6.3).
- Scoped `UPDATE`s for the §3 silence seeds, by slug, and only where the column is still NULL, so a
  tenant who has configured something is never overwritten.
- The corrected comment on `Contact.previousStageId`.

Nothing in PRD 15's two columns is altered for an existing row — the only writes to `reentersOnReply`
and `reentryTargetId` are on the two rows this migration creates.

Same `DO`-block idiom as PRDs 11–15. The directory name must sort after
`20260916120000_funnel_reentry`, which is the latest applied — the registry is keyed by directory
name, so the name is both the ordering and the identity, and renaming it later makes it run again.

**Pre-apply to tenant databases before deploying the code, not with it.** The migration runner
applies unregistered migrations to every active tenant in the background at boot and swallows
failures into a `logger.warn`; a tenant that misses this one has a bot whose classifier prompt names
stages its database does not have.

The index is the piece worth watching: `Message` is the largest table per tenant, and a plain
`CREATE INDEX` holds a write lock for its duration. `CONCURRENTLY` cannot run inside the runner's
transaction, so for a large tenant this is a maintenance-window operation rather than a boot-time
one.

---

## 9. Testing

`tsc` proves nothing here. `db` is `any` throughout these services, so a wrong query shape
type-checks clean — the same thing that made every real defect in the FAQ work show up only when
adversarial inputs were actually run.

- **The business-days module**, adversarially: a Friday-evening last message, a cutoff landing on a
  Sunday, a window spanning two weekends, a `silenceDays` of 1.
- **The sweep decision**, as a pure function: a stage with no rule, a target that is disabled, a
  config shortened mid-window, a contact whose only message is outbound.
- **Stage caps and seed validation**, next to PRD 8's in
  [funnel-criteria.spec.ts](../soylaika.backend/src/funnel/funnel-criteria.spec.ts): four enabled
  out-stages allowed, five rejected, a `silenceTargetId` pointing at a pipeline stage rejected, a
  `reentryTargetId` pointing at an out-stage rejected.

Two tests exist specifically because what they pin is invisible and will rot silently:

- **`previousStageId` is written for all four out-stages**, not just `perdido` (§5). If this
  regresses, the dimension the requirement was built for disappears with no error.
- **Re-entry fires for each new out-stage, with the bot still active** (§6, §6.1). Two distinct
  failures hide here: a new stage seeded without `reentersOnReply` never re-enters, and a
  disqualification path that pauses the bot disables re-entry for every stage. Both leave the lead
  stuck outside the funnel with no error, and both would pass every other test in the suite.
- **`no-calificado` re-enters at `nuevo`, and every other out-stage still re-enters at the global
  target** (§6.3). The second half is what pins the fallback: an override that accidentally applies
  to all rows looks correct for spam and silently demotes every returning lead.

---

## 10. What this PRD does not do

- **No Argentine holiday calendar** (§4.2).
- **No frontend panel** for the three new `FunnelStage` fields (§7.1).
- **No backfill.** Leads currently sitting in `no-contesta` keep their empty `previousStageId`;
  `ContactEvent` has no history before PRD 12 shipped and was deliberately not backfilled. The gap
  closes on its own as events accumulate.
- **No structured-reason enum** (§5).
- **No fix for spam that keeps writing** (§6.5).
- **No `Opportunity` entity.** As with PRDs 11–15, the tables here gain a nullable `opportunityId` if
  it ever lands. PRD 15 §2 is the record of why it has not.

The review findings this work left deliberately unfixed — the ones that are about the code rather
than the product — are listed in [hallazgos-diferidos.md](hallazgos-diferidos.md), with the reason
each was parked. Several explain why the obvious fix is the wrong one.

Two more, both decided during implementation rather than planned:

- **The dimension is recorded but not yet surfaced for the three new stages.** §5 cites the bulk-send
  origin filter ([crm.service.ts:531](../soylaika.backend/src/crm/crm.service.ts)) as
  `previousStageId`'s existing consumer. Implementation converted the two *write* gates from `isLost`
  to `kind === 'out'` and deliberately left that *read* gated on `isLost`, so it still applies to
  `perdido` alone. Nothing over-sends — the CRM gates identically — but choosing which stages become
  bulk-reachable changes who receives real WhatsApp templates, and that is a product decision, not a
  consequence of this PRD.
- **A tenant with a custom `out` stage ends up one over the cap.** The migration inserts
  `no-calificado` and `no-interesado` enabled, unconditionally. A tenant already at three enabled
  `out` stages lands at five against a `MAX_OUT_ENABLED` of four — `checkCaps` only forbids making
  things worse, so they are not trapped, but their classifier prompt carries five criteria blocks on
  every message, above what §2.3 budgeted, without anyone choosing it. Seeding the new stages disabled
  would contradict this PRD's own intent that they work on day one, so the trade was taken knowingly.
  Worth a per-tenant check before deploying.
