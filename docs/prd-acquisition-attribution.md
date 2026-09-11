# PRD 13 — Acquisition attribution

**Status:** proposed
**Repos:** `soylaika.backend` + this doc — two commits
**Related:**
- [PRD 11 — Message origin](prd-message-origin.md) — §1 there describes `Contact.source` recording the
  channel rather than the acquisition source. This PRD is where that is answered, though it does not
  change the column.
- [PRD 12 — Contact lifecycle event log](prd-contact-lifecycle-events.md) — §3 there is the principle
  this PRD reuses: a closed set of mechanics, open values.

---

## 1. Why this exists

Third of four projects from the ~90-field customer-data dictionary (PRD 11 §1). It covers the
Acquisition group: channel, source, campaign, ad set, ad, and the advertising identifiers.

The group is empty today. Nothing in the codebase records where a lead came from, and the one column
that looks like it does, does not.

### 1.1 The data is arriving and being thrown away

Meta delivers a `referral` object on the inbound webhook message when a customer arrives by clicking a
click-to-WhatsApp ad. Per the
[text messages webhook reference](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/text)
it carries:

| Field | Meaning |
|---|---|
| `source_id` | the ad's id |
| `source_url` | the ad's URL (`fb.me/...`) |
| `source_type` | `"ad"` |
| `headline`, `body` | the ad's headline and primary text |
| `media_type`, `image_url`, `video_url`, `thumbnail_url` | the creative |
| `ctwa_clid` | click-to-WhatsApp click id — omitted for Status placements |
| `welcome_message` | the greeting text |

The strings `referral`, `ctwa`, `source_url` and `utm` appear nowhere in `src/`.
`handleMessage` at [whatsapp.service.ts:152](../soylaika.backend/src/whatsapp/whatsapp.service.ts) reads
`message.id`, `message.type` and the body, and never looks at `message.referral`.

So every ad-attributed lead the business has ever received arrived with its campaign identity attached,
and was stored without it.

One thing the documentation does not settle: whether `referral` arrives on every message from that
contact or only the first after the click. §4.2 makes the design correct either way rather than
depending on the answer.

### 1.2 `Contact.source` is the channel, under a misleading name

`source` is written as a literal at contact creation — `'WhatsApp'` in
[whatsapp.service.ts:105](../soylaika.backend/src/whatsapp/whatsapp.service.ts) and
[crm.service.ts:525](../soylaika.backend/src/crm/crm.service.ts), `'Instagram'` in
[instagram.service.ts:111](../soylaika.backend/src/instagram/instagram.service.ts). It holds the channel,
which `platform` already holds, and never the acquisition source. The `clientsBySource` breakdown on the
resultados screen therefore has two buckets and neither one is a source.

**This PRD does not repurpose the column.** Rewriting `source` to mean acquisition source would leave a
column where old rows mean *channel* and new rows mean *source*, with no way to tell which is which —
the same objection that stopped the backfill in PRD 11 §6.1. Acquisition source is derived from the new
table instead. That `source` duplicates `platform` is a pre-existing wart and not this PRD's to fix.

---

## 2. The design, in one line

One table, `AcquisitionTouch`, holding one row per attribution signal observed for a contact, never
overwritten.

---

## 3. Four paths, one of which captures itself

The business acquires WhatsApp leads four ways. They are not four variations of one mechanism:

| Path | How it would be captured | Buildable now |
|---|---|---|
| Ads | the `referral` object on the inbound message | **yes** |
| Website | a `wa.me/...?text=<token>` link whose token arrives in the first message body | no — the links must exist first |
| Direct | a first message with no signal at all | **yes**, as an absence |
| Referrals | somebody has to state it | no — no mechanism exists |

### 3.1 Direct and referral are the same event

A customer who found the number on the packaging and a customer whose friend forwarded them the
business contact card produce an **identical** inbound message: no `referral` object, nothing else to go
on. WhatsApp exposes no signal for how a person came to have the number, and contact-card sharing —
which is exactly what word of mouth looks like on WhatsApp — arrives as an ordinary first message.

So the system can honestly record *"arrived with no attribution signal"*. It cannot split that into
direct versus referred without someone stating which, and two labels backed by the same absence of
evidence are worse than one honest label: they invite a report that looks like measurement.

`referral` therefore exists as a **value** in the model, settable when a mechanism to state it exists
(§6), and is never inferred.

---

## 4. The schema

