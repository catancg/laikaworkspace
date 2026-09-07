# FAQ Content Ingestion — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make filling the FAQ knowledge base practical: bulk-import Q&A pairs from a CSV, run every incoming chunk through a content lint, and land them in a review queue where nothing reaches a customer until a human approves it.

**Architecture:** Three new backend units in the existing `src/faq/` module. `faq-lint.ts` is a pure, dependency-free rule set (no DB, no network) used at two moments — intake and approval. `FaqImportService` parses an uploaded CSV, lints each row, and hands the survivors to PRD 1's existing `FaqIngestionService.upsertBatch()` stamped `source_type: IMPORT` / `review_status: PENDING_REVIEW`. `FaqReviewService` owns the queue and the state transitions (approve / reject / edit-and-approve). The controller gains the HTTP surface. **No schema migration is needed** — `review_status`, `source_type`, `source_ref` and `source_ordinal` all shipped in PRD 1 as forward-compat columns, and this is the phase that finally uses them.

**Tech Stack:** NestJS 11, Prisma 7, `csv-parse` (already a dependency, used by `ProductsService`), `multer` + `FileInterceptor` (already used by `POST /api/products/import`), Jest.

**Spec:** `docs/prd-faq-content-ingestion.md` (workspace root, one level above this repo — `f:\docs\Laika\docs\prd-faq-content-ingestion.md`). Read it alongside this plan; `§` references point into it. Its predecessor `docs/prd-rag-knowledge-layer.md` (PRD 1) is already implemented — its §15 records where the shipped code differs from its own spec, which matters here because this plan calls PRD 1's `FaqIngestionService`.

## Global Constraints

- Comments and log messages in Spanish, matching the rest of `soylaika.backend`.
- Every service method that touches tenant data takes `tenantDb?: any` as a trailing parameter and resolves it through a private `db(tenantDb)` helper — the established pattern in `RulesService`/`BusinessService`/`FaqIngestionService`.
- **Imported chunks MUST land as `review_status: PENDING_REVIEW`.** PRD 1's retrieval query filters on `review_status = 'APPROVED'`, so pending content is structurally unreachable by the bot. That filter is the enforcement mechanism for the spec's G3 ("nothing reaches a customer unreviewed") — never work around it in application code.
- **`source_ordinal` is the CSV row number and `source_ref` is the import batch id.** PRD 1's `@@unique([source_ref, source_ordinal])` makes re-importing the same batch id idempotent. Batch ids must be unique per import (`import:<uuid>`), so two separate uploads of the same file create two batches rather than colliding — deliberate for phase 1; true re-ingestion versioning is phase 4 (see Deferred).
- The lint's **currency rule is a hard error, not a warning**: the product catalog is the single source of price truth (PRD 1 §13, PRD 2 §11). A chunk whose answer contains a price figure is rejected at intake and cannot be approved.
- **Unresolved `{{placeholders}}` block approval but not import** (PRD 2 §8): templates legitimately carry them until copy-time resolution, so intake tolerates them while approval refuses them. This is why `lintChunk` takes a `stage` parameter.
- **CSV parsing duplicates ~12 lines from `ProductsService`** (encoding sniffing, BOM strip, delimiter detection). Deliberate: extracting a shared helper would mean refactoring a 570-line service this plan otherwise never touches, and the two parsers have different column semantics and will likely diverge. Reviewers should not flag this as a DRY violation — a comment in the code points at the original.
- **Scope: PRD 2 phase 1 only** (spec §9). Phases 2–4 are out of scope for this plan.
- **Known deviations from the spec, deliberate — call these out, don't silently drop them:**
  - **CSV only, no XLSX.** §4.2 says "CSV or XLSX". `csv-parse` is already a dependency; XLSX would add a new one for a format every spreadsheet exports out of. CSV first is exactly §4.2's own argument ("cheapest to build and the most reliable — build it first"). Add XLSX when a real client can't produce a CSV.
  - **No language check in the lint.** §4.3 wants rioplatense Spanish enforced. Reliable language/register detection is not a regex, and getting it wrong blocks legitimate content. The imperative-phrasing rule catches the specific failure that actually matters (text that reads as instructions to the bot); leave register to the human reviewer.
  - **Imperative detection is a warning, not an error.** It's a keyword heuristic over a fuzzy category. Blocking on it would reject legitimate answers; surfacing it puts the decision in front of the reviewer, which is where §5 wants it.
  - **Deferred to later phases, not built here:** vertical starter packs and the control-plane `FaqTemplate` tables (§8, phase 2); LLM document extraction and `source_span` (§4.3, phase 3); duplicate detection via embedding similarity and re-ingestion versioning (§5, §6, phase 4). **Phase 4's versioning needs a schema change** — PRD 1's `upsertBatch` overwrites a matched row in place, which cannot satisfy both "nothing unreviewed goes live" and "the bot never has a gap" at once; the recommended fix is a new-version row plus a partial unique index scoped to approved rows. Do not attempt it inside this plan.
  - **No frontend.** The review screen (§5, "the review screen is the whole feature") lives in `soylaika.frontend`, a separate repo. This plan builds the API it will consume. Building that screen is a separate track.

---

## File Structure

```
soylaika.backend/src/faq/
  faq-lint.ts                  [CREATE] pure content rules, no DB/network, used at intake AND approval
  faq-lint.spec.ts             [CREATE]
  faq-import.service.ts        [CREATE] CSV → rows → lint → FaqIngestionService.upsertBatch(PENDING_REVIEW)
  faq-import.service.spec.ts   [CREATE]
  faq-review.service.ts        [CREATE] queue listing + APPROVED/ARCHIVED transitions
  faq-review.service.spec.ts   [CREATE]
  faq.controller.ts            [MODIFY] +import, +CSV template download, +review endpoints, +list filters
  faq.controller.spec.ts       [MODIFY] tests for the new handlers
  faq.module.ts                [MODIFY] register the two new services
  faq-ingestion.service.ts     [MODIFY] (Task 5 only) run the lint on the manual-entry path too
```

`faq-lint.ts` is deliberately a plain module, not an injectable service: it has no dependencies, and keeping it pure means Tasks 2, 3 and 5 can all use it without DI wiring, and its tests need no mocks at all.

---

### Task 1: Content lint

**Files:**
- Create: `src/faq/faq-lint.ts`
- Test: `src/faq/faq-lint.spec.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `lintChunk(question: string, answer: string, stage: 'intake' | 'approval'): LintFinding[]`, `hasErrors(findings: LintFinding[]): boolean`, and the exported caps `MAX_QUESTION_CHARS` (300) / `MAX_ANSWER_CHARS_CONTENT` (800). Consumed by Tasks 2, 3 and 5.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-lint.spec.ts`:

