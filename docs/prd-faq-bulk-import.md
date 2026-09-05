# PRD 4 — Bulk knowledge loading

**Status:** draft
**Repos:** `soylaika.backend` and `soylaika.frontend`
**Depends on:** [prd-rag-knowledge-layer.md](prd-rag-knowledge-layer.md), [prd-faq-content-ingestion.md](prd-faq-content-ingestion.md), [prd-faq-admin-ui.md](prd-faq-admin-ui.md)

---

## 1. Problem

Loading knowledge is one entry at a time. A client with 28 articles — the real Ideas Todo Terreno case — is one to two hours of typing into a dialog, field by field. That cost falls on the person least able to justify it, and it is why in practice the knowledge does not get loaded at all.

The backend already has nearly everything required, and nobody can reach it:

| Piece | State |
|---|---|
| `FaqImportService.importCsv` | exists: columns `pregunta, respuesta, agentes, tags`, 500-row cap, returns rejected rows with their reason |
| `POST /api/faq/import` | exists, but is tenant-context — a superadmin cannot call it |
| `GET /api/faq/import/template` | exists: a sample CSV |
| Superadmin route | **missing** |
| Any screen at all | **missing** |
| Preview before committing | **does not exist**: `importCsv` validates and writes in the same call |

A second problem sits behind the first, and it is the more important one. **The business cannot see what its own bot knows.** Everything built so far is superadmin-only. A client whose bot answers a customer badly has no way to look at the knowledge base, find the wrong answer, and say "that is not what we do". They have to ask us.

## 2. Goals

1. Upload one file and load dozens of articles at once, instead of one at a time.
2. **See what will happen before it happens**: which rows go in, which do not, and why.
3. **Give the business permanent visibility of what is in its RAG** — not as a favour, as a standing property of the product.
4. Give the client a template they can fill without knowing anything about the system, reachable from the CRM rather than sent over email.
5. Leave intact the guarantee that nothing reaches an end customer without a human approving it.

## 3. Non-goals

- Replacing the review queue. Bulk loading fills the queue faster; it does not bypass it.
- Bulk approval. Still out of scope, for the reason it has always been out of scope: a queue that gets approved in one click is not a queue.
- Replacing the catalogue. Prices stay out of the RAG (§8).
- Letting the business edit chunks directly in phase 1 of the visibility work. Seeing and changing are different rights and are earned separately (§4).

---

## 4. Two different review questions

The vision — the client shares documents and the tool curates them — moves work onto a surface that has deliberately been superadmin-only ([prd-faq-admin-ui.md](prd-faq-admin-ui.md) §3). It is worth being precise about what changes, because it is not one more screen: it is who stands behind the guarantee.

**This section amends two earlier documents** and should be read as doing so explicitly: [prd-faq-admin-ui.md](prd-faq-admin-ui.md) §13 question 1 leaves "should a tenant admin get a read-only view?" open, and [prd-faq-content-ingestion.md](prd-faq-content-ingestion.md) §5 recommends human review without saying *which* human. This is the answer to both.

### 4.1 The mistake worth avoiding

The tempting framing is a single axis — "who is allowed to approve?" — settled by who wrote the content. It does not survive contact with the actual cases.

The motivating example in §1 is a staff member typing a client's document into the tool for one to two hours. Who wrote that? The client authored the source; an operator transcribed it. **Transcription is not verification** — nobody checked whether the client's document was still accurate, and a typo in an operator's transcription is invisible to everyone downstream.

Worse, **the system cannot tell the cases apart after the fact.** `FaqChunk` records `reviewed_by`, `updated_by` and `superseded_by`. There is **no `created_by`**. Any rule keyed on authorship is unenforceable today, because the data to enforce it does not exist.

### 4.2 The framing that does work

There are **two different questions** a reviewer can be asked, they need different people, and collapsing them is the error:

| Question | What it takes | Who can answer it | Applies to |
|---|---|---|---|
| **Is this grounded in the source it claims to come from?** | the source document, next to the candidate | whoever holds the source — today, the operator who uploaded it | only content a model derived (`source_type = DOCUMENT`) |
| **Is this true about this business?** | knowing the business | **only the business.** Nobody else, ever | **everything**, regardless of origin |

