# FAQ import phases D, E, H (backend) — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finish the backend for PRD 4 — let a business report a wrong answer, accept a spreadsheet instead of a CSV, and extract knowledge from an uploaded document.

**Architecture:** Three independent additions. A `FaqReport` table and routes so the business can say "that answer is wrong" (phase D's other half). Spreadsheet parsing behind the existing import path, so XLSX and CSV are the same feature (phase E). File parsing in front of the existing extraction pipeline, so PDF and DOCX reuse span verification unchanged (phase H).

**Tech Stack:** NestJS 11, Prisma 7, PostgreSQL + pgvector, Jest/ts-jest, `exceljs`, `mammoth`, `pdf-parse`.

**Spec:** `f:\docs\Laika\docs\prd-faq-bulk-import.md` — phases D, E and H of §9.

## Why phases F and G need no backend

Verified before writing this: the tenant-context `/api/faq` routes already grant the tenant `admin` role. `GET /api/faq` and `/review` and `/review/counts` are open to any authenticated user; `POST /api/faq/import`, `/:id/approve`, `/:id/reject` and `/extract` all carry `@Roles('admin', 'superadmin')`.

So a tenant admin can already list, import, approve and reject over the API. **F and G are frontend-only** — what was missing was the screen, exactly as PRD 3 §3 intended. They live in the frontend plan.

## Global Constraints

- Comments and user-facing strings in **español rioplatense (voseo)**.
- **The load-bearing invariant: only `review_status = 'APPROVED'` AND `active = true` chunks are retrievable by the bot.** Nothing here may publish content without a human.
- Migrations are **hand-written SQL**, idempotent, applied to every tenant database at boot by `TenantMigrationsService`.
- **`tsc` cannot see query shapes** — DB access goes through an `any`-typed accessor. Verify column names against `prisma/schema.prisma` by hand.
- Both gates on every task: `npm test -- src/faq/ src/tenants/` and `npx tsc --noEmit -p tsconfig.json`.
- Never run `git stash`. Stage only files you edited. **`CLAUDE.md` has uncommitted user changes — do not touch it.**

---

### Task 1: `FaqReport` — let the business say "that is wrong"

PRD 4 §5. The lint validates shape; nobody validates truth. The business is the only party who knows the warranty is six months and not nine, and today they have no way to tell us. §11 lists "visibility without a channel" as a risk that produces frustration rather than corrections.

**Files:**
- Create: `prisma/migrations/20260906120000_faq_report/migration.sql`, `src/faq/faq-report.service.ts`, `src/faq/faq-report.service.spec.ts`
- Modify: `prisma/schema.prisma`, `src/faq/faq.controller.ts`, `src/faq/faq.module.ts`, `src/tenants/tenants.controller.ts`, `src/tenants/tenants.service.ts`

- [ ] **Step 1: Migration**

```sql
-- Un reporte del negocio sobre una respuesta suya: "esto no es asi".
--
-- Vive en la base DEL TENANT junto al chunk que reporta. No es una tabla de
-- soporte global: el dato que corrige es del cliente y el que lo corrige es el
-- cliente. Mantenerlo al lado del contenido tambien hace que se borre con el
-- tenant, sin dejar huerfanos en master.
CREATE TABLE IF NOT EXISTS "FaqReport" (
  "id"          TEXT PRIMARY KEY,
  "chunk_id"    TEXT NOT NULL,
  "note"        TEXT,
  "status"      TEXT NOT NULL DEFAULT 'OPEN',
  "reported_by" TEXT,
  "created_at"  TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "resolved_at" TIMESTAMP(3),
  "resolved_by" TEXT
);

-- La consulta caliente es "reportes abiertos de este tenant".
CREATE INDEX IF NOT EXISTS "FaqReport_status_idx" ON "FaqReport" ("status");
CREATE INDEX IF NOT EXISTS "FaqReport_chunk_id_idx" ON "FaqReport" ("chunk_id");
```

**No foreign key to `FaqChunk` on purpose.** Nothing in this system is hard-deleted, so a dangling `chunk_id` is not a real risk — and a FK would make the report disappear if a chunk row were ever removed, which is exactly the history worth keeping.

- [ ] **Step 2: Schema**

```prisma
model FaqReport {
  id          String    @id @default(uuid())
  chunk_id    String
  note        String?
  status      String    @default("OPEN")
  reported_by String?
  created_at  DateTime  @default(now())
  resolved_at DateTime?
  resolved_by String?

  @@index([status])
  @@index([chunk_id])
}
```

Then `npx prisma generate`.

- [ ] **Step 3: Write the failing test**

Create `src/faq/faq-report.service.spec.ts`:

```ts
import { FaqReportService } from './faq-report.service';

describe('FaqReportService', () => {
  function makeDb(chunk: any = { id: 'c1', question: '¿Q?' }) {
    return {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(chunk) },
      faqReport: {
        create: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'r1', ...data })),
        findMany: jest.fn().mockResolvedValue([]),
        update: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'r1', ...data })),
        count: jest.fn().mockResolvedValue(2),
      },
    };
  }

  it('creates a report against an existing chunk', async () => {
    const service = new FaqReportService({} as any);
    const db = makeDb();
    const r = await service.create('c1', 'La garantia es de 9 meses, no 6.', db, 'user-7');
    expect(db.faqReport.create).toHaveBeenCalled();
    expect(db.faqReport.create.mock.calls[0][0].data.chunk_id).toBe('c1');
    expect(db.faqReport.create.mock.calls[0][0].data.reported_by).toBe('user-7');
    expect(r.status).toBe('OPEN');
  });

  // Reportar algo que no existe es un 404, no una fila huerfana.
  it('refuses to report a chunk that does not exist', async () => {
    const service = new FaqReportService({} as any);
    const db = makeDb(null);
    await expect(service.create('nope', 'algo', db)).rejects.toThrow();
    expect(db.faqReport.create).not.toHaveBeenCalled();
  });

  // El reporte NO toca el chunk. Que el negocio diga "esto esta mal" no puede,
  // por si solo, sacar una respuesta del aire: eso lo decide quien cura, y
  // sacarla automaticamente convertiria el canal en un boton de despublicar.
  it('does not modify the chunk it reports', async () => {
    const service = new FaqReportService({} as any);
    const db: any = makeDb();
    db.faqChunk.update = jest.fn();
    db.$executeRaw = jest.fn();
    await service.create('c1', 'mal', db);
    expect(db.faqChunk.update).not.toHaveBeenCalled();
    expect(db.$executeRaw).not.toHaveBeenCalled();
  });

  it('lists open reports by default', async () => {
    const service = new FaqReportService({} as any);
    const db = makeDb();
    await service.list({}, db);
    expect(db.faqReport.findMany.mock.calls[0][0].where).toEqual({ status: 'OPEN' });
  });

  it('resolves a report and records who did it', async () => {
    const service = new FaqReportService({} as any);
    const db = makeDb();
    await service.resolve('r1', db, 'user-9');
    const data = db.faqReport.update.mock.calls[0][0].data;
    expect(data.status).toBe('RESOLVED');
    expect(data.resolved_by).toBe('user-9');
    expect(data.resolved_at).toBeInstanceOf(Date);
  });

  it('counts open reports for a badge', async () => {
    const service = new FaqReportService({} as any);
    const db = makeDb();
    expect(await service.openCount(db)).toBe(2);
    expect(db.faqReport.count.mock.calls[0][0].where).toEqual({ status: 'OPEN' });
  });
});
```

- [ ] **Step 4: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-report.service.spec.ts
```

- [ ] **Step 5: Implement**

Create `src/faq/faq-report.service.ts`, following the shape of `faq-review.service.ts` (same `private db(tenantDb?: any)` accessor, same `PrismaService` injection):

```ts
// El negocio marcando "esta respuesta esta mal".
//
// Es el unico control que existe sobre la VERDAD del contenido. El lint mira la
// forma —largo, precios, imperativos, placeholders— y no puede mirar otra cosa;
// nosotros tampoco. Quien sabe que la garantia es de 6 meses y no de 9 es el
// cliente, y hasta ahora no tenia como decirlo (PRD 4 §5).
//
// Un reporte NO cambia el chunk. Si lo desactivara, el canal seria un boton de
// despublicar y un malentendido dejaria al bot sin una respuesta correcta.
// Reporta; decide quien cura.
```

Methods: `create(chunkId, note, tenantDb?, reportedBy?)` (404s if the chunk does not exist), `list(filters, tenantDb?)` (defaults to `status: 'OPEN'`, newest first), `resolve(id, tenantDb?, resolvedBy?)`, `openCount(tenantDb?)`.

Register it in `FaqModule`'s providers **and exports** — `TenantsService` injects it in step 7.

- [ ] **Step 6: Tenant-context routes**

In `faq.controller.ts` — this is the half the business uses, so the create route must be reachable by **any authenticated tenant user**, not just `admin`. A seller who spots a wrong answer should be able to say so:

```ts
  @Post(':id/report')
  report(@Param('id') id: string, @Body() body: { note?: string }, @Req() req: any) {
    return this.reports.create(id, (body?.note ?? '').trim() || null, req.tenantDb, req.user?.id);
  }

  @Get('reports')
  listReports(@Query('status') status: string | undefined, @Req() req: any) {
    return this.reports.list({ status }, req.tenantDb);
  }
```

`@Get('reports')` must be declared **before** `@Get(':id')` if one exists — check, and note the result in your report.

- [ ] **Step 7: Superadmin routes**

In `tenants.controller.ts` and `TenantsService`, mirroring the existing FAQ delegations (`faqDb(slug)` → pass the client through). Add `faqReports` as the **eleventh** constructor parameter:

```ts
  @Get(':slug/faq/reports')
  getFaqReports(@Param('slug') slug: string, @Query('status') status: string | undefined) { … }

  @Post(':slug/faq/reports/:id/resolve')
  resolveFaqReport(@Param('slug') slug: string, @Param('id') id: string, @Req() req: any) { … }
```

`TenantsService`'s constructor currently takes **ten** parameters: `prisma, railway, funnel, agents, factory, ai, faqIngestion, faqReview, faqRetrieval, faqImport`. Append `faqReports`. Every mock in the spec is `any`, so a wrong position passes the test while injecting the wrong mock — state all eleven in your report.

- [ ] **Step 8: Both gates, then commit**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
git commit -m "feat(faq): let the business report an answer that is wrong"
```

---

### Task 2: XLSX — the same import, a friendlier file

PRD 4 §6.2 and phase E. CSV is a programmer's format: it breaks on commas in text, on encoding, and on Excel choosing its own separator. The client filling this in is not a programmer.

**Dependency:** `exceljs`. Neither candidate is ideal — the npm `xlsx` package has a CVE history since SheetJS moved newer releases off npm, and `exceljs` 4.4.0 has not been published since late 2024. `exceljs` wins on being pure JS with no native build step and no known critical advisories, and PRD §11 already says to **scope what is parsed** rather than support Excel: one sheet, four columns, nothing else. Record that reasoning in the code.

**Files:**
- Modify: `package.json`, `src/faq/faq-import.service.ts`, `src/faq/faq.controller.ts`, `src/faq/faq-import.service.spec.ts`

- [ ] **Step 1: Install**

```bash
npm install exceljs
```

- [ ] **Step 2: Write the failing test**

Append to `src/faq/faq-import.service.spec.ts`. Build a real workbook in the test rather than a fixture file, so the test carries its own input:

```ts
import * as ExcelJS from 'exceljs';

async function xlsxBuffer(rows: string[][]): Promise<Buffer> {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('FAQ');
  ws.addRow(['pregunta', 'respuesta', 'agentes', 'tags']);
  rows.forEach((r) => ws.addRow(r));
  return Buffer.from(await wb.xlsx.writeBuffer());
}

describe('analyzeSpreadsheet — XLSX', () => {
  it('reads an xlsx with the same columns as the csv', async () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const buf = await xlsxBuffer([['¿Hacen envios?', 'Si, a todo el pais.', '', 'envios']]);
    const result = await service.analyzeFile(buf, 'lista.xlsx');
    expect(result.total).toBe(1);
    expect(result.willImport).toBe(1);
  });

  // Un xlsx con precios se rechaza igual que un csv con precios: el lint es el
  // mismo, y el formato del archivo no cambia que dato puede entrar.
  it('applies the same lint to an xlsx', async () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const buf = await xlsxBuffer([['¿Precio?', 'Sale $56.000 el m2.', '', 'precios']]);
    const result = await service.analyzeFile(buf, 'lista.xlsx');
    expect(result.willImport).toBe(0);
    expect(result.rejected[0].findings.some((f) => f.rule === 'precio')).toBe(true);
  });

  it('still reads a csv, chosen by extension', async () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    const buf = Buffer.from('pregunta,respuesta\n¿Q?,A.\n', 'utf8');
    const result = await service.analyzeFile(buf, 'lista.csv');
    expect(result.total).toBe(1);
  });

  it('rejects a file type it cannot read, naming what it accepts', async () => {
    const service = new FaqImportService({ upsertBatch: jest.fn() } as any);
    await expect(service.analyzeFile(Buffer.from('x'), 'lista.pdf')).rejects.toThrow(/csv|xlsx/i);
  });
});
```

- [ ] **Step 3: Implement**

`analyzeCsv` becomes the CSV branch of a format-dispatching `analyzeFile(buffer, filename)`. **`analyzeCsv` and `importCsv` keep working unchanged** — other code and tests call them.

```ts
  // Elige el parser por extension y despues sigue el MISMO camino: mismas
  // columnas, mismo lint, mismo tope de filas. El formato del archivo no
  // cambia que dato puede entrar, solo como se lee.
  async analyzeFile(buffer: Buffer, filename?: string): Promise<ImportAnalysis> { … }
```

For the XLSX branch: read the **first worksheet only**, treat row 1 as headers, map through the existing `findColumn` so Spanish/English headers work identically, and hand the resulting records to the same row loop the CSV path uses. Do not support formulas, multiple sheets, or anything beyond flat cell text — read `cell.text`, not `cell.value`, so a number or a date arrives as the string a human typed.

Add `importFile(buffer, filename, tenantDb?, tenant?, createdBy?)` alongside `importCsv`, and point the controller and `TenantsService` at it, passing `file.originalname`.

- [ ] **Step 4: Serve an XLSX template**

Add `GET /api/faq/import/template.xlsx` generating a workbook with the four headers, three example rows, and **an instructions sheet** whose first line is the price rule — PRD §8 says the template must say it *before* the client writes, not after twenty rows bounce:

> Poné la política, no el número: "hay descuento por transferencia" sí, "20% de descuento" no.

Keep the existing CSV template endpoint working.

- [ ] **Step 5: Both gates, then commit**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
git commit -m "feat(faq): accept a spreadsheet, not just a csv"
```

---

### Task 3: PDF and DOCX — a document instead of pasted text

PRD 4 §7 and phase H. `POST /api/faq/extract` already takes pasted text, asks a model for candidates, requires each to cite a verbatim excerpt, verifies the excerpt really exists, and **drops** anything ungrounded. All of that is built and reviewed. What is missing is the front door.

**Dependencies:** `mammoth` (DOCX, actively maintained) and `pdf-parse` (PDF).

**Files:**
- Modify: `package.json`, `src/faq/faq-extraction.service.ts`, `src/faq/faq.controller.ts`, `src/tenants/tenants.controller.ts`, `src/tenants/tenants.service.ts`
- Create: `src/faq/faq-document.ts`, `src/faq/faq-document.spec.ts`

- [ ] **Step 1: Install**

```bash
npm install mammoth pdf-parse
```

- [ ] **Step 2: Write the failing test**

Create `src/faq/faq-document.spec.ts`. Test the dispatch and the failure modes; do not test that `mammoth` parses DOCX — that is its job, not ours:

```ts
import { documentToText, MIN_DOCUMENT_CHARS } from './faq-document';

describe('documentToText', () => {
  it('passes plain text through', async () => {
    const text = 'a'.repeat(MIN_DOCUMENT_CHARS + 10);
    expect(await documentToText(Buffer.from(text, 'utf8'), 'politica.txt')).toContain('aaa');
  });

  it('rejects a file type it cannot read, naming what it accepts', async () => {
    await expect(documentToText(Buffer.from('x'), 'foto.png')).rejects.toThrow(/pdf|docx|txt/i);
  });

  // Un PDF de puras imagenes extrae texto vacio. Sin este chequeo, el pipeline
  // gastaria una llamada al modelo sobre nada y devolveria "0 candidatos", que
  // le suena al usuario a que su documento no servia — cuando el problema es
  // que es un escaneo.
  it('rejects a document that yielded almost no text', async () => {
    await expect(documentToText(Buffer.from('%PDF-1.4 fake'), 'escaneo.pdf')).rejects.toThrow();
  });
});
```

- [ ] **Step 3: Implement `faq-document.ts`**

```ts
/** Minimo de texto util para que valga la pena mandarlo al modelo. Por debajo
 *  de esto casi siempre es un PDF escaneado (puras imagenes, sin capa de
 *  texto): extraer 12 caracteres y seguir gastaria una llamada al modelo y le
 *  devolveria al usuario "0 candidatos", que suena a que su documento no
 *  servia en vez de a que es un escaneo. */
export const MIN_DOCUMENT_CHARS = 200;

export async function documentToText(buffer: Buffer, filename?: string): Promise<string> { … }
```

Dispatch on extension: `.txt`/`.md` decode as UTF-8; `.docx` through `mammoth.extractRawText`; `.pdf` through `pdf-parse`. Anything else throws a `BadRequestException` naming the accepted types. After extraction, throw if the text is under `MIN_DOCUMENT_CHARS`, with a message that says a scanned PDF is the likely cause — that is the difference between a user retrying usefully and giving up.

**Do not touch the extraction pipeline itself.** This produces text; `extractFromText` already handles everything after that, including the span verification that makes the whole thing safe.

- [ ] **Step 4: Wire the upload routes**

Tenant-context on `FaqController` (`@Roles('admin','superadmin')`, `FileInterceptor('file')`), and the superadmin mirror on `TenantsController`. Both read the file, call `documentToText`, and pass the result plus `sourceName` (default to the original filename) into the existing extraction call.

- [ ] **Step 5: Both gates, then commit**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
git commit -m "feat(faq): extract knowledge from an uploaded pdf or docx"
```

---

## Self-review notes

**Spec coverage:** phase D's backend half is Task 1 (the read side already exists — `GET /api/faq` is open to any authenticated tenant user). Phase E is Task 2. Phase H is Task 3. Phases F and G need no backend at all, verified against the route roles.

**Deliberately not here:**
- No change to who may approve. `POST /api/faq/:id/approve` already allows the tenant `admin`; PRD §13 question 4 asks whether it *should*, and that is a product decision, not something to change in either direction here.
- No auto-unpublish on report. Task 1's third test pins that a report leaves the chunk alone — otherwise the channel becomes a despublish button and one misunderstanding takes a correct answer off the air.
- The 500-row cap is unchanged, and applies to XLSX identically.

**Known risks:** three new dependencies, which is the largest supply-chain change this feature has made — `exceljs` is stale (see Task 2's reasoning), and `pdf-parse` pulls a PDF engine. All three are confined to parsing a file into text; none touches retrieval, the lint, or the review gate.
