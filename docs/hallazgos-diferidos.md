# Hallazgos diferidos

Findings that a review judged **real but not blocking**, and that were deliberately left
unfixed. This is not a wishlist and not a changelog: everything here was seen, weighed, and
parked with a reason. If you are about to touch one of these files, read the line first —
several of them explain why the obvious "fix" is wrong.

Anything fixed gets deleted from this file rather than crossed out; git remembers.

---

## From PRD 16 (lead disqualification), 2026-09-12

The whole-branch review triaged these. Ordered by how likely you are to trip on them, not by
severity.

### Worth doing when you are next in the file

- **`ART_OFFSET_MS` is defined twice** — `src/queue/business-days.ts` and
  `src/queue/business-hours.ts`. The duplication was deliberate: the brief for `business-days`
  said "pure, no dependencies", and the two modules answer different questions. A third copy is
  the signal to extract it.
- **`silence-sweep.ts`'s `sinValor` is weaker than what it replaced.** It catches `null` and
  `undefined`, where the previous guard was `typeof !== 'number'`. `silenceDays` is `Int?` in
  the schema so `undefined` is the only value that actually occurs, but a non-nullish non-number
  would now slip through.
- **`computePreviousStageId` has no test for the case its two arguments disagree.** It returns
  `fromStageId`, not `fromStage.id`. They coincide at all three call sites today, so nothing
  pins the distinction.
- **Negative `silenceDays` is untested** in both `business-days.ts` and `silence-sweep.ts`.
  Behaviour is correct — it takes the same branch as `0` — and every caller guards `< 1`.

### Understood, and the obvious fix is worse

- **`updateAdmin` passes a spliced stage list to `checkSilenceRule` and an un-spliced one to
  `checkReentryOverride`** (`src/funnel/funnel.service.ts`). Inert today: both functions
  short-circuit a self-reference before ever looking the id up in the list, so the splice is
  never exercised. It reads as suspicious and would matter if either validator started
  inspecting the row through `all`.
- **Editing only `reentryTargetId` re-validates the stored silence pair**, so an
  already-invalid stored pair rejects the write with a message naming a field the caller never
  touched. This is what the brief specified; it is a wording problem, not a logic one.
- **`resolveReentryTarget` cannot reject a stage pointing at itself** — its parameter carries no
  `id`. That protection lives in `checkReentryOverride` at write time, which is the intended
  design. There is no defence-in-depth if someone edits the column directly in SQL.
- **A stage whose `kind` is flipped away from `pipeline` while still enabled** can still be
  resolved into by an existing `reentryTargetId`, recreating the loop the write-time check
  prevents. Pre-existing shape, not introduced by PRD 16.
- **The `perdido` criteria `UPDATE` in the migration reports `UPDATE 1` on every re-run.** It is
  a fixed point — the end state is stable — so this only misleads someone reading migration
  logs and expecting `UPDATE 0` to mean "nothing to do".
- **The sweep's re-read of the contact sits outside `moveStage`'s transaction.** `moveStage`
  re-reads `fromStageId` inside its own, so a few milliseconds remain in which the
  `previousStageId` column and the event's `fromStageId` could disagree. The window used to be
  the whole sweep; closing it entirely means pushing an expected-from guard into `moveStage`.

### Structural, and bigger than a cleanup

- **`MessageProcessor` has no spec at all.** Pre-existing. It is why the re-entry rule is pinned
  only through the pure `resolveReentryTarget`, and why "re-entry fires with the bot still
  active" is half-covered.
- **No test ever asserted the 24-hour stage move**, which is why deleting a customer-facing
  behaviour turned nothing red. Historical, but a fair indicator of where coverage is thin.
- **eslint counts on `funnel.service.ts`, `crm.service.ts` and `ai.service.ts` grew** by the
  `no-unsafe-*` class, because the tenant `db` handle is `any` throughout. `NotificationsService`
  shows the way out — declare the shape you use — but applying that to the large services is its
  own piece of work.

### Not fixed on purpose — see the PRD

`docs/prd-lead-disqualification.md` §10 records two decisions that belong to the product, not to
the code: the bulk-send origin filter still keyed on `isLost`, and the migration seeding two
enabled `out` stages, which pushes a tenant with a custom one over the classifier cost cap.

### Never verified

**Nobody has looked at two screens.** The `no-calificado` / `no-interesado` badges on the
contacts screen, and the "Tiempo y reingreso" section added to the superadmin funnel dialog.
Both chains are sound by construction and type-check clean, but neither was opened in a browser —
it needs superadmin credentials. `soylaika.backend/CLAUDE.md` documents the ones `start:dev`
seeds.