The first question is what the extraction pipeline's `source_span` exists for, and the review queue already lays it out for exactly that comparison. It is bounded and mechanical: does the answer say what the excerpt says?

The second is unbounded and cannot be delegated. **The lint validates shape — length, prices, imperatives, placeholders. It cannot validate truth, and neither can we.** The only party who knows the warranty is six months and not nine is the client. This is why §5 is not a courtesy feature: business visibility *is* the mechanism for the second question, and without it the second question simply never gets asked.

### 4.3 What follows

- **Extraction-derived content requires a comparison review before approval**, performed by whoever holds the source. The system can enforce this: it records `source_type`, so `DOCUMENT` chunks are identifiable without any new field.
- **All content requires business confirmation**, but confirmation is not the same as gating. Making it a blocking step for every chunk would stall loading behind a client's inbox. The workable shape is: content is approved by the operator, is immediately visible to the business (§5), and the business has a low-friction way to say "that is wrong" — which is a correction loop, not an approval queue.
- **`created_by` should be recorded** — who loaded a chunk, and when a superadmin acts for a tenant, on whose behalf. Without it none of the above is auditable, and PRD 3 §12 already flagged the identical gap for *edits*. Adding it for imports at the same time is cheap; discovering later that no import can be traced is not.

**A design consequence that survives all of this:** the two ingest modes do not share one screen, even though the vision naturally merges them. They carry different risks, different warnings and different review paths, and presenting both as "upload a file" makes them look more alike than they are.

### 4.4 The honest limit of this document

Letting a tenant admin *approve* — rather than see and flag — needs a tenant-facing review screen, and **no phase in §9 builds one.** PRD 3 gates every review surface to superadmin at the controller. Until that screen exists, "the client can approve their own content" is an aspiration, not a shipped capability, and this document should not imply otherwise. §9 phase G scopes it; §13 question 4 is where the decision belongs.

---

## 5. Visibility for the business

A read-only view of the knowledge base for the tenant's own `admin` role, on their own CRM — not the superadmin panel.

It shows, for their tenant: every active chunk as a question/answer pair, its status, which agents can use it, where it came from, and when it last changed. Filters and search, the same as the superadmin list.

**Read-only, in the first phase, on purpose.** The point is to close the "I cannot see what my bot knows" gap, which is worth delivering on its own and carries no risk. Editing is a separate decision with a separate failure mode — see open question 3.

What the business gets that nobody else can provide:

- **"That answer is wrong."** Only they know. A flag or comment against a chunk, routed to whoever curates, is a cheap way to capture it — cheaper than the support conversation it replaces.
- **"We stopped doing that."** Knowledge decays. The people who know it decayed are not us.
- **Trust.** A bot answering on your behalf from a knowledge base you cannot inspect is an uncomfortable product, and the discomfort is legitimate.

This also changes what "the tool curates the content" should mean. The tool can extract, validate and organise. **It cannot verify.** Making the knowledge visible to the only party who can verify it is what makes the curation trustworthy rather than merely tidy.

---

## 6. Mode A — structured template (the main path)

The client fills in a template and uploads it. The tool validates and shows the outcome before writing anything.

### 6.1 The template lives in the product

Today the template is a CSV behind an endpoint. It should be **visible on the CRM page where the loading happens**, so nobody has to be told it exists or be sent one by email.

The upload screen shows, before any file is chosen:

- **The expected columns, inline**, with a filled-in example row. Someone should be able to look at the page and understand the shape without downloading anything.
- **A download button** for the template file itself.
- **The rules that matter, stated as instructions rather than as future errors** — most importantly the one about prices (§8).

Making the structure visible on the page is not decoration. The cost of a bad upload is not the failed upload, it is the twenty rows the client has to fix and re-send, and most of those are avoidable by showing the shape first.

### 6.2 The file format

The template is offered as CSV today. CSV is a programmer's format: it breaks on commas inside text, on encoding, and on Excel choosing its own separators. For someone who just wants to fill in questions and answers, that is gratuitous friction.

