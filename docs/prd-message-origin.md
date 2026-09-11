# PRD 11 — Message origin

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 8 — Per-tenant funnel stage names and criteria](prd-funnel-stage-criteria.md) — §8 there is the
  same hazard this PRD avoids by staying backend-only: a list duplicated across repos ships broken.
- **PRD 12 — Event log** (not yet written) — owns *when things happened*; this PRD owns *what sent a
  message*. The boundary is §3. Changing one means re-reading the other's boundary section.

---

## 1. Why this exists

The requirement came in as a data dictionary — roughly ninety fields across eleven groups
(Identity, Opportunity, Acquisition, Conversation, AI Qualification, Product/Interest, Quote, Sale,
Human Team, AI/Costs, Historical), tagged MVP1 through MVP3, to be stored now and built on later.

Most of it is a storage question with a known answer: `Contact`, `Message`, `Sale`, `SaleItem`,
`AiUsage` and `Product` already hold a large share of the MVP1 rows. Three groups need entities that
do not exist (opportunity, quote, and any history of anything). One group — Acquisition — is empty,
and its one existing field is a constant: `Contact.source` is written as the literal `'WhatsApp'` at
[crm.service.ts:525](../soylaika.backend/src/crm/crm.service.ts), which means the `clientsBySource`
breakdown on the resultados screen has always had exactly one bucket.

This PRD is none of those. It is the smallest piece, taken first, because it is the only one that is
**losing data every day it is not built**.

### 1.1 Every outbound message claims to be the same thing

Four different mechanisms send messages to a customer. All four write the same row shape:

| Mechanism | Write site | What distinguishes it today |
|---|---|---|
| Bot reply | [message.processor.ts:150](../soylaika.backend/src/queue/message.processor.ts) | `jobId` is set |
| Human, manual send | [whatsapp.service.ts:128](../soylaika.backend/src/whatsapp/whatsapp.service.ts) | **nothing** |
| Automatic follow-up (6h / 23h) | [followup.processor.ts:78](../soylaika.backend/src/queue/followup.processor.ts) and `:95` | **nothing structural** (see §6.1) |
| Bulk template campaign | [bulk.processor.ts:41](../soylaika.backend/src/queue/bulk.processor.ts) | `type: 'template'` |

Every one of them is `role: 'assistant'`. `agentType` cannot stand in for the missing field: it names
*which AI agent* produced a reply and is null for the three non-bot mechanisms, so "null `agentType`"
means "human, or follow-up, or campaign, or a bot reply from before
`20260901120001_add_message_agent_type`".

### 1.2 What that costs

Eight rows of the requested MVP1 set are not answerable from the data, and cannot be made answerable
retroactively:

- Author of each message (Lead / Bot / Human)
- Mode at that time (Bot / Manual)
- Bot → Manual timestamp, and the handoff time derived from it
- Recontact yes/no
- Number of recontact attempts
- Recontact timestamp
- Lead responded to recontact
- Timestamp of first human message after handoff

A sales team's response time, the effectiveness of each follow-up attempt, and the qualification
handoff are all measurements of *who said what, when*. The timestamps are all present. The "who" is
not.

---

## 2. The design, in one line

Two nullable columns on `Message`: `origin`, which says what mechanism produced the message, and
`sentByUserId`, which says which person, when there was one.

That is the whole feature. Everything below is the boundary around it.

---

## 3. Scope — the raw fact, not the events

The eight rows in §1.2 divide into two kinds. "Author of each message" is a **fact about a row**:
the message is there, something sent it, and the sender can be recorded at write time. "Bot → Manual
timestamp" is a **fact about a transition**: the bot was paused, and that happened whether or not a
message followed.

This PRD stores only the first kind. The derived numbers come from queries over it — handoff time is
the first `origin = 'human'` message after the last `origin = 'bot'` one; recontact attempts are the
count of `origin = 'followup'` rows; a lead answered a recontact if a `role = 'user'` row follows one.

**The limitation this accepts, stated plainly:** a handoff that produces no message is not recorded.
A salesperson who takes a conversation over and never types anything leaves no trace, because
`botActive` flips in six places with no history — [handoff.service.ts:44](../soylaika.backend/src/notifications/handoff.service.ts),
[crm.service.ts:433](../soylaika.backend/src/crm/crm.service.ts), `:527`, `:553`, `:721`,
and [bulk.processor.ts:53](../soylaika.backend/src/queue/bulk.processor.ts).

Recording those flips is PRD 12's job, and it is deliberately not started here. Half an event log —
two event types added now because they were convenient — is worse than none: it fixes the shape
before the design that needs it exists, and PRD 12 then inherits a schema it did not choose.

---

## 4. The schema

```prisma
model Message {
  // ...
  origin        String?   // 'bot' | 'human' | 'followup' | 'bulk'
  sentByUserId  String?
  sentBy        User?     @relation("SentBy", fields: [sentByUserId], references: [id])

  @@index([contactId, origin])
  @@index([sentByUserId])
}
```

