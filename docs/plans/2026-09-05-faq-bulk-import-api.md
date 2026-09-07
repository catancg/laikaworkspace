# FAQ bulk import API (PRD 4 phases A–C, backend) — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a superadmin upload a file of FAQ entries for any tenant, see exactly what will happen before anything is written, undo a batch, and leave a record of who loaded it.

**Architecture:** Almost no new logic. `FaqImportService.importCsv` already parses, lints per row and reports rejections — it just also writes, in the same call. This splits analysis from writing, exposes both through superadmin routes with the slug in the path, adds a batch withdraw, and records `created_by` so an import is attributable.

**Tech Stack:** NestJS 11, Prisma 7 (raw tagged templates for vector I/O), PostgreSQL + pgvector, Jest/ts-jest, multer via `@nestjs/platform-express`.

**Spec:** `f:\docs\Laika\docs\prd-faq-bulk-import.md` — phases A, B and C of §9. Phases D–H are out of scope: D needs a reporting-channel design, E is blocked on §13 question 1, F depends on D, G is blocked on §13 question 4, H is deferred.

## Global Constraints

- Comments and user-facing strings in **español rioplatense (voseo)**. Match the surrounding file.
- **The load-bearing invariant: only `review_status = 'APPROVED'` AND `active = true` chunks are retrievable by the bot.** Import writes `PENDING_REVIEW`; nothing here may change that.
- **Cross-tenant routes are security-sensitive.** Everything new goes on `TenantsController`, which carries class-level `@UseGuards(JwtAuthGuard, RolesGuard)` and `@Roles('superadmin')`. Never add a slug-in-path FAQ route to `FaqController`, and never add a per-method guard — the class-level one is the gate.
- Migrations are **hand-written SQL** in `prisma/migrations/<timestamp>_<nombre>/`, idempotent (`IF NOT EXISTS`), because `TenantMigrationsService` applies them to every tenant database at boot.
- **`tsc` cannot see query shapes.** DB access goes through an `any`-typed accessor, so wrong column names and misaligned raw-SQL value lists are invisible to `tsc` and to `ts-jest`. Verify column names against `prisma/schema.prisma` by hand and count INSERT columns against values.
- Both gates on every task, reported separately: `npm test -- src/faq/ src/tenants/` and `npx tsc --noEmit -p tsconfig.json`.
- Never run `git stash`. Stage only files you edited, by explicit path. **`CLAUDE.md` has uncommitted user changes — do not touch it.**

---

## What already exists (do not rebuild)

| Piece | Where |
|---|---|
| CSV parsing, column detection ES/EN, per-row lint, 500-row cap | `FaqImportService.importCsv` |
| `ImportResult` = `{ batchId, total, imported, rejected[], warnings[] }` | same file |
| Static CSV template, **reachable by a superadmin already** (needs no `tenantDb`) | `GET /api/faq/import/template` |
| Archiving every active chunk of a `source_ref` | `FaqIngestionService.supersede(sourceRef, [])` |
| Nine superadmin FAQ routes to mirror | `TenantsController` |

---

### Task 1: `created_by` — make an import attributable

PRD 4 §4.3. `FaqChunk` records `reviewed_by`, `updated_by` and `superseded_by`; there is **no** record of who created a row. Every accountability claim in that PRD depends on this existing.

**On-behalf-of needs no second column.** Each tenant has its own database, so a superadmin's user id appearing inside a tenant's `FaqChunk` table *is* the on-behalf-of signal: that id does not exist in that tenant's `User` table. One column carries both facts.

**Files:**
- Create: `prisma/migrations/20260905120000_faq_chunk_created_by/migration.sql`
- Modify: `prisma/schema.prisma`, `src/faq/faq-ingestion.service.ts`, `src/faq/faq-import.service.ts`, `src/faq/faq.controller.ts`, `src/tenants/tenants.service.ts`
- Test: `src/faq/faq-ingestion.service.spec.ts`

**Interfaces:**
- Produces: `UpsertOpts.createdBy?: string`. Consumed by Tasks 2 and 3.

- [ ] **Step 1: Migration**

