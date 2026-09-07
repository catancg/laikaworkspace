# FAQ content ingestion — phase 4: re-ingestion versioning and duplicate detection

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make re-uploading a corrected document update the existing FAQ entries instead of duplicating them, without ever taking an approved answer away from the bot, and surface likely-duplicate questions for a human instead of silently merging them.

**Architecture:** A stable `source_ref` per document makes re-uploads *match* existing chunks. Matching alone is not safe, so a `version` column lands in the same migration: changed content becomes a **new row** at the next version, `PENDING_REVIEW`, while the previously approved row stays `APPROVED` and `active` and keeps serving. Approval of the new version is what supersedes the old one. Chunks absent from a re-upload are reported, never auto-archived. Duplicate detection is a pure heuristic over content words that **flags** — the inverse of the support check's drop-don't-flag rule, for a reason stated in Task 6.

**Tech Stack:** NestJS 11, Prisma 7 (raw tagged templates for all vector I/O), PostgreSQL + pgvector, Jest/ts-jest.

**Spec:** `f:\docs\Laika\docs\prd-faq-content-ingestion.md` — §6 (Re-ingestion and versioning) and §9 phase 4.

## Global Constraints

- Comments and user-facing strings in **español rioplatense (voseo)**. Match the surrounding file.
- **The load-bearing invariant: only `review_status = 'APPROVED'` AND `active = true` chunks are bot-retrievable.** No task may create a state where a customer-visible answer was never approved, or where an approved answer silently stops being served.
- Migrations are **hand-written SQL** in `prisma/migrations/<timestamp>_<nombre>/migration.sql`. Master migrations use `IF NOT EXISTS`. Tenant DBs migrate themselves at boot via `TenantMigrationsService`, which records applied migrations in each tenant's `_tenant_migrations`.
- Multi-tenant by database: every DB call takes the request's `tenantDb`. Never fall back to `this.prisma` (master) by accident.
- `FaqChunk.embedding` is `Unsupported("vector(1536)")` — all reads/writes of it go through `$queryRaw`/`$executeRaw` tagged templates. Never string-concatenate a value into SQL.
- **`ts-jest` does not full type-check.** Every task runs BOTH `npm test -- src/faq/` and `npx tsc --noEmit -p tsconfig.json`, and reports both outputs separately. A green suite is not evidence the branch compiles.
- Never run `git stash`. Stage only the files you edited, by explicit path — never `git add -A`. **`CLAUDE.md` has uncommitted user changes: do not stage, commit, or revert it.**

---

## Why this phase exists (read before Task 1)

Today **no intake path can ever match an existing chunk**, because every one derives a `source_ref` that is unique per upload:

| Path | `source_ref` | Consequence |
|---|---|---|
| CSV import | `batchId` (new per upload) | re-uploading a corrected CSV duplicates every row |
| Extraction | `document:${randomUUID()}` | re-uploading a corrected document duplicates every candidate |
| Template apply | `template:${id}:v${version}` | stable *within* a version — this one is already correct and deliberate |

So PRD §6's whole flow — match, skip unchanged, version what changed, surface what's missing — is unimplemented. The review queue fills with duplicates and the reviewer does the deduplication by hand.

**The sequencing that matters:** `upsertBatch` currently handles a matched-but-changed chunk by `UPDATE`-ing the row in place and setting `review_status` to the caller's value. With today's always-unique refs that code path is unreachable. Introduce a stable `source_ref` on its own and it becomes reachable immediately — and it would take an approved answer, overwrite it, and flip it to `PENDING_REVIEW`, removing it from the bot until a human re-approves. That is exactly the gap §6.3 forbids.

**Therefore the stable ref (Task 2) must not land before versioning (Tasks 1 and 3).** Task order in this plan is a safety constraint, not a preference.

---

## File structure

| File | Responsibility |
|---|---|
| `prisma/migrations/<ts>_faq_chunk_version/migration.sql` | new | `version` column, widened unique, index |
| `prisma/schema.prisma` | modify | mirror the migration |
| `src/faq/faq-source-ref.ts` | new | pure: derive a stable `source_ref` from a document name |
| `src/faq/faq-ingestion.service.ts` | modify | versioning in `upsertBatch`; `supersede` writes `superseded_by`; `unchanged` refreshes `source_span` |
| `src/faq/faq-review.service.ts` | modify | approving a version supersedes its predecessor |
| `src/faq/faq-duplicates.ts` | new | pure: near-duplicate question detection |
| `src/faq/faq-extraction.service.ts` | modify | use the stable ref; report missing chunks |
| `src/faq/faq-import.service.ts` | modify | use the stable ref |
| `src/faq/faq.controller.ts` | modify | expose the missing-chunks report |

---

### Task 1: `version` column and widened uniqueness

The schema change that makes two live versions of one chunk representable. Nothing uses it yet — this task is deliberately inert so the migration can be reviewed on its own.

**Files:**
- Create: `prisma/migrations/20260903000000_faq_chunk_version/migration.sql`
- Modify: `prisma/schema.prisma` (the `FaqChunk` model)

**Interfaces:**
- Produces: `FaqChunk.version: Int` (default 1) and the unique key `(source_ref, source_ordinal, version)`. Consumed by Tasks 3, 4, 5.

- [ ] **Step 1: Write the migration**

Create `prisma/migrations/20260903000000_faq_chunk_version/migration.sql`:

```sql
-- Una fila por VERSION de un chunk. Hasta ahora (source_ref, source_ordinal) era
-- unico, con lo cual no habia forma de tener la version aprobada sirviendo al bot
-- y la nueva esperando revision al mismo tiempo: habia que pisar una con la otra.
-- Eso dejaba al bot sin esa respuesta hasta que alguien aprobara (PRD 2 §6.3).
ALTER TABLE "FaqChunk" ADD COLUMN IF NOT EXISTS "version" INTEGER NOT NULL DEFAULT 1;

-- El unique viejo es justo lo que impide dos versiones vivas. Hay que sacarlo de
-- las DOS formas posibles, y esto no es paranoia: la migracion original lo creo
-- con `CREATE UNIQUE INDEX`, o sea un indice suelto SIN entrada en pg_constraint,
-- y `ALTER TABLE ... DROP CONSTRAINT IF EXISTS` sobre un indice suelto no falla:
-- avisa "does not exist, skipping" y sigue de largo. El indice queda vivo, y el
-- primer INSERT de una version 2 revienta con unique violation en produccion —
-- con todos los tests en verde, porque los tests mockean la base.
-- Verificado contra pgvector/pg16 antes de escribir esto.
-- El orden importa: si en alguna base SI es constraint, DROP INDEX solo fallaria
-- ("cannot drop index ... because constraint ... requires it").
ALTER TABLE "FaqChunk" DROP CONSTRAINT IF EXISTS "FaqChunk_source_ref_source_ordinal_key";
DROP INDEX IF EXISTS "FaqChunk_source_ref_source_ordinal_key";

DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_constraint WHERE conname = 'FaqChunk_source_ref_source_ordinal_version_key'
  ) THEN
    ALTER TABLE "FaqChunk"
      ADD CONSTRAINT "FaqChunk_source_ref_source_ordinal_version_key"
      UNIQUE ("source_ref", "source_ordinal", "version");
  END IF;
END $$;

-- Buscar "la version viva de este ordinal" es la consulta caliente de la re-ingesta.
CREATE INDEX IF NOT EXISTS "FaqChunk_source_ref_source_ordinal_idx"
  ON "FaqChunk" ("source_ref", "source_ordinal");
```

- [ ] **Step 2: Mirror it in `schema.prisma`**

In the `FaqChunk` model, add the field next to `superseded_by`:

```prisma
  version        Int       @default(1)
```

and replace the `@@unique` line:

```prisma
  @@unique([source_ref, source_ordinal, version])
  @@index([source_ref, source_ordinal])
```

- [ ] **Step 3: Regenerate the client and type-check**

```bash
npx prisma generate
npx tsc --noEmit -p tsconfig.json
```
Expected: `tsc` passes, and that is worth understanding rather than taking as good news.

`faq-ingestion.service.ts:86` uses the generated compound key `source_ref_source_ordinal`, which this migration removes — so it *looks* like `tsc` should catch it. It does not, because every DB call in these services goes through `private db(tenantDb?: any) { return tenantDb ?? this.prisma; }`, whose return type is **`any`**. Nothing about `db.faqChunk.findUnique(...)` is type-checked at all.

**The consequence is worth carrying for the rest of this phase:** the "run `tsc` to catch stale call sites" discipline protects *constructor arity* (DI is typed) but gives **zero protection on any query shape** — a wrong column, a wrong where-key, a removed compound key are all invisible to `tsc` and to `ts-jest` alike. Those get caught only by a test that asserts on the emitted query, or by production. Step 4's change is therefore required by *runtime* correctness, not by the compiler.

- [ ] **Step 4: Fix the one call site, minimally**

In `faq-ingestion.service.ts`, the `findUnique` becomes a `findFirst` — the lookup we want is "the live version of this ordinal", which is no longer a unique-key lookup. Replace the `existing` assignment (currently lines 84-88):

```ts
      // La version VIVA de este ordinal: la mas nueva que siga activa. Ojo que
      // ya no es un findUnique — con el unique widened a (ref, ordinal, version)
      // puede haber varias filas para el mismo ordinal.
      const existing = (input.sourceOrdinal !== undefined && opts.sourceRef)
        ? await db.faqChunk.findFirst({
            where: { source_ref: opts.sourceRef, source_ordinal: input.sourceOrdinal, active: true },
            orderBy: { version: 'desc' },
          })
        : null;
```

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```
Expected: suite passes unchanged (behaviour is identical while only one version per ordinal exists), `tsc` clean.

- [ ] **Step 6: Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260903000000_faq_chunk_version/migration.sql src/faq/faq-ingestion.service.ts
git commit -m "feat(faq): let a chunk have versions, so an approved answer keeps serving"
```

---

### Task 2: stable `source_ref` for documents

**Files:**
- Create: `src/faq/faq-source-ref.ts`
- Test: `src/faq/faq-source-ref.spec.ts`

**Interfaces:**
- Produces: `documentSourceRef(sourceName?: string): string`. Consumed by Tasks 5 and 7.

**Do not wire this into any service in this task.** Task 1 must be merged first, and the wiring happens in Task 5 — see "Why this phase exists": a stable ref without versioning creates the exact gap this phase exists to close.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-source-ref.spec.ts`:

```ts
import { documentSourceRef } from './faq-source-ref';

