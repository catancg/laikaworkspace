# FAQ Content Ingestion — Phase 3 (Document Extraction) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a client hand us the material they already have — a policy document, an FAQ page, the paragraph they paste to customers ten times a day — and turn it into reviewable Q&A candidates, without the LLM being able to invent an answer the source never supported.

**Architecture:** Text in → chunked → one LLM extraction pass per chunk producing candidates that each carry a **verbatim `source_span`** → a deterministic support check that **drops** candidates whose answer isn't grounded in its own span → survivors land through phase 1's `upsertBatch` as `source_type: DOCUMENT`, `review_status: PENDING_REVIEW`. Phase 1's review queue already returns `source_span`, so the reviewer sees the source text beside the candidate with no further work.

**Tech Stack:** NestJS 11, Prisma 7, the existing `openai` SDK pointed at OpenRouter (same client and per-tenant key path as the rest of the system), Jest. **No new dependencies.**

**Spec:** `docs/prd-faq-content-ingestion.md` §4.3 is this phase, with §5 (review), §7 (cost) and §11 (risks) constraining it. Read `2026-09-02-faq-content-ingestion-phase1.md` and `-phase2.md` — this plan calls services both built.

## Global Constraints

- Comments and log messages in Spanish, matching the codebase.
- Services take `tenantDb?: any` trailing, resolved through a private `db(tenantDb)` helper.
- **Every candidate carries a verbatim `source_span`.** §4.3 calls this non-negotiable: it is what makes review possible, because the reviewer approves against the source text rather than against whether the answer sounds plausible. A candidate without a span is dropped, not stored with an empty one.
- **Unsupported candidates are dropped, not flagged.** §4.3 is explicit and the reasoning is worth preserving: flagged-but-present content gets bulk-approved by a tired reviewer, so surfacing weak candidates is worse than losing them. This means the pipeline deliberately accepts **lower recall for higher precision** — the opposite of the usual instinct, and not a knob to quietly retune later.
- **Extraction output is constrained by the same rules the lint enforces**, so candidates don't die at the `upsertBatch` gate: no prices/currency/discounts (the catalog is the only price source), answers under 600 characters, rioplatense Spanish, no imperative phrasing aimed at the bot.
- **Cap candidates per document.** §4.3 says ~60 regardless of length: "a 40-page PDF producing 200 candidates guarantees the review is not done properly." The cap is a review-quality mechanism, not a performance one.
- **Cost is metered against the same ledger** as the rest (`AiUsage`), with its own `kind` so ingestion spend stays separable from chat spend (§7). Use a **mid-tier** model, not the cheapest — §7 is explicit that this is a one-off per-document cost paid to avoid recurring review burden, the inverse of the per-message economics.
- **Deliberately deferred:**
  - **PDF and DOCX parsing.** §4.3 says "PDF, DOCX or pasted text". Pasted text needs zero new dependencies and exercises the entire risky path (chunk → extract → span → support check → queue); file parsing is mechanical text-acquisition bolted on the front. Same reasoning that put CSV before XLSX in phase 1, and it lets the extraction quality be judged before adding parser dependencies. Add `pdf-parse`/`mammoth` when a real client can't paste.
  - **An LLM-based support scorer.** §4.3 permits "a second cheap LLM pass scoring support, **or** a heuristic on token overlap". This plan ships the heuristic: deterministic, free, unit-testable, and no second round trip. If it proves too blunt against real documents, swapping it is a contained change behind one function.
  - **Re-extraction / versioning of a document.** That's phase 4's `source_ref` versioning work.
  - **Document retention.** §12's open question (keep the uploaded source or only spans?) is unresolved and is a data-handling decision, not an engineering one. This plan stores only the spans.

---

## File Structure

```
soylaika.backend/src/faq/
  faq-support.ts                  [CREATE] pure: is this answer grounded in this span?
  faq-support.spec.ts             [CREATE]
  faq-extraction.service.ts       [CREATE] chunk → LLM → candidates → support filter → upsertBatch
  faq-extraction.service.spec.ts  [CREATE]
  faq.controller.ts               [MODIFY] +POST /api/faq/extract
  faq.controller.spec.ts          [MODIFY]
  faq.module.ts                   [MODIFY] register FaqExtractionService
  doc/api-frontend.md             [MODIFY] document the endpoint
```

---

### Task 1: Support check

The gate that decides whether a generated answer is grounded in the text it claims to come from. Pure and deterministic so it can be tested hard — this is the component standing between a confident hallucination and a customer.

**Files:**
- Create: `src/faq/faq-support.ts`
- Test: `src/faq/faq-support.spec.ts`

**Interfaces:**
- Produces: `isSupported(answer: string, span: string): boolean`, `supportRatio(answer: string, span: string): number`, and `SUPPORT_THRESHOLD`. Consumed by Task 3.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-support.spec.ts`:

```ts
import { isSupported, supportRatio, SUPPORT_THRESHOLD } from './faq-support';

const span =
  'La garantia de los empapelados es de 6 meses desde la fecha de compra. ' +
  'Para hacerla valida hay que presentar el ticket original.';