```sql
-- Quien cargo la fila. `reviewed_by` lo escribe la aprobacion y `updated_by` la
-- edicion, pero de la creacion no quedaba rastro: una importacion de 80 filas no
-- se podia atribuir a nadie (PRD 4 §4.3).
--
-- No hace falta una segunda columna para "en nombre de": cada tenant tiene su
-- propia base, asi que un id de superadmin dentro de la tabla de un tenant YA
-- significa que alguien de plataforma cargo eso para ese cliente — ese id no
-- existe en la tabla User de esa base.
ALTER TABLE "FaqChunk" ADD COLUMN IF NOT EXISTS "created_by" TEXT;
```

- [ ] **Step 2: Schema**

Beside `updated_by` in the `FaqChunk` model:

```prisma
  created_by     String?
```

Then `npx prisma generate`.

- [ ] **Step 3: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService createdBy', () => {
  function makeService() {
    const embeddingClient = { embed: jest.fn().mockResolvedValue({ vectors: [[0.1]], usage: {}, model: 'm' }) } as any;
    return new FaqIngestionService({} as any, embeddingClient);
  }

  function makeDb() {
    return {
      faqChunk: { findFirst: jest.fn().mockResolvedValue(null), aggregate: jest.fn() },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };
  }

  it('records who created a chunk', async () => {
    const service = makeService();
    const db = makeDb();
    await service.upsertBatch(
      [{ question: '¿Q?', answer: 'A.' }],
      { sourceType: 'IMPORT', sourceRef: 'import:1', reviewStatus: 'PENDING_REVIEW', createdBy: 'user-7' },
      db,
    );
    const sql = db.$executeRaw.mock.calls[0][0].join('?');
    expect(sql).toContain('created_by');
    expect(db.$executeRaw.mock.calls[0].slice(1)).toContain('user-7');
  });

  it('writes null when nobody is supplied', async () => {
    const service = makeService();
    const db = makeDb();
    await service.upsertBatch([{ question: '¿Q?', answer: 'A.' }], { sourceType: 'MANUAL' }, db);
    // El INSERT tiene otros nullables, asi que `toContain(null)` no discrimina.
    // Se afirma sobre la POSICION de created_by en la lista de columnas.
    const call = db.$executeRaw.mock.calls[0];
    const cols = call[0].join('?').match(/\(([^)]*created_by[^)]*)\)/)![1].split(',').map((c: string) => c.trim());
    const idx = cols.indexOf('created_by');
    expect(idx).toBeGreaterThan(-1);
    expect(call.slice(1)[idx]).toBeNull();
  });
});
```

- [ ] **Step 4: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — `createdBy` is not part of `UpsertOpts` and no INSERT mentions the column.

- [ ] **Step 5: Implement**

Add to `UpsertOpts`:

```ts
  /** Quien carga. En una base de tenant, un id de superadmin significa que
   *  plataforma cargo esto para el cliente (PRD 4 §4.3). */
  createdBy?: string;
```

**There are TWO INSERT statements in `upsertBatch`** — the create branch and the new-version branch. Both need `created_by` added to their column list and `${opts.createdBy ?? null}` to their value list, in matching positions.

**Both INSERTs are currently 16 columns / 16 values. After this they are 17 / 17.** Nothing in the toolchain checks that: hand-count both, and state both counts in your report.

- [ ] **Step 6: Thread it through the callers**

- `FaqImportService.importCsv` — add a `createdBy?: string` parameter and pass it into `upsertBatch`'s opts.
- `FaqController` create and import handlers — pass `req.user?.id`.
- `TenantsService.createTenantFaq` — accept and forward it; its controller passes `req.user?.id`.

- [ ] **Step 7: Both gates, then commit**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
git add prisma/schema.prisma prisma/migrations/20260905120000_faq_chunk_created_by/migration.sql src/faq/faq-ingestion.service.ts src/faq/faq-import.service.ts src/faq/faq.controller.ts src/tenants/tenants.service.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): record who loaded a chunk, so an import can be attributed"
```

---

### Task 2: split analysis from writing, and add batch withdraw

PRD 4 §6.3 and §6.5. Today a client learns twenty rows were rejected once eighty are already written.

**Files:**
- Modify: `src/faq/faq-import.service.ts`, `src/faq/faq-ingestion.service.ts`
- Test: `src/faq/faq-import.service.spec.ts`