**Proposal: ship the template as XLSX** with the four columns already labelled in Spanish, an instructions sheet, and three example rows. Accept both XLSX and CSV on upload.

That means adding XLSX parsing. Worth being accurate about its status: [prd-faq-content-ingestion.md](prd-faq-content-ingestion.md) §4.2 specifies "CSV or XLSX" and its phasing lists "Structured file upload (CSV/XLSX)" — XLSX was in scope from the start. Shipping CSV-only was a ruling made during implementation, on the grounds that the consumer was an internal operator who can export to CSV without complaining. **So this is not new scope; it is finishing PRD 2 phase 1.** And the reason it now matters is that the consumer changed: the client is the one filling it in, and the format decides whether the feature gets used at all.

**Cheaper alternative, to be decided:** ship the template as a Google Sheet with "download as CSV" spelled out on the instructions tab. Zero new code, more steps for the client. It is a product decision about who absorbs the friction (§13, question 1).

The XLSX detail above is written out because it is the option that needs design; it should not be read as the decision having been made. If the Sheet wins, phase E collapses to writing the instructions tab and everything else in this subsection is discarded.

### 6.3 The preview — the core of this proposal

Today `importCsv` validates and writes in the same call: the client learns that twenty rows were rejected once eighty are already loaded. It needs to be split in two.

```
POST /tenants/:slug/faq/import?dryRun=true   → analyse, do NOT write
POST /tenants/:slug/faq/import               → write
```

Those signatures are **illustrative, not settled** — they assume the superadmin path. If the client-facing upload of phase F lands on a tenant-context route instead, the shape changes; the split into analyse-then-write is the requirement, not the URL.

The response already carries almost everything needed (`total`, `imported`, `rejected[]` with row number and findings, `warnings[]`). What is missing is not writing — and that part is genuinely small: the write is a single isolated block in `importCsv`, so `dryRun` skips it without touching the validation path.

**Two consequences of splitting it that the implementation has to answer:**

**The file gets uploaded twice.** Preview and confirm are separate requests, and nothing carries the parsed file between them. Re-uploading is the simple answer and keeps the file as the source of truth, but it means nothing guarantees the confirmed file is the previewed one. The alternative — holding the parsed batch server-side against a token — adds state and an expiry policy. Re-uploading is probably right; it should be a decision rather than an accident.

**A confirmed import can still fail halfway.** `upsertBatch` writes row by row **without a transaction**. The code already knows this and says so: on a mid-import failure it logs and returns an error naming the `batchId`, precisely so the partial rows can be found. But a preview that says "80 rows will be loaded" followed by a partial failure leaves the user with an unknown subset and an error message full of an identifier they did not ask for.

The preview makes this worse rather than better, because it raises the expectation that the outcome is known in advance. Minimum acceptable handling: the result screen reports what actually landed rather than what was promised, and offers to withdraw the batch (§6.5).

Before confirming, the screen shows:

- **How many rows will go in**, and how many will not.
- **Every rejected row, with its row number and its reason in plain Spanish.** The lint messages are already written for a person: *"La pregunta o la respuesta menciona un precio o descuento. Los precios salen del catálogo, no de las FAQ: se desactualizan y contradicen al bot."*
- **The warnings**, which do not block but are worth reading.
- A confirm button that writes only the valid rows.

**The preview can make a strong promise, and the screen should say so.** The import lint and the approval lint are nearly the same check. Verified rule by rule: `vacio`, `largo`, `precio` and `imperativo` behave identically at both stages. The only difference is `placeholder` — an unresolved `{{token}}` — which is a warning at import and blocks at approval.

So: **if a row passes the preview, it will pass review, unless it carries an unresolved placeholder.** That is what makes the preview worth building. It is not an estimate; it is the same control run earlier.

### 6.4 After confirming

Everything lands as `PENDING_REVIEW` under a shared `source_ref` (`import:<uuid>`), which is what lets it be reviewed as a batch and, later, re-uploaded in a corrected version without duplicating — PRD 2 phase 4 already does that.

