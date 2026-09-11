# PRD 12 — Contact lifecycle event log

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 11 — Message origin](prd-message-origin.md) — the sibling. 11 owns *what sent a message*; this one
  owns *when things happened to the lead*. §3 of PRD 11 is the boundary; changing one means re-reading
  the other's.
- [PRD 8 — Per-tenant funnel stage names and criteria](prd-funnel-stage-criteria.md) — `kind`,
  `MAX_OUT_ENABLED` and the seeded stage set all constrain §5.1 below.

---

## 1. Why this exists

Second of four projects carved out of a ~90-field customer-data dictionary (see PRD 11 §1). This one
covers every row in that dictionary phrased as *when did this happen* or *who did it*:

- Opportunity: complete stage history, timestamp entering each stage, timestamp leaving each stage,
  reactivation timestamp, loss timestamp, disqualification timestamp
- Conversation: bot → manual timestamp, manual → bot timestamp, handoff reason
- Human Team: assignment timestamp, user who changed the stage, reassignment history
- Historical: modification timestamp, who/what modified the data

None of it is recorded today.

### 1.1 Eleven places change a lead's state, and none of them leave a trace

Twelve call sites write to `Contact`. Eleven change something a person would call a lifecycle fact:

| Site | What changes |
|---|---|
| [crm.service.ts:383](../soylaika.backend/src/crm/crm.service.ts) `updateContact` | stage, `botActive`, `lostReason` — the human path |
| [crm.service.ts:209](../soylaika.backend/src/crm/crm.service.ts) `claimContact` | `claimedById`, `claimedAt` (via `updateMany`) |
| [crm.service.ts:238](../soylaika.backend/src/crm/crm.service.ts) `releaseContact` | claim cleared |
| [crm.service.ts:636](../soylaika.backend/src/crm/crm.service.ts) | stage → `no-contesta`, test-chat simulation |
| [message.processor.ts:157](../soylaika.backend/src/queue/message.processor.ts) | stage, from the classifier |
| [message.processor.ts:90](../soylaika.backend/src/queue/message.processor.ts) | stage `out` → `interesado`, reactivation |
| [message.processor.ts:325](../soylaika.backend/src/queue/message.processor.ts) | claim + `botActive: false`, delegation to a vendor |
| [handoff.service.ts:42](../soylaika.backend/src/notifications/handoff.service.ts) | `botActive: false` + claim |
| [followup.processor.ts:53](../soylaika.backend/src/queue/followup.processor.ts) | stage → `no-contesta` after 24h |
| [bulk.processor.ts:51](../soylaika.backend/src/queue/bulk.processor.ts) | `reactivationPending`, `botActive: true` |
| [ai.service.ts:584](../soylaika.backend/src/ai/ai.service.ts) | stage, from the classifier (test-chat path) |

The twelfth, [ai.service.ts:349](../soylaika.backend/src/ai/ai.service.ts), writes `agentType` and
`agentTurn` — internal routing state, not a lifecycle fact, and out of scope.

Six of those eleven move a lead between funnel stages. The lead's current position is stored; the path
it took to get there is not. `previousStageId` is a single step backwards and only populated on the way
into a lost stage.

### 1.2 Two MVP1 rows are missing outright

"Loss timestamp" and "disqualification timestamp" are not underivable — they are absent. `lostReason`
and `lostNote` hold a current value with no time attached, and `Contact.updatedAt` is overwritten by
the next unrelated write to the row.

### 1.3 A log over a mass-assignment hole — fixed, and why it mattered here

**Resolved before this PRD, as it required.** Recorded because the reasoning is the precondition for
the log meaning anything.

`updateContact` used to spread the raw request body:

```ts
const updateData: any = { ...data };
```

The `@Body()` annotation at [crm.controller.ts:51](../soylaika.backend/src/crm/crm.controller.ts) is
TypeScript and erased at runtime, and this project has no global `ValidationPipe`. So any authenticated
CRM user could write `value`, `claimedById`, `claimedAt`, `agentTurn` or `createdAt` — and writing
`claimedById` directly bypassed the conflict check in `claimContact`, which is the control that stops
two salespeople taking the same conversation.

It is now a named allowlist of the nine declared fields, with eleven tests — two for the hole and nine
pinning the stage-sync behaviour that had no coverage at all.

The reason it blocked this PRD: **an event log over a mass-assignment endpoint records arbitrary writes
as legitimate history.** It would have made the hole more convincing, not more visible. Any future
endpoint that mutates a lead has the same precondition — the log is only as trustworthy as the writes
it faithfully records.

