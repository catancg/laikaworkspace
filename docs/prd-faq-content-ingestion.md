# PRD 2 — FAQ content authoring and ingestion

**Version:** 0.1 draft
**Owner:** TBD
**Status:** for review
**Depends on:** `prd-rag-knowledge-layer.md` (PRD 1) phase 2 shipped and stable. The seams in PRD 1 §10a are hard prerequisites. Uses the same OpenRouter client and control-plane database as the rest of the system — no new infrastructure.
**Can run in parallel with:** PRD 1 phase 3 (`BusinessProfile.extra` migration)

---

## 1. Problem

PRD 1 makes the bot able to answer from a knowledge base. It does not make filling that knowledge base practical.

The authoring story it ships with is one question at a time, typed into a panel. For a single pilot tenant that is fine. Across a growing client base it fails in three specific ways:

**Onboarding cost scales linearly with clients.** A useful FAQ set for a decoración business is 40–80 questions. Someone has to write all of them, per client, from scratch, before the feature does anything. That cost falls on whoever is doing implementation, and it is the difference between onboarding a client in an afternoon and onboarding one over two weeks.

**Clients already have the content, in the wrong format.** Policy documents, an FAQ page on their website, a Google Doc the sales team uses, a WhatsApp message they paste to customers ten times a day. Asking them to retype it as Q&A pairs is asking them to do work they will not do.

**Nothing improves across clients.** The tenth decoración client's warranty question is nearly identical to the first's. With no shared asset, that knowledge is re-derived every time.

The result is the failure mode where a well-built feature sits unused because the content pipeline was left as an exercise.

---

## 2. Goals

| # | Goal | Measure |
|---|---|---|
| G1 | A new tenant reaches a useful index without manual authoring | Time from tenant creation to ≥30 approved chunks |
| G2 | Clients can contribute their own existing material | % of tenants with ≥1 document-sourced chunk |
| G3 | Nothing reaches a customer unreviewed | Zero unreviewed chunks retrievable (enforced by PRD 1 §7.3) |
| G4 | Review is fast enough that people do it | Median review time per document; % of candidates approved without edit |
| G5 | Knowledge compounds across clients | Number of vertical templates; % of new tenants seeded from one |

## 3. Non-goals

- Automatic website crawling. Tempting, and it produces the worst content — marketing copy, stale prices, navigation text. Revisit only with a strong extraction filter.
- Bot-authored content reaching customers without human approval. Every path terminates in review.
- Real-time sync with an external doc (Google Docs, Notion). Import is a point-in-time action with explicit re-import.
- Editing agent prompts. Different lifecycle, different owner, out of scope.

---

## 4. Intake channels

Four ways content enters, all landing on `FaqIngestionService.upsertBatch()` from PRD 1 §6.

### 4.1 Vertical starter packs

Curated Q&A sets per business vertical, copied into a tenant at onboarding. This is the highest-leverage channel and the one that makes client count sublinear in effort.

- Stored in the **control-plane database** as `FaqTemplate` / `FaqTemplateEntry`, never in a tenant database — see §8.
- Copied, not referenced. Once in a tenant, chunks are theirs to edit; template updates do not propagate. Propagation across live tenants is a much harder problem and not worth it here.
- Seeded as `source_type: TEMPLATE`, `review_status: PENDING_REVIEW`. Even template content needs a pass, because a template says "consultá con un asesor por plazos de envío" and this client might actually publish theirs.
- Placeholders in template answers (`{{horario}}`, `{{politica_cambios}}`) resolved from `BusinessProfile` at copy time where possible, left visible in the review queue where not.

### 4.2 Structured file upload

CSV or XLSX with columns `pregunta`, `respuesta`, and optionally `agentes`, `tags`. A downloadable template file with the right headers and three example rows.

This is the channel that works for a client who has a list, and for an implementation person migrating an existing knowledge base. It needs no extraction step, so it is the cheapest to build and the most reliable — build it first.

Ingested as `source_type: IMPORT`, `PENDING_REVIEW`, with row number as `source_ordinal`.

### 4.3 Unstructured document upload

PDF, DOCX or pasted text. An LLM extraction pass converts it into Q&A candidates.

```
document → text extraction → chunking → LLM extraction → candidates → review queue
```

**The extraction pass is the risky part**, because an LLM asked to produce FAQ answers from a document will produce plausible answers to questions the document doesn't address. Three constraints on it:

1. **Every candidate must carry `source_span`** — the verbatim excerpt the answer came from. Non-negotiable. It is what makes review possible.
2. **The reviewer sees the span beside the candidate.** They approve against the source text, not against whether the answer sounds right. An answer that sounds right and isn't is exactly the failure this catches.
3. **Candidates whose answer is not substantially supported by their span are dropped, not flagged.** A second cheap LLM pass scoring support, or a heuristic on token overlap. Flagged-but-present content gets bulk-approved by a tired reviewer.

Extraction prompt constraints, mirroring PRD 1 §13:

- No prices, stock figures, SKUs or promotional terms in generated answers. Those belong to the catalog and go stale. Reject candidates containing a currency figure.
- Answers under 600 characters, matching `MAX_ANSWER_CHARS`.
- Answers in rioplatense Spanish, matching the agents' register.
- No imperative language directed at the bot — the retrieved block is data, not instructions, and extraction is where imperative phrasing sneaks in.
- Cap at ~60 candidates per document regardless of length. A 40-page PDF producing 200 candidates guarantees the review is not done properly.

### 4.4 Manual entry

Already built in PRD 1. Unchanged, still the right tool for a one-off correction.

---

## 5. Review workflow

States, already in the PRD 1 schema:

```
DRAFT → PENDING_REVIEW → APPROVED → ARCHIVED
                ↓
            (rejected → ARCHIVED)
```

Only `APPROVED` chunks are retrievable — enforced in the PRD 1 retrieval query, not in application logic.

**The review screen is the whole feature.** Everything else is plumbing. It needs:

- Candidate question and answer, both editable in place
- The `source_span` alongside, for document-sourced candidates
- Approve / edit-and-approve / reject, on the keyboard
- Batch operations, but *not* a single "approve all" button — approve-visible-page is the right granularity
- Duplicate detection against existing approved chunks, using the embedding index that already exists. Surfacing "this is 0.94 similar to an existing question" prevents the index filling with near-duplicates that fight each other at retrieval time.

**Who reviews.** Recommendation: the tenant reviews their own content, with two backstops — an automated lint (prices, imperative language, length, language) that runs before anything reaches the queue, and superadmin spot-checks on the first batch for a new tenant. Requiring superadmin review of everything does not scale and will become a rubber stamp, which is worse than no review because it looks like control.

**Review fatigue is the main threat to quality.** The candidate cap, the duplicate detection and the lint exist to keep the queue small enough that review stays real.

---

## 6. Re-ingestion and versioning

A client updates their returns policy and re-uploads the document. What happens:

1. New extraction produces candidates with the same `source_ref`, ordinals reassigned.
2. `upsertBatch` matches on `(source_ref, source_ordinal)`. Unchanged content hashes are skipped and stay approved — this is important, or every re-upload dumps the entire document back into the review queue.
3. Changed content goes to `PENDING_REVIEW` as a new version, with the previous chunk still `APPROVED` and live until the new one is approved. The bot never has a gap.
4. On approval, `supersede()` sets the old chunk `active: false` and `superseded_by` to the new id.
5. Chunks absent from the new document are surfaced for explicit archival, not auto-archived. A missing section is usually a re-upload artifact, not an intentional deletion.

Nothing is ever hard-deleted. Disputes about what the bot told a customer resolve against this history.

---

## 7. Cost

Per document, roughly:

| Item | Cost |
|---|---|
| Text extraction | Free (local libraries) |
| LLM extraction pass, 20-page doc | ~$0.02–0.10 depending on model |
| Support-scoring pass | ~$0.01 |
| Embeddings, 60 chunks | ~$0.001 |

Trivial per document. Two things to guard anyway: a per-tenant cap on documents per month, and idempotency so a retry loop cannot re-extract the same document repeatedly. Both charged to the same ledger as PRD 1 §8, as a distinct line item so ingestion spend is separable from chat spend.

Use a mid-tier model for extraction, not the cheapest. This is a one-off cost per document paid to avoid recurring review burden and bad answers — the economics are the opposite of the per-message path.

---

## 8. Where the template library lives — resolved

Vertical starter packs are a cross-tenant asset, and the architecture is multi-tenant by database with no shared tables. **A superadmin/control-plane database exists**, which is the right home.

Schema, in the control plane (not in any tenant DB):