`User` gains the opposite side: `sentMessages Message[] @relation("SentBy")`.

### 4.1 `origin` is for outbound only

Inbound messages leave `origin` null. `role` already says the customer wrote them, and a second
field saying `'customer'` would be a copy of `role` that can disagree with it — two fields, one fact,
no way to tell which is right when they differ.

This overloads null with two meanings: "inbound" and "outbound, written before this PRD". They are
separable by `role`, which is never null. Any query about origin is a query about outbound traffic
and must filter `role = 'assistant'` first.

### 4.2 No database default

`origin` gets no default. A default of `'bot'` would mean any write site added later, by anyone who
does not know this field exists, produces rows that **claim to be the bot** — which is the exact
defect this PRD exists to remove, reintroduced in a form that no longer looks like a bug.

Null is the honest value for "nobody recorded this". The backend has already made this argument
once, in commit `99662ce` — *"business es un dep requerido de verdad, no un default que miente"*.

The cost is that a missed writer fails silently rather than loudly. §8.2 is the control for that.

### 4.3 `sentBy` is a real relation, and what that costs

`sentByUserId` is a foreign key to `User`, matching `Contact.claimedById`. Prisma's default
referential action for an optional relation is `SetNull`, so **deleting a user erases their
attribution from every message they ever sent**.

For an audit-flavoured field that is a real loss, and a bare `String` with no constraint would
survive it. The relation is chosen anyway, for two reasons: it matches the pattern already in the
schema, so a reader does not have to ask why this one is different; and an unconstrained id can
point at a user that never existed, which is a worse failure than a null.

If user deletion ever becomes routine, this decision should be revisited — the fix is to soft-delete
users, not to drop the constraint.

---

## 5. The nine write sites

Sixteen calls create `Message` rows. Seven are inbound (`role: 'user'`) and are not touched:
[whatsapp.service.ts](../soylaika.backend/src/whatsapp/whatsapp.service.ts) `:174`, `:242`, `:374`;
[instagram.service.ts](../soylaika.backend/src/instagram/instagram.service.ts) `:136`, `:146`;
[crm.service.ts](../soylaika.backend/src/crm/crm.service.ts) `:538`, `:574`.

The other nine are outbound and each gets an `origin`:

| Write site | What it sends | `origin` |
|---|---|---|
| [message.processor.ts:150](../soylaika.backend/src/queue/message.processor.ts) | bot reply | `bot` |
| message.processor.ts:190 | bot image attachment | `bot` |
| [crm.service.ts:582](../soylaika.backend/src/crm/crm.service.ts) | test-chat bot reply | `bot` |
| crm.service.ts:590 | test-chat image attachment | `bot` |
| crm.service.ts:647 | test-chat follow-up simulation | `followup` |
| [followup.processor.ts:78](../soylaika.backend/src/queue/followup.processor.ts) | 6h follow-up | `followup` |
| followup.processor.ts:95 | 23h follow-up | `followup` |
| [bulk.processor.ts:41](../soylaika.backend/src/queue/bulk.processor.ts) | template campaign | `bulk` |
| [whatsapp.service.ts:128](../soylaika.backend/src/whatsapp/whatsapp.service.ts) | manual send | `human` + `sentByUserId` |

Instagram needs no change. It writes inbound rows only; its bot replies go through
`message.processor` like WhatsApp's.

### 5.1 The one non-mechanical edit

Eight of the nine are a literal added to an object. The ninth is not:

```ts
async sendManual(contactId: string, text: string, db?: any): Promise<{ ok: boolean }>
```

`sendManual` does not know who called it. The user is available one frame up — the controller at
[crm.controller.ts:63](../soylaika.backend/src/crm/crm.controller.ts) already takes `@Request() req`
and passes only `req.tenantDb` down. So the change is a fourth parameter and one argument at the call
site, not a plumbing project.

The parameter is **required**, not optional-with-a-fallback. An optional `userId` would let a future
caller omit it and produce a `human` message attributed to nobody, which is the §4.2 argument again.

---

## 6. Existing rows stay null

The migration adds columns and writes nothing. Every message already in every tenant database keeps
a null `origin`, and any analysis of message authorship begins on the day this ships.

This is a deliberate choice in favour of a migration that cannot corrupt anything, taken with the
knowledge that most of the history *is* recoverable.

### 6.1 The backfill that is not being run

Recorded here so the analysis is not lost if the decision is revisited. Applied in order, against
`role = 'assistant'` rows only:

| Condition | Origin | Confidence |
|---|---|---|
| `type = 'template'` | `bulk` | certain — only `bulk.processor` writes this type |
| `jobId IS NOT NULL` | `bot` | certain — only `message.processor` sets `jobId` |
| `agentType IS NOT NULL` | `bot` | certain — only the two bot-reply sites set it |
| `type = 'image'` | `bot` | certain — the two `📷` attachment writes; `sendManual` only sends text |
| `content` equals `DEFAULT_6H_MESSAGE` or `DEFAULT_23H_MESSAGE` | `followup` | certain *while* those remain two fixed constants with no per-tenant override ([followup.messages.ts](../soylaika.backend/src/queue/followup.messages.ts)) |
| anything remaining | `human` | **inference** |

The last row is why this is not being run. The remainder is manual sends, plus any bot reply from
before `jobId` existed (`20260601070000_add_job_id`, hours after init) and any test-chat reply from
before `agentType` existed. A small, old population — but writing `human` over it would put a wrong
name on messages a person never sent, permanently and invisibly.

If the backfill is ever run, the first five rules are safe on their own and the sixth should be left
null.

---

## 7. Migration

One hand-written SQL file, `prisma/migrations/20260911120000_message_origin/migration.sql`:

```sql
ALTER TABLE "Message" ADD COLUMN IF NOT EXISTS "origin" TEXT;
ALTER TABLE "Message" ADD COLUMN IF NOT EXISTS "sentByUserId" TEXT;
CREATE INDEX IF NOT EXISTS "Message_contactId_origin_idx" ON "Message"("contactId", "origin");
CREATE INDEX IF NOT EXISTS "Message_sentByUserId_idx" ON "Message"("sentByUserId");
```

No `UPDATE`, no `NOT NULL`, no backfill, no data movement. On an existing tenant database this is
two nullable columns and two indexes.

`TenantMigrationsService.onApplicationBootstrap` applies unregistered migrations to **every active
tenant database** in the background on every backend start, sorting directory names
lexicographically and swallowing failures into a `logger.warn`. Three consequences for this file:

- It must be complete before anyone starts the dev server on this branch, because it will propagate
  the moment they do.
- The directory name is both the ordering and the identity. `20260911120000_message_origin` sorts
  after `20260909120000_business_profile_draft`, the current last entry. If another migration is
  authored the same day, the timestamps must differ.
- **Renaming the directory makes it run again** — the `_tenant_migrations` registry is keyed by
  name. Once pushed, the name is fixed.

The `IF NOT EXISTS` clauses let it be pre-applied to a tenant database before deploying, which is the
sequence a live tenant requires: migration first, code second.

---

## 8. Testing

### 8.1 Per-path

One spec covering the nine sites, in the shape
[audio-transcription.spec.ts](../soylaika.backend/src/whatsapp/audio-transcription.spec.ts) already
uses: a fake `db` that captures the arguments to `message.create`, then an assertion on the `origin`
each path wrote. The manual-send case additionally asserts `sentByUserId` is the caller's id.

This is worth stating because the alternative is tempting and useless: `npx tsc --noEmit` will not
catch a missing `origin`. `db` is typed `any` throughout — `db.message.create({})` type-checks
cleanly. A green compile proves nothing about what was written.

### 8.2 The tenth write site

The nine edits are easy and will be correct. The defect this PRD is actually exposed to is the
outbound write site somebody adds in six months, who does not know `origin` exists. Because there is
no default (§4.2) and no `NOT NULL`, that writer produces silent nulls forever and no test fails.

So: **a source-level guard test**. It reads the files under `src/`, finds every `message.create`
whose data object contains `role: 'assistant'`, and fails if that same call does not also set
`origin`.

It is an unusual test — it asserts on source text rather than behaviour, and it will need care to
survive reformatting. It is proposed anyway because it is the only mechanism that constrains code
that has not been written yet, and this repository has the scar that makes the case: `@Roles` on a
controller class was decoration and not a control until `RolesGuard` was changed to read
`getAllAndOverride`, and three controllers sat unguarded in the meantime. A convention nothing
asserts on is a convention that will be broken.

If the source-scanning approach proves too brittle in practice, the fallback is a runtime assertion
in a thin `createOutboundMessage` helper that every outbound site must call — stronger, but a larger
refactor than this PRD wants.

### 8.3 What is not tested

No test asserts that `origin` is *correct* in production traffic, only that each code path sets the
value it intends. A follow-up mislabelled as a bot reply at the write site would pass. Accepted: the
nine values are one literal each, visible in review, and §8.2 covers the failure that is actually
likely.

---

## 9. What this PRD does not do

- **No frontend.** `lib/api.ts` and the chat view are untouched; `origin` is stored and not yet
  displayed. Rendering it is a separate, small PRD once there is data worth showing. PRD 8 §8 is the
  precedent for why a half-applied change across both repos ships looking broken.
- **No events.** See §3. Handoffs, stage transitions and field history are PRD 12.
- **No acquisition data.** Channel, campaign, UTMs and the hardcoded `Contact.source` (§1) are a
  separate project.
- **No opportunity or quote entities.** The largest piece of the original dictionary, and the one
  that restructures what a "lead" means. Last, deliberately.