```prisma
model AcquisitionTouch {
  id            String   @id @default(uuid())
  contactId     String
  contact       Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  channel       String   // 'whatsapp' today; 'instagram' | 'facebook' later
  mechanism     String   // 'ad' | 'unattributed' | 'website' | 'referral'
  whatsappMsgId String?  @unique
  sourceId      String?  // ad id
  sourceUrl     String?
  headline      String?
  body          String?
  mediaType     String?
  clickId       String?  // ctwa_clid
  token         String?  // website tracking token, when that mechanism lands
  note          String?  // who referred, when a person records it
  recordedById  String?  // set when a human stated it; null when captured automatically
  recordedBy    User?    @relation("AcquisitionRecordedBy", fields: [recordedById], references: [id])
  createdAt     DateTime @default(now())

  @@index([contactId, createdAt])
  @@index([sourceId])
  @@index([channel, mechanism])
}
```

### 4.1 One row per signal, never overwritten

A lead who clicks an ad in March, goes quiet, and clicks a different ad in July has two rows. First
touch is the earliest, last touch the latest, and the whole path is visible.

This is deliberately not a choice between first-touch and last-touch attribution. That is a marketing
decision, it changes, and encoding today's answer in a schema means the next answer costs a migration
across every tenant database. Storing every touch lets the question be answered at read time, and
answered differently later.

### 4.2 Dedup keys on the message, not the ad

`whatsappMsgId` is unique, so a row is written at most once per carrying message.

`ctwa_clid` would be the obvious key and is the wrong one twice over: it is omitted for Status ad
placements, and Postgres treats nulls as distinct, so a unique index over it would not constrain the
rows that matter. Keying on the message id also makes webhook retries idempotent for free — the same
protection `handleMessage` already gives itself by looking up `whatsappMsgId` before processing.

It has a third benefit: it makes §1.1's open question irrelevant. If `referral` turns out to repeat on
later messages, each produces its own row keyed to its own message, and the history is simply more
detailed. Nothing has to be guessed about Meta's behaviour.

The column is nullable because a `referral` or `website` row written by a human (§6) has no carrying
message.

### 4.2.1 The ad id does not fit in a JavaScript number

**Found during implementation, by running the mapper rather than reading it.**

Meta's ad ids are around 18 digits — the documented example is `120226305854810726`. That is larger
than `Number.MAX_SAFE_INTEGER`, so as a JavaScript number it becomes `120226305854810720`. The last
digits are gone and cannot be recovered.

This matters because the natural defensive move — "accept a number and convert it, a mistyped JSON
is no reason to lose the attribution" — is exactly wrong here. It would store **an ad id that is
wrong but looks right**, which is worse than storing nothing: it silently corrupts the one
measurement this PRD exists to produce, in a way no report would reveal.

So the mapper accepts a number only when `Number.isSafeInteger` holds, and otherwise stores null.
Meta sends ids as JSON strings, so this branch almost never runs — it exists so that the day it
does, the failure is visible rather than plausible.

ESLint's `no-loss-of-precision` flags the literal in the test independently, which is confirmation
of the hazard rather than an inconvenience; the test suppresses the rule with that reason stated.

This is also the argument for `sourceId` being `TEXT` in the database and never a numeric column.

### 4.3 `channel` from day one

Only `'whatsapp'` is ever written by this PRD. The column exists anyway because Instagram and Facebook
capture are planned: a column added now costs one word in a `CREATE TABLE`, and added later costs
another migration reaching every tenant database in the background on boot.

### 4.4 What is dropped from the referral payload

`image_url`, `video_url` and `thumbnail_url` are not stored — they are CDN links to the creative that
expire, and `sourceId` identifies the creative permanently. `welcome_message` is not stored: it is the
business's own copy, echoed back.

`source_type` is not stored either. It is `"ad"` for every row this PRD writes, which is what
`mechanism` already says.

---

## 5. What ships

Two mechanisms, both requiring nothing outside this repo:

- **`ad`** — when `message.referral` is present, one row with the ad fields, deduped by `whatsappMsgId`.
- **`unattributed`** — when a contact has **no touches at all** and a message arrives with no
  referral, one row recording that, so "no signal" is a fact in the data rather than an absent join.

  **Corrected during implementation.** This said "a contact's *first inbound message*", which is both
  harder to determine and wrong at the edges: `upsertContact` does not report whether it created or
  found the contact, and a lead whose first message carried a referral would later get a spurious
  `unattributed` row from their second. Keying on "has no touches yet" is one query, needs nothing
  from the upsert, and is self-correcting — once any signal exists, no further `unattributed` row can
  be written. Without that cut there would be one `unattributed` row per message the customer sends.

### 5.1 Capture point, and not breaking the message flow