describe('documentSourceRef', () => {
  it('is stable for the same document name', () => {
    expect(documentSourceRef('Politica de devoluciones')).toBe(documentSourceRef('Politica de devoluciones'));
  });

  it('ignores case, accents and surrounding whitespace', () => {
    expect(documentSourceRef('  Política de Devoluciones  ')).toBe(documentSourceRef('politica de devoluciones'));
  });

  it('distinguishes different documents', () => {
    expect(documentSourceRef('devoluciones')).not.toBe(documentSourceRef('envios'));
  });

  it('is prefixed so it cannot collide with a template or import ref', () => {
    expect(documentSourceRef('envios')).toMatch(/^document:/);
  });

  // Sin nombre no hay identidad estable posible: dos documentos distintos sin
  // nombre no son "el mismo documento". Cae a un id unico, que es el
  // comportamiento viejo — duplica en vez de pisar, que es el lado seguro.
  it('falls back to a unique ref when there is no name', () => {
    expect(documentSourceRef()).not.toBe(documentSourceRef());
  });

  it('does not produce an empty slug for a name with no alphanumerics', () => {
    expect(documentSourceRef('***')).not.toBe('document:');
    expect(documentSourceRef('***')).not.toBe(documentSourceRef('***'));
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-source-ref.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

Create `src/faq/faq-source-ref.ts`:

```ts
import { randomUUID } from 'crypto';

// La identidad de un documento para la re-ingesta. Antes cada subida generaba
// `document:${randomUUID()}`, con lo cual re-subir el MISMO documento corregido
// nunca matcheaba nada y duplicaba todo en la cola de revision (PRD 2 §6).
//
// La identidad es el nombre del documento normalizado. Es del cliente, no
// nuestra: si sube "Politica de devoluciones" dos veces, es el mismo documento
// aunque el texto haya cambiado — de eso se trata la re-ingesta.
export function documentSourceRef(sourceName?: string): string {
  const slug = (sourceName ?? '')
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '');

  // Sin nombre utilizable no hay identidad estable: dos documentos anonimos
  // distintos no son el mismo documento. Volvemos al id unico, que duplica en
  // vez de pisar — el lado seguro del error.
  if (!slug) return `document:${randomUUID()}`;
  return `document:${slug}`;
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-source-ref.spec.ts
npx tsc --noEmit -p tsconfig.json
```
Expected: PASS (6 tests), `tsc` clean.

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-source-ref.ts src/faq/faq-source-ref.spec.ts
git commit -m "feat(faq): give a re-uploaded document a stable identity"
```

---

### Task 3: version instead of overwrite

The heart of the phase. A matched chunk whose content changed no longer overwrites the approved row.

**Files:**
- Modify: `src/faq/faq-ingestion.service.ts`
- Test: `src/faq/faq-ingestion.service.spec.ts`

**Interfaces:**
- Consumes: `FaqChunk.version` (Task 1).
- Produces: `upsertBatch` result gains `versioned: number`. Consumed by Tasks 4 and 5.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService versioning on re-ingestion', () => {
  function makeService() {
    const embeddingClient = { embed: jest.fn().mockResolvedValue({ vectors: [[0.1]], usage: {}, model: 'm' }) } as any;
    return { service: new FaqIngestionService({} as any, embeddingClient), embeddingClient };
  }

  function makeDb(existing: any, maxVersion?: number) {
    return {
      faqChunk: {
        findFirst: jest.fn().mockResolvedValue(existing),
        // El maximo historico incluye versiones ya rechazadas/inactivas.
        aggregate: jest.fn().mockResolvedValue({ _max: { version: maxVersion ?? existing?.version ?? null } }),
      },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };
  }

  const approved = {
    id: 'viejo-1', question: '¿Cuanto tarda el envio?', answer: 'Tarda 48 horas.',
    content_hash: 'hash-viejo', review_status: 'APPROVED', active: true,
    version: 1, source_ordinal: 1, source_span: 'Tarda 48 horas.',
  };

  // Lo que justifica toda la fase: la respuesta aprobada NO se puede pisar.
  it('inserts a new version instead of overwriting an approved chunk', async () => {
    const { service } = makeService();
    const tenantDb = makeDb(approved);

    const res = await service.upsertBatch(
      [{ question: '¿Cuanto tarda el envio?', answer: 'Tarda 72 horas.', sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'PENDING_REVIEW' },
      tenantDb,
    );

    expect(res.versioned).toBe(1);
    expect(res.updated).toBe(0);
    const sql = tenantDb.$executeRaw.mock.calls[0][0].join('?');
    expect(sql).toContain('INSERT INTO "FaqChunk"');
    expect(sql).not.toContain('UPDATE "FaqChunk"');
  });

  // El escenario que rompe: rechazar una version y despues volver a subir.
  // reject() deja la v2 ARCHIVED + active=false, la viva vuelve a ser la v1, y
  // contar desde la viva reusa el 2 — que ya existe. Unique violation, y el
  // documento queda roto para siempre. El numero tiene que salir del maximo
  // historico, no de la viva.
  it('numbers a new version above every version ever used, not just the live one', async () => {
    const { service } = makeService();
    const tenantDb = makeDb(approved, 2);   // viva = v1 aprobada, pero ya existio una v2 rechazada

    await service.upsertBatch(
      [{ question: '¿Cuanto tarda el envio?', answer: 'Tarda 96 horas.', sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'PENDING_REVIEW' },
      tenantDb,
    );

    const values = tenantDb.$executeRaw.mock.calls[0].slice(1);
    expect(values).toContain(3);
    expect(values).not.toContain(2);   // 2 esta tomado por la rechazada
  });

  it('gives the new version the next version number', async () => {
    const { service } = makeService();
    const tenantDb = makeDb({ ...approved, version: 3 });

    await service.upsertBatch(
      [{ question: '¿Cuanto tarda el envio?', answer: 'Tarda 72 horas.', sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'PENDING_REVIEW' },
      tenantDb,
    );

    const values = tenantDb.$executeRaw.mock.calls[0].slice(1);
    expect(values).toContain(4);
  });

  // El viejo tiene que seguir sirviendo: no lo tocamos hasta que aprueben el nuevo.
  it('leaves the approved chunk untouched while the new version waits', async () => {
    const { service } = makeService();
    const tenantDb = makeDb(approved);

    await service.upsertBatch(
      [{ question: '¿Cuanto tarda el envio?', answer: 'Tarda 72 horas.', sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'PENDING_REVIEW' },
      tenantDb,
    );

    const statements = tenantDb.$executeRaw.mock.calls.map((c: any[]) => c[0].join('?'));
    expect(statements.some((s: string) => s.includes('active=false'))).toBe(false);
    expect(statements.some((s: string) => s.includes(`WHERE id=`))).toBe(false);
  });

  // Si la version viva NO esta aprobada, no hay nada que proteger: pisarla es
  // correcto, y evita una pila de versiones pendientes por cada re-subida.
  it('updates in place when the live version is still pending review', async () => {
    const { service } = makeService();
    const tenantDb = makeDb({ ...approved, review_status: 'PENDING_REVIEW' });

    const res = await service.upsertBatch(
      [{ question: '¿Cuanto tarda el envio?', answer: 'Tarda 72 horas.', sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'PENDING_REVIEW' },
      tenantDb,
    );

    expect(res.versioned).toBe(0);
    expect(res.updated).toBe(1);
    expect(tenantDb.$executeRaw.mock.calls[0][0].join('?')).toContain('UPDATE "FaqChunk"');
  });

  // Fix arrastrado de la fase 3: el atajo de "sin cambios" nunca refrescaba el
  // span, con lo cual un span corregido en una re-subida quedaba viejo y el
  // revisor comparaba la respuesta contra el fragmento equivocado.
  it('refreshes the source span even when the content hash is unchanged', async () => {
    const { service } = makeService();
    const tenantDb = makeDb({ ...approved, content_hash: undefined, source_span: 'span viejo' });
    // el hash real lo calcula el servicio; forzamos igualdad leyendola de un primer run
    const first = await service.upsertBatch(
      [{ question: approved.question, answer: approved.answer, sourceOrdinal: 1 }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'APPROVED' },
      makeDb(null),
    );
    const realHash = (first.chunks[0] as any).content_hash;

    const db2 = makeDb({ ...approved, content_hash: realHash, source_span: 'span viejo' });
    const res = await service.upsertBatch(
      [{ question: approved.question, answer: approved.answer, sourceOrdinal: 1, sourceSpan: 'span corregido' }],
      { sourceType: 'DOCUMENT', sourceRef: 'document:envios', reviewStatus: 'APPROVED' },
      db2,
    );

    expect(res.unchanged).toBe(1);
    const spanUpdates = db2.$executeRaw.mock.calls.filter((c: any[]) => c[0].join('?').includes('source_span'));
    expect(spanUpdates).toHaveLength(1);
    expect(spanUpdates[0].slice(1)).toContain('span corregido');
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — `versioned` is undefined; the approved chunk is overwritten with `UPDATE`.

- [ ] **Step 3: Implement**

In `faq-ingestion.service.ts`:

(a) Add `versioned` to the counters initialised alongside `created`/`updated`/`unchanged`:

```ts
    let versioned = 0;
```

(b) Replace the unchanged short-circuit (currently lines 90-94) so a corrected span is still written:

```ts
      if (existing && existing.content_hash === hash) {
        unchanged++;
        results[i] = existing;
        // El hash cubre pregunta+respuesta, no el span. Una re-subida puede traer
        // el MISMO texto con un span corregido, y el revisor compara la respuesta
        // contra el span: dejarlo viejo lo hace revisar contra el fragmento
        // equivocado. Es un UPDATE puntual, sin re-embeber (el texto no cambio).
        if (input.sourceSpan !== undefined && input.sourceSpan !== existing.source_span) {
          await db.$executeRaw`
            UPDATE "FaqChunk" SET source_span=${input.sourceSpan}, updated_at=NOW()
            WHERE id=${existing.id}`;
          results[i] = { ...existing, source_span: input.sourceSpan };
        }
        continue;
      }
```

(c) Replace the `if (existing)` branch of the write loop (currently lines 110-119) with the versioning decision:

```ts
      // La decision central de la fase 4. Si la version viva ya esta APROBADA,
      // pisarla la sacaria del aire hasta que alguien apruebe la nueva — el bot
      // se queda sin esa respuesta justo despues de que el cliente "actualizo"
      // su documento. Asi que la aprobada se queda sirviendo intacta y lo nuevo
      // entra como version siguiente, a la cola (PRD 2 §6.3).
      //
      // Si la viva NO esta aprobada no hay nada que proteger: pisarla es
      // correcto y evita apilar una version pendiente por cada re-subida.
      if (existing && existing.review_status === 'APPROVED') {
        const id = randomUUID();
        // El numero sale del MAXIMO historico, no de la version viva. Si alguna
        // version fue rechazada, `reject()` la deja ARCHIVED + active=false pero
        // la fila sigue ahi (aca no se borra nada, PRD 2 §6), y entonces la viva
        // vuelve a ser una mas vieja. Contar desde la viva reusaria un numero ya
        // tomado: unique violation en (source_ref, source_ordinal, version), y no
        // una sola vez — ese documento queda roto para siempre.
        // Verificado contra pgvector/pg16: v1 aprobada + v2 rechazada => la viva
        // es v1, +1 da 2, y el INSERT explota. Con max(version)+1 da 3 y entra.
        const top = await db.faqChunk.aggregate({
          where: { source_ref: opts.sourceRef, source_ordinal: input.sourceOrdinal },
          _max: { version: true },
        });
        const nextVersion = (top?._max?.version ?? existing.version ?? 1) + 1;
        await db.$executeRaw`
          INSERT INTO "FaqChunk"
            (id, question, answer, agents, tags, content_hash, review_status, source_type, source_ref, source_ordinal, source_span, version, embedding, embedded_at, created_at, updated_at)
          VALUES
            (${id}, ${input.question}, ${input.answer}, ${agents}, ${tags}, ${hash},
             ${reviewStatus}::"ReviewStatus", ${opts.sourceType}::"SourceType",
             ${opts.sourceRef ?? null}, ${input.sourceOrdinal ?? null}, ${input.sourceSpan ?? null},
             ${nextVersion}, ${vectorLiteral}::vector, NOW(), NOW(), NOW())`;
        versioned++;
        results[idx] = {
          id, question: input.question, answer: input.answer, agents, tags,
          content_hash: hash, active: true, version: nextVersion, supersedes: existing.id,
        };
      } else if (existing) {
        await db.$executeRaw`
          UPDATE "FaqChunk" SET question=${input.question}, answer=${input.answer},
            agents=${agents}, tags=${tags}, content_hash=${hash},
            review_status=${reviewStatus}::"ReviewStatus", source_span=${input.sourceSpan ?? null},
            embedding=${vectorLiteral}::vector,
            embedded_at=NOW(), updated_at=NOW()
          WHERE id=${existing.id}`;
        updated++;
        results[idx] = { ...existing, question: input.question, answer: input.answer, agents, tags, content_hash: hash };
      } else {
```

(the existing `else` INSERT branch stays as-is, but add `version` to its column and value lists with the literal `1`).

(d) Return `versioned` from both return statements:

```ts
    if (!toEmbed.length) return { created, updated, unchanged, versioned, chunks: results };
    ...
    return { created, updated, unchanged, versioned, chunks: results };
```

- [ ] **Step 4: Run the tests**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
npx tsc --noEmit -p tsconfig.json
```
Expected: PASS, `tsc` clean.

**If `tsc` fails:** callers destructure `upsertBatch`'s result. Adding a field is additive and should not break them — but `ExtractionResult` in `faq-extraction.service.ts` builds its return from those fields, and Task 5 extends it. If `tsc` points there, report rather than editing `faq-extraction.service.ts` in this task.

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-ingestion.service.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): re-ingesting changed content adds a version instead of blanking the live answer"
```

---

### Task 4: approval supersedes the previous version

**Files:**
- Modify: `src/faq/faq-ingestion.service.ts` (`supersede`), `src/faq/faq-review.service.ts` (`approve`)
- Test: `src/faq/faq-review.service.spec.ts`

**Interfaces:**
- Consumes: `FaqChunk.version`, `superseded_by`.
- Produces: `supersedePrevious(chunk, tenantDb?)` on `FaqIngestionService`.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-review.service.spec.ts`:

```ts
describe('approve supersedes the previous version', () => {
  it('deactivates the older approved version and links it to the new one', async () => {
    const nuevo = {
      id: 'nuevo-1', review_status: 'PENDING_REVIEW', active: true, version: 2,
      source_ref: 'document:envios', source_ordinal: 1,
      question: '¿Cuanto tarda?', answer: 'Tarda 72 horas.',
    };
    const { service, tenantDb } = makeService(nuevo);
    tenantDb.faqChunk.findFirst = jest.fn().mockResolvedValue({ id: 'viejo-1', version: 1 });

    await service.approve('nuevo-1', {}, tenantDb);

    const sql = tenantDb.$executeRaw.mock.calls.map((c: any[]) => c[0].join('?')).join(' | ');
    expect(sql).toContain('superseded_by');
    expect(sql).toContain('active=false');
    const values = tenantDb.$executeRaw.mock.calls.flatMap((c: any[]) => c.slice(1));
    expect(values).toContain('nuevo-1');
    expect(values).toContain('viejo-1');
  });

  // Aprobar la version 1 no tiene predecesor: no puede desactivar nada.
  it('does nothing extra when there is no previous version', async () => {
    const primero = {
      id: 'unico', review_status: 'PENDING_REVIEW', active: true, version: 1,
      source_ref: 'document:envios', source_ordinal: 1,
      question: '¿Cuanto tarda?', answer: 'Tarda 48 horas.',
    };
    const { service, tenantDb } = makeService(primero);
    tenantDb.faqChunk.findFirst = jest.fn().mockResolvedValue(null);

    await service.approve('unico', {}, tenantDb);

    const sql = tenantDb.$executeRaw.mock.calls.map((c: any[]) => c[0].join('?')).join(' | ');
    expect(sql).not.toContain('superseded_by');
  });

  // Un chunk manual (sin source_ref) no participa del versionado.
  it('skips superseding for a chunk with no source_ref', async () => {
    const manual = {
      id: 'manual-1', review_status: 'PENDING_REVIEW', active: true, version: 1,
      source_ref: null, source_ordinal: null,
      question: '¿Hola?', answer: 'Hola.',
    };
    const { service, tenantDb } = makeService(manual);
    tenantDb.faqChunk.findFirst = jest.fn();

    await service.approve('manual-1', {}, tenantDb);

    expect(tenantDb.faqChunk.findFirst).not.toHaveBeenCalled();
  });
});
```

Reuse the file's existing `makeService` helper; if it does not accept the chunk to return from `findUnique`, extend it rather than duplicating it.

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-review.service.spec.ts
```
Expected: FAIL — nothing writes `superseded_by`.

- [ ] **Step 3: Implement `supersedePrevious` in `faq-ingestion.service.ts`**

Add beside the existing `supersede`:

```ts
  // Al aprobar una version nueva, la anterior deja de servir y queda apuntando a
  // la que la reemplazo. No se borra nada: si mañana hay una discusion sobre lo
  // que el bot contesto, la respuesta vieja tiene que seguir siendo legible
  // (PRD 2 §6.4 y el "nada se borra duro" del final de §6).
  async supersedePrevious(chunk: any, tenantDb?: any): Promise<string | null> {
    const db = this.db(tenantDb);
    if (!chunk?.source_ref || chunk.source_ordinal === null || chunk.source_ordinal === undefined) return null;

    const previous = await db.faqChunk.findFirst({
      where: {
        source_ref: chunk.source_ref,
        source_ordinal: chunk.source_ordinal,
        active: true,
        version: { lt: chunk.version ?? 1 },
      },
      orderBy: { version: 'desc' },
    });
    if (!previous) return null;

    await db.$executeRaw`
      UPDATE "FaqChunk" SET active=false, superseded_by=${chunk.id}, updated_at=NOW()
      WHERE id=${previous.id}`;
    return previous.id;
  }
```

- [ ] **Step 4: Call it from `approve`**

In `faq-review.service.ts`, after the chunk has been marked `APPROVED` and before returning, add:

```ts
    // El orden importa: primero queda aprobada la nueva, despues se baja la
    // vieja. Al reves habria un instante sin ninguna version sirviendo.
    await this.ingestion.supersedePrevious({ ...chunk, ...patch, id }, tenantDb);
```

If `FaqReviewService` does not already hold a `FaqIngestionService`, inject it — and update every construction site in `faq-review.service.spec.ts` and any other spec, then run `tsc` (see Global Constraints: `ts-jest` will not catch a stale arity).

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 6: Commit**

```bash
git add src/faq/faq-ingestion.service.ts src/faq/faq-review.service.ts src/faq/faq-review.service.spec.ts
git commit -m "feat(faq): approving a new version retires the one it replaces"
```

---

### Task 5: wire the stable ref and report what is missing

**Files:**
- Modify: `src/faq/faq-extraction.service.ts`
- Test: `src/faq/faq-extraction.service.spec.ts`

**Interfaces:**
- Consumes: `documentSourceRef` (Task 2), `upsertBatch(...).versioned` (Task 3).
- Produces: `ExtractionResult` gains `versioned: number` and `missing: Array<{ id: string; question: string }>`.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-extraction.service.spec.ts`:

```ts
describe('re-ingestion of a named document', () => {
  it('reuses the same source ref for the same document name', async () => {
    const { service, ingestion } = makeService([goodCandidate]);
    await service.extractFromText(DOC, { sourceName: 'Politica de garantia' }, {});
    await service.extractFromText(DOC, { sourceName: 'Politica de garantia' }, {});

    const first = ingestion.upsertBatch.mock.calls[0][1].sourceRef;
    const second = ingestion.upsertBatch.mock.calls[1][1].sourceRef;
    expect(first).toBe(second);
    expect(first).toBe('document:politica-de-garantia');
  });

  it('still uses a unique ref when the document has no name', async () => {
    const { service, ingestion } = makeService([goodCandidate]);
    await service.extractFromText(DOC, {}, {});
    await service.extractFromText(DOC, {}, {});

    expect(ingestion.upsertBatch.mock.calls[0][1].sourceRef)
      .not.toBe(ingestion.upsertBatch.mock.calls[1][1].sourceRef);
  });

  // §6.5: lo que falta se INFORMA, no se archiva solo. Una seccion ausente casi
  // siempre es un artefacto de la re-subida, no una decision del cliente.
  it('reports chunks absent from the new upload without archiving them', async () => {
    const { service, ingestion } = makeService([goodCandidate]);
    const tenantDb = {
      faqChunk: {
        findMany: jest.fn().mockResolvedValue([
          { id: 'viejo-9', question: '¿Seccion que ya no esta?', source_ordinal: 9 },
        ]),
        // Mockeados para poder afirmar que NO se llaman: son las vias por las
        // que un archivado accidental pasaria si alguien "mejora" esto despues.
        update: jest.fn(),
        updateMany: jest.fn(),
      },
      $executeRaw: jest.fn(),
      aiUsage: { create: jest.fn() },
    };
    ingestion.upsertBatch.mockResolvedValue({ created: 1, updated: 0, unchanged: 0, versioned: 0, chunks: [] });

    const result = await service.extractFromText(DOC, { sourceName: 'garantia' }, tenantDb);

    expect(result.missing).toEqual([{ id: 'viejo-9', question: '¿Seccion que ya no esta?' }]);
    // Nada se archiva, por ninguna via. Afirmar solo sobre $executeRaw seria
    // flojo: `ingestion` esta mockeado, asi que casi nada lo llamaria igual y el
    // test pasaria aunque el codigo archivara por el cliente de Prisma.
    expect(tenantDb.$executeRaw).not.toHaveBeenCalled();
    expect(tenantDb.faqChunk.update).not.toHaveBeenCalled();
    expect(tenantDb.faqChunk.updateMany).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-extraction.service.spec.ts
```
Expected: FAIL — `sourceRef` differs between calls; `missing` is undefined.

- [ ] **Step 3: Implement**

In `faq-extraction.service.ts`:

(a) Import the helper:

```ts
import { documentSourceRef } from './faq-source-ref';
```

(b) Replace the `sourceRef` line:

```ts
    const sourceRef = documentSourceRef(opts.sourceName);
```

(c) Extend `ExtractionResult`:

```ts
export interface ExtractionResult {
  sourceRef: string;
  candidates: number;
  kept: number;
  dropped: DroppedCandidate[];
  created: number;
  updated: number;
  unchanged: number;
  versioned: number;
  /** Chunks vivos de este documento que la subida nueva ya no trae. Se informan
   *  para que un humano decida: una seccion ausente suele ser un artefacto de la
   *  re-subida, no una baja intencional (PRD 2 §6.5). NUNCA se archivan solos. */
  missing: Array<{ id: string; question: string }>;
}
```

(d) After the `upsertBatch` call, compute `missing`:

```ts
    const keptOrdinals = kept.map((k) => k.sourceOrdinal);
    let missing: Array<{ id: string; question: string }> = [];
    if (tenantDb?.faqChunk?.findMany) {
      try {
        const survivors = await tenantDb.faqChunk.findMany({
          where: {
            source_ref: sourceRef,
            active: true,
            source_ordinal: { notIn: keptOrdinals.length ? keptOrdinals : [-1] },
          },
          select: { id: true, question: true },
        });
        missing = survivors.map((s: any) => ({ id: s.id, question: s.question }));
      } catch (err: any) {
        // Informar que falta algo es util, pero no vale romper una extraccion
        // que ya se hizo bien por no poder calcularlo.
        this.logger.warn(`No se pudo calcular los chunks ausentes de ${sourceRef}: ${err?.message ?? err}`);
      }
    }
```

(e) Add `versioned: result.versioned ?? 0` and `missing` to the returned object.

- [ ] **Step 4: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-extraction.service.ts src/faq/faq-extraction.service.spec.ts
git commit -m "feat(faq): re-uploading a named document updates it and reports what is gone"
```

---

### Task 6: duplicate detection

**Files:**
- Create: `src/faq/faq-duplicates.ts`
- Test: `src/faq/faq-duplicates.spec.ts`

**Interfaces:**
- Produces: `questionSimilarity(a, b): number`, `findDuplicates(candidates, existing): DuplicateFlag[]`, `DUPLICATE_THRESHOLD`.

**This module FLAGS. It must never drop.** That is the opposite of `faq-support.ts`, deliberately, and the reason is worth holding onto: dropping an ungrounded answer protects a customer from a confident fabrication, so a false drop is cheap. A "duplicate" that is not one is just two different questions that share vocabulary — dropping it silently loses real content the client wrote, and nobody ever sees that it happened. Different cost, different default.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-duplicates.spec.ts`:

```ts
import { questionSimilarity, findDuplicates, DUPLICATE_THRESHOLD } from './faq-duplicates';

describe('questionSimilarity', () => {
  it('scores 1 for the same question written the same way', () => {
    expect(questionSimilarity('¿Cuanto dura la garantia?', '¿Cuanto dura la garantia?')).toBe(1);
  });

  it('ignores accents, case and punctuation', () => {
    expect(questionSimilarity('¿Cuánto dura la garantía?', 'cuanto dura la garantia')).toBe(1);
  });

  it('scores 0 for questions with nothing in common', () => {
    expect(questionSimilarity('¿Cuanto dura la garantia?', '¿Hacen envios a Chile?')).toBe(0);
  });

  it('is symmetric', () => {
    const a = '¿Cuanto dura la garantia del empapelado?';
    const b = '¿Que garantia tiene el empapelado?';
    expect(questionSimilarity(a, b)).toBeCloseTo(questionSimilarity(b, a));
  });

  it('returns 0 when either side is empty', () => {
    expect(questionSimilarity('', '¿Cuanto dura?')).toBe(0);
    expect(questionSimilarity('¿Cuanto dura?', '')).toBe(0);
  });
});

describe('findDuplicates', () => {
  const existing = [
    { id: 'e1', question: '¿Cuanto dura la garantia del empapelado?' },
    { id: 'e2', question: '¿Hacen envios a todo el pais?' },
  ];

  it('flags a candidate that restates an existing question', () => {
    const flags = findDuplicates([{ question: '¿Cuanto dura la garantia del empapelado?' }], existing);
    expect(flags).toHaveLength(1);
    expect(flags[0].existingId).toBe('e1');
    expect(flags[0].similarity).toBe(1);
  });

  it('does not flag a genuinely different question', () => {
    expect(findDuplicates([{ question: '¿Puedo pagar en cuotas?' }], existing)).toHaveLength(0);
  });

  it('reports the index of the candidate so the caller can map it back', () => {
    const flags = findDuplicates(
      [{ question: '¿Puedo pagar en cuotas?' }, { question: '¿Hacen envios a todo el pais?' }],
      existing,
    );
    expect(flags).toHaveLength(1);
    expect(flags[0].index).toBe(1);
  });

  it('flags against the most similar existing question, not merely the first', () => {
    const flags = findDuplicates([{ question: '¿Hacen envios a todo el pais?' }], existing);
    expect(flags[0].existingId).toBe('e2');
  });

  it('returns nothing when there is nothing to compare against', () => {
    expect(findDuplicates([{ question: '¿Algo?' }], [])).toEqual([]);
  });

  // El contrato que importa: esto INFORMA, no filtra. El llamador recibe la
  // lista completa de candidatos y aparte las marcas.
  it('never removes candidates — it only describes them', () => {
    const candidates = [{ question: '¿Cuanto dura la garantia del empapelado?' }];
    const flags = findDuplicates(candidates, existing);
    expect(candidates).toHaveLength(1);
    expect(flags[0].index).toBe(0);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-duplicates.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

Create `src/faq/faq-duplicates.ts`:

```ts
// ¿Esta pregunta ya esta en la base? Con volumen, la misma pregunta entra varias
// veces por caminos distintos — el pack de la vertical, el CSV del cliente y el
// documento que subio despues (PRD 2 §9, fase 4).
//
// ESTO MARCA, NO DESCARTA — al reves que faq-support.ts, y a proposito. Tirar una
// respuesta sin respaldo protege al cliente final de una invencion, asi que
// equivocarse ahi sale barato. Un "duplicado" que en realidad no lo es son dos
// preguntas distintas que comparten vocabulario: descartarla pierde contenido
// real que el cliente escribio, y nadie se entera de que paso.

/** Arriba de esto, dos preguntas se consideran la misma para mostrarselo a un humano. */
export const DUPLICATE_THRESHOLD = 0.8;

export interface DuplicateFlag {
  index: number;
  existingId: string;
  similarity: number;
}

const STOPWORDS = new Set([
  'a','al','como','con','cual','cuales','cuando','cuanto','cuanta','de','del','desde','donde','el','en',
  'es','esta','hay','la','las','lo','los','me','mi','o','para','por','que','se','si','sobre','su','sus',
  'te','tiene','un','una','uno','vos','y','ya',
]);

function contentWords(text: string): Set<string> {
  return new Set(
    (text ?? '')
      .normalize('NFD')
      .replace(/[\u0300-\u036f]/g, '')
      .toLowerCase()
      .split(/[^a-z0-9]+/)
      .filter((w) => w.length > 1 && !STOPWORDS.has(w)),
  );
}

/** Jaccard sobre palabras de contenido: simetrico, que es lo que uno espera de
 *  "parecido" — a diferencia del ratio de faq-support, que es direccional a
 *  proposito (la respuesta se apoya en el span, no al reves). */
export function questionSimilarity(a: string, b: string): number {
  const wa = contentWords(a);
  const wb = contentWords(b);
  if (!wa.size || !wb.size) return 0;

  let shared = 0;
  for (const w of wa) if (wb.has(w)) shared++;
  return shared / (wa.size + wb.size - shared);
}

export function findDuplicates(
  candidates: Array<{ question: string }>,
  existing: Array<{ id: string; question: string }>,
): DuplicateFlag[] {
  if (!existing.length) return [];

  const flags: DuplicateFlag[] = [];
  for (let index = 0; index < candidates.length; index++) {
    let best: DuplicateFlag | null = null;
    for (const e of existing) {
      const similarity = questionSimilarity(candidates[index].question, e.question);
      if (similarity >= DUPLICATE_THRESHOLD && (!best || similarity > best.similarity)) {
        best = { index, existingId: e.id, similarity };
      }
    }
    if (best) flags.push(best);
  }
  return flags;
}
```

- [ ] **Step 4: Run the tests**

```bash
npm test -- src/faq/faq-duplicates.spec.ts
npx tsc --noEmit -p tsconfig.json
```
Expected: PASS (11 tests), `tsc` clean.

**Report the observed similarity** for `'¿Cuanto dura la garantia del empapelado?'` vs `'¿Que garantia tiene el empapelado?'` — a near-miss pair. If it lands just under or just over `DUPLICATE_THRESHOLD`, that is a design signal about the threshold and I want the number, not a threshold nudged to make a test green.

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-duplicates.ts src/faq/faq-duplicates.spec.ts
git commit -m "feat(faq): flag questions that restate one already in the base"
```

---

### Task 7: surface duplicate flags on extraction

**Files:**
- Modify: `src/faq/faq-extraction.service.ts`
- Test: `src/faq/faq-extraction.service.spec.ts`

**Interfaces:**
- Consumes: `findDuplicates` (Task 6), `ExtractionResult` (Task 5).
- Produces: `ExtractionResult.duplicates: Array<{ question: string; existingId: string; similarity: number }>`.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-extraction.service.spec.ts`:

```ts
describe('duplicate flagging', () => {
  it('flags a kept candidate that restates an existing question, without dropping it', async () => {
    const { service, ingestion } = makeService([goodCandidate]);
    const tenantDb = {
      faqChunk: {
        findMany: jest.fn().mockResolvedValue([{ id: 'e1', question: goodCandidate.question }]),
      },
      aiUsage: { create: jest.fn() },
    };

    const result = await service.extractFromText(DOC, { sourceName: 'garantia' }, tenantDb);

    expect(result.kept).toBe(1);                      // NO se descarta
    expect(result.duplicates).toHaveLength(1);
    expect(result.duplicates[0].existingId).toBe('e1');
    expect(ingestion.upsertBatch).toHaveBeenCalled();  // igual se ingesta
  });

  it('reports no duplicates when nothing matches', async () => {
    const { service } = makeService([goodCandidate]);
    const tenantDb = {
      faqChunk: { findMany: jest.fn().mockResolvedValue([{ id: 'e1', question: '¿Puedo pagar en cuotas?' }]) },
      aiUsage: { create: jest.fn() },
    };
    const result = await service.extractFromText(DOC, { sourceName: 'garantia' }, tenantDb);
    expect(result.duplicates).toEqual([]);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-extraction.service.spec.ts
```
Expected: FAIL — `duplicates` is undefined.

- [ ] **Step 3: Implement**

(a) Import: `import { findDuplicates } from './faq-duplicates';`

(b) Add to `ExtractionResult`:

```ts
  /** Candidatos que repiten una pregunta que ya existe. Se informan y se
   *  ingestan igual: el revisor decide cual se queda (PRD 2 §9 fase 4). */
  duplicates: Array<{ question: string; existingId: string; similarity: number }>;
```

(c) Before the `upsertBatch` call — the flags describe what is about to be ingested, and it is ingested either way:

```ts
    let duplicates: ExtractionResult['duplicates'] = [];
    if (kept.length && tenantDb?.faqChunk?.findMany) {
      try {
        const already = await tenantDb.faqChunk.findMany({
          where: { active: true },
          select: { id: true, question: true },
        });
        duplicates = findDuplicates(kept, already).map((f) => ({
          question: kept[f.index].question,
          existingId: f.existingId,
          similarity: f.similarity,
        }));
      } catch (err: any) {
        this.logger.warn(`No se pudo chequear duplicados: ${err?.message ?? err}`);
      }
    }
```

(d) Add `duplicates` to the returned object.

- [ ] **Step 4: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-extraction.service.ts src/faq/faq-extraction.service.spec.ts
git commit -m "feat(faq): tell the reviewer which extracted questions already exist"
```

---

## Self-review notes (written with the spec open)

**Spec coverage of §6:**

| §6 clause | Task |
|---|---|
| 1. re-extraction reuses `source_ref` | 2, 5 |
| 2. unchanged hashes skipped, stay approved | already shipped in phase 1; Task 3 additionally refreshes a corrected span |
| 3. changed content → new version, previous stays approved and live | **3** |
| 4. on approval, `supersede()` sets `active:false` + `superseded_by` | **4** |
| 5. absent chunks surfaced, not auto-archived | **5** |
| "nothing is ever hard-deleted" | held: every path here is an `UPDATE` of `active`/`superseded_by` or an `INSERT` |

§9 phase 4's "duplicate detection" is Tasks 6-7.

**Known gaps, stated rather than hidden:**

- **CSV import keeps its per-upload `batchId`.** Task 2's helper is document-shaped, and giving imports a stable identity means deciding whether identity is the filename (renames break it) or something the user picks. That is a product question, not an engineering one, so this phase does not answer it silently. Re-uploading a CSV still duplicates.
- **`missing` is computed but nothing consumes it yet.** The endpoint returns it; there is no UI. That is the correct order — the frontend repo is a separate deploy.
- **Duplicate detection loads every active chunk's question into memory.** Fine at the volumes this phase is scoped for (a few thousand per tenant); it becomes a pgvector query if it ever isn't.
- **Two APPROVED+active versions could coexist** if `supersedePrevious` failed after `approve` committed. `approve` and the supersede are not in one transaction. Retrieval would then return both versions of the same chunk. Worth a follow-up; not introduced blindly — Task 4's ordering comment says why approve-then-supersede is nonetheless the right order.