```ts
import { lintChunk, hasErrors, MAX_QUESTION_CHARS, MAX_ANSWER_CHARS_CONTENT } from './faq-lint';

describe('lintChunk', () => {
  const ok = ['¿Hacen envios?', 'Si, a todo el pais. El plazo lo confirma un asesor.'] as const;

  it('passes a clean chunk with no findings', () => {
    expect(lintChunk(ok[0], ok[1], 'intake')).toEqual([]);
  });

  it('rejects an empty question or answer', () => {
    expect(lintChunk('  ', 'algo', 'intake').some((f) => f.rule === 'vacio')).toBe(true);
    expect(lintChunk('algo', '   ', 'intake').some((f) => f.rule === 'vacio')).toBe(true);
  });

  it('rejects an over-long question or answer', () => {
    const longQ = 'a'.repeat(MAX_QUESTION_CHARS + 1);
    const longA = 'b'.repeat(MAX_ANSWER_CHARS_CONTENT + 1);
    expect(lintChunk(longQ, ok[1], 'intake').some((f) => f.rule === 'largo')).toBe(true);
    expect(lintChunk(ok[0], longA, 'intake').some((f) => f.rule === 'largo')).toBe(true);
  });

  it.each([
    'Sale $45.000 el rollo.',
    'Cuesta 45000 pesos.',
    'Tenemos 20% de descuento.',
    'Vale u$s 300.',
    'Cuesta US$ 300.',
    'El precio es USD 300.',
    'Descuento del 20% pagando en efectivo.',
    'Te queda 20% menos si llevas dos.',
    'Te sale 20% menos si compras dos rollos.',
    'Te hacemos 20% menos, pagando al contado.',
    'Bonificacion del 15 %.',
    'Descuento: 20%.',
  ])('rejects an answer containing a price figure: %j', (answer) => {
    const findings = lintChunk(ok[0], answer, 'intake');
    expect(findings.some((f) => f.rule === 'precio' && f.severity === 'error')).toBe(true);
  });

  // Numeros que NO son precios. Un falso positivo aca bloquea contenido
  // legitimo con un error duro y sin apelacion, asi que estos casos importan
  // MAS que los de arriba: una FAQ de precio que se escapa la ve despues un
  // humano en la cola de revision, pero una spec de producto rechazada frena
  // el trabajo de quien esta cargando contenido.
  //
  // "N% menos <atributo>" es comparativo generico en español (menos energia,
  // menos agua, menos brillo) y no habla de plata: no debe bloquearse.
  it.each([
    'Son 6 meses desde la compra.',
    'La pared mide 3 x 2,5 metros.',
    'Llamanos al 011 4555-1234.',
    'Es 100% algodon.',
    'Atendemos de 9 a 18.',
    'El rollo cubre 5,3 m2.',
    'La humedad no puede superar el 80% para colocar.',
    'El proceso usa 20% menos energia.',
    'Este modelo pesa 20% menos que el anterior.',
    'Gasta 20% menos agua por lavado.',
    'El rollo nuevo tiene 15% menos brillo.',
    'Sin descuento, cubre 20% mas superficie.',
    'No hay descuento pero rinde 20% mas que el standard.',
    'Este tejido encoge 20% menos, aunque cuesta lo mismo.',
    'El nuevo modelo pesa 20% menos, segun el fabricante.',
    'Este modelo pesa 20% menos.',
    'El vinilo refleja 30% menos luz.',
    'Rinde 20% menos siempre que este seco.',
    // Condicionales de cuidado/uso: "si" NO implica una condicion de pago.
    'Encoge 20% menos si se lava en frio.',
    'Rinde 20% menos si el ambiente esta humedo.',
    'Pesa 20% menos si se usa el modelo compacto.',
    'Sin descuento: el rollo cubre 20% mas.',
  ])('does not flag content that is not a price: %j', (answer) => {
    const findings = lintChunk('¿Cuanto dura la garantia?', answer, 'intake');
    expect(findings.some((f) => f.rule === 'precio')).toBe(false);
  });

  it('warns (not errors) on imperative language aimed at the bot', () => {
    const findings = lintChunk(ok[0], 'Decile al cliente que consulte con un asesor.', 'intake');
    const imperative = findings.find((f) => f.rule === 'imperativo');
    expect(imperative).toBeDefined();
    expect(imperative!.severity).toBe('warning');
  });

  it('tolerates an unresolved placeholder at intake but errors at approval', () => {
    const withPlaceholder = 'Atendemos {{horario}}.';
    const atIntake = lintChunk(ok[0], withPlaceholder, 'intake').find((f) => f.rule === 'placeholder');
    const atApproval = lintChunk(ok[0], withPlaceholder, 'approval').find((f) => f.rule === 'placeholder');
    expect(atIntake!.severity).toBe('warning');
    expect(atApproval!.severity).toBe('error');
  });

  it('hasErrors is true only when a finding has error severity', () => {
    expect(hasErrors([{ rule: 'imperativo', severity: 'warning', message: 'x' }])).toBe(false);
    expect(hasErrors([{ rule: 'precio', severity: 'error', message: 'x' }])).toBe(true);
    expect(hasErrors([])).toBe(false);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-lint.spec.ts
```
Expected: FAIL — `Cannot find module './faq-lint'`.

- [ ] **Step 3: Implement `faq-lint.ts`**

```ts
// Reglas de contenido para los chunks de FAQ. Modulo puro: sin DB, sin red, sin
// DI — se usa en dos momentos distintos (ver `stage`) y desde tres lugares
// (import CSV, alta manual, aprobacion en la cola de revision).
//
// Por que existe (PRD 2 §5, §11): el contenido lo escribe el negocio, y los dos
// modos de falla caros son (a) un precio que se desactualiza y contradice al
// catalogo, y (b) texto imperativo que el modelo lee como instruccion en vez de
// como dato. El resto lo decide un humano en la cola.

export type LintSeverity = 'error' | 'warning';

export interface LintFinding {
  rule: 'vacio' | 'largo' | 'precio' | 'imperativo' | 'placeholder';
  severity: LintSeverity;
  message: string;
}

// Topes de tamaño. PRD 1 §5: "si una respuesta necesita mas de ~800 caracteres,
// es señal de que hay que partirla en dos preguntas". La pregunta se inyecta
// entera en el prompt (MAX_ANSWER_CHARS solo recorta la respuesta), asi que
// tambien tiene tope.
export const MAX_QUESTION_CHARS = 300;
export const MAX_ANSWER_CHARS_CONTENT = 800;

// Precios: el catalogo es la unica fuente de verdad (PRD 1 §13). La regla es un
// error duro, asi que tiene que cubrir como escribe precios un negocio argentino
// de verdad — incluyendo "u$s", que es la forma corriente de escribir dolares —
// sin marcar numeros que no son precios ("6 meses", "100% algodon", un telefono).
const PRECIO_RE = new RegExp(
  [
    // "$45.000", "$ 1.500"
    String.raw`\$\s?\d`,
    // "u$s 300", "U$S300", "us$ 300" — abreviatura local de dolares
    String.raw`(\bu\s?\$\s?s|\bus\s?\$)\s?\d`,
    // "45000 pesos", "300 usd", "300 dolares" — moneda DESPUES del numero
    String.raw`\b\d[\d.,]*\s?(pesos|usd|d[oó]lares)\b`,
    // "USD 300", "dolares 300" — moneda ANTES del numero
    String.raw`\b(usd|d[oó]lares)\s?\d`,
    // "20% de descuento", "20% off", "15% de bonificacion"
    String.raw`\b\d+\s?%\s?(de\s+)?(descuento|off|bonificaci[oó]n)\b`,
    // "20% menos" SOLO si le sigue una condicion de pago ("si llevas dos",
    // "pagando en efectivo"). Deliberadamente NO alcanza con que cierre la
    // frase: "pesa 20% menos." es tan plausible como "sale 20% menos." y nada
    // en la oracion los distingue, asi que un "20% menos" suelto se deja pasar
    // a la cola de revision en vez de bloquearlo. "N% menos <atributo>"
    // (energia, agua, brillo) es comparativo comun y no habla de plata.
    // Ojo con el `si`: tiene que ser "si COMPRAS/LLEVAS", no el "si"
    // condicional a secas. Un textil escribe "encoge 20% menos si se lava en
    // frio" o "rinde 20% menos si el ambiente esta humedo" — instrucciones de
    // cuidado, no promociones — y un `si` suelto las bloquearia.
    String.raw`\b\d+\s?%\s?menos\b(?=\s*,?\s*(si\s+(te\s+)?(llev[aá]s?|compr[aá]s?|pag[aá]s?|abon[aá]s?)|pagando|abonando|al\s+contado|en\s+efectivo)\b)`,
    // "descuento del 20%", "Descuento: 20%" — la palabra ANTES del porcentaje,
    // pero solo con los conectores propios de esa construccion (: / de / del /
    // de un). Una ventana generica uniria clausulas sueltas: "sin descuento,
    // cubre 20% mas superficie" no habla de un precio.
    String.raw`\b(descuento|bonificaci[oó]n)\s*:?\s*(de\s+un\s+|del\s+|de\s+)?\d+\s?%`,
  ].join('|'),
  'i',
);

// Texto imperativo dirigido al bot ("decile al cliente que...", "no ofrezcas...").
// Heuristica por verbo inicial: solo el arranque de una oracion, para no marcar
// una respuesta que legitimamente usa el verbo mas adelante.
const IMPERATIVO_RE = /(^|[.!?]\s+)(decile|deciles|aclarale|explicale|respondele|record[aá]le|ofrecele|derivalo|derivala|pedile|mandale|contale|no\s+(ofrezcas|menciones|digas|prometas))\b/i;

const PLACEHOLDER_RE = /\{\{\s*[\w.\-]+\s*\}\}/;

export function lintChunk(question: string, answer: string, stage: 'intake' | 'approval'): LintFinding[] {
  const findings: LintFinding[] = [];
  const q = (question ?? '').trim();
  const a = (answer ?? '').trim();

  if (!q || !a) {
    findings.push({ rule: 'vacio', severity: 'error', message: 'La pregunta y la respuesta no pueden estar vacias.' });
    return findings; // sin contenido no tiene sentido seguir evaluando
  }

  if (q.length > MAX_QUESTION_CHARS) {
    findings.push({ rule: 'largo', severity: 'error', message: `La pregunta supera los ${MAX_QUESTION_CHARS} caracteres (${q.length}).` });
  }
  if (a.length > MAX_ANSWER_CHARS_CONTENT) {
    findings.push({ rule: 'largo', severity: 'error', message: `La respuesta supera los ${MAX_ANSWER_CHARS_CONTENT} caracteres (${a.length}). Si necesita mas, conviene partirla en dos preguntas.` });
  }
  if (PRECIO_RE.test(a)) {
    findings.push({ rule: 'precio', severity: 'error', message: 'La respuesta menciona un precio o descuento. Los precios salen del catalogo, no de las FAQ: se desactualizan y contradicen al bot.' });
  }
  if (IMPERATIVO_RE.test(a)) {
    findings.push({ rule: 'imperativo', severity: 'warning', message: 'La respuesta parece darle instrucciones al bot ("decile que...", "no ofrezcas..."). El bloque recuperado es informacion, no instrucciones: conviene redactarla como el dato en si.' });
  }
  if (PLACEHOLDER_RE.test(a) || PLACEHOLDER_RE.test(q)) {
    findings.push({
      rule: 'placeholder',
      // En intake un placeholder es esperable (plantillas sin resolver); al
      // aprobar no, porque de ahi sale directo al cliente (PRD 2 §8).
      severity: stage === 'approval' ? 'error' : 'warning',
      message: 'Hay un placeholder {{...}} sin resolver.',
    });
  }

  return findings;
}

export function hasErrors(findings: LintFinding[]): boolean {
  return findings.some((f) => f.severity === 'error');
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-lint.spec.ts
```
Expected: PASS (all cases, including the 3 `it.each` price variants).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-lint.ts src/faq/faq-lint.spec.ts
git commit -m "feat(faq): add content lint (precios, largo, imperativo, placeholders)"
```

---

### Task 2: CSV import service

**Files:**
- Create: `src/faq/faq-import.service.ts`
- Test: `src/faq/faq-import.service.spec.ts`

**Interfaces:**
- Consumes: `lintChunk`/`hasErrors` (Task 1); `FaqIngestionService.upsertBatch(chunks, opts, tenantDb, tenant)` from PRD 1 — note its real signature takes the tenant's Prisma client, not a `tenantId` (PRD 1 §15).
- Produces: `FaqImportService.importCsv(buffer: Buffer, tenantDb?: any, tenant?: any): Promise<ImportResult>` and the static `CSV_TEMPLATE` string. Consumed by Task 4's controller.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-import.service.spec.ts`:

```ts
import { FaqImportService, CSV_TEMPLATE, MAX_IMPORT_ROWS } from './faq-import.service';

function makeService() {
  const ingestion = { upsertBatch: jest.fn().mockResolvedValue({ created: 0, updated: 0, unchanged: 0, chunks: [] }) } as any;
  return { service: new FaqImportService(ingestion), ingestion };
}

const csv = (body: string) => Buffer.from(body, 'utf8');

describe('FaqImportService.importCsv', () => {
  it('imports valid rows as PENDING_REVIEW with row-numbered ordinals', async () => {
    const { service, ingestion } = makeService();
    const result = await service.importCsv(
      csv('pregunta,respuesta\n¿Hacen envios?,Si a todo el pais.\n¿Cuanto dura la garantia?,6 meses con ticket.\n'),
      {},
    );

    expect(result.imported).toBe(2);
    expect(result.rejected).toHaveLength(0);
    const [chunks, opts] = ingestion.upsertBatch.mock.calls[0];
    expect(opts.sourceType).toBe('IMPORT');
    expect(opts.reviewStatus).toBe('PENDING_REVIEW');
    expect(opts.sourceRef).toBe(result.batchId);
    expect(chunks).toEqual([
      { question: '¿Hacen envios?', answer: 'Si a todo el pais.', agents: [], tags: [], sourceOrdinal: 1 },
      { question: '¿Cuanto dura la garantia?', answer: '6 meses con ticket.', agents: [], tags: [], sourceOrdinal: 2 },
    ]);
  });

  it('rejects rows that fail the lint and imports the rest', async () => {
    const { service, ingestion } = makeService();
    const result = await service.importCsv(
      csv('pregunta,respuesta\n¿Precio?,Sale $45.000 el rollo.\n¿Hacen envios?,Si a todo el pais.\n'),
      {},
    );

    expect(result.imported).toBe(1);
    expect(result.rejected).toHaveLength(1);
    expect(result.rejected[0].row).toBe(1);
    expect(result.rejected[0].findings.some((f: any) => f.rule === 'precio')).toBe(true);
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks).toHaveLength(1);
    expect(chunks[0].question).toBe('¿Hacen envios?');
  });

  it('parses agentes and tags as comma-separated lists', async () => {
    const { service, ingestion } = makeService();
    await service.importCsv(
      csv('pregunta,respuesta,agentes,tags\n¿Garantia?,6 meses.,"soporte, devolucion","postventa, garantia"\n'),
      {},
    );
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].agents).toEqual(['soporte', 'devolucion']);
    expect(chunks[0].tags).toEqual(['postventa', 'garantia']);
  });

  it('accepts English headers as an alias', async () => {
    const { service, ingestion } = makeService();
    await service.importCsv(csv('question,answer\n¿Hacen envios?,Si.\n'), {});
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].question).toBe('¿Hacen envios?');
  });

  it('throws when the file has no recognisable question/answer columns', async () => {
    const { service } = makeService();
    await expect(service.importCsv(csv('foo,bar\n1,2\n'), {})).rejects.toThrow(/pregunta.*respuesta/i);
  });

  it('refuses a file with more rows than the cap, before writing anything', async () => {
    const { service, ingestion } = makeService();
    const rows = Array.from({ length: MAX_IMPORT_ROWS + 1 }, (_, i) => `¿Pregunta ${i}?,Respuesta ${i}.`).join('\n');
    await expect(service.importCsv(csv(`pregunta,respuesta\n${rows}\n`), {})).rejects.toThrow(
      new RegExp(String(MAX_IMPORT_ROWS)),
    );
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });

  it('does not call upsertBatch when every row is rejected', async () => {
    const { service, ingestion } = makeService();
    const result = await service.importCsv(csv('pregunta,respuesta\n¿Precio?,Sale $45.000.\n'), {});
    expect(result.imported).toBe(0);
    expect(result.rejected).toHaveLength(1);
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });

  // La plantilla se PARSEA, no se cuenta por lineas: es el archivo que el
  // cliente descarga como ejemplo, asi que si tiene una coma sin comillas
  // dentro de un campo, el ejemplo mismo enseña un formato roto. Contar lineas
  // no detecta eso.
  it('CSV_TEMPLATE parses back into well-formed rows', async () => {
    const { service, ingestion } = makeService();
    const result = await service.importCsv(csv(CSV_TEMPLATE), {});

    expect(result.rejected).toHaveLength(0);
    expect(result.imported).toBe(3);
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].question).toBe('¿Hacen envios a todo el pais?');
    expect(chunks[0].answer).toBe('Si, enviamos a todo el pais. El plazo te lo confirma un asesor segun la zona.');
    expect(chunks[0].tags).toEqual(['envios']);
    expect(chunks[1].agents).toEqual(['soporte', 'devolucion']);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-import.service.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-import.service.ts`**