`handleMessage` calls `upsertContact` before the `type` switch, and `message.referral` is available
there. The touch write goes next to it, inside its own guard.

**Attribution must never be able to cost a customer message.** The failure shape is the one PRD 9's
audio work just fixed: a network or database error in a nice-to-have step escaping and taking the
message write and the queue push with it. The row write and the enqueue stay outside the guard and
always run; a failed attribution write is a `logger.warn` and nothing more.

---

## 6. Reserved, not built

`website` and `referral` are valid values of `mechanism` from day one, and no code writes them yet.

- **`website`** needs `wa.me/...?text=<token>` links deployed wherever the traffic originates, and a
  parser that strips the token from the first message body. The blocker is outside the backend: the
  links have to exist before there is anything to parse.
- **`referral`** needs someone to state it — a CRM control, or a bot question whose answer the
  classifier extracts. `note` and `recordedById` exist so that landing either one is a new row, not a
  schema change.

Both are columns already present and values already valid, so adding them is code, not a migration.

---

## 7. UTMs are not capturable, and the columns are not being added

The dictionary lists `utm_source`, `utm_medium`, `utm_campaign`, `utm_content` and `utm_term` as MVP2.
None of them can be filled by any path the business uses.

WhatsApp delivers an ad id, not query parameters. UTM parameters live in a URL, and the only URL in this
flow is `source_url` — Meta's own `fb.me/...` shortlink, not a link the business controls. Adding the
five columns would produce five fields that are null for every row ever written, which reads as missing
data rather than as an absent mechanism.

They become possible only through the `website` path in §6, where the business controls the link and
can encode parameters into the token. At that point they are a parsed form of `token`, not five new
columns.

---

## 8. Ad names live in Meta, not in the webhook

`sourceId` is an ad id. The dictionary asks for campaign, ad set and ad *names* — those are in Meta's
Marketing API, not the webhook payload, and resolving id → name means an authenticated API integration
with its own credentials, rate limits and failure modes.

That is a separate project. This PRD stores the identifier, which is the part that is being lost; the
names can be resolved later against rows that already exist, which is not true of the ids.

---

## 9. Migration

One migration, `prisma/migrations/20260913120000_acquisition_touch/migration.sql`: `CREATE TABLE IF NOT
EXISTS "AcquisitionTouch"`, three indexes, one unique constraint on `whatsappMsgId`, and **two**
foreign keys — `contactId` with `ON DELETE CASCADE`, `recordedById` with `ON DELETE SET NULL` — both
inside the `DO` block idiom PRD 11 §7 introduced, so the file can be pre-applied by hand before a
deploy. Purely additive.

Verified against a throwaway database: it runs **twice** with no errors and leaves exactly those two
constraints with those two actions.

No backfill. The referral payloads were never stored and are not recoverable from anything — unlike PRD
11 §6.1, there is no residue to reconstruct from. Attribution starts on the day this ships.

The directory name must sort after PRD 11's `20260911120000_message_origin` and PRD 12's
`20260912120000_contact_event` if those land first, and must be settled **before it is pushed** —
`TenantMigrationsService` keys its registry on the directory name, so renaming it later makes it run
again.

---

## 10. Testing

Per-path unit tests with a fake `db`, in the shape
[audio-transcription.spec.ts](../soylaika.backend/src/whatsapp/audio-transcription.spec.ts) uses:

- a message with a full `referral` writes one row with the fields mapped correctly
- a message with a `referral` missing `ctwa_clid` (the Status placement case) still writes a row
- the same message delivered twice writes one row, not two
- a first message with no referral writes an `unattributed` row
- **a failing attribution write does not prevent the message row or the queue push** — the §5.1
  guarantee, and the one test that matters most, because the defect it guards against is invisible
  until a customer's message disappears

As in PRD 11 §8.1, `tsc` proves nothing here: `db` is `any`, so a write with every field misspelled
type-checks clean.

---

## 11. What this PRD does not do

- **No frontend.** Nothing displays attribution; `lib/api.ts` is untouched. The resultados screen keeps
  showing its two-bucket `clientsBySource` breakdown until a separate PRD replaces it.
- **No change to `Contact.source`.** See §1.2.
- **No Instagram or Facebook capture.** `channel` is ready for it; the webhook handling is not written.
  Instagram's referral payload has its own shape and is its own piece of work.
- **No Marketing API integration.** See §8.
- **No opportunity entity.** Touches hang off `Contact`. When project four introduces opportunities,
  `AcquisitionTouch` gains a nullable `opportunityId` — additive, same reasoning as PRD 12 §10.