**Interfaces:**
- Produces: `analyzeCsv(buffer)` returning `ImportAnalysis`; `importCsv(buffer, tenantDb?, tenant?, createdBy?)` unchanged in behaviour; `FaqIngestionService.withdrawBatch(sourceRef, tenantDb?)`. Consumed by Task 3.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-import.service.spec.ts`, following that file's existing helpers:

```ts
describe('analyzeCsv', () => {
  const csv = (rows: string) => Buffer.from('pregunta,respuesta\n' + rows, 'utf8');

  it('reports what would happen without touching the database', async () => {
    const ingestion = { upsertBatch: jest.fn() } as any;
    const service = new FaqImportService(ingestion);
    const result = service.analyzeCsv(csv('¿Hacen envios?,Si a todo el pais.\n'));

    expect(result.total).toBe(1);
    expect(result.willImport).toBe(1);
    expect(result.rejected).toHaveLength(0);
    // Lo que justifica la funcion entera: NO escribe.
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });

  it('reports a rejected row with its row number and reason', () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const result = service.analyzeCsv(csv('¿Precio?,El papel sale $56.000 el m2.\n'));

    expect(result.willImport).toBe(0);
    expect(result.rejected).toHaveLength(1);
    expect(result.rejected[0].row).toBe(1);
    expect(result.rejected[0].findings.some((f) => f.rule === 'precio')).toBe(true);
  });

  // La promesa que hace la pantalla: si pasa la vista previa, pasa la revision.
  // La UNICA excepcion es placeholder, que en intake avisa y en aprobacion corta.
  it('flags a placeholder as a warning, matching intake', () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const result = service.analyzeCsv(csv('¿Donde estan?,Estamos en {{direccion}}.\n'));

    expect(result.willImport).toBe(1);
    expect(result.warnings[0].findings.some((f) => f.rule === 'placeholder')).toBe(true);
  });

  it('rejects a file over the row cap without analysing it row by row', () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const rows = Array.from({ length: MAX_IMPORT_ROWS + 1 }, (_, i) => `¿P${i}?,R${i}.`).join('\n') + '\n';
    expect(() => service.analyzeCsv(csv(rows))).toThrow(BadRequestException);
  });
});

describe('importCsv still writes', () => {
  it('analyses and then writes in one call', async () => {
    const ingestion = { upsertBatch: jest.fn().mockResolvedValue({ chunks: [] }) } as any;
    const service = new FaqImportService(ingestion);
    const result = await service.importCsv(Buffer.from('pregunta,respuesta\n¿Q?,A.\n', 'utf8'), {}, {}, 'user-7');

    expect(ingestion.upsertBatch).toHaveBeenCalled();
    expect(ingestion.upsertBatch.mock.calls[0][1].createdBy).toBe('user-7');
    expect(result.imported).toBe(1);
  });
});
```

Import `MAX_IMPORT_ROWS` and `BadRequestException` if the spec does not already.

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-import.service.spec.ts
```

- [ ] **Step 3: Implement the split**

Extract everything up to the write into a pure method. The existing `importCsv` keeps its behaviour by calling it:

```ts
export interface ImportAnalysis {
  total: number;
  willImport: number;
  rejected: RejectedRow[];
  warnings: { row: number; findings: LintFinding[] }[];
  /** Las filas que entrarian, ya normalizadas. `importCsv` las reusa para no
   *  parsear ni lintear dos veces. */
  chunks: FaqChunkInput[];
}

// Analiza sin escribir NADA. Es la mitad de `importCsv` que la pantalla necesita
// poder correr sola: hoy el que sube se entera de que se rechazaron veinte filas
// cuando ya hay ochenta escritas (PRD 4 §6.3).
analyzeCsv(buffer: Buffer): ImportAnalysis { … }
```

Move the parse, the cap check, the column detection and the per-row lint loop into `analyzeCsv`. Then:

```ts
  async importCsv(buffer: Buffer, tenantDb?: any, tenant?: any, createdBy?: string): Promise<ImportResult> {
    const analysis = this.analyzeCsv(buffer);
    const batchId = `import:${randomUUID()}`;
    // … el bloque de escritura existente, usando analysis.chunks y createdBy …
  }
```