From the result screen, a direct link to the queue filtered by that `source_ref`: the work just created, scoped.

### 6.5 Withdrawing a batch

**There is no way to undo an upload.** Deletion is per-chunk, and a bad batch of eighty rows is eighty clicks. That is a gap in the current system, not just in this proposal, and bulk loading is what makes it urgent: the whole point is that a single action now creates many rows, and the inverse action does not exist.

Every import already carries a shared `source_ref`, so the grouping is there — nothing new is needed to identify a batch. What is missing is an operation over it.

Two cases need it:

- **"I uploaded the wrong file."** Noticed immediately, nothing approved yet. Withdrawing should be one action.
- **A partial write after a failed confirm** (§6.3). The error already names the `batchId`; without a withdraw operation the only remedy is manual cleanup, which is exactly what the error message currently asks a human to do.

**Withdraw means archive, not delete** — nothing in this system is hard-deleted, and a batch that was partly approved and served to customers is precisely the history the retention design exists to keep. Chunks already approved and serving need a confirmation naming how many are live, not a silent archive.

---

## 7. Mode B — unstructured document

The client uploads a PDF or DOCX and the tool extracts candidates. This **already exists** for pasted text (`POST /faq/extract`): an LLM proposes question/answer pairs, each must cite a verbatim excerpt from the original, the excerpt is verified to actually exist in the text, and anything ungrounded is dropped rather than flagged.

What is missing is the front door: PDF/DOCX parsing. The rest of the path is built and tested.

**This mode needs a different review and the screen has to say so.** The content was written by a model. The queue already shows the source excerpt beside the answer precisely for this, and that is the moment where somebody has to compare. A client approving this without reading the excerpt is exactly the failure the whole design exists to prevent.

So, until there is evidence to the contrary: **mode B stays in the superadmin panel.** The client can upload the document; approving what comes out of it is not theirs. It is an uncomfortable restriction and it should be revisited with real usage data, not removed for convenience.

---

## 8. What still cannot go in

Prices. The lint rejects prices and discounts at both stages, and that does not change here. It is the rule that forced four Ideas Todo Terreno articles to be rewritten, and the reason is in the error message itself: a price in a FAQ goes stale and ends up contradicting the bot.

Practical consequence for the template: **the instructions sheet has to say this before the client writes anything**, not after they upload and see twenty rejected rows. *"Write the policy, not the number: 'there is a discount for bank transfer' yes, '20% discount' no."*

The lint catching it is the safety net. The template explaining it is what stops the net being needed.

---

## 9. Phasing

| Phase | Scope | Why in this order |
|---|---|---|
| **A** | Superadmin route `POST /tenants/:slug/faq/import` + upload and result screen, CSV, with the template visible on the page | Removes today's 1-2 hours of typing. Small: the service exists and the route mirrors the nine already there |
| **B** | Preview (`dryRun`) before committing, plus batch withdraw (§6.5) | The core request. Withdraw ships with it because the preview raises the expectation that outcomes are known in advance, and a partial write breaks that promise |
| **C** | `created_by` on chunks, and on-behalf-of when a superadmin acts for a tenant (§4.3) | Small, and everything about accountability depends on it. Cheap now, expensive to backfill |
| **D** | Read-only knowledge view for the tenant's `admin` role, with a "this is wrong" reporting channel | Closes the "I cannot see what my bot knows" gap. Independent of the upload work and deliverable on its own |
| **E** | XLSX template + XLSX parsing + instructions sheet | Turns it into something the client uses without help |
| **F** | Client-facing upload (tenant `admin`), mode A only | Needs phases B and D working: no preview and no visibility, no client upload |
| **G** | Tenant-facing review screen — approve/reject for the tenant's own `admin` | **The phase that would make §4.2's second row real.** Deliberately last, and gated on question 4 in §13 |
| **H** | Mode B: PDF/DOCX parsing over the existing extraction pipeline | The riskiest and least needed: the good content is usually already written down |

Phase A alone changes today's working day. It is worth shipping before the rest is agreed.