```prisma
model FaqTemplate {
  id          String   @id @default(uuid())
  vertical    String                    // "decoracion" | "indumentaria" | ...
  name        String
  version     Int      @default(1)
  active      Boolean  @default(true)
  entries     FaqTemplateEntry[]
  created_at  DateTime @default(now())
  updated_at  DateTime @updatedAt

  @@unique([vertical, name, version])
}

model FaqTemplateEntry {
  id          String   @id @default(uuid())
  templateId  String
  template    FaqTemplate @relation(fields: [templateId], references: [id])
  ordinal     Int
  question    String
  answer      String                    // may contain {{placeholders}}
  agents      String[] @default([])
  tags        String[] @default([])

  @@unique([templateId, ordinal])
}
```

**Copy, don't reference.** Applying a template writes rows into the tenant's `FaqChunk` with `source_type: TEMPLATE`, `source_ref: "template:<id>:v<version>"`, and the entry ordinal. Once copied they are the tenant's to edit; template updates do not propagate to live tenants. Propagation across tenants that have edited their copies is a much harder problem and not worth solving here.

**Versioning is why `version` is on the template, not just `updated_at`.** Bumping a version lets you see which tenants were seeded from which version, and offer a diff-and-reapply later if that ever becomes worth building. `source_ref` carries the version so this stays answerable.

**Authoring.** Superadmin CRUD in the control panel. This is the argument for the control-plane database over repo seed files: templates are content, they will be edited by whoever does implementation, and requiring a deploy per edit means they stop being edited.

**Placeholder resolution at copy time.** Template answers may contain `{{horario}}`, `{{politica_cambios}}`, `{{direccion}}`. Resolve from `BusinessProfile` where a matching field exists; where it does not, leave the placeholder visible and let the review queue surface it as an unresolved item. Never let an unresolved placeholder reach `APPROVED` — add it to the lint in §5.

---

## 9. Phasing

| Phase | Scope | Rationale |
|---|---|---|
| **1** | Structured file upload (CSV/XLSX) + review queue + lint | No extraction risk. Delivers the review screen, which everything else needs. |
| **2** | Vertical starter packs, seeded from repo files | Highest leverage per unit of work once the review queue exists |
| **3** | Document upload with LLM extraction | The risky one. Ships last, on top of a review flow already proven by phases 1–2. |
| **4** | Duplicate detection, re-ingestion versioning | Refinements that only matter once volume exists |

Phase 1 alone resolves most of the onboarding cost problem, because implementation people can work in a spreadsheet. Phase 3 is what clients touch directly.

---

## 10. Metrics

| Metric | Target |
|---|---|
| Time from tenant creation to ≥30 approved chunks | < 1 day |
| Candidates approved without edit | > 60% |
| Median review time per 20-page document | < 15 min |
| Extraction candidates rejected as unsupported | Track; a rising rate means the extraction prompt is drifting |
| Near-duplicate chunks in an approved index | < 5% |
| Retrieval recall (PRD 1 metric) after seeding vs. before | Should improve materially — this is the point |

---

## 11. Risks

| Risk | Mitigation |
|---|---|
| LLM extraction invents answers the document doesn't support | Mandatory `source_span`, support-scoring drop pass, side-by-side review |
| Reviewer rubber-stamps a long queue | Candidate cap, duplicate detection, pre-queue lint, no approve-all button |
| Client uploads a document with stale prices | Currency-figure rejection in extraction; catalog remains the only price source |
| Template content is wrong for a specific client | Templates land as `PENDING_REVIEW`, never auto-approved |
| Re-upload floods the review queue | Content-hash matching skips unchanged chunks |
| Extraction produces imperative text that reads as instructions to the bot | Extraction prompt constraint plus lint; PRD 1's consumption clause is the second line of defence |
| Template placeholders reach a customer unresolved | Lint blocks `{{...}}` from reaching APPROVED (§5, §8) |

---

## 12. Open questions

1. **Which verticals get templates first?** Should follow the current client mix rather than a guess.
2. **Does the tenant review their own content, or does implementation?** Recommendation is tenant with backstops; depends on the actual client relationship.
3. **Document retention.** Keep the uploaded source file, or only the extracted spans? Keeping it helps re-extraction after an extraction-prompt improvement, and costs storage plus a data-handling commitment.
4. **Per-tenant document quota.** Needed as an abuse guard; the number is a business decision.
5. **Which model for extraction?** Available through the same OpenRouter client as everything else. Use a mid-tier model, not the cheapest — §7.