Keep every existing behaviour: the same errors with the same messages, the same `batchId` format, the same partial-failure error naming the batch.

- [ ] **Step 4: Add `withdrawBatch`**

In `faq-ingestion.service.ts`, beside `supersede`:

```ts
  // Deshacer una carga entera. `supersede(ref, [])` ya desactiva todo lo de un
  // source_ref, pero deja el review_status como estaba: una fila retirada
  // quedaria "pendiente" e inactiva a la vez, que en pantalla no se entiende.
  // Esto la deja ARCHIVED, igual que reject().
  //
  // Archiva, no borra: un lote que llego a estar aprobado y contestando es
  // justamente el historial que el diseño de retencion quiere conservar.
  async withdrawBatch(sourceRef: string, tenantDb?: any): Promise<{ withdrawn: number }> {
    const db = this.db(tenantDb);
    const affected = await db.$executeRaw`
      UPDATE "FaqChunk"
      SET active=false, review_status='ARCHIVED'::"ReviewStatus", updated_at=NOW()
      WHERE source_ref=${sourceRef} AND active=true`;
    return { withdrawn: Number(affected) || 0 };
  }
```

Add a test asserting it filters by `source_ref`, sets both columns, and returns the count.

- [ ] **Step 5: Both gates, then commit**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
git add src/faq/faq-import.service.ts src/faq/faq-ingestion.service.ts src/faq/faq-import.service.spec.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): analyse an import without writing it, and allow withdrawing a batch"
```

---

### Task 3: superadmin routes

PRD 4 phase A. The tenth, eleventh and twelfth FAQ routes on `TenantsController`.

**Files:**
- Modify: `src/tenants/tenants.controller.ts`, `src/tenants/tenants.service.ts`, `src/tenants/tenants.module.ts`
- Test: `src/tenants/tenants.service.spec.ts`

- [ ] **Step 1: `MulterModule` — the thing that will bite you**

`FileInterceptor` needs multer options registered in the module that owns the controller. `FaqModule` and `ProductsModule` each do `MulterModule.register({ limits: { fileSize: 10 * 1024 * 1024 } })`. **`TenantsModule` does not**, so the interceptor would fail there. Add the same registration to `TenantsModule`'s `imports`, matching the existing convention rather than passing inline options.

- [ ] **Step 2: Write the failing test**

Append to `src/tenants/tenants.service.spec.ts`:

```ts
describe('TenantsService import delegation', () => {
  function make() {
    const tenantDb = { marker: 'tenant-db' } as any;
    const factory = { getClient: jest.fn().mockReturnValue(tenantDb) } as any;
    const ingestion = { withdrawBatch: jest.fn().mockResolvedValue({ withdrawn: 3 }) } as any;
    const importSvc = {
      analyzeCsv: jest.fn().mockReturnValue({ total: 2, willImport: 2, rejected: [], warnings: [], chunks: [] }),
      importCsv: jest.fn().mockResolvedValue({ batchId: 'import:1', total: 2, imported: 2, rejected: [], warnings: [] }),
    } as any;
    const prisma = {
      tenant: { findUnique: jest.fn().mockResolvedValue({ slug: 'itt', database_url: 'postgres://x', active: true }) },
    } as any;
    const service = new TenantsService(
      prisma, {} as any, {} as any, {} as any, factory, {} as any,
      ingestion, {} as any, {} as any, importSvc,
    );
    return { service, factory, ingestion, importSvc, tenantDb };
  }

  // Lo que hace que la vista previa sirva: no toca la base del tenant.
  it('analysing does not resolve or touch the tenant database', async () => {
    const { service, factory, importSvc } = make();
    await service.analyzeTenantImport('itt', Buffer.from('x'));
    expect(importSvc.analyzeCsv).toHaveBeenCalled();
    expect(importSvc.importCsv).not.toHaveBeenCalled();
  });

  it('importing writes to the tenant database and records the actor', async () => {
    const { service, importSvc, tenantDb } = make();
    await service.importTenantFaq('itt', Buffer.from('x'), 'user-7');
    expect(importSvc.importCsv).toHaveBeenCalledWith(expect.anything(), tenantDb, expect.anything(), 'user-7');
  });

  it('withdrawing a batch goes through the tenant database', async () => {
    const { service, ingestion, tenantDb } = make();
    const r = await service.withdrawTenantImport('itt', 'import:1');
    expect(ingestion.withdrawBatch).toHaveBeenCalledWith('import:1', tenantDb);
    expect(r.withdrawn).toBe(3);
  });
});
```

**`TenantsService`'s constructor currently takes nine parameters** in this order: `prisma, railway, funnel, agents, factory, ai, faqIngestion, faqReview, faqRetrieval`. Append `faqImport` as the tenth. Every mock is `any`, so a wrong position passes the test while injecting the wrong mock — get the order right and state it in your report.

- [ ] **Step 3: Implement the service methods**

```ts
  // Analiza sin resolver la base del tenant: no hay nada que consultar, y no
  // pedir el cliente deja explicito que esto no puede escribir.
  analyzeTenantImport(_slug: string, buffer: Buffer) {
    return this.faqImport.analyzeCsv(buffer);
  }

  async importTenantFaq(slug: string, buffer: Buffer, createdBy?: string) {
    const { tenant, db } = await this.faqDb(slug);
    return this.faqImport.importCsv(buffer, db, tenant, createdBy);
  }

  async withdrawTenantImport(slug: string, batchId: string) {
    const { db } = await this.faqDb(slug);
    return this.faqIngestion.withdrawBatch(batchId, db);
  }