**Phase D is deliberately placed before the client can upload anything.** Letting the business see the knowledge base is worth more than letting it load into one, and it is the prerequisite for the trust that phase F assumes. Its reporting channel is part of the phase, not an extra — §11 lists visibility-without-a-channel as a risk, and a phase that delivers the first without the second creates that risk rather than avoiding it.

**Phases A through F deliver bulk loading; none of them delivers client approval.** Until phase G, a client-uploaded batch lands in a queue only a superadmin can clear. That is still a useful feature — the client stops waiting on us to type — but it is a smaller one than §4.2's table suggests, and the gap should be visible in the plan rather than discovered during phase F.

## 10. Metrics

- **Articles loaded per session.** If it stays near 1, the feature is not being used.
- **Share of rows rejected on first upload.** High means the template is not explaining itself; it is a metric about the instructions, not about the client.
- **Time from load to first approval.** If it grows, bulk loading is filling a queue nobody works — which is worse than not having bulk loading.
- **Uploads that use the preview and then do not confirm.** High is good: the preview is doing its job.
- **Corrections reported by the business** through the visibility view. This is the number that says whether §5 delivered anything — it measures wrong answers found by the only people who can find them.

## 11. Risks

- **Flooding the queue.** Five hundred rows in one go is five hundred decisions for somebody. The 500 cap exists for this; with real bulk loading it is worth asking whether 500 is still sensible or should come down.
- **Whoever approves does not read.** The risk scales with volume, and §4.1 rules out the comfortable answer that authorship makes it safe — a transcribed batch was written by nobody who checked it. Mitigated for extracted content by the comparison review (§4.2) and for everything else by the business seeing it afterwards (§5), which is a correction loop rather than a gate.
- **Visibility without a channel is worse than no visibility.** If the business can see a wrong answer and has no way to say so, the feature produces frustration rather than corrections. The reporting path in §5 is not optional, which is why it ships inside phase D rather than after it.
- **The template becomes a contract.** Once a client has filled it in, changing the columns breaks their work. Version it from the start.
- **A confirmed import can land partially.** `upsertBatch` is not transactional (§6.3). The preview makes this sharper, not softer: it promises a known outcome and a mid-write failure breaks that promise. Batch withdraw (§6.5) is the floor, not a nicety.
- **XLSX adds a dependency.** The first in this PRD. Pick a maintained library and scope what is parsed — one sheet, four columns — rather than supporting Excel.

## 12. Already built — do not rebuild

Worth listing, because it is most of the work:

- CSV parsing with column detection in Spanish or English, and errors that name the columns actually found.
- Per-row lint with a human-readable reason, at both stages.
- Row cap with an explanation.
- `upsertBatch` with `PENDING_REVIEW`, a batch `source_ref` and a per-row `source_ordinal`.
- Re-upload versioning: a corrected second load updates rather than duplicates, and what was approved keeps serving until the new version is approved.
- Review queue with source excerpt and per-item lint findings.
- Extraction from text with verbatim-excerpt verification.

## 13. Open questions

1. **XLSX or Google Sheet?** XLSX is new code and a dependency; the Sheet is no code and more steps for the client. A product decision about who absorbs the friction.
2. **Does the 500-row cap still make sense** once loading is genuinely bulk? The number was chosen for review fatigue, not for a technical limit.
3. **Should the business be able to edit, not just see?** §5 proposes read-only first. Editing brings back the question §4 answers for uploads — and for an edit there is no template and no author to point at, so it is a harder case, not an easier one.
4. **Can the client approve what they loaded themselves by template?** §4 proposes yes. It is the most consequential question in this document and should be decided with the business, not here.
5. **What happens to a rejected row?** Today it simply does not go in. Is it worth offering an in-screen fix and retry, or is it corrected in the file and re-uploaded? The latter is simpler and keeps the file as the source of truth.
6. **Should the template carry aliases?** The Ideas Todo Terreno document has 165 and there is currently nowhere to put them ([plan-ingesta-rag-itt.md](plan-ingesta-rag-itt.md) §2). If measurement says they are needed, the template is the natural place to ask for them — but first we need to know whether they help.