describe('supportRatio', () => {
  it('scores 1 when the answer only reuses words from the span', () => {
    expect(supportRatio('La garantia es de 6 meses desde la compra.', span)).toBe(1);
  });

  it('scores 0 for an answer with no content words in common', () => {
    expect(supportRatio('Aceptamos tarjeta de credito y debito.', span)).toBe(0);
  });

  it('ignores accents and case', () => {
    expect(supportRatio('LA GARANTÍA ES DE 6 MESES.', span)).toBe(1);
  });

  it('ignores stopwords, so filler does not inflate the score', () => {
    // "de la y en" son puras stopwords: no deberian contar como respaldo.
    expect(supportRatio('de la y en', span)).toBe(0);
  });

  it('returns 0 for an empty span', () => {
    expect(supportRatio('cualquier cosa', '')).toBe(0);
  });

  it('returns 0 for an empty answer', () => {
    expect(supportRatio('', span)).toBe(0);
  });
});

describe('isSupported', () => {
  it('accepts an answer drawn from the span', () => {
    expect(isSupported('La garantia es de 6 meses desde la fecha de compra.', span)).toBe(true);
  });

  // El caso que justifica todo el modulo: suena bien, el documento no lo dice.
  it('rejects a plausible answer the span never supports', () => {
    expect(isSupported('La garantia se puede extender a 12 meses pagando un adicional.', span)).toBe(false);
  });

  it('rejects an answer that only borrows a word or two', () => {
    expect(isSupported('La garantia cubre robo, incendio y daños por mascotas.', span)).toBe(false);
  });

  it('rejects when the span is empty, whatever the answer says', () => {
    expect(isSupported('La garantia es de 6 meses.', '')).toBe(false);
  });

  it('is consistent with the exported threshold when no hard rule applies', () => {
    const ratio = supportRatio('La garantia es de 6 meses desde la fecha de compra.', span);
    expect(ratio >= SUPPORT_THRESHOLD).toBe(isSupported('La garantia es de 6 meses desde la fecha de compra.', span));
  });
});

// Las dos reglas duras. El ratio por si solo no puede ver una sola palabra
// cambiada en una respuesta larga — cambiar "6 meses" por "9 meses" deja el
// ratio en 0.8 y pasaria. Numeros y polaridad se verifican exacto.
describe('isSupported — numeros', () => {
  it('rejects an answer that changes a number the span states', () => {
    expect(isSupported('La garantia es de 9 meses desde la fecha de compra.', span)).toBe(false);
  });

  it('rejects a changed multi-digit number even inside an otherwise correct answer', () => {
    const s = 'El envio cuesta 5000 y tarda 48 horas habiles.';
    expect(isSupported('El envio cuesta 8000 y tarda 48 horas.', s)).toBe(false);
  });

  // Este es el que prueba la REGLA DURA y no el ratio: la respuesta es casi
  // verbatim (ratio 0.857, muy por encima del umbral) y lo unico malo es el
  // numero. Sin la regla dura este test pasaria igual y no probaria nada.
  it('rejects a wrong number inside an otherwise near-verbatim answer', () => {
    const envios = 'Los envios salen dentro de las 48 horas habiles a todo el pais.';
    expect(isSupported('Los envios salen dentro de las 72 horas habiles a todo el pais.', envios)).toBe(false);
  });

  it('accepts a short answer whose number the span supports', () => {
    expect(isSupported('6 meses.', span)).toBe(true);
  });

  it('accepts an answer with no numbers at all', () => {
    const s = 'Enviamos a todo el pais sin excepcion.';
    expect(isSupported('Enviamos a todo el pais.', s)).toBe(true);
  });
});

// El riesgo propio de la regla dura de numeros: rechazar una respuesta CORRECTA
// porque el numero esta escrito distinto. En pesos el separador de miles es la
// forma normal de escribir un precio.
describe('isSupported — separador de miles', () => {
  const conPunto = 'El costo es de 1.500 pesos por unidad segun el reglamento vigente hoy.';
  const sinPunto = 'El costo es de 1500 pesos por unidad segun el reglamento vigente hoy.';

  it('treats 1.500 and 1500 as the same number', () => {
    expect(isSupported(sinPunto, conPunto)).toBe(true);
  });

  it('works in the other direction too', () => {
    expect(isSupported(conPunto, sinPunto)).toBe(true);
  });

  it('collapses more than one separator', () => {
    expect(
      isSupported('El total del combo es de 1500000 pesos.', 'El total del combo es de 1.500.000 pesos.'),
    ).toBe(true);
  });

  // Que tolere el formato no puede volverlo ciego a un numero realmente distinto.
  it('still rejects a genuinely different number', () => {
    expect(isSupported('El costo es de 1.501 pesos por unidad.', 'El costo es de 1.500 pesos por unidad.')).toBe(false);
  });

  // Tres digitos exactos: un decimal no es un separador de miles.
  it('leaves decimals alone', () => {
    expect(isSupported('Tarda 48.5 horas habiles.', 'El envio tarda 48.5 horas habiles a todo el pais.')).toBe(true);
  });

  // Una cola de ceros no afirma nada, asi que no puede ser un dato sin respaldo.
  it('treats 1500,00 and 1.500 as the same number', () => {
    expect(
      isSupported('El costo es de 1500,00 pesos por unidad segun el reglamento.',
                  'El costo es de 1.500 pesos por unidad segun el reglamento.'),
    ).toBe(true);
  });

  it('works with the zero tail on the other side too', () => {
    expect(
      isSupported('El costo es de 1.500,00 pesos por unidad segun el reglamento.',
                  'El costo es de 1500 pesos por unidad segun el reglamento.'),
    ).toBe(true);
  });

  // Pero una cola que NO es cero si afirma algo: aca la respuesta se invento una
  // precision que el documento no da, y eso tiene que seguir cayendo.
  it('still rejects an answer that invents precision the span does not give', () => {
    expect(
      isSupported('El envio tarda 48.5 horas habiles a todo el pais.',
                  'El envio tarda 48 horas habiles a todo el pais.'),
    ).toBe(false);
  });
});