```ts
import { BadRequestException, Injectable, InternalServerErrorException, Logger } from '@nestjs/common';
import { parse } from 'csv-parse/sync';
import { randomUUID } from 'crypto';
import { FaqIngestionService } from './faq-ingestion.service';
import { lintChunk, hasErrors, type LintFinding } from './faq-lint';

export interface RejectedRow {
  row: number;            // numero de fila del CSV (1 = primera fila de datos)
  question: string;
  answer: string;
  findings: LintFinding[];
}

export interface ImportResult {
  batchId: string;
  total: number;
  imported: number;
  rejected: RejectedRow[];
  warnings: { row: number; findings: LintFinding[] }[];
}

// Tope de filas por archivo. Generoso para una migracion real (un set de FAQ
// util son 40-80 preguntas) pero acotado, para que una carga enorme no deje una
// cola de revision que nadie va a mirar de verdad.
export const MAX_IMPORT_ROWS = 500;

// Archivo de ejemplo que se descarga desde el panel (PRD 2 §4.2): cabeceras
// correctas + tres filas de muestra, para que el cliente no adivine el formato.
// OJO: cualquier campo con coma adentro va entre comillas — este archivo es el
// que el cliente copia, asi que un ejemplo mal formado enseña el error.
export const CSV_TEMPLATE = `pregunta,respuesta,agentes,tags
¿Hacen envios a todo el pais?,"Si, enviamos a todo el pais. El plazo te lo confirma un asesor segun la zona.",,envios
¿Cuanto dura la garantia?,6 meses desde la fecha de compra presentando el ticket.,"soporte, devolucion",postventa
¿Puedo cambiar un producto abierto?,Solo si tiene falla de fabrica. Sin uso y con el embalaje original se cambia sin problema.,devolucion,cambios
`;

@Injectable()
export class FaqImportService {
  private readonly logger = new Logger(FaqImportService.name);

  constructor(private readonly ingestion: FaqIngestionService) {}

  async importCsv(buffer: Buffer, tenantDb?: any, tenant?: any): Promise<ImportResult> {
    const records = this.parseCsv(buffer);
    const batchId = `import:${randomUUID()}`;

    if (!records.length) throw new BadRequestException('El archivo no tiene filas de datos.');
    // Tope de filas: sin esto una carga grande inunda la cola de revision — y
    // una cola que nadie puede revisar de verdad es peor que no tenerla
    // (PRD 2 §5, "review fatigue is the main threat to quality"). Ademas evita
    // mandar miles de textos en una sola llamada de embeddings.
    if (records.length > MAX_IMPORT_ROWS) {
      throw new BadRequestException(
        `El archivo tiene ${records.length} filas y el maximo es ${MAX_IMPORT_ROWS}. Partilo en varios archivos.`,
      );
    }

    const columns = Object.keys(records[0]);
    const qKey = this.findColumn(columns, ['pregunta', 'question']);
    const aKey = this.findColumn(columns, ['respuesta', 'answer']);
    if (!qKey || !aKey) {
      throw new BadRequestException(
        `El archivo necesita columnas "pregunta" y "respuesta" (o "question"/"answer"). Encontradas: ${columns.join(', ')}`,
      );
    }
    const agentsKey = this.findColumn(columns, ['agentes', 'agents']);
    const tagsKey = this.findColumn(columns, ['tags', 'etiquetas']);

    const chunks: any[] = [];
    const rejected: RejectedRow[] = [];
    const warnings: { row: number; findings: LintFinding[] }[] = [];

    records.forEach((record, i) => {
      const row = i + 1;
      const question = String(record[qKey] ?? '').trim();
      const answer = String(record[aKey] ?? '').trim();

      const findings = lintChunk(question, answer, 'intake');
      if (hasErrors(findings)) {
        rejected.push({ row, question, answer, findings });
        return;
      }
      if (findings.length) warnings.push({ row, findings });

      chunks.push({
        question,
        answer,
        agents: this.splitList(agentsKey ? record[agentsKey] : ''),
        tags: this.splitList(tagsKey ? record[tagsKey] : ''),
        sourceOrdinal: row,
      });
    });

    if (chunks.length) {
      try {
        // PENDING_REVIEW es lo que mantiene el contenido fuera del alcance del bot:
        // la query de retrieval filtra por review_status = 'APPROVED' (PRD 1 §7.3).
        await this.ingestion.upsertBatch(
          chunks,
          { sourceType: 'IMPORT', sourceRef: batchId, reviewStatus: 'PENDING_REVIEW' },
          tenantDb,
          tenant,
        );
      } catch (err: any) {
        // upsertBatch escribe fila por fila sin transaccion: si falla en el medio,
        // quedan chunks ya insertados con este source_ref. Sin el batchId en el
        // error no hay forma de encontrarlos, y reintentar genera un batchId nuevo
        // (duplicando lo que ya entro). Con el batchId, limpiar es una query.
        this.logger.error(`Import ${batchId} fallo despues de escribir parcialmente: ${err?.message ?? err}`);
        throw new InternalServerErrorException(
          `La importacion fallo a mitad de camino. Puede haber filas cargadas bajo el lote "${batchId}": ` +
          `revisalas o borralas por ese identificador antes de reintentar. Detalle: ${err?.message ?? err}`,
        );
      }
    }

    this.logger.log(
      `Import ${batchId}: ${chunks.length}/${records.length} filas a revision, ${rejected.length} rechazadas, ${warnings.length} con advertencias`,
    );

    return { batchId, total: records.length, imported: chunks.length, rejected, warnings };
  }

  private findColumn(columns: string[], candidates: string[]): string | null {
    return columns.find((c) => candidates.includes(c.toLowerCase().trim())) ?? null;
  }

  private splitList(raw: any): string[] {
    return String(raw ?? '')
      .split(',')
      .map((s) => s.trim())
      .filter(Boolean);
  }

  // Mismo tratamiento que ProductsService.parseCsv: los exports de Excel en
  // español vienen con `;` y a veces en Latin-1, y con BOM. Se duplica a
  // proposito en vez de compartir helper: las columnas y la validacion son
  // distintas y es probable que los dos parsers diverjan.
  private parseCsv(buffer: Buffer): any[] {
    let text = buffer.toString('utf8');
    if (text.includes('\ufffd')) {
      text = buffer.toString('latin1');
      this.logger.log('CSV decodificado como Latin-1 (no era UTF-8 valido)');
    }
    if (text.charCodeAt(0) === 0xfeff) text = text.slice(1);

    const firstLine = text.slice(0, (text.indexOf('\n') + 1 || text.length + 1) - 1);
    const delimiter = this.detectDelimiter(firstLine);

    return parse(text, {
      columns: true,
      skip_empty_lines: true,
      trim: true,
      delimiter,
      relax_quotes: true,
      relax_column_count: true,
    });
  }

  private detectDelimiter(headerLine: string): string {
    const candidates = [';', ',', '\t', '|'];
    let best = ',';
    let bestCount = -1;
    for (const d of candidates) {
      const count = headerLine.split(d).length - 1;
      if (count > bestCount) { bestCount = count; best = d; }
    }
    return best;
  }
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-import.service.spec.ts
```
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-import.service.ts src/faq/faq-import.service.spec.ts
git commit -m "feat(faq): add CSV import landing chunks in the review queue"
```

---

### Task 3: Review queue service

**Files:**
- Create: `src/faq/faq-review.service.ts`
- Test: `src/faq/faq-review.service.spec.ts`

**Interfaces:**
- Consumes: `lintChunk`/`hasErrors` (Task 1); `FaqIngestionService.updateOne(id, patch, tenantDb, tenant)` from PRD 1 (re-embeds when content changed).
- Produces: `FaqReviewService.listQueue(filters, tenantDb)`, `.counts(tenantDb)`, `.approve(id, patch, tenantDb, tenant)`, `.reject(id, tenantDb)`. Consumed by Task 4's controller.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-review.service.spec.ts`:

```ts
import { BadRequestException, NotFoundException } from '@nestjs/common';
import { FaqReviewService } from './faq-review.service';

function makeService(chunk: any = null) {
  const ingestion = { updateOne: jest.fn().mockResolvedValue(chunk) } as any;
  const prisma = {} as any;
  const tenantDb = {
    faqChunk: {
      findUnique: jest.fn().mockResolvedValue(chunk),
      findMany: jest.fn().mockResolvedValue([]),
      update: jest.fn().mockImplementation(({ data }) => Promise.resolve({ ...chunk, ...data })),
      groupBy: jest.fn().mockResolvedValue([
        { review_status: 'PENDING_REVIEW', _count: { _all: 3 } },
        { review_status: 'APPROVED', _count: { _all: 7 } },
      ]),
    },
  };
  return { service: new FaqReviewService(prisma, ingestion), ingestion, tenantDb };
}

const pending = {
  id: 'faq-1',
  question: '¿Hacen envios?',
  answer: 'Si, a todo el pais.',
  review_status: 'PENDING_REVIEW',
  active: true,
};

describe('FaqReviewService.listQueue', () => {
  it('defaults to PENDING_REVIEW and orders oldest first', async () => {
    const { service, tenantDb } = makeService();
    await service.listQueue({}, tenantDb);
    const arg = tenantDb.faqChunk.findMany.mock.calls[0][0];
    expect(arg.where).toEqual({ active: true, review_status: 'PENDING_REVIEW' });
    expect(arg.orderBy).toEqual({ created_at: 'asc' });
  });

  it('filters by review_status and source_type when given', async () => {
    const { service, tenantDb } = makeService();
    await service.listQueue({ reviewStatus: 'APPROVED', sourceType: 'IMPORT' }, tenantDb);
    const arg = tenantDb.faqChunk.findMany.mock.calls[0][0];
    expect(arg.where).toEqual({ active: true, review_status: 'APPROVED', source_type: 'IMPORT' });
  });
});

describe('FaqReviewService.approve', () => {
  it('approves a pending chunk without edits', async () => {
    const { service, ingestion, tenantDb } = makeService(pending);
    const result = await service.approve('faq-1', {}, tenantDb);
    expect(ingestion.updateOne).not.toHaveBeenCalled();
    expect(tenantDb.faqChunk.update).toHaveBeenCalledWith({
      where: { id: 'faq-1' },
      data: { review_status: 'APPROVED', reviewed_at: expect.any(Date) },
    });
    expect(result.review_status).toBe('APPROVED');
  });

  it('edits before approving when a patch is supplied', async () => {
    const edited = { ...pending, answer: 'Si, a todo el pais sin cargo.' };
    const { service, ingestion, tenantDb } = makeService(pending);
    ingestion.updateOne.mockResolvedValue(edited);

    await service.approve('faq-1', { answer: 'Si, a todo el pais sin cargo.' }, tenantDb, { slug: 't' });

    expect(ingestion.updateOne).toHaveBeenCalledWith(
      'faq-1',
      { answer: 'Si, a todo el pais sin cargo.' },
      tenantDb,
      { slug: 't' },
    );
    expect(tenantDb.faqChunk.update).toHaveBeenCalled();
  });

  it('refuses to approve content that fails the approval lint', async () => {
    const withPlaceholder = { ...pending, answer: 'Atendemos {{horario}}.' };
    const { service, tenantDb } = makeService(withPlaceholder);
    await expect(service.approve('faq-1', {}, tenantDb)).rejects.toBeInstanceOf(BadRequestException);
    expect(tenantDb.faqChunk.update).not.toHaveBeenCalled();
  });

  it('throws NotFound for an unknown id', async () => {
    const { service, tenantDb } = makeService(null);
    await expect(service.approve('nope', {}, tenantDb)).rejects.toBeInstanceOf(NotFoundException);
  });

  // Un chunk archivado ya fue rechazado. Aprobarlo lo dejaria APPROVED pero
  // active:false — incoherente, invisible en la cola, y con la decision previa
  // pisada sin rastro.
  it('refuses to approve an archived chunk', async () => {
    const archived = { ...pending, review_status: 'ARCHIVED', active: false };
    const { service, tenantDb } = makeService(archived);
    await expect(service.approve('faq-1', {}, tenantDb)).rejects.toBeInstanceOf(BadRequestException);
    expect(tenantDb.faqChunk.update).not.toHaveBeenCalled();
  });

  // El lint de aprobacion tiene que correr sobre el contenido EDITADO. Este
  // caso lo distingue de verdad: el original pasa el lint y la edicion no, asi
  // que si se linteara el pre-edicion la aprobacion saldria bien y entraria un
  // precio a produccion.
  it('lints the post-edit content, not the original', async () => {
    const { service, ingestion, tenantDb } = makeService(pending);
    ingestion.updateOne.mockResolvedValue({ ...pending, answer: 'Ahora sale $45.000.' });

    await expect(
      service.approve('faq-1', { answer: 'Ahora sale $45.000.' }, tenantDb),
    ).rejects.toBeInstanceOf(BadRequestException);
    expect(tenantDb.faqChunk.update).not.toHaveBeenCalled();
  });

  it('records who approved it', async () => {
    const { service, tenantDb } = makeService(pending);
    await service.approve('faq-1', {}, tenantDb, undefined, 'user-7');
    expect(tenantDb.faqChunk.update).toHaveBeenCalledWith({
      where: { id: 'faq-1' },
      data: { review_status: 'APPROVED', reviewed_at: expect.any(Date), reviewed_by: 'user-7' },
    });
  });
});

describe('FaqReviewService.reject', () => {
  it('archives instead of deleting, recording who rejected it', async () => {
    const { service, tenantDb } = makeService(pending);
    await service.reject('faq-1', tenantDb, 'user-7');
    expect(tenantDb.faqChunk.update).toHaveBeenCalledWith({
      where: { id: 'faq-1' },
      data: { review_status: 'ARCHIVED', active: false, reviewed_at: expect.any(Date), reviewed_by: 'user-7' },
    });
  });
});

describe('FaqReviewService.counts', () => {
  it('returns a status → count map', async () => {
    const { service, tenantDb } = makeService();
    expect(await service.counts(tenantDb)).toEqual({ PENDING_REVIEW: 3, APPROVED: 7 });
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-review.service.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-review.service.ts`**