---

## 2. The design, in one line

One table, `ContactEvent`, written only by a new `ContactLifecycleService` that owns every state change
and writes the contact row and the event row in the same transaction.

---

## 3. The principle: closed types, open values

The event `type` set is **closed and small** — it names mechanics: a state changed, the bot was paused,
someone claimed the lead. Everything the type points at is **open data**: which stage, which category,
what reason.

This matters because the disqualification policy does not exist yet. Leads will move through the funnel,
be disqualified into some category, and come back into the funnel, on rules that will live in the agent
prompts — and the number of categories is expected to change. A schema that encoded today's guess at
that taxonomy would be a schema the policy work has to fight.

With this split, defining the taxonomy later adds rows and at most one nullable reference. It never adds
an event type and never rewrites history.

---

## 4. The schema

```prisma
model ContactEvent {
  id          String   @id @default(uuid())
  contactId   String
  contact     Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  type        String   // 'stage_changed' | 'bot_paused' | 'bot_resumed' | 'claimed' | 'released' | 'reactivated'
  fromStageId String?
  toStageId   String?
  reason      String?
  actorKind   String   // 'human' | 'bot' | 'system'
  actorUserId String?
  actor       User?    @relation("EventActor", fields: [actorUserId], references: [id])
  createdAt   DateTime @default(now())

  @@index([contactId, createdAt])
  @@index([type, createdAt])
}
```

`User` gains `lifecycleEvents ContactEvent[] @relation("EventActor")`.

### 4.1 Transitions, not enter/exit rows

The dictionary asks for both "timestamp entering each stage" and "timestamp leaving each stage". One
transition row answers both: a lead entered stage X at the row where `toStageId = X`, and left it at the
next row for that contact where `fromStageId = X`. Time-in-stage is the difference.

Writing an enter row and an exit row would be two records of one fact, which can disagree — and the
first thing that makes them disagree is a stage change that fails halfway.

A lead leaving the funnel and returning needs no special handling. It is more transitions, in the order
they actually happened.

### 4.2 The actor mirrors PRD 11

`actorKind` + `actorUserId` are deliberately the same shape as PRD 11's `origin` + `sentByUserId`: a
category saying what kind of thing acted, and a nullable user for when it was a person. Same question,
same answer, one idea to learn instead of two.

`actorKind` is **not** nullable. Unlike PRD 11's `origin`, which had to tolerate rows written before it
existed, every `ContactEvent` row is created by this feature — there is no history to be honest about.
A required column is the stronger control and costs nothing here.

---

## 5. What this deliberately does not model

### 5.1 The disqualification taxonomy

`toStageId` is a foreign key to `FunnelStage`, so an event can only point at one of the seven seeded
stages. Disqualification categories are not modelled; a disqualification is recorded as whatever stage
change actually happens, with its `reason`.

When the policy exists it has two shapes available, and one of them has a wall in front of it:

- **Categories as `FunnelStage` rows** with `kind = 'out'`. PRD 8 caps enabled `out` stages at three
  ([funnel-criteria.ts:97](../soylaika.backend/src/funnel/funnel-criteria.ts)) and `no-contesta` and
  `perdido` already use two. **That leaves exactly one slot.** More than one category does not fit
  without raising `MAX_OUT_ENABLED`, which is a PRD 8 decision with its own reasons — the cap exists
  because every enabled stage's criteria text ships in the classifier prompt on every message, so the
  cap is a per-conversation cost control, not an arbitrary limit.
- **Categories as their own table**, which adds one nullable FK column to `ContactEvent`.

Either way the change is additive. Guessing now is not.

### 5.2 Arbitrary field history

This log records lifecycle events, not field diffs. Changes to `value`, `notes`, `name` or `details`
leave no record. That covers the dictionary's Historical group for the lifecycle fields — stage,
`botActive`, claim — and not for anything else.

If literal prev/new history for other columns is wanted later, it is a sibling table, not a rework of
this one. It was left out because a generic field-diff log over the mass-assignment endpoint in §1.3
would mostly be a faithful record of writes nobody intended.

---

## 6. The write path

A new `ContactLifecycleService` owns the semantic operations:

| Method | Writes | Event |
|---|---|---|
| `moveStage` | `stageId`, `status`, `previousStageId` | `stage_changed` |
| `pauseBot` | `botActive: false` | `bot_paused` |
| `resumeBot` | `botActive: true` | `bot_resumed` |
| `claim` | `claimedById`, `claimedAt` | `claimed` |
| `release` | claim cleared | `released` |
| `reactivate` | `reactivationPending`, `botActive` | `reactivated` |

Each writes the `Contact` row and the `ContactEvent` row inside one `$transaction`, so a lead cannot
move without its event. The eleven sites in §1.1 call these instead of `contact.update`.

Two of the eleven do more than one thing at once —
[message.processor.ts:325](../soylaika.backend/src/queue/message.processor.ts) and
[handoff.service.ts:42](../soylaika.backend/src/notifications/handoff.service.ts) both claim a lead and
pause the bot in a single write. These produce **two events**, not one compound event: "claimed" and
"bot_paused" are separately meaningful, and a compound type would be a third thing to query for.

`claimContact` is the one structural change. It uses `updateMany` with a `WHERE` that enforces the
claim-conflict rule, and only writes when `count` is 1 — so the event write must be conditional on that
same result, inside the transaction. It is the only site where the event is not unconditional, and it is
the one most worth a test.

### 6.1 The actor is a required argument

Every method takes an explicit actor. No default, no inference, no optional parameter — the same
argument as PRD 11 §4.2 and §5.1. An optional actor produces events attributed to nobody, and an event
log whose actor column is sometimes empty answers "who did this" with "sometimes".

---

## 7. Migration

One migration, `prisma/migrations/20260912120000_contact_event/migration.sql`: `CREATE TABLE IF NOT
EXISTS "ContactEvent"`, two indexes, two foreign keys. Purely additive — it creates a table and touches
no existing row.

No backfill is possible. The history genuinely is not there: nothing recorded the transitions, and
unlike PRD 11 §6.1 there is no residue in other columns to reconstruct them from. The log starts on the
day it ships.

That name sorts after `20260909120000_business_profile_draft` and after PRD 11's
`20260911120000_message_origin`; if the two land in the other order, this one must still be renamed to
sort last **before it is pushed**, never after. `TenantMigrationsService` sorts directory names
lexicographically and applies anything unregistered to every active tenant database on boot, swallowing
failures into a `logger.warn` — and **renaming the directory makes it run again**, because the
`_tenant_migrations` registry is keyed by name.

---

## 8. Testing

### 8.1 Per-path

A spec per lifecycle method with a fake `db` capturing the transaction's writes: the contact update and
the event row, asserted together. The `claim` case specifically asserts that a **losing** claim — where
`updateMany` matches nothing — writes no event, which is the one path where a contact write and an event
write come apart.

As in PRD 11 §8.1, `tsc` proves nothing here: `db` is `any` throughout, so a call that forgets the event
entirely type-checks clean.

### 8.2 The guard

A source-level test: fail if any file outside `ContactLifecycleService` calls `contact.update` or
`contact.updateMany` with `stageId`, `status`, `botActive`, `claimedById` or `claimedAt` in its data.

This matters more here than in PRD 11. There the risk was a tenth write site producing null `origin` on
new rows. Here a bypassing writer moves a lead with **no event at all**, which does not look like missing
data — it looks like the lead was never moved. A log with silent holes is worse than no log, because it
gets trusted.

---

## 9. What this PRD covers, from the dictionary

Directly stored: complete stage history, timestamp entering each stage, timestamp leaving each stage,
reactivation timestamp, bot → manual timestamp, manual → bot timestamp, handoff reason, assignment
timestamp, user who changed the stage, reassignment history, modification timestamp for lifecycle
fields, who/what modified them.

Closes PRD 11's known hole: a handoff that produces no message is now recorded, because the pause is an
event rather than an inference from message traffic.

Still not covered, and named so they are not assumed: disqualification category (§5.1), disqualification
and loss *reasons* beyond free text, field history outside the lifecycle set (§5.2), and everything in
the Acquisition, Identity, Opportunity-entity, Quote and Sale groups — projects three and four.

---

## 10. What this PRD does not do

- **No frontend.** Nothing renders the timeline. The CRM is untouched.
- **No mass-assignment fix.** §1.3 is a separate change with its own test; this PRD depends on it
  landing but does not contain it.
- **No opportunity entity.** Events hang off `Contact`, which remains the lead. When project four
  introduces a real opportunity, `ContactEvent` gains a nullable `opportunityId` — additive, and the
  reason this table is keyed on contact rather than on something that does not exist yet.