```

- [ ] **Step 4: Implement the routes**

In `tenants.controller.ts`, in the FAQ block. **Add no guard decorators** — the class-level `@Roles('superadmin')` is the gate.

```ts
  @Post(':slug/faq/import')
  @UseInterceptors(FileInterceptor('file'))
  async importFaq(
    @Param('slug') slug: string,
    @UploadedFile() file: Express.Multer.File,
    @Query('dryRun') dryRun: string | undefined,
    @Req() req: any,
  ) {
    if (!file?.buffer) throw new BadRequestException('Subi un archivo CSV en el campo "file".');
    // dryRun analiza y no escribe. La pantalla lo usa para mostrar que va a
    // pasar ANTES de confirmar (PRD 4 §6.3).
    if (dryRun === 'true') return this.tenants.analyzeTenantImport(slug, file.buffer);
    return this.tenants.importTenantFaq(slug, file.buffer, req.user?.id);
  }

  @Post(':slug/faq/import/:batchId/withdraw')
  withdrawFaqImport(@Param('slug') slug: string, @Param('batchId') batchId: string) {
    return this.tenants.withdrawTenantImport(slug, batchId);
  }
```

**Route ordering:** `:slug/faq/import` is two segments after the slug and cannot collide with `:slug/faq/:id/approve` (three). `:slug/faq/import/:batchId/withdraw` is four and collides with nothing. Verify by listing the declared routes and reasoning about depth, as previous tasks did.

- [ ] **Step 5: Both gates, then commit**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
git add src/tenants/tenants.controller.ts src/tenants/tenants.service.ts src/tenants/tenants.module.ts src/tenants/tenants.service.spec.ts
git commit -m "feat(tenants): let a superadmin preview, import and withdraw a tenant's FAQ batch"
```

---

## Self-review notes

**Spec coverage:** PRD 4 phase A is Task 3 (routes; the template endpoint already works for a superadmin and needs nothing). Phase B is Task 2 (`dryRun` split and withdraw) plus Task 3's route wiring. Phase C is Task 1.

**Deliberately not here:**
- No frontend. Phase A's screen is a separate plan in `soylaika.frontend`.
- No XLSX. Blocked on PRD 4 §13 question 1.
- No tenant-facing anything. Phases D, F and G, and two of them are blocked on product decisions.
- The 500-row cap is unchanged. PRD 4 §13 question 2 asks whether it should come down; that is a decision, not an implementation detail.

**Known risk:** Task 1 touches both INSERT statements in `upsertBatch`, which are the exact statements a previous phase's review had to verify by hand because nothing type-checks a raw column list. Both go 16→17. The plan asks for the counts explicitly for that reason.