```ts
import { BadRequestException, Injectable, Logger, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { FaqIngestionService } from './faq-ingestion.service';
import { lintChunk, hasErrors } from './faq-lint';

export interface QueueFilters {
  reviewStatus?: string;
  sourceType?: string;
  sourceRef?: string;
}

export interface ApprovePatch {
  question?: string;
  answer?: string;
  agents?: string[];
  tags?: string[];
}

// Campos que ve la pantalla de revision. `content_hash` y el resto de la
// procedencia interna no se exponen.
const QUEUE_SELECT = {
  id: true, question: true, answer: true, agents: true, tags: true, active: true,
  review_status: true, source_type: true, source_ref: true, source_ordinal: true,
  source_span: true, embedded_at: true, created_at: true, updated_at: true,
} as const;

@Injectable()
export class FaqReviewService {
  private readonly logger = new Logger(FaqReviewService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly ingestion: FaqIngestionService,
  ) {}

  private db(tenantDb?: any) { return tenantDb ?? this.prisma; }

  // Cola de revision. Por defecto lo pendiente, lo mas viejo primero: es una
  // cola, no un listado.
  listQueue(filters: QueueFilters, tenantDb?: any) {
    const where: any = { active: true, review_status: filters.reviewStatus ?? 'PENDING_REVIEW' };
    if (filters.sourceType) where.source_type = filters.sourceType;
    if (filters.sourceRef) where.source_ref = filters.sourceRef;

    return this.db(tenantDb).faqChunk.findMany({
      where,
      orderBy: { created_at: 'asc' },
      select: QUEUE_SELECT,
    });
  }

  // Contadores por estado, para el badge de "pendientes" del panel.
  // El catch tolera al tenant que todavia no corrio la migracion de FaqChunk
  // (misma convencion que RulesService.listActive), pero LOGUEA: si no, un
  // error real de base se ve igual que "no hay nada para revisar", y el
  // operador lee un cero tranquilizador cuando en realidad esta roto.
  async counts(tenantDb?: any): Promise<Record<string, number>> {
    const rows = await this.db(tenantDb).faqChunk
      .groupBy({ by: ['review_status'], where: { active: true }, _count: { _all: true } })
      .catch((err: any) => {
        this.logger.warn(`No se pudieron contar los chunks de FAQ (¿tenant sin migrar?): ${err?.message ?? err}`);
        return [];
      });
    const out: Record<string, number> = {};
    for (const r of rows) out[r.review_status] = r._count._all;
    return out;
  }

  // Aprobar, opcionalmente editando antes (el "edit and approve" de la pantalla).
  // El lint corre en modo 'approval': ahi un placeholder sin resolver SI bloquea,
  // porque lo que se aprueba sale derecho al cliente.
  async approve(id: string, patch: ApprovePatch, tenantDb?: any, tenant?: any, reviewerId?: string) {
    const db = this.db(tenantDb);
    let chunk = await db.faqChunk.findUnique({ where: { id } });
    if (!chunk) throw new NotFoundException('FAQ no encontrada');

    // Un chunk archivado ya fue rechazado por alguien. Aprobarlo desde aca lo
    // dejaria en un estado incoherente (APPROVED pero active:false), invisible
    // en la cola y con la decision de rechazo pisada sin rastro. Si de verdad
    // hay que recuperarlo, es una accion explicita, no un approve mas.
    if (chunk.review_status === 'ARCHIVED') {
      throw new BadRequestException(
        'Esta FAQ fue archivada. Para volver a usarla hay que reactivarla explicitamente, no aprobarla.',
      );
    }

    const hasEdits = Object.values(patch ?? {}).some((v) => v !== undefined);
    if (hasEdits) {
      // updateOne re-embebe solo si cambio el contenido (compara content_hash).
      chunk = await this.ingestion.updateOne(id, patch, tenantDb, tenant);
      if (!chunk) throw new NotFoundException('FAQ no encontrada');
    }

    const findings = lintChunk(chunk.question, chunk.answer, 'approval');
    if (hasErrors(findings)) {
      throw new BadRequestException({
        message: 'No se puede aprobar: el contenido no pasa las reglas.',
        findings,
      });
    }

    const updated = await db.faqChunk.update({
      where: { id },
      data: { review_status: 'APPROVED', reviewed_at: new Date(), reviewed_by: reviewerId ?? null },
    });
    this.logger.log(`FAQ ${id} aprobada por ${reviewerId ?? 'desconocido'}${hasEdits ? ' (con edicion)' : ''}`);
    return updated;
  }

  // Rechazar = archivar. Nunca se borra: si un cliente discute que le dijo el
  // bot, la unica forma de responder es que el contenido siga existiendo.
  async reject(id: string, tenantDb?: any, reviewerId?: string) {
    const db = this.db(tenantDb);
    const chunk = await db.faqChunk.findUnique({ where: { id } });
    if (!chunk) throw new NotFoundException('FAQ no encontrada');

    const updated = await db.faqChunk.update({
      where: { id },
      data: { review_status: 'ARCHIVED', active: false, reviewed_at: new Date(), reviewed_by: reviewerId ?? null },
    });
    this.logger.log(`FAQ ${id} rechazada (archivada) por ${reviewerId ?? 'desconocido'}`);
    return updated;
  }
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-review.service.spec.ts
```
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-review.service.ts src/faq/faq-review.service.spec.ts
git commit -m "feat(faq): add review queue service (approve/reject/edit-and-approve)"
```

---

### Task 4: HTTP surface

**Files:**
- Modify: `src/faq/faq.controller.ts`
- Modify: `src/faq/faq.module.ts`
- Modify: `src/faq/faq.controller.spec.ts`

**Interfaces:**
- Consumes: `FaqImportService` (Task 2), `FaqReviewService` (Task 3).
- Produces: `POST /api/faq/import`, `GET /api/faq/import/template`, `GET /api/faq/review`, `GET /api/faq/review/counts`, `POST /api/faq/:id/approve`, `POST /api/faq/:id/reject`, plus `review_status`/`source_type` filters on the existing `GET /api/faq`.

- [ ] **Step 1: Widen the three existing constructor calls, then write the failing tests**

The constructor grows from 3 params to 5. The three pre-existing tests call it with 3 arguments, which still *runs* fine (the extra params are just `undefined`, and those tests don't touch them) but does **not type-check** — `tsc --noEmit` correctly rejects a 3-arg call to a 5-param constructor. `ts-jest` doesn't full-type-check, so `npm test` alone would hide this.

Resist the temptation to make the new params optional (`?`) or give them defaults to keep the old calls valid: Nest always injects them, so typing them as possibly-undefined would be a lie in production code told for the benefit of a test double. Update the three call sites instead — in `src/faq/faq.controller.spec.ts`, at each of the three `new FaqController(...)` lines, append two more `{} as any` arguments:

```ts
const controller = new FaqController(ingestion, {} as any, {} as any, {} as any, {} as any);
```
```ts
const controller = new FaqController(ingestion, {} as any, {} as any, {} as any, {} as any);
```
```ts
const controller = new FaqController({} as any, retrieval, {} as any, {} as any, {} as any);
```

The assertions in those three tests stay exactly as they are — only the constructor calls change.

Then append the new tests:

```ts
describe('FaqController — import and review', () => {
  function makeController() {
    const ingestion = { upsertBatch: jest.fn(), updateOne: jest.fn(), remove: jest.fn() } as any;
    const retrieval = { retrieve: jest.fn() } as any;
    const importSvc = { importCsv: jest.fn().mockResolvedValue({ batchId: 'import:1', total: 2, imported: 2, rejected: [], warnings: [] }) } as any;
    const review = {
      listQueue: jest.fn().mockResolvedValue([]),
      counts: jest.fn().mockResolvedValue({ PENDING_REVIEW: 2 }),
      approve: jest.fn().mockResolvedValue({ id: 'faq-1', review_status: 'APPROVED' }),
      reject: jest.fn().mockResolvedValue({ id: 'faq-1', review_status: 'ARCHIVED' }),
    } as any;
    const prisma = {} as any;
    const controller = new (require('./faq.controller').FaqController)(ingestion, retrieval, prisma, importSvc, review);
    return { controller, importSvc, review };
  }

  const req = { tenantDb: {}, tenant: { slug: 'itt' }, user: { id: 'user-7' } } as any;

  it('import() passes the uploaded buffer through to the import service', async () => {
    const { controller, importSvc } = makeController();
    const file = { buffer: Buffer.from('pregunta,respuesta\na,b\n') } as any;
    const result = await controller.import(file, req);
    expect(importSvc.importCsv).toHaveBeenCalledWith(file.buffer, req.tenantDb, req.tenant);
    expect(result.batchId).toBe('import:1');
  });

  it('import() rejects a missing file', async () => {
    const { controller, importSvc } = makeController();
    await expect(controller.import(undefined as any, req)).rejects.toThrow();
    expect(importSvc.importCsv).not.toHaveBeenCalled();
  });

  it('template() returns the CSV with a download header', () => {
    const { controller } = makeController();
    const res = { setHeader: jest.fn(), send: jest.fn() } as any;
    controller.template(res);
    expect(res.setHeader).toHaveBeenCalledWith('Content-Type', 'text/csv; charset=utf-8');
    expect(res.setHeader).toHaveBeenCalledWith(
      'Content-Disposition',
      'attachment; filename="plantilla-faq.csv"',
    );
    expect(res.send).toHaveBeenCalledWith(expect.stringContaining('pregunta,respuesta'));
  });

  it('reviewQueue() forwards filters', async () => {
    const { controller, review } = makeController();
    await controller.reviewQueue('APPROVED', 'IMPORT', 'import:1', req);
    expect(review.listQueue).toHaveBeenCalledWith(
      { reviewStatus: 'APPROVED', sourceType: 'IMPORT', sourceRef: 'import:1' },
      req.tenantDb,
    );
  });

  it('approve() forwards the optional edit patch and the reviewer id', async () => {
    const { controller, review } = makeController();
    await controller.approve('faq-1', { answer: 'nueva' } as any, req);
    expect(review.approve).toHaveBeenCalledWith('faq-1', { answer: 'nueva' }, req.tenantDb, req.tenant, 'user-7');
  });

  it('reject() delegates to the review service with the reviewer id', async () => {
    const { controller, review } = makeController();
    await controller.reject('faq-1', req);
    expect(review.reject).toHaveBeenCalledWith('faq-1', req.tenantDb, 'user-7');
  });

  // Un valor invalido tiene que ser 400 con el detalle, no un 500 opaco de
  // Prisma: el que consume esta API (la pantalla de revision) va a mandar un
  // typo alguna vez y necesita saber cual fue.
  it('rejects an unknown review_status with 400 instead of letting Prisma throw', async () => {
    const { controller, review } = makeController();
    expect(() => controller.reviewQueue('BOGUS', undefined, undefined, req))
      .toThrow(BadRequestException);
    expect(review.listQueue).not.toHaveBeenCalled();
  });

  it('accepts a valid review_status', async () => {
    const { controller, review } = makeController();
    await controller.reviewQueue('APPROVED', undefined, undefined, req);
    expect(review.listQueue).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq.controller.spec.ts
```
Expected: FAIL — the constructor now takes five arguments; the new handlers don't exist.

- [ ] **Step 3: Extend the controller**

In `src/faq/faq.controller.ts`, update the imports and constructor, and add the new handlers. The existing `list`/`create`/`update`/`remove`/`test` handlers stay as they are, except `list` gains two filters:

```ts
import {
  BadRequestException, Body, Controller, Delete, Get, Param, Patch, Post, Query, Req, Res,
  UploadedFile, UseGuards, UseInterceptors,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import type { Response } from 'express';
import { FaqIngestionService } from './faq-ingestion.service';
import { FaqRetrievalService } from './faq-retrieval.service';
import { FaqImportService, CSV_TEMPLATE } from './faq-import.service';
import { FaqReviewService } from './faq-review.service';
import { PrismaService } from '../prisma/prisma.service';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { RolesGuard } from '../auth/roles.guard';
import { Roles } from '../auth/roles.decorator';
import { ReviewStatus, SourceType } from '@prisma/client';
```

And a module-level validator above the class. The filters land in a Prisma `where` against enum columns, so an unknown value makes Prisma throw a `PrismaClientValidationError` — and with no global Prisma exception filter in this app, that reaches the caller as a bare 500. A typo in a query string should be a 400 that names the problem:

```ts
// Los valores validos salen de los enums generados por Prisma, no de una lista
// escrita a mano: asi no se desincronizan del schema si mañana se agrega un estado.
function assertEnumParam(value: string | undefined, allowed: string[], field: string) {
  if (value !== undefined && !allowed.includes(value)) {
    throw new BadRequestException(
      `${field} invalido: "${value}". Valores validos: ${allowed.join(', ')}.`,
    );
  }
}
```

Constructor — all five params plainly required, no defaults and no `?`. Nest injects every one of them, so anything softer would misstate the contract (see Step 1 for why the existing tests are updated rather than the constructor loosened):

```ts
  constructor(
    private readonly ingestion: FaqIngestionService,
    private readonly retrieval: FaqRetrievalService,
    private readonly prisma: PrismaService,
    private readonly importService: FaqImportService,
    private readonly review: FaqReviewService,
  ) {}
```

Replace the existing `list()` with the filtered version (PRD 1 §9 asked for these filters and PRD 1 §15 recorded them as missing):

```ts
  @Get()
  list(
    @Query('active') active: string | undefined,
    @Query('agent') agent: string | undefined,
    @Query('review_status') reviewStatus: string | undefined,
    @Query('source_type') sourceType: string | undefined,
    @Req() req: any,
  ) {
    assertEnumParam(reviewStatus, Object.values(ReviewStatus), 'review_status');
    assertEnumParam(sourceType, Object.values(SourceType), 'source_type');

    const where: any = {};
    if (active !== undefined) where.active = active === 'true';
    if (agent) where.agents = { has: agent };
    if (reviewStatus) where.review_status = reviewStatus;
    if (sourceType) where.source_type = sourceType;
    return this.db(req).faqChunk.findMany({
      where,
      orderBy: { created_at: 'desc' },
      select: {
        id: true, question: true, answer: true, agents: true, tags: true, active: true,
        review_status: true, source_type: true, embedded_at: true, updated_at: true,
      },
    });
  }
```

Add the new handlers (place them before the `:id` routes so `import`/`review` aren't captured as ids):

```ts
  // Importacion masiva desde CSV. Todo entra como PENDING_REVIEW: nada de esto
  // es alcanzable por el bot hasta que alguien lo apruebe (PRD 2 §4.2, §5).
  @Post('import')
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  @UseInterceptors(FileInterceptor('file'))
  // `async` a proposito: el throw de abajo tiene que llegar como promesa
  // rechazada, no como excepcion sincronica (el test lo espera asi, y Nest
  // maneja ambas pero la asincronica es la que compone con el resto).
  async import(@UploadedFile() file: Express.Multer.File, @Req() req: any) {
    if (!file?.buffer) throw new BadRequestException('Subi un archivo CSV en el campo "file".');
    return this.importService.importCsv(file.buffer, req.tenantDb, req.tenant);
  }

  // Plantilla de ejemplo para que el cliente no adivine el formato.
  @Get('import/template')
  template(@Res() res: Response) {
    res.setHeader('Content-Type', 'text/csv; charset=utf-8');
    res.setHeader('Content-Disposition', 'attachment; filename="plantilla-faq.csv"');
    res.send(CSV_TEMPLATE);
  }

  @Get('review')
  reviewQueue(
    @Query('review_status') reviewStatus: string | undefined,
    @Query('source_type') sourceType: string | undefined,
    @Query('source_ref') sourceRef: string | undefined,
    @Req() req: any,
  ) {
    assertEnumParam(reviewStatus, Object.values(ReviewStatus), 'review_status');
    assertEnumParam(sourceType, Object.values(SourceType), 'source_type');
    return this.review.listQueue({ reviewStatus, sourceType, sourceRef }, req.tenantDb);
  }

  @Get('review/counts')
  reviewCounts(@Req() req: any) {
    return this.review.counts(req.tenantDb);
  }

  @Post(':id/approve')
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  approve(
    @Param('id') id: string,
    @Body() body: { question?: string; answer?: string; agents?: string[]; tags?: string[] },
    @Req() req: any,
  ) {
    // req.user.id lo pone JwtStrategy.validate — queda en reviewed_by para que
    // el historial diga QUIEN aprobo, no solo cuando.
    return this.review.approve(id, body ?? {}, req.tenantDb, req.tenant, req.user?.id);
  }

  @Post(':id/reject')
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  reject(@Param('id') id: string, @Req() req: any) {
    return this.review.reject(id, req.tenantDb, req.user?.id);
  }
```

- [ ] **Step 4: Register the services in `faq.module.ts`**

Add the two imports and list them in `providers` (and `exports`, matching how the module already exports its services):

```ts
import { FaqImportService } from './faq-import.service';
import { FaqReviewService } from './faq-review.service';
```
```ts
  providers: [FaqEmbeddingClient, FaqIngestionService, FaqRetrievalService, FaqImportService, FaqReviewService],
  exports: [FaqIngestionService, FaqRetrievalService, FaqImportService, FaqReviewService],
```

- [ ] **Step 5: Run the controller tests, confirm they pass**

```bash
npm test -- src/faq/faq.controller.spec.ts
```
Expected: PASS — the 3 original tests plus the 6 new ones.

- [ ] **Step 6: Type-check**

```bash
npx tsc --noEmit -p tsconfig.json
```
Expected: no errors.

- [ ] **Step 7: Manual verification against a running server**

With the dev environment up (`.\start-dev.ps1`) and a tenant to test against, run this end-to-end check. Substitute your tenant slug and credentials — `tenant-dev` / `dev@localhost.test` / `testpass123` for the current local setup:

```bash
node -e "
(async () => {
  const BASE='http://localhost:3000', SLUG='tenant-dev';
  const h0={'Content-Type':'application/json','X-Tenant-Slug':SLUG};
  const {token}=await fetch(BASE+'/auth/login',{method:'POST',headers:h0,body:JSON.stringify({identifier:'dev@localhost.test',password:'testpass123'})}).then(r=>r.json());
  const h={...h0,Authorization:'Bearer '+token};

  const csv='pregunta,respuesta\n¿Hacen envios?,Si a todo el pais.\n¿Precio?,Sale \$45.000.\n';
  const fd=new FormData();
  fd.append('file',new Blob([csv],{type:'text/csv'}),'faq.csv');
  const imported=await fetch(BASE+'/api/faq/import',{method:'POST',headers:{Authorization:h.Authorization,'X-Tenant-Slug':SLUG},body:fd}).then(r=>r.json());
  console.log('IMPORT:',JSON.stringify(imported,null,2));

  const queue=await fetch(BASE+'/api/faq/review',{headers:h}).then(r=>r.json());
  console.log('QUEUE:',queue.map(c=>({id:c.id,q:c.question,status:c.review_status})));

  const approved=await fetch(BASE+'/api/faq/'+queue[0].id+'/approve',{method:'POST',headers:h,body:'{}'}).then(r=>r.json());
  console.log('APPROVED:',approved.review_status);
})();
"
```

Expected: the import reports `imported: 1` with the price row in `rejected` (rule `precio`); the queue shows one `PENDING_REVIEW` chunk; approving it returns `APPROVED`. Then confirm the bot can now retrieve it — `POST /api/faq/test` with `{"message":"hacen envios a todo el pais?"}` should return `fired: true` (assuming `faq_rag_enabled` is on for that tenant), and the same query *before* approval should have returned nothing.

- [ ] **Step 8: Commit**

```bash
git add src/faq/faq.controller.ts src/faq/faq.module.ts src/faq/faq.controller.spec.ts
git commit -m "feat(faq): expose CSV import, review queue and approve/reject endpoints"
```

---

### Task 5: Apply the lint to the manual-entry path

PRD 1's `POST /api/faq` and `PATCH /api/faq/:id` accept any content at any length — PRD 1 §14 flagged this as an open question, and it matters more now that a review queue exists (two intake paths with different rules is how inconsistent content gets in).

**Files:**
- Modify: `src/faq/faq-ingestion.service.ts`
- Modify: `src/faq/faq-ingestion.service.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService lint enforcement', () => {
  function makeService() {
    const embeddingClient = { embed: jest.fn() } as any;
    const prisma = {} as any;
    return { service: new FaqIngestionService(prisma, embeddingClient), embeddingClient };
  }

  it('refuses to upsert a chunk whose answer contains a price', async () => {
    const { service, embeddingClient } = makeService();
    const tenantDb = { faqChunk: { findUnique: jest.fn() }, $executeRaw: jest.fn() };
    await expect(
      service.upsertBatch([{ question: '¿Precio?', answer: 'Sale $45.000.' }], { sourceType: 'MANUAL' as any }, tenantDb),
    ).rejects.toThrow(/precio/i);
    expect(embeddingClient.embed).not.toHaveBeenCalled();
  });

  it('refuses to update a chunk into invalid content', async () => {
    const { service } = makeService();
    const existing = { id: 'faq-1', question: '¿Envios?', answer: 'Si.', content_hash: 'x', agents: [], tags: [], active: true, review_status: 'APPROVED' };
    const tenantDb = { faqChunk: { findUnique: jest.fn().mockResolvedValue(existing), update: jest.fn() }, $executeRaw: jest.fn() };
    await expect(service.updateOne('faq-1', { answer: 'Ahora sale $9.999.' }, tenantDb)).rejects.toThrow(/precio/i);
  });

  // Pin de la semantica "valida el MERGE, no el patch": el patch aca es
  // impecable (solo tags) y aun asi tiene que fallar, porque el contenido ya
  // guardado viola la regla. Si alguien cambiara updateOne para validar solo
  // lo que llega en el patch, este test se cae.
  it('validates the merged result, not just the patch', async () => {
    const { service } = makeService();
    const existing = { id: 'faq-1', question: '¿Precio?', answer: 'Sale $45.000.', content_hash: 'x', agents: [], tags: [], active: true, review_status: 'APPROVED' };
    const tenantDb = { faqChunk: { findUnique: jest.fn().mockResolvedValue(existing), update: jest.fn() }, $executeRaw: jest.fn() };
    await expect(service.updateOne('faq-1', { tags: ['nuevo'] }, tenantDb)).rejects.toThrow(/precio/i);
    expect(tenantDb.faqChunk.update).not.toHaveBeenCalled();
  });

  // El agujero que encontro la review de la Task 5: el alta manual no pasa
  // `reviewStatus`, upsertBatch cae a APPROVED, y eso sale derecho al cliente.
  // Con el lint atado al destino, un placeholder sin resolver ahi es error.
  it('blocks an unresolved placeholder when the chunk lands APPROVED', async () => {
    const { service, embeddingClient } = makeService();
    const tenantDb = { faqChunk: { findUnique: jest.fn() }, $executeRaw: jest.fn() };
    await expect(
      service.upsertBatch(
        [{ question: '¿Horarios?', answer: 'Atendemos {{horario}}.' }],
        { sourceType: 'MANUAL' as any },
        tenantDb,
      ),
    ).rejects.toThrow(/placeholder/i);
    expect(embeddingClient.embed).not.toHaveBeenCalled();
  });

  // El mismo contenido, pero destinado a la cola: ahi el placeholder es
  // advertencia, porque despues lo mira un humano que puede resolverlo.
  it('allows an unresolved placeholder when the chunk lands PENDING_REVIEW', async () => {
    const { service, embeddingClient } = makeService();
    embeddingClient.embed.mockResolvedValue({ vectors: [[0.1]], usage: {}, model: 'm' });
    const tenantDb = { faqChunk: { findUnique: jest.fn().mockResolvedValue(null) }, $executeRaw: jest.fn().mockResolvedValue(1), aiUsage: { create: jest.fn() } };
    const result = await service.upsertBatch(
      [{ question: '¿Horarios?', answer: 'Atendemos {{horario}}.' }],
      { sourceType: 'IMPORT' as any, reviewStatus: 'PENDING_REVIEW' as any },
      tenantDb,
    );
    expect(result.created).toBe(1);
  });

  it('names which item of a batch failed', async () => {
    const { service } = makeService();
    const tenantDb = { faqChunk: { findUnique: jest.fn().mockResolvedValue(null) }, $executeRaw: jest.fn() };
    await expect(
      service.upsertBatch(
        [
          { question: '¿Envios?', answer: 'Si, a todo el pais.' },
          { question: '¿Precio?', answer: 'Sale $45.000.' },
        ],
        { sourceType: 'MANUAL' as any },
        tenantDb,
      ),
    ).rejects.toThrow(/item 2/i);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — no validation happens today, so both calls proceed instead of throwing.

- [ ] **Step 3: Add the lint check**

In `src/faq/faq-ingestion.service.ts`, import the lint and add a private guard, then call it from both write paths:

```ts
import { BadRequestException } from '@nestjs/common';
import { lintChunk, hasErrors } from './faq-lint';
```

```ts
  // Mismas reglas que la importacion masiva y la aprobacion: un solo criterio de
  // contenido para todas las vias de entrada (PRD 2 §5).
  //
  // La etapa del lint la decide el DESTINO del chunk, no la via de entrada. Si
  // va a quedar APPROVED sale derecho al cliente — no hay humano despues — asi
  // que se le exige la misma vara que a una aprobacion en la cola, donde un
  // {{placeholder}} sin resolver es error y no advertencia. Si va a
  // PENDING_REVIEW alcanza con 'intake', porque la cola lo vuelve a mirar.
  //
  // Atarlo al destino y no al endpoint es lo que evita que la proxima via de
  // entrada que alguien agregue tenga el mismo agujero: el alta manual escribia
  // APPROVED directo y se linteaba como si fuera a revisarse.
  private assertValid(question: string, answer: string, reviewStatus: string, index?: number) {
    const stage = reviewStatus === 'APPROVED' ? 'approval' : 'intake';
    const findings = lintChunk(question, answer, stage);
    if (hasErrors(findings)) {
      // Las reglas que fallaron van EN el mensaje, no solo en `findings`:
      // HttpException expone `message` como el texto del error, asi que un
      // cliente que solo loguea err.message igual ve por que se rechazo. El
      // indice importa cuando entra un lote: sin el, con 40 filas no se sabe
      // cual fallo.
      const rules = findings.filter((f) => f.severity === 'error').map((f) => f.rule).join(', ');
      const where = index === undefined ? '' : ` (item ${index + 1})`;
      throw new BadRequestException({
        message: `El contenido no pasa las reglas de FAQ${where}: ${rules}.`,
        findings,
        index,
      });
    }
  }
```

In `upsertBatch`, hoist the resolved `reviewStatus` above the loop (it's currently computed further down, inside the write loop — move it up and reuse the same variable there), then validate each input before anything else happens per chunk, before the hash, so nothing is embedded or written:

```ts
    const reviewStatus = opts.reviewStatus ?? 'APPROVED';

    for (let i = 0; i < chunks.length; i++) {
      const input = chunks[i];
      this.assertValid(input.question, input.answer, reviewStatus, i);
      const hash = this.computeHash(input.question, input.answer);
      // ... resto igual
```

Further down, the write loop's existing `const reviewStatus = opts.reviewStatus ?? 'APPROVED';` line must be **removed** (it's now hoisted above) — leaving both would shadow and is a needless duplicate.

In `updateOne`, validate the merged result after resolving `question`/`answer` but before hashing. The row's **current** status is what decides the stage: editing a row that is already `APPROVED` publishes immediately, so it faces the approval bar; editing one still in the queue only needs `intake`, because approval will re-check it.

```ts
    const question = patch.question?.trim() ?? existing.question;
    const answer = patch.answer?.trim() ?? existing.answer;
    this.assertValid(question, answer, existing.review_status);
    const hash = this.computeHash(question, answer);
    // ... resto igual
```

- [ ] **Step 4: Run the full FAQ suite**

```bash
npm test -- src/faq
```
Expected: PASS. Every existing FAQ test still passes — the fixtures they use ("¿Hacen envios?" / "Si, a todo el pais", the garantía pair) are all lint-clean, which is why this task comes last: it validates that the lint's thresholds don't reject the content the rest of the suite already treats as normal.

- [ ] **Step 5: Type-check and commit**

```bash
npx tsc --noEmit -p tsconfig.json
git add src/faq/faq-ingestion.service.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): enforce the content lint on manual create and edit too"
```

---

## Spec Coverage Check (self-review)

| PRD 2 section | Covered by |
|---|---|
| §4.1 vertical starter packs, §8 control-plane templates | **Phase 2 — not this plan** |
| §4.2 structured upload (CSV) + downloadable template | Tasks 2, 4 |
| §4.2 XLSX | **Deferred** — see Global Constraints |
| §4.3 document upload + LLM extraction + `source_span` | **Phase 3 — not this plan** |
| §4.4 manual entry | Already shipped in PRD 1; Task 5 brings it under the same lint |
| §5 review states, only APPROVED retrievable | Task 3 (transitions); enforcement is PRD 1's retrieval query, unchanged |
| §5 review screen (UI) | **Separate track** — different repo; this plan ships the API it consumes |
| §5 duplicate detection | **Phase 4 — not this plan** (needs a similarity function that doesn't exist yet) |
| §5 lint before the queue | Tasks 1, 2, 5 |
| §6 re-ingestion versioning | **Phase 4 — not this plan.** Needs a schema change; see Global Constraints for why the current `upsertBatch` can't express it |
| §7 cost / ledger | Inherited: `upsertBatch` already records embedding spend via `recordUsage('faq_index')` |
| §7 per-tenant document quota | **Deferred** — no documents in phase 1; the number is an open business decision (§12.4) |
| §9 phase 1 | This plan |
| §11 risks: stale prices, imperative text, re-upload flood | Lint (Tasks 1, 5); flood is inherently limited in phase 1 since each upload is its own batch |

## Execution

1. **Subagent-Driven (recommended)** — a fresh subagent per task, with a review pass between tasks.
2. **Inline Execution** — work through the tasks in this session, batch by batch.