describe('isSupported — polaridad', () => {
  it('rejects an answer that negates what its span asserts', () => {
    expect(isSupported('La garantia no es de 6 meses.', 'La garantia es de 6 meses.')).toBe(false);
  });

  it('rejects other negation words too', () => {
    expect(
      isSupported('Nunca aceptamos devoluciones abiertas.', 'Aceptamos devoluciones abiertas con falla de fabrica.'),
    ).toBe(false);
  });

  it('accepts a negation the span also makes', () => {
    expect(isSupported('No hacemos envios a Uruguay.', 'No hacemos envios a Uruguay ni a Chile.')).toBe(true);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-support.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-support.ts`**

```ts
// ¿Esta respuesta esta realmente respaldada por el fragmento del que dice salir?
//
// Esto es lo unico que separa una alucinacion segura de si misma de un cliente.
// Un LLM al que se le pide "sacá preguntas frecuentes de este documento"
// produce con total naturalidad respuestas plausibles a preguntas que el
// documento nunca contesto. El span es la defensa, y esta funcion es la que la
// hace efectiva: si la respuesta no reusa el vocabulario del span, se descarta.
//
// Deliberadamente DESCARTA en vez de marcar (PRD 2 §4.3): contenido dudoso pero
// presente en la cola termina aprobado en lote por alguien cansado. Preferimos
// perder un candidato bueno que colar uno inventado — recall mas bajo a cambio
// de precision, al reves de lo que uno haria por instinto.
//
// Heuristica y no una segunda pasada de LLM (§4.3 permite cualquiera de las
// dos): es determinista, gratis y testeable. Si contra documentos reales
// resulta demasiado gruesa, cambiarla es tocar una sola funcion.

/** Proporcion de palabras de contenido de la respuesta que aparecen en el span. */
export const SUPPORT_THRESHOLD = 0.7;

// Stopwords del español rioplatense. No cuentan como respaldo: una respuesta
// puede compartir "de la que en" con cualquier texto sin decir lo mismo.
//
// OJO: "no" NO esta en esta lista, a proposito. Es gramaticalmente una palabra
// vacia pero semanticamente lo cambia todo, y tratarla como relleno hacia que
// "la garantia NO es de 6 meses" tuviera respaldo perfecto contra un span que
// dice exactamente lo contrario.
const STOPWORDS = new Set([
  'a','al','ante','asi','aca','aunque','cada','como','con','contra','cual','cuando','de','del','desde',
  'donde','dos','el','ella','ellas','ellos','en','entre','era','eran','es','esa','ese','eso','esta',
  'estan','este','esto','estos','fue','ha','hasta','hay','la','las','le','les','lo','los','mas','me',
  'mi','mientras','muy','nos','o','para','pero','por','porque','que','se','segun','ser','si','sin',
  'sobre','solo','son','su','sus','tambien','te','tiene','tienen','todo','todos','tu','un','una','uno',
  'unos','vos','y','ya',
]);

const NEGACIONES = new Set(['no', 'nunca', 'tampoco', 'ningun', 'ninguna', 'jamas']);

// "1.500" y "1500" son el mismo numero, pero el tokenizer parte en "." y dejaria
// {1, 500} de un lado contra {1500} del otro — con lo cual la regla dura de
// numeros rechazaba una respuesta correcta solo por el formato. En pesos el
// separador de miles es lo normal, no un caso raro, y los precios son
// justamente el dato que este modulo tiene que proteger.
//
// Colapsa solo cuando agrupa de a 3 digitos EXACTOS, para no romper decimales
// ("48.5" queda como esta). El bucle es porque un replace global de una pasada
// deja "1.500.000" a medio colapsar.
function collapseThousands(text: string): string {
  let out = text;
  let prev = '';
  do {
    prev = out;
    out = out.replace(/(\d)[.,](\d{3})(?!\d)/g, '$1$2');
  } while (out !== prev);
  // Y despues la cola decimal de ceros: "1500,00" es el mismo numero que "1500",
  // pero dejaba un token "00" suelto que la regla dura leia como un numero sin
  // respaldo y rechazaba una respuesta correcta. Una cola de ceros no afirma
  // nada, asi que no puede ser una afirmacion sin respaldo.
  // Ojo con el orden: los miles primero, si no "100.000" se comeria como cola.
  // Solo ceros: "48.5" contra un span que dice "48" SIGUE rechazando, porque ahi
  // la respuesta se invento precision que el documento no da.
  return out.replace(/(\d)[.,]0+(?!\d)/g, '$1');
}

function normalize(text: string): string {
  return collapseThousands(
    (text ?? '')
      .normalize('NFD')
      .replace(/[̀-ͯ]/g, '')   // fuera acentos: "garantía" y "garantia" son la misma palabra
      .toLowerCase(),
  );
}

function allWords(text: string): string[] {
  return normalize(text).split(/[^a-z0-9]+/).filter(Boolean);
}

function contentWords(text: string): string[] {
  // Los digitos sueltos SI cuentan (`w.length > 1 || /^[0-9]$/`): "6 meses" es
  // justo el tipo de dato que esto tiene que verificar, y descartarlo hacia que
  // cambiar un 6 por un 9 no costara nada.
  return allWords(text).filter((w) => (w.length > 1 || /^[0-9]$/.test(w)) && !STOPWORDS.has(w));
}

function numbers(text: string): string[] {
  return allWords(text).filter((w) => /^[0-9]+$/.test(w));
}

function negates(text: string): boolean {
  return allWords(text).some((w) => NEGACIONES.has(w));
}

export function supportRatio(answer: string, span: string): number {
  const spanWords = new Set(contentWords(span));
  const answerWords = contentWords(answer);

  if (!spanWords.size || !answerWords.length) return 0;

  const covered = answerWords.filter((w) => spanWords.has(w)).length;
  return covered / answerWords.length;
}

/**
 * El ratio solo mide respaldo GENERAL, y por construccion no puede ver una sola
 * palabra cambiada dentro de una respuesta larga: cambiar "6 meses" por "9
 * meses" en una oracion por lo demas correcta deja el ratio en 0.8 y pasa.
 *
 * Por eso hay dos reglas DURAS ademas del ratio. No son heuristicas: numeros y
 * polaridad son las dos cosas que se pueden verificar exacto, y son justo las
 * dos que mas caro salen si el modelo las inventa.
 */
export function isSupported(answer: string, span: string): boolean {
  if (!normalize(span).trim() || !normalize(answer).trim()) return false;

  // 1. Todo numero que afirme la respuesta tiene que estar en el span. Un plazo,
  //    un precio o una cantidad inventada es el error mas caro de este dominio.
  const spanNumbers = new Set(numbers(span));
  if (numbers(answer).some((n) => !spanNumbers.has(n))) return false;

  // 2. Si la respuesta niega y el span no, la respuesta esta diciendo lo
  //    contrario a su propia evidencia.
  if (negates(answer) && !negates(span)) return false;

  return supportRatio(answer, span) >= SUPPORT_THRESHOLD;
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-support.spec.ts
```
Expected: PASS (11 tests).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-support.ts src/faq/faq-support.spec.ts
git commit -m "feat(faq): add the support check that drops ungrounded extraction candidates"
```

---

### Task 2: Plumb `source_span` through ingestion

`FaqChunk.source_span` has existed as a column since PRD 1, and phase 1's review queue already returns it — but **nothing has ever written to it.** `FaqChunkInput` has no such field and `upsertBatch`'s INSERT doesn't include the column, so without this task every extracted candidate would store `source_span = NULL` and the reviewer would have nothing to check the answer against. That silently destroys the entire premise of phase 3, while every test still passes.

This is a small change to a phase 1 file, kept as its own task so it gets its own review.

**Files:**
- Modify: `src/faq/faq-ingestion.service.ts`
- Modify: `src/faq/faq-ingestion.service.spec.ts`

**Interfaces:**
- Produces: `FaqChunkInput.sourceSpan?: string`, persisted to `FaqChunk.source_span`. Consumed by Task 3.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService source_span', () => {
  function makeService() {
    const embeddingClient = { embed: jest.fn().mockResolvedValue({ vectors: [[0.1]], usage: {}, model: 'm' }) } as any;
    return { service: new FaqIngestionService({} as any, embeddingClient), embeddingClient };
  }

  it('persists the source span when one is supplied', async () => {
    const { service } = makeService();
    const tenantDb = {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(null) },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };
    await service.upsertBatch(
      [{ question: '¿Garantia?', answer: 'Son 6 meses.', sourceSpan: 'La garantia es de 6 meses.' }],
      { sourceType: 'DOCUMENT' as any, reviewStatus: 'PENDING_REVIEW' as any },
      tenantDb,
    );

    // $executeRaw es un tagged template: (strings, ...values). El span tiene que
    // viajar como valor ligado, no quedarse en el camino.
    const values = tenantDb.$executeRaw.mock.calls[0].slice(1);
    expect(values).toContain('La garantia es de 6 meses.');
  });

  it('writes null when no span is supplied', async () => {
    const { service } = makeService();
    const tenantDb = {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(null) },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };
    await service.upsertBatch(
      [{ question: '¿Envios?', answer: 'Si, a todo el pais.' }],
      { sourceType: 'MANUAL' as any },
      tenantDb,
    );
    const values = tenantDb.$executeRaw.mock.calls[0].slice(1);
    expect(values).toContain(null);
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — the span never reaches `$executeRaw`.

- [ ] **Step 3: Add the field and persist it**

In `src/faq/faq-ingestion.service.ts`:

1. Add to `FaqChunkInput`:
```ts
  /** Cita textual del documento del que salio la respuesta (PRD 2 §4.3). */
  sourceSpan?: string;
```

2. In `upsertBatch`'s INSERT, add `source_span` to the column list and `${input.sourceSpan ?? null}` to the corresponding position in `VALUES`. **Keep the column and value positions aligned** — a mismatch here writes the wrong data into the wrong column, which no type-checker will catch.

3. In `upsertBatch`'s UPDATE branch, add `source_span=${input.sourceSpan ?? null}` alongside the other updated fields, so re-ingesting a document refreshes the span too.

Leave `updateOne` alone: a manual edit doesn't change where the content originally came from.

- [ ] **Step 4: Run the full FAQ suite**

```bash
npm test -- src/faq
```
Every pre-existing test must still pass — the column addition must not disturb the existing INSERT.

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-ingestion.service.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): persist source_span so extracted candidates are reviewable"
```

---

### Task 3: Extraction service

**Files:**
- Create: `src/faq/faq-extraction.service.ts`
- Test: `src/faq/faq-extraction.service.spec.ts`

**Interfaces:**
- Consumes: `isSupported` (Task 1); `FaqIngestionService.upsertBatch` (phase 1); the `openai` SDK against OpenRouter, mirroring `AiService`'s client construction and per-tenant key fallback.
- Produces: `FaqExtractionService.extractFromText(text, opts, tenantDb?, tenant?)` returning `ExtractionResult`. Consumed by Task 4.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-extraction.service.spec.ts`:

```ts
import { BadRequestException } from '@nestjs/common';
import { FaqExtractionService, MAX_CANDIDATES, MIN_TEXT_CHARS, chunkText } from './faq-extraction.service';

// 281 caracteres: tiene que pasar MIN_TEXT_CHARS (200) o cada test de extraccion
// muere en el guard de "texto muy corto" sin llegar nunca al filtrado. Hay un test
// abajo que lo verifica, porque cuando esto falla fallan seis tests con un error
// que no menciona el fixture.
const DOC =
  'La garantia de los empapelados es de 6 meses desde la fecha de compra. ' +
  'Para hacerla valida hay que presentar el ticket original. ' +
  'Los envios salen dentro de las 48 horas habiles a todo el pais. ' +
  'Aceptamos cambios dentro de los 30 dias si el producto esta sin abrir y con su etiqueta.';

// Un candidato "bueno": la respuesta reusa el vocabulario de su span.
const goodCandidate = {
  question: '¿Cuanto dura la garantia?',
  answer: 'La garantia es de 6 meses desde la fecha de compra.',
  source_span: 'La garantia de los empapelados es de 6 meses desde la fecha de compra.',
};

// El caso peligroso: suena perfectamente razonable y el documento no lo dice.
const hallucinated = {
  question: '¿Se puede extender la garantia?',
  answer: 'Si, se puede extender a 24 meses contratando el plan premium.',
  source_span: 'La garantia de los empapelados es de 6 meses desde la fecha de compra.',
};

function makeService(candidates: any[] = [goodCandidate]) {
  const completion = {
    choices: [{ message: { content: JSON.stringify({ candidates }) } }],
    usage: { total_tokens: 100, cost: 0.01 },
    model: 'test-model',
  };
  const client = { chat: { completions: { create: jest.fn().mockResolvedValue(completion) } } };
  // El SDK de openai TIRA en el constructor si no resuelve una api key (ni de las
  // opciones ni de OPENAI_API_KEY). Eso esta bien en produccion — es mejor reventar
  // al arrancar que con un 401 confuso en el primer request — asi que el test le da
  // una key falsa en vez de que el servicio se invente un default.
  // El resto sigue devolviendo undefined a proposito, para que el `?? 'openai/gpt-4o-mini'`
  // de FAQ_EXTRACTION_MODEL se ejercite de verdad.
  const config = {
    get: jest.fn((key: string) => (key === 'OPENROUTER_API_KEY' ? 'test-key' : undefined)),
  } as any;
  const ingestion = { upsertBatch: jest.fn().mockResolvedValue({ created: 1, updated: 0, unchanged: 0, chunks: [] }) } as any;
  const service = new FaqExtractionService(config, ingestion);
  (service as any).defaultClient = client;
  return { service, client, ingestion };
}

describe('chunkText', () => {
  it('returns one chunk for short text', () => {
    expect(chunkText('hola', 1000)).toEqual(['hola']);
  });

  it('splits long text into chunks no larger than the limit', () => {
    const long = 'a'.repeat(2500);
    const chunks = chunkText(long, 1000);
    expect(chunks.length).toBeGreaterThan(1);
    for (const c of chunks) expect(c.length).toBeLessThanOrEqual(1000);
  });

  it('prefers to split on a paragraph boundary', () => {
    const text = 'parrafo uno.\n\nparrafo dos.';
    expect(chunkText(text, 20)).toEqual(['parrafo uno.', 'parrafo dos.']);
  });
});

describe('FaqExtractionService.extractFromText', () => {
  // Guard del fixture, no del codigo: si DOC cae por debajo del minimo, los demas
  // tests fallan todos con "texto muy corto" y nada apunta al fixture.
  it('uses a fixture long enough to reach the extraction path', () => {
    expect(DOC.length).toBeGreaterThanOrEqual(MIN_TEXT_CHARS);
  });

  it('rejects text too short to be worth extracting', async () => {
    const { service, client } = makeService();
    await expect(service.extractFromText('hola', {}, {})).rejects.toBeInstanceOf(BadRequestException);
    expect(client.chat.completions.create).not.toHaveBeenCalled();
  });

  it('keeps a candidate whose answer is grounded in its span', async () => {
    const { service, ingestion } = makeService([goodCandidate]);
    const result = await service.extractFromText(DOC, {}, {});

    expect(result.kept).toBe(1);
    expect(result.dropped).toHaveLength(0);
    const [chunks, opts] = ingestion.upsertBatch.mock.calls[0];
    expect(opts.sourceType).toBe('DOCUMENT');
    expect(opts.reviewStatus).toBe('PENDING_REVIEW');
    expect(chunks[0].sourceSpan).toBe(goodCandidate.source_span);
  });

  // La razon de ser de toda la fase.
  it('drops a plausible-sounding candidate its span does not support', async () => {
    const { service, ingestion } = makeService([hallucinated]);
    const result = await service.extractFromText(DOC, {}, {});

    expect(result.kept).toBe(0);
    expect(result.dropped).toHaveLength(1);
    expect(result.dropped[0].reason).toBe('sin respaldo');
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });

  it('drops a candidate with no span at all', async () => {
    const { service } = makeService([{ question: '¿Y?', answer: 'Algo.', source_span: '   ' }]);
    const result = await service.extractFromText(DOC, {}, {});
    expect(result.kept).toBe(0);
    expect(result.dropped[0].reason).toBe('sin span');
  });

  it('drops a candidate whose span is not verbatim in the source text', async () => {
    const { service } = makeService([
      { ...goodCandidate, source_span: 'Un fragmento que el documento nunca dijo.' },
    ]);
    const result = await service.extractFromText(DOC, {}, {});
    expect(result.kept).toBe(0);
    expect(result.dropped[0].reason).toBe('span inventado');
  });

  it('caps the number of candidates regardless of what the model returns', async () => {
    const many = Array.from({ length: MAX_CANDIDATES + 10 }, () => ({ ...goodCandidate }));
    const { service, ingestion } = makeService(many);
    const result = await service.extractFromText(DOC, {}, {});
    expect(result.kept).toBeLessThanOrEqual(MAX_CANDIDATES);
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks.length).toBeLessThanOrEqual(MAX_CANDIDATES);
  });

  it('survives a model reply that is not valid JSON', async () => {
    const { service, ingestion } = makeService();
    (service as any).defaultClient = {
      chat: { completions: { create: jest.fn().mockResolvedValue({
        choices: [{ message: { content: 'lo siento, no puedo' } }], usage: {}, model: 'm',
      }) } },
    };
    const result = await service.extractFromText(DOC, {}, {});
    expect(result.kept).toBe(0);
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-extraction.service.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-extraction.service.ts`**

```ts
import { BadRequestException, Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import OpenAI from 'openai';
import { randomUUID } from 'crypto';
import { FaqIngestionService } from './faq-ingestion.service';
import { isSupported } from './faq-support';

/** Tope de candidatos por documento (PRD 2 §4.3). Es un mecanismo de calidad de
 *  revision, no de performance: 200 candidatos garantizan que nadie los mire en
 *  serio. */
export const MAX_CANDIDATES = 60;

/** Texto minimo para que valga la pena gastar una llamada al modelo. */
export const MIN_TEXT_CHARS = 200;

/** Tamaño de cada pedazo que se manda al modelo. */
export const CHUNK_CHARS = 6000;

export interface DroppedCandidate {
  question: string;
  reason: 'sin span' | 'span inventado' | 'sin respaldo';
}

export interface ExtractionResult {
  sourceRef: string;
  candidates: number;   // lo que devolvio el modelo, antes de filtrar
  kept: number;
  dropped: DroppedCandidate[];
  created: number;
  updated: number;
  unchanged: number;
}

// Parte el texto en pedazos, prefiriendo cortar en limite de parrafo para no
// romper una idea al medio (un span cortado deja de ser verificable).
export function chunkText(text: string, limit = CHUNK_CHARS): string[] {
  const clean = (text ?? '').trim();
  if (clean.length <= limit) return clean ? [clean] : [];

  const chunks: string[] = [];
  let rest = clean;
  while (rest.length > limit) {
    const window = rest.slice(0, limit);
    const cut = window.lastIndexOf('\n\n') > 0 ? window.lastIndexOf('\n\n') : window.lastIndexOf('. ') + 1;
    const at = cut > limit * 0.4 ? cut : limit;   // si no hay corte natural razonable, corte duro
    chunks.push(rest.slice(0, at).trim());
    rest = rest.slice(at).trim();
  }
  if (rest) chunks.push(rest);
  return chunks.filter(Boolean);
}

@Injectable()
export class FaqExtractionService {
  private readonly logger = new Logger(FaqExtractionService.name);
  private readonly defaultClient: OpenAI;

  constructor(
    private readonly config: ConfigService,
    private readonly ingestion: FaqIngestionService,
  ) {
    this.defaultClient = new OpenAI({
      baseURL: 'https://openrouter.ai/api/v1',
      apiKey: this.config.get<string>('OPENROUTER_API_KEY'),
      timeout: 60_000,
      maxRetries: 0,
      defaultHeaders: { 'HTTP-Referer': 'https://asapmarketing.com', 'X-Title': 'ASAP Marketing Bot' },
    });
  }

  async extractFromText(
    text: string,
    opts: { sourceName?: string } = {},
    tenantDb?: any,
    tenant?: any,
  ): Promise<ExtractionResult> {
    const clean = (text ?? '').trim();
    if (clean.length < MIN_TEXT_CHARS) {
      throw new BadRequestException(`El texto es muy corto para extraer preguntas (minimo ${MIN_TEXT_CHARS} caracteres).`);
    }

    const sourceRef = `document:${randomUUID()}`;
    const model = this.config.get<string>('FAQ_EXTRACTION_MODEL') ?? 'openai/gpt-4o-mini';
    const client = this.buildClient(tenant);

    const raw: any[] = [];
    for (const chunk of chunkText(clean)) {
      if (raw.length >= MAX_CANDIDATES) break;
      raw.push(...(await this.extractChunk(client, model, chunk, tenantDb)));
    }

    const dropped: DroppedCandidate[] = [];
    const kept: any[] = [];

    for (const c of raw) {
      if (kept.length >= MAX_CANDIDATES) break;

      const question = String(c?.question ?? '').trim();
      const answer = String(c?.answer ?? '').trim();
      const span = String(c?.source_span ?? '').trim();
      if (!question || !answer) continue;

      // Sin span no hay forma de revisar: el revisor no tendria contra que
      // comparar. §4.3 lo llama no negociable.
      if (!span) {
        dropped.push({ question, reason: 'sin span' });
        continue;
      }
      // El span tiene que estar TEXTUAL en el documento. Un modelo que parafrasea
      // el span ya perdio la propiedad que lo hacia util como evidencia.
      if (!clean.includes(span)) {
        dropped.push({ question, reason: 'span inventado' });
        continue;
      }
      // Y la respuesta tiene que estar respaldada por ese span, no meramente
      // acompañada por el.
      if (!isSupported(answer, span)) {
        dropped.push({ question, reason: 'sin respaldo' });
        continue;
      }

      kept.push({
        question,
        answer,
        agents: [],
        tags: [],
        sourceOrdinal: kept.length + 1,
        sourceSpan: span,
      });
    }

    const result = kept.length
      ? await this.ingestion.upsertBatch(
          kept,
          { sourceType: 'DOCUMENT', sourceRef, reviewStatus: 'PENDING_REVIEW' },
          tenantDb,
          tenant,
        )
      : { created: 0, updated: 0, unchanged: 0, chunks: [] };

    this.logger.log(
      `Extraccion ${sourceRef} (${opts.sourceName ?? 'texto pegado'}): ` +
      `${raw.length} candidatos, ${kept.length} conservados, ${dropped.length} descartados ` +
      `(${dropped.filter((d) => d.reason === 'sin respaldo').length} sin respaldo)`,
    );

    return {
      sourceRef,
      candidates: raw.length,
      kept: kept.length,
      dropped,
      created: result.created,
      updated: result.updated,
      unchanged: result.unchanged,
    };
  }

  private buildClient(tenant?: any): OpenAI {
    if (!tenant?.openrouter_api_key) return this.defaultClient;
    return new OpenAI({
      baseURL: 'https://openrouter.ai/api/v1',
      apiKey: tenant.openrouter_api_key,
      timeout: 60_000,
      maxRetries: 0,
      defaultHeaders: { 'HTTP-Referer': 'https://asapmarketing.com', 'X-Title': 'ASAP Marketing Bot' },
    });
  }

  private async extractChunk(client: OpenAI, model: string, chunk: string, tenantDb?: any): Promise<any[]> {
    try {
      const completion: any = await client.chat.completions.create({
        model,
        messages: [
          { role: 'system', content: EXTRACTION_PROMPT },
          { role: 'user', content: chunk },
        ],
        max_tokens: 2000,
        usage: { include: true },
      } as any);

      void this.recordUsage(tenantDb, completion, 'faq_extract');

      const text = completion.choices?.[0]?.message?.content ?? '';
      const match = text.match(/\{[\s\S]*\}/);
      if (!match) {
        this.logger.warn('La respuesta del modelo no traia JSON — no se extrae nada de este pedazo');
        return [];
      }
      const parsed = JSON.parse(match[0]);
      return Array.isArray(parsed?.candidates) ? parsed.candidates : [];
    } catch (err: any) {
      this.logger.warn(`Extraccion de un pedazo fallo: ${err?.message ?? err}`);
      return [];
    }
  }

  // Mismo ledger que el resto (AiUsage), con `kind` propio para que el gasto de
  // ingesta quede separable del gasto de chat (PRD 2 §7).
  private async recordUsage(db: any, completion: any, kind: string): Promise<void> {
    try {
      const u = completion?.usage;
      if (!u || !db) return;
      await db.aiUsage.create({
        data: {
          model: completion.model ?? 'desconocido',
          kind,
          promptTokens: u.prompt_tokens ?? 0,
          completionTokens: u.completion_tokens ?? 0,
          totalTokens: u.total_tokens ?? 0,
          cost: typeof u.cost === 'number' ? u.cost : 0,
        },
      });
    } catch (err: any) {
      this.logger.warn(`No se pudo registrar el uso de IA (extraccion): ${err?.message ?? err}`);
    }
  }
}

// El prompt lleva las mismas restricciones que despues aplica el lint, para que
// los candidatos no mueran en el gate de upsertBatch (PRD 2 §4.3).
const EXTRACTION_PROMPT = `Sos un asistente que extrae preguntas frecuentes de documentos de un negocio, para cargarlas en la base de conocimiento de un bot de atencion.

Te paso un fragmento de documento. Devolve SOLO JSON con esta forma, sin texto alrededor:
{"candidates":[{"question":"...","answer":"...","source_span":"..."}]}

REGLAS (obligatorias):
- "source_span" tiene que ser una cita TEXTUAL del fragmento, copiada caracter por caracter. Si no podes citar la parte exacta que respalda la respuesta, no generes ese candidato.
- La respuesta tiene que salir del span. NO completes con conocimiento general ni con lo que suene razonable: si el documento no lo dice, no existe.
- Preferi pocas preguntas bien respaldadas antes que muchas dudosas.
- NADA de precios, importes, porcentajes de descuento ni codigos de producto: esos datos salen del catalogo y se desactualizan.
- Respuestas de menos de 600 caracteres.
- Español rioplatense (voseo), tono cercano y profesional, como le hablarias a un cliente por WhatsApp.
- La respuesta es informacion para el cliente, NO instrucciones para el bot: nada de "decile al cliente que...".
- Si el fragmento no tiene nada que sirva como pregunta frecuente, devolve {"candidates":[]}.`;
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-extraction.service.spec.ts
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-extraction.service.ts src/faq/faq-extraction.service.spec.ts
git commit -m "feat(faq): extract Q&A candidates from pasted text with span verification"
```

---

### Task 4: Endpoint and wiring

**Files:**
- Modify: `src/faq/faq.controller.ts`, `src/faq/faq.module.ts`, `src/faq/faq.controller.spec.ts`, `doc/api-frontend.md`

- [ ] **Step 1: Write the failing test**

Append a new describe block to `src/faq/faq.controller.spec.ts`, and update every existing `new FaqController(...)` call site to pass a **seventh** argument (`{} as any`). The constructor grows again; `ts-jest` will not catch a stale call site, only `tsc` will.

```ts
describe('FaqController — extraction', () => {
  function makeController() {
    const extraction = {
      extractFromText: jest.fn().mockResolvedValue({ sourceRef: 'document:1', candidates: 8, kept: 5, dropped: [], created: 5, updated: 0, unchanged: 0 }),
    } as any;
    const controller = new (require('./faq.controller').FaqController)(
      {} as any, {} as any, {} as any, {} as any, {} as any, {} as any, extraction,
    );
    return { controller, extraction };
  }

  const req = { tenantDb: {}, tenant: { slug: 'itt' }, user: { id: 'user-7' } } as any;

  it('extract() forwards the text and source name', async () => {
    const { controller, extraction } = makeController();
    const result = await controller.extract({ text: 'un documento largo', sourceName: 'politicas.txt' } as any, req);
    expect(extraction.extractFromText).toHaveBeenCalledWith(
      'un documento largo', { sourceName: 'politicas.txt' }, req.tenantDb, req.tenant,
    );
    expect(result.kept).toBe(5);
  });

  it('extract() rejects a missing text body', async () => {
    const { controller, extraction } = makeController();
    await expect(controller.extract({} as any, req)).rejects.toThrow();
    expect(extraction.extractFromText).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run, confirm failure**

```bash
npm test -- src/faq/faq.controller.spec.ts
```

- [ ] **Step 3: Add the handler**

Import `FaqExtractionService`, add it as a seventh **plainly required** constructor param (no `?`, no default — see the phase 1 and 2 notes on why), and add this handler with the other static-segment routes, before the `:id` routes:

```ts
  // Extraccion desde texto pegado. Todo lo que sobrevive al chequeo de respaldo
  // entra a la cola de revision con su span, para que el revisor apruebe contra
  // el texto original y no contra lo bien que suena la respuesta.
  @Post('extract')
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  async extract(@Body() body: { text?: string; sourceName?: string }, @Req() req: any) {
    const text = (body?.text ?? '').trim();
    if (!text) throw new BadRequestException('Mandá el texto a procesar en el campo "text".');
    return this.extraction.extractFromText(text, { sourceName: body?.sourceName }, req.tenantDb, req.tenant);
  }
```

- [ ] **Step 4: Register in `faq.module.ts`** — add `FaqExtractionService` to `providers` and `exports`.

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq
npx tsc --noEmit -p tsconfig.json
```
Both must be clean. Run both separately — a stale constructor call site is invisible to the first.

- [ ] **Step 6: Document it**

In `doc/api-frontend.md`, document `POST /api/faq/extract`: admin-only; body `{ text, sourceName? }`; returns `ExtractionResult`. Say plainly that candidates land as `PENDING_REVIEW` with a `source_span`, and that **candidates whose answer isn't supported by their span are dropped rather than surfaced** — with the reason, because a UI that doesn't explain the gap between "8 candidates found" and "5 kept" will read as a bug.

- [ ] **Step 7: Commit**

```bash
git add src/faq/faq.controller.ts src/faq/faq.module.ts src/faq/faq.controller.spec.ts doc/api-frontend.md
git commit -m "feat(faq): expose the document extraction endpoint"
```

---

## Spec Coverage Check (self-review)

| PRD 2 §4.3 requirement | Covered by |
|---|---|
| document → text → chunking → LLM extraction → candidates → review queue | Tasks 3, 4 (phase 1's queue is the destination) |
| Every candidate carries a verbatim `source_span` | Task 3 (missing span → dropped; non-verbatim span → dropped) |
| Reviewer sees the span beside the candidate | Already shipped — phase 1's `QUEUE_SELECT` returns `source_span` |
| Unsupported candidates dropped, not flagged | Tasks 1, 3 |
| No prices/currency in generated answers | Extraction prompt + phase 1's lint as the backstop |
| Answers under 600 chars | Extraction prompt + phase 1's lint warning at the retrieval cap |
| Rioplatense Spanish | Extraction prompt (no automated check — same reasoning as phase 1) |
| No imperative language aimed at the bot | Extraction prompt + phase 1's lint warning |
| ~60 candidate cap per document | Task 3 (`MAX_CANDIDATES`, enforced at both the fetch loop and the keep loop) |
| Cost metered separately (§7) | Task 3 (`recordUsage` with `kind: 'faq_extract'`) |
| Mid-tier model, not cheapest (§7) | Task 3 (`FAQ_EXTRACTION_MODEL`, defaulting to a mid-tier model) |
| PDF/DOCX parsing | **Deferred** — see Global Constraints |
| LLM-based support scorer | **Deferred** — heuristic ships first, §4.3 permits either |
| Document retention policy | **Deferred** — §12 open question, a data-handling decision |

## Execution

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks.
2. **Inline Execution** — batch through with checkpoints.
