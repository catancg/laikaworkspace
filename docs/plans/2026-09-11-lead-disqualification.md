# Lead Disqualification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close leads that will not become commercial opportunities automatically, on a
business-configurable timer, recording which stage they were lost from and leaving them able to come
back.

**Architecture:** The rule lives on the `FunnelStage` row the lead is *standing in* (`silenceDays` +
`silenceTargetId`), not on the destination. A periodic BullMQ sweep evaluates it per tenant; the
decision itself is a pure function so it can be tested with adversarial inputs. Reactivation rides
on PRD 15's shipped re-entry mechanism, plus one nullable per-source override so spam does not come
back as "interested".

**Tech Stack:** NestJS, Prisma (multi-tenant: one Postgres database per tenant), BullMQ on Redis,
Jest. Frontend is Next.js App Router.

**Spec:** [docs/prd-lead-disqualification.md](../prd-lead-disqualification.md) (PRD 16)

## Global Constraints

- **Run every command from `soylaika.backend/`**, not the workspace root. Frontend commands run from
  `soylaika.frontend/`.
- **Never run `npm run build`** while `start-dev.ps1` is running — it competes with `nest start
  --watch` over `dist/` and takes the running backend down.
- **`npm run lint` in the backend is `eslint --fix` and rewrites the whole tree.** Use
  `npx eslint <files>` on the files you touched. The frontend's `npm run lint` starts from a
  non-zero baseline (`react-hooks/set-state-in-effect`) — record the count before you change
  anything and do not grow it.
- **`npx tsc --noEmit -p tsconfig.json` proves almost nothing here.** Every `db` handle is typed
  `any`, so a wrong Prisma query shape type-checks clean. Behaviour must be pinned by a test that
  actually runs.
- **Stage explicit paths when committing.** Never `git add -A`, `git add .`, or `git stash` — both
  code repos carry uncommitted local edits to their own `CLAUDE.md` that are not yours to move.
- **Do not run migrations against an existing tenant or production database.** Create a throwaway
  database, use it, drop it.
- **Comments and commit messages in Spanish**, matching the surrounding file. Chat/PR prose to the
  user is English.
- **Migration directory name is both the ordering and the identity.** It must sort after
  `20260916120000_funnel_reentry`, and renaming it later makes it run again on every tenant.
- Backend tests: `npm test -- <path>`. Single test: `npm test -- <path> -t "<name>"`.

---

## File Structure

**New files**

| File | Responsibility |
| --- | --- |
| `src/queue/business-days.ts` | Pure: how much *business* time has elapsed. Distinct from `business-hours.ts`, which answers "when may the bot send". |
| `src/queue/business-days.spec.ts` | Adversarial tests for the above. |
| `src/queue/silence-sweep.ts` | Pure: given a stage's rule and a contact's silence, what should happen. |
| `src/queue/silence-sweep.spec.ts` | Adversarial tests for the above. |
| `src/queue/silence.processor.ts` | The sweep: walks tenants, runs the queries, applies the decision. |
| `src/funnel/funnel-silence.spec.ts` | Tests for the new validators and the seed invariants. |
| `prisma/migrations/20260917120000_lead_disqualification/migration.sql` | Columns, index, seed rows. |

**Modified files**

| File | Change |
| --- | --- |
| `prisma/schema.prisma` | 3 nullable columns on `FunnelStage`, 1 index on `Message`, corrected comment on `Contact.previousStageId`. |
| `src/funnel/funnel.service.ts` | Export `DEFAULT_STAGES`; add two stage rows; accept the new fields in `updateAdmin`. |
| `src/funnel/funnel-criteria.ts` | `MAX_OUT_ENABLED` 3→4; add `checkSilenceRule`, `checkReentryOverride`, `resolveReentryTarget`. |
| `src/ai/stage-prompt.ts` | `.find` → `.filter` for the "markable from any stage" exception. |
| `src/ai/ai.service.ts:583-586` | `previousStageId` gate: `isLost` → `kind === 'out'`. |
| `src/crm/crm.service.ts:408-410` | Same gate, human path. |
| `src/queue/message.processor.ts:93-96` | Re-entry destination honours the per-source override. |
| `src/queue/followup.processor.ts` | `no-answer-24h` stops moving the stage. |
| `src/queue/queue.constants.ts` | Drop `'no-answer-24h'` from `FollowupJobData['type']`. |
| `src/queue/queue.module.ts` | Register `SILENCE_QUEUE` and `SilenceProcessor`. |
| `src/crm/crm.service.ts` (`testFollowup`) | Demo chat follows the same removal. |
| `soylaika.frontend/lib/stage-style.ts` | Two new slugs. |
| `soylaika.frontend/app/(crm)/contacts/page.tsx` | Two new slugs. |

---

## Task 1: Business-days module

Pure, no dependencies, nothing else needs it yet. It exists first because every later timing
decision is expressed in its terms.

**Files:**
- Create: `src/queue/business-days.ts`
- Test: `src/queue/business-days.spec.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `businessDaysBefore(fromMs: number, days: number): number` — the epoch-ms instant that is
  `days` business days before `fromMs`, counting only Mon–Fri **in Argentine local time**. Used as a
  cutoff: a contact whose last inbound message is at or before it has been silent that long.

- [ ] **Step 1: Write the failing test**

Create `src/queue/business-days.spec.ts`:

```ts
import { businessDaysBefore } from './business-days';

// Fechas en hora ARGENTINA (UTC-3). 2026-09-16 es MIERCOLES.
// Se construyen en UTC y se les suma el offset para que la "hora de pared"
// argentina sea la que dice el nombre de la constante.
const ART = (iso: string) => new Date(`${iso}-03:00`).getTime();

describe('businessDaysBefore', () => {
  it('resta dias corridos cuando no hay fin de semana en el medio', () => {
    // miercoles 16 → lunes 14
    expect(businessDaysBefore(ART('2026-09-16T12:00:00'), 2)).toBe(ART('2026-09-14T12:00:00'));
  });

  it('salta el fin de semana', () => {
    // martes 15 menos 2 habiles: lunes 14 (1), viernes 11 (2). Sabado y domingo no cuentan.
    expect(businessDaysBefore(ART('2026-09-15T12:00:00'), 2)).toBe(ART('2026-09-11T12:00:00'));
  });

  it('cinco dias habiles cruzan exactamente un fin de semana', () => {
    // viernes 18 menos 5 habiles → viernes 11
    expect(businessDaysBefore(ART('2026-09-18T10:00:00'), 5)).toBe(ART('2026-09-11T10:00:00'));
  });

  it('diez dias habiles cruzan dos fines de semana', () => {
    // viernes 25 menos 10 habiles → viernes 11
    expect(businessDaysBefore(ART('2026-09-25T10:00:00'), 10)).toBe(ART('2026-09-11T10:00:00'));
  });

  it('partiendo de un sabado, el primer dia habil hacia atras es el viernes', () => {
    // sabado 19 menos 1 habil → viernes 18
    expect(businessDaysBefore(ART('2026-09-19T09:00:00'), 1)).toBe(ART('2026-09-18T09:00:00'));
  });

  it('partiendo de un domingo, el primer dia habil hacia atras es el viernes', () => {
    expect(businessDaysBefore(ART('2026-09-20T09:00:00'), 1)).toBe(ART('2026-09-18T09:00:00'));
  });

  it('cero dias devuelve el mismo instante', () => {
    expect(businessDaysBefore(ART('2026-09-16T12:00:00'), 0)).toBe(ART('2026-09-16T12:00:00'));
  });

  it('conserva la hora del dia', () => {
    const r = businessDaysBefore(ART('2026-09-16T23:45:00'), 1);
    expect(r).toBe(ART('2026-09-15T23:45:00'));
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/queue/business-days.spec.ts`
Expected: FAIL — `Cannot find module './business-days'`.

- [ ] **Step 3: Write the implementation**

Create `src/queue/business-days.ts`:

```ts
//
// PRD 16 §4.2 — cuanto tiempo HABIL paso, que no es la pregunta que contesta
// business-hours.ts.
//
// `nextBusinessTime` contesta "cuando puede mandar el bot": corre un instante
// hacia adelante hasta caer en la franja 9–20 ART, y NO mira el dia de la
// semana, porque un follow-up un sabado a la tarde esta bien.
//
// Esto es lo otro: contar dias habiles hacia atras para decidir si un lead se
// quedo callado el tiempo suficiente. Ahi el fin de semana SI importa — cinco
// dias corridos desde un miercoles caen en lunes, y el cliente tuvo dos dias
// menos de los que el negocio cree para contestar.
//
// FERIADOS NO (§4.2, §10). Un feriado hace que el bot espere un poco menos que
// los dias habiles configurados. Esta dicho en el PRD para que nadie asuma que
// se contemplo.
//

const ART_OFFSET_MS = 3 * 60 * 60 * 1000; // Argentina = UTC-3, sin DST desde 2009
const DAY_MS = 24 * 60 * 60 * 1000;

// 0 = domingo, 6 = sabado, en hora de pared argentina.
function artWeekday(ms: number): number {
  return new Date(ms - ART_OFFSET_MS).getUTCDay();
}

function isWeekend(ms: number): boolean {
  const d = artWeekday(ms);
  return d === 0 || d === 6;
}

/**
 * El instante que esta `days` dias HABILES antes de `fromMs`, conservando la
 * hora del dia. Sabados y domingos no se cuentan: se saltean.
 *
 * Se usa como corte — un contacto cuyo ultimo mensaje entrante es <= a esto
 * estuvo callado esa cantidad de dias habiles.
 *
 * Se resta de a un dia en vez de calcular semanas enteras a proposito: la
 * version cerrada se equivoca en los bordes (empezar un sabado, o justo sobre
 * el cambio de semana) y esto se corre pocas veces por barrido.
 */
export function businessDaysBefore(fromMs: number, days: number): number {
  let cursor = fromMs;
  let restantes = days;

  while (restantes > 0) {
    cursor -= DAY_MS;
    if (!isWeekend(cursor)) restantes -= 1;
  }

  return cursor;
}
```

- [ ] **Step 4: Run the test and confirm it passes**

Run: `npm test -- src/queue/business-days.spec.ts`
Expected: PASS, 8 tests.

- [ ] **Step 5: Lint only the files you touched**

Run: `npx eslint src/queue/business-days.ts src/queue/business-days.spec.ts`
Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add src/queue/business-days.ts src/queue/business-days.spec.ts
git commit -m "feat(queue): contar dias habiles hacia atras, que no es lo que hace business-hours"
```

---

## Task 2: Schema, migration and the two new stages

**Files:**
- Modify: `prisma/schema.prisma`
- Modify: `src/funnel/funnel.service.ts`
- Create: `prisma/migrations/20260917120000_lead_disqualification/migration.sql`
- Create: `src/funnel/funnel-silence.spec.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `DEFAULT_STAGES` exported from `src/funnel/funnel.service.ts` as
  `readonly { name: string; slug: string; kind: string; isLost: boolean; isWon: boolean;
  reentersOnReply?: boolean; criteria: string; order: number; color: string;
  notifyOnEnter: boolean }[]`. Three new nullable `FunnelStage` columns: `silenceDays Int?`,
  `silenceTargetId String?`, `reentryTargetId String?`.

- [ ] **Step 1: Write the failing test**

Create `src/funnel/funnel-silence.spec.ts`:

```ts
import { DEFAULT_STAGES } from './funnel.service';

// PRD 16 §2.2 y §6 — dos invariantes de la siembra que el codigo asume en otro
// lado y que fallan EN SILENCIO si se rompen.
describe('DEFAULT_STAGES', () => {
  const bySlug = (s: string) => DEFAULT_STAGES.find((x) => x.slug === s);

  it('siembra las cuatro etapas fuera del flujo', () => {
    const out = DEFAULT_STAGES.filter((s) => s.kind === 'out').map((s) => s.slug).sort();
    expect(out).toEqual(['no-calificado', 'no-contesta', 'no-interesado', 'perdido']);
  });

  it('SOLO perdido tiene isLost', () => {
    // §2.2: stage-prompt hace find(isLost) y asume que hay uno solo.
    expect(DEFAULT_STAGES.filter((s) => s.isLost).map((s) => s.slug)).toEqual(['perdido']);
  });

  it('todas las etapas fuera del flujo reingresan', () => {
    // §6: PRD 15 cambio la condicion kind==='out' por una flag explicita, y una
    // etapa nueva nace en false. El lead quedaria trabado afuera sin error.
    const sinReingreso = DEFAULT_STAGES
      .filter((s) => s.kind === 'out' && !s.reentersOnReply)
      .map((s) => s.slug);
    expect(sinReingreso).toEqual([]);
  });

  it('la criteria de perdido ya no habla de falta de interes', () => {
    // §2.1: "no me interesa" / "no gracias" ahora son no-interesado.
    expect(bySlug('perdido')!.criteria).not.toMatch(/no me interesa|no gracias/i);
  });

  it('no-interesado describe la falta de interes durante el funnel', () => {
    expect(bySlug('no-interesado')!.criteria).toMatch(/no me interesa|no gracias|lo dejo/i);
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/funnel/funnel-silence.spec.ts`
Expected: FAIL — `DEFAULT_STAGES` is not exported.

- [ ] **Step 3: Add the schema columns**

In `prisma/schema.prisma`, inside `model FunnelStage`, after the `enabled` line:

```prisma
  // PRD 16 §3 — la regla de tiempo vive en la etapa donde el lead ESTA PARADO,
  // no en la etapa destino. NULL = esta etapa no tiene regla.
  silenceDays     Int?
  // A donde se lo manda al vencer. NULL con silenceDays seteado = solo avisa (§4.3).
  silenceTargetId String?
  // PRD 16 §6.3 — override del destino de reingreso de PRD 15, por etapa de
  // ORIGEN. NULL = usa el destino global (`isReentryTarget`), que es lo normal.
  reentryTargetId String?
```

In `model Contact`, correct the stale comment on `previousStageId`:

```prisma
  previousStageId     String?      // etapa en la que estaba justo antes de salir del flujo (PRD 16 §5)
```

In `model Message`, add to the index block:

```prisma
  @@index([contactId, role, createdAt])
```

- [ ] **Step 4: Add the two stages and export the list**

In `src/funnel/funnel.service.ts`, change `const DEFAULT_STAGES = [` to
`export const DEFAULT_STAGES = [`.

Replace the `perdido` row and add the two new rows so the `out` block reads:

```ts
  { name: 'No contesta', reentersOnReply: true, slug: 'no-contesta', color: '#a855f7', order: 6, notifyOnEnter: false, isWon: false, isLost: false, kind: 'out',
    criteria: 'dejo de responder; la maneja el sistema por tiempo, casi nunca la marques vos.' },
  // PRD 16 §2: no-calificado junta dos cosas distintas — el que mando un solo
  // mensaje y desaparecio (lo decide el tiempo) y el spam/promo/consulta ajena
  // al negocio (lo decide el clasificador, en el primer mensaje).
  { name: 'No calificado', reentersOnReply: true, slug: 'no-calificado', color: '#78716c', order: 8, notifyOnEnter: false, isWon: false, isLost: false, kind: 'out',
    criteria: 'spam, promociones, mensajes mandados por error, o consultas que no tienen nada que ver con lo que vende el negocio.' },
  // PRD 16 §2.1: esto ANTES estaba en perdido. Es la falta de interes DURANTE el
  // funnel, antes de que el cliente se comprometa.
  { name: 'No interesado', reentersOnReply: true, slug: 'no-interesado', color: '#f97316', order: 9, notifyOnEnter: false, isWon: false, isLost: false, kind: 'out',
    criteria: 'dice que no quiere avanzar: no me interesa, lo dejo, esta muy caro, no gracias, compre en otro lado.' },
  // PRD 16 §2.1: perdido se angosta a la perdida de FINAL de embudo — el cliente
  // queria comprar y se cayo por algo nuestro (no llegamos con el envio, el
  // producto no daba). La falta de interes temprana ya no entra aca.
  { name: 'Perdido',     reentersOnReply: true, slug: 'perdido',     color: '#ef4444', order: 7, notifyOnEnter: false, isWon: false, isLost: true,  kind: 'out',
    criteria: 'el cliente queria comprar y la venta se cayo por una razon del negocio: no llegamos con la entrega, el producto no servia para lo que necesitaba, no habia stock.' },
```

- [ ] **Step 5: Run the test and confirm it passes**

Run: `npm test -- src/funnel/funnel-silence.spec.ts`
Expected: PASS, 5 tests.

- [ ] **Step 6: Write the migration**

Create `prisma/migrations/20260917120000_lead_disqualification/migration.sql`:

```sql
-- PRD 16: la descalificacion automatica pasa a ser dato.
--
-- Tres columnas nullable, un indice, y dos etapas nuevas. Lo unico que se
-- reescribe de lo que ya existia es la criteria de `perdido` (§2.1), que es el
-- unico cambio de este PRD que altera el comportamiento del bot el dia del
-- deploy.
ALTER TABLE "FunnelStage" ADD COLUMN IF NOT EXISTS "silenceDays"     INTEGER;
ALTER TABLE "FunnelStage" ADD COLUMN IF NOT EXISTS "silenceTargetId" TEXT;
ALTER TABLE "FunnelStage" ADD COLUMN IF NOT EXISTS "reentryTargetId" TEXT;

-- §4.1: el barrido pregunta "este contacto tiene algun mensaje ENTRANTE despues
-- del corte". Sin este indice eso es un scan de la tabla mas grande del tenant.
CREATE INDEX IF NOT EXISTS "Message_contactId_role_createdAt_idx"
  ON "Message" ("contactId", "role", "createdAt");

-- Las dos etapas nuevas. ON CONFLICT DO NOTHING porque la siembra del codigo
-- (seedDefaults, upsert con update:{}) puede haber corrido antes en un tenant
-- nuevo.
INSERT INTO "FunnelStage" ("id", "name", "slug", "color", "order", "notifyOnEnter", "isWon", "isLost", "kind", "enabled", "reentersOnReply", "isReentryTarget", "criteria", "criteriaDefault")
VALUES
  (gen_random_uuid(), 'No calificado', 'no-calificado', '#78716c', 8, false, false, false, 'out', true, true, false,
   'spam, promociones, mensajes mandados por error, o consultas que no tienen nada que ver con lo que vende el negocio.',
   'spam, promociones, mensajes mandados por error, o consultas que no tienen nada que ver con lo que vende el negocio.'),
  (gen_random_uuid(), 'No interesado', 'no-interesado', '#f97316', 9, false, false, false, 'out', true, true, false,
   'dice que no quiere avanzar: no me interesa, lo dejo, esta muy caro, no gracias, compre en otro lado.',
   'dice que no quiere avanzar: no me interesa, lo dejo, esta muy caro, no gracias, compre en otro lado.')
ON CONFLICT ("slug") DO NOTHING;

-- §2.1: perdido se angosta. Solo si el tenant NO la edito a mano — se compara
-- contra criteriaDefault, que es la copia de lo sembrado.
UPDATE "FunnelStage" SET
  "criteria"        = 'el cliente queria comprar y la venta se cayo por una razon del negocio: no llegamos con la entrega, el producto no servia para lo que necesitaba, no habia stock.',
  "criteriaDefault" = 'el cliente queria comprar y la venta se cayo por una razon del negocio: no llegamos con la entrega, el producto no servia para lo que necesitaba, no habia stock.'
  WHERE "slug" = 'perdido' AND "criteria" IS NOT DISTINCT FROM "criteriaDefault";

-- §3: la siembra de los plazos. El guard es NOT EXISTS sobre CUALQUIER regla ya
-- configurada, no `IS NULL` por fila.
--
-- Es la leccion de la migracion de PRD 15: `= false` / `IS NULL` por fila no
-- distingue "nunca se configuro" de "el tenant lo apago a proposito", y
-- corriendo la migracion dos veces con configuracion en el medio se repone lo
-- sembrado encima de la decision del tenant. Con pre-aplicado a mano + corrida
-- automatica al bootear, eso pasa siempre.
UPDATE "FunnelStage" s SET
  "silenceDays" = v.dias,
  "silenceTargetId" = (SELECT t."id" FROM "FunnelStage" t WHERE t."slug" = v.destino)
FROM (VALUES
  ('nuevo',      5,  'no-calificado'),
  ('interesado', 5,  'no-contesta'),
  ('cerrando',   5,  'no-contesta'),
  ('cotizado',   10, NULL)
) AS v(slug, dias, destino)
WHERE s."slug" = v.slug
  AND NOT EXISTS (SELECT 1 FROM "FunnelStage" WHERE "silenceDays" IS NOT NULL);

-- §6.3: el unico override. El resto cae al destino global de PRD 15.
UPDATE "FunnelStage" SET
  "reentryTargetId" = (SELECT "id" FROM "FunnelStage" WHERE "slug" = 'nuevo')
  WHERE "slug" = 'no-calificado'
    AND NOT EXISTS (SELECT 1 FROM "FunnelStage" WHERE "reentryTargetId" IS NOT NULL);
```

- [ ] **Step 7: Apply the migration to a throwaway database and verify it**

Never point this at an existing tenant or at production.

```bash
createdb laika_prd16_scratch
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/laika_prd16_scratch" npx prisma migrate deploy
```

Then confirm the seeds landed, and confirm the migration is idempotent by running the seed block a
second time and checking nothing changed:

```bash
psql laika_prd16_scratch -c 'SELECT slug, "silenceDays", "silenceTargetId" IS NOT NULL AS tiene_destino, "reentryTargetId" IS NOT NULL AS tiene_override FROM "FunnelStage" ORDER BY "order";'
psql laika_prd16_scratch -f prisma/migrations/20260917120000_lead_disqualification/migration.sql
psql laika_prd16_scratch -c 'SELECT count(*) FROM "FunnelStage";'
```

Expected: 9 stages; `nuevo`/`interesado`/`cerrando` with `silenceDays = 5` and a target;
`cotizado` with `10` and no target; only `no-calificado` with an override. Count is still 9 after
the second run.

```bash
dropdb laika_prd16_scratch
```

- [ ] **Step 8: Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260917120000_lead_disqualification/migration.sql src/funnel/funnel.service.ts src/funnel/funnel-silence.spec.ts
git commit -m "feat(funnel): cuatro etapas fuera del flujo y el plazo de silencio como dato"
```

---

## Task 3: Caps and the two new validators

**Files:**
- Modify: `src/funnel/funnel-criteria.ts`
- Modify: `src/funnel/funnel-silence.spec.ts`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `MAX_OUT_ENABLED = 4`
  - `type SilenceShape = { id: string; slug: string; kind: string; enabled: boolean; silenceDays?: number | null; silenceTargetId?: string | null }`
  - `checkSilenceRule(row: SilenceShape, all: SilenceShape[]): string | null`
  - `checkReentryOverride(row: { id: string; reentryTargetId?: string | null }, all: SilenceShape[]): string | null`

- [ ] **Step 1: Write the failing tests**

Append to `src/funnel/funnel-silence.spec.ts`:

```ts
import {
  MAX_OUT_ENABLED,
  checkCaps,
  checkSilenceRule,
  checkReentryOverride,
  type SilenceShape,
} from './funnel-criteria';

const etapa = (over: Partial<SilenceShape> = {}): SilenceShape => ({
  id: 'id-1',
  slug: 'nuevo',
  kind: 'pipeline',
  enabled: true,
  silenceDays: null,
  silenceTargetId: null,
  ...over,
});

describe('MAX_OUT_ENABLED', () => {
  it('permite las cuatro etapas fuera del flujo de PRD 16', () => {
    expect(MAX_OUT_ENABLED).toBe(4);
    const cuatro = Array.from({ length: 4 }, () => ({ kind: 'out', enabled: true }));
    expect(checkCaps([...cuatro, { kind: 'pipeline', enabled: true }])).toBeNull();
  });

  it('cinco siguen siendo demasiadas', () => {
    const cinco = Array.from({ length: 5 }, () => ({ kind: 'out', enabled: true }));
    expect(checkCaps([...cinco, { kind: 'pipeline', enabled: true }])).toMatch(/fuera del flujo/i);
  });
});

describe('checkSilenceRule', () => {
  const destino = etapa({ id: 'out-1', slug: 'no-contesta', kind: 'out' });

  it('acepta una regla completa que apunta a una etapa fuera del flujo', () => {
    const row = etapa({ silenceDays: 5, silenceTargetId: 'out-1' });
    expect(checkSilenceRule(row, [row, destino])).toBeNull();
  });

  it('acepta silenceDays sin destino: es el caso "solo avisa"', () => {
    const row = etapa({ slug: 'cotizado', silenceDays: 10, silenceTargetId: null });
    expect(checkSilenceRule(row, [row, destino])).toBeNull();
  });

  it('rechaza un destino sin plazo, que se lee como regla y no hace nada', () => {
    const row = etapa({ silenceDays: null, silenceTargetId: 'out-1' });
    expect(checkSilenceRule(row, [row, destino])).toMatch(/sin plazo/i);
  });

  it('rechaza cero dias', () => {
    const row = etapa({ silenceDays: 0, silenceTargetId: 'out-1' });
    expect(checkSilenceRule(row, [row, destino])).toMatch(/al menos 1/i);
  });

  it('rechaza un destino que es una etapa del flujo comercial', () => {
    const pipeline = etapa({ id: 'pipe-2', slug: 'interesado', kind: 'pipeline' });
    const row = etapa({ silenceDays: 5, silenceTargetId: 'pipe-2' });
    expect(checkSilenceRule(row, [row, pipeline])).toMatch(/fuera del flujo/i);
  });

  it('rechaza un destino deshabilitado', () => {
    const apagada = etapa({ id: 'out-1', slug: 'no-contesta', kind: 'out', enabled: false });
    const row = etapa({ silenceDays: 5, silenceTargetId: 'out-1' });
    expect(checkSilenceRule(row, [row, apagada])).toMatch(/desactivada/i);
  });

  it('rechaza apuntarse a si misma', () => {
    const row = etapa({ id: 'x', kind: 'out', silenceDays: 5, silenceTargetId: 'x' });
    expect(checkSilenceRule(row, [row])).toMatch(/a si misma|sí misma/i);
  });

  it('rechaza un destino inexistente', () => {
    const row = etapa({ silenceDays: 5, silenceTargetId: 'no-existe' });
    expect(checkSilenceRule(row, [row])).toMatch(/no existe/i);
  });
});

describe('checkReentryOverride', () => {
  const pipeline = etapa({ id: 'pipe-2', slug: 'nuevo', kind: 'pipeline' });

  it('NULL es valido: cae al destino global de PRD 15', () => {
    expect(checkReentryOverride({ id: 'a', reentryTargetId: null }, [pipeline])).toBeNull();
  });

  it('acepta un override que apunta al flujo comercial', () => {
    expect(checkReentryOverride({ id: 'a', reentryTargetId: 'pipe-2' }, [pipeline])).toBeNull();
  });

  it('rechaza reingresar a otra etapa fuera del flujo, que seria un loop', () => {
    const out = etapa({ id: 'out-9', kind: 'out' });
    expect(checkReentryOverride({ id: 'a', reentryTargetId: 'out-9' }, [out]))
      .toMatch(/flujo comercial/i);
  });

  it('rechaza un destino deshabilitado', () => {
    const apagada = etapa({ id: 'pipe-2', kind: 'pipeline', enabled: false });
    expect(checkReentryOverride({ id: 'a', reentryTargetId: 'pipe-2' }, [apagada]))
      .toMatch(/desactivada/i);
  });

  it('rechaza apuntarse a si misma', () => {
    expect(checkReentryOverride({ id: 'a', reentryTargetId: 'a' }, [etapa({ id: 'a' })]))
      .toMatch(/a si misma|sí misma/i);
  });
});
```

- [ ] **Step 2: Run the tests and confirm they fail**

Run: `npm test -- src/funnel/funnel-silence.spec.ts`
Expected: FAIL — `checkSilenceRule` is not exported, and `MAX_OUT_ENABLED` is 3.

- [ ] **Step 3: Write the implementation**

In `src/funnel/funnel-criteria.ts`, change the cap and add its reason:

```ts
// PRD 16 §2.3: sube a 4 para que entren no-calificado y no-interesado junto a
// no-contesta y perdido. El tope es control de COSTO — la criteria de cada
// etapa activa viaja en el prompt del clasificador en CADA mensaje — asi que
// esto es gasto por conversacion que el negocio elige pagar.
export const MAX_OUT_ENABLED = 4;
```

Append at the end of the file:

```ts
// PRD 16 §3.1 — la forma que necesitan los chequeos de las reglas de silencio.
export type SilenceShape = {
  id: string;
  slug: string;
  kind: string;
  enabled: boolean;
  silenceDays?: number | null;
  silenceTargetId?: string | null;
};

/**
 * PRD 16 §3.1 — valida la regla de tiempo de UNA etapa contra el funnel entero.
 *
 * `silenceDays` sin destino es LEGAL: es el caso de `cotizado`, que avisa a un
 * humano en vez de descalificar (§4.3). Lo que no es legal es lo inverso — un
 * destino sin plazo se lee como una regla configurada y no hace nada nunca.
 */
export function checkSilenceRule(row: SilenceShape, all: SilenceShape[]): string | null {
  const dias = row.silenceDays ?? null;
  const destinoId = row.silenceTargetId ?? null;

  if (dias === null && destinoId === null) return null;

  if (dias === null && destinoId !== null) {
    return 'La etapa tiene destino de silencio pero no tiene plazo: así no se aplica nunca';
  }
  if (dias !== null && dias < 1) {
    return 'El plazo de silencio tiene que ser de al menos 1 día hábil';
  }
  if (destinoId === null) return null;

  if (destinoId === row.id) return 'Una etapa no puede mandarse a sí misma por silencio';

  const destino = all.find((s) => s.id === destinoId);
  if (!destino) return 'El destino de silencio no existe';
  if (!destino.enabled) return `La etapa destino "${destino.slug}" está desactivada`;
  if (destino.kind !== 'out') {
    return `El destino de silencio tiene que ser una etapa fuera del flujo; "${destino.slug}" no lo es`;
  }
  return null;
}

/**
 * PRD 16 §6.3 — el override del destino de reingreso.
 *
 * NULL es el caso normal y no se valida: significa "usá el destino global de
 * PRD 15". Solo se valida cuando alguien lo setea.
 *
 * Tiene que ser `pipeline`: reingresar a otra etapa fuera del flujo es un loop
 * — el lead escribe, "vuelve" a otra etapa out, y sigue fuera del embudo.
 */
export function checkReentryOverride(
  row: { id: string; reentryTargetId?: string | null },
  all: SilenceShape[],
): string | null {
  const destinoId = row.reentryTargetId ?? null;
  if (destinoId === null) return null;

  if (destinoId === row.id) return 'Una etapa no puede reingresar a sí misma';

  const destino = all.find((s) => s.id === destinoId);
  if (!destino) return 'El destino de reingreso no existe';
  if (!destino.enabled) return `La etapa destino "${destino.slug}" está desactivada`;
  if (destino.kind !== 'pipeline') {
    return `El reingreso tiene que ser a una etapa del flujo comercial; "${destino.slug}" no lo es`;
  }
  return null;
}
```

- [ ] **Step 4: Run the tests and confirm they pass**

Run: `npm test -- src/funnel/funnel-silence.spec.ts src/funnel/funnel-criteria.spec.ts`
Expected: PASS. `funnel-criteria.spec.ts` must still pass — if a test there asserted the old cap of
3, update it to 4 and leave a comment naming PRD 16 §2.3.

- [ ] **Step 5: Wire the validators into `updateAdmin`**

In `src/funnel/funnel.service.ts`, add the fields to the `updateAdmin` signature after
`isReentryTarget?: boolean;`:

```ts
      silenceDays?: number | null;
      silenceTargetId?: string | null;
      reentryTargetId?: string | null;
```

And before the `isReentryTarget === true` transaction block, after the `criteria` handling:

```ts
    // PRD 16 §3.1 / §6.3: se valida contra el funnel COMPLETO, porque las tres
    // columnas apuntan a otras filas.
    if (
      data.silenceDays !== undefined ||
      data.silenceTargetId !== undefined ||
      data.reentryTargetId !== undefined
    ) {
      const todas = await db.funnelStage.findMany();
      const propuesta = {
        ...stage,
        ...(data.silenceDays !== undefined ? { silenceDays: data.silenceDays } : {}),
        ...(data.silenceTargetId !== undefined ? { silenceTargetId: data.silenceTargetId } : {}),
        ...(data.reentryTargetId !== undefined ? { reentryTargetId: data.reentryTargetId } : {}),
      };
      const error =
        checkSilenceRule(propuesta, todas.map((s: any) => (s.id === stage.id ? propuesta : s))) ??
        checkReentryOverride(propuesta, todas);
      if (error) throw new BadRequestException(error);

      if (data.silenceDays !== undefined) patch.silenceDays = data.silenceDays;
      if (data.silenceTargetId !== undefined) patch.silenceTargetId = data.silenceTargetId;
      if (data.reentryTargetId !== undefined) patch.reentryTargetId = data.reentryTargetId;
    }
```

Add `checkSilenceRule, checkReentryOverride` to the existing import from `./funnel-criteria`.

- [ ] **Step 6: Type-check and lint**

Run: `npx tsc --noEmit -p tsconfig.json`
Run: `npx eslint src/funnel/funnel-criteria.ts src/funnel/funnel.service.ts src/funnel/funnel-silence.spec.ts`
Expected: clean.

- [ ] **Step 7: Commit**

```bash
git add src/funnel/funnel-criteria.ts src/funnel/funnel.service.ts src/funnel/funnel-silence.spec.ts src/funnel/funnel-criteria.spec.ts
git commit -m "feat(funnel): validar las reglas de silencio y el override de reingreso"
```

---

## Task 4: The classifier prompt names every out-stage

**Files:**
- Modify: `src/ai/stage-prompt.ts:78-83`
- Modify: `src/ai/stage-prompt.spec.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: no new exports. `buildStagePrompt` emits one `EXCEPCION:` line naming every enabled
  `kind === 'out'` stage instead of only the single `isLost` one.

- [ ] **Step 1: Write the failing test**

Append to `src/ai/stage-prompt.spec.ts` (reuse the `row` helper already in that file):

```ts
describe('PRD 16 §2.2 — la excepcion nombra TODAS las etapas fuera del flujo', () => {
  const conOut = [
    row({ slug: 'nuevo', name: 'Nuevo', order: 1, criteria: 'saluda.' }),
    row({ slug: 'perdido', name: 'Perdido', order: 7, criteria: 'se cayo.', kind: 'out', isLost: true }),
    row({ slug: 'no-interesado', name: 'No interesado', order: 9, criteria: 'no quiere.', kind: 'out' }),
  ];

  it('nombra perdido Y no-interesado', () => {
    const out = buildStagePrompt(conOut);
    expect(out).toMatch(/EXCEPCION:.*"perdido"/);
    expect(out).toMatch(/EXCEPCION:.*"no-interesado"/);
  });

  it('no nombra una etapa out desactivada', () => {
    const apagada = conOut.map((s) => (s.slug === 'no-interesado' ? { ...s, enabled: false } : s));
    expect(buildStagePrompt(apagada)).not.toMatch(/EXCEPCION:.*no-interesado/);
  });

  it('sin etapas out no emite la excepcion', () => {
    const sinOut = [row({ slug: 'nuevo', name: 'Nuevo', order: 1, criteria: 'saluda.' })];
    expect(buildStagePrompt(sinOut)).not.toContain('EXCEPCION:');
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/ai/stage-prompt.spec.ts`
Expected: FAIL — the `no-interesado` assertion, because only the `isLost` stage is named.

- [ ] **Step 3: Write the implementation**

In `src/ai/stage-prompt.ts`, replace the `const lost = ...` block:

```ts
  // §6.2 original: la etapa salia del dato (`isLost`), no de un literal. PRD 16
  // §2.2 corrige la otra mitad — era `find`, y ahora hay CUATRO etapas fuera del
  // flujo. Con `find` las otras tres perdian la excepcion en silencio.
  //
  // Se nombran todas las `out` activas, incluida `no-contesta`: su propia
  // criteria ya dice "la maneja el sistema por tiempo, casi nunca la marques
  // vos", asi que el texto la cubre sin una regla aparte en el codigo.
  const fueraDelFlujo = enabled.filter((s) => s.kind === 'out');
  if (fueraDelFlujo.length) {
    const nombres = fueraDelFlujo.map((s) => `"${s.slug}"`).join(', ');
    parts.push(
      `EXCEPCION: ${nombres} se pueden marcar desde CUALQUIER etapa apenas el cliente lo deja claro.`,
    );
  }
```

- [ ] **Step 4: Run the tests and confirm they pass**

Run: `npm test -- src/ai/stage-prompt.spec.ts`
Expected: PASS. An existing test asserting the exact singular sentence for `perdido` will need its
expectation widened to the new plural form — update it, do not delete it.

- [ ] **Step 5: Commit**

```bash
git add src/ai/stage-prompt.ts src/ai/stage-prompt.spec.ts
git commit -m "fix(ai): la excepcion de etapas fuera del flujo era un find y ahora hay cuatro"
```

---

## Task 5: `previousStageId` fires for every out-stage

This is the dimension the whole requirement was built for, and it has been silently missing since
the 24-hour rule existed.

**Files:**
- Modify: `src/ai/ai.service.ts:583-586`
- Modify: `src/crm/crm.service.ts:408-410`
- Modify: `src/crm/crm.service.update-contact.spec.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: no new exports. Behaviour change only.

- [ ] **Step 1: Write the failing test**

First widen the file's stage fixture type. At the top of
`src/crm/crm.service.update-contact.spec.ts`, line 21:

```ts
// PRD 16 §5: `kind` entra al doble porque la condicion de previousStageId dejo
// de mirar `isLost`. Sin esto los fixtures existentes quedan sin `kind` y la
// condicion nueva da false para todos.
type StageRow = { id: string; slug: string; isLost: boolean; kind: string };
```

That makes the two existing `previousStageId` tests fail to compile. Add `kind` to their stage rows
— `'out'` for `perdido` and `descartado`, `'pipeline'` for `cotizado` — and leave them otherwise
untouched: they are the regression guard that the old behaviour still works.

Then append the new cases, using the file's real `makeDb` (which *returns* `{ db, updates, events }`):

```ts
describe('PRD 16 §5 — previousStageId en TODAS las etapas fuera del flujo', () => {
  it('guarda de que etapa venia al pasar a no-contesta, que NO es isLost', async () => {
    // Este es el agujero: los dos sitios miraban isLost, y no-contesta se
    // siembra con isLost:false. El lead que se vencia por tiempo perdia
    // exactamente el dato que el requerimiento pide cruzar.
    const { db, updates } = makeDb({
      contact: { id: 'c1', stageId: 's2' },
      stages: [
        { id: 's2', slug: 'interesado',  isLost: false, kind: 'pipeline' },
        { id: 's6', slug: 'no-contesta', isLost: false, kind: 'out' },
      ],
    });

    await makeService().updateContact('c1', body({ status: 'no-contesta' }), 'u1', db);

    expect(updates[0].data.previousStageId).toBe('s2');
  });

  it('no lo pisa si el lead ya venia de otra etapa fuera del flujo', async () => {
    const { db, updates } = makeDb({
      contact: { id: 'c1', stageId: 's6' },
      stages: [
        { id: 's6', slug: 'no-contesta',   isLost: false, kind: 'out' },
        { id: 's9', slug: 'no-interesado', isLost: false, kind: 'out' },
      ],
    });

    await makeService().updateContact('c1', body({ status: 'no-interesado' }), 'u1', db);

    expect(updates[0].data.previousStageId).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/crm/crm.service.update-contact.spec.ts`
Expected: FAIL — `previousStageId` is `undefined` in the first test, because `no-contesta` is not
`isLost`.

- [ ] **Step 3: Change both gates**

In `src/crm/crm.service.ts` around line 408:

```ts
      // PRD 16 §5: la condicion era `isLost`, y solo `perdido` lo tiene. Las
      // otras tres etapas fuera del flujo no registraban de donde venia el lead
      // — justo la dimension que el requerimiento pide cruzar. `kind` es el dato
      // que dice "fuera del flujo"; isLost dice otra cosa (§2.2).
      if (targetStage.kind === 'out' && contact.stageId && contact.stageId !== targetStage.id) {
        const prev = await db.funnelStage.findUnique({ where: { id: contact.stageId } });
        if (prev && prev.kind !== 'out') updateData.previousStageId = contact.stageId;
      }
```

In `src/ai/ai.service.ts` around line 583, the same change:

```ts
          // PRD 16 §5: ver crm.service.ts — `isLost` dejaba afuera tres de las
          // cuatro etapas fuera del flujo.
          if (stage.kind === 'out' && contact.stageId && contact.stageId !== stage.id) {
            const prev = [...stageMap.values()].find((s: any) => s.id === contact.stageId);
            if (prev && prev.kind !== 'out') data.previousStageId = contact.stageId;
          }
```

- [ ] **Step 4: Run the tests and confirm they pass**

Run: `npm test -- src/crm/crm.service.update-contact.spec.ts`
Expected: PASS, including the pre-existing `al pasar a perdido guarda de que etapa venia` test —
`perdido` is `kind: 'out'`, so it still qualifies.

- [ ] **Step 5: Commit**

```bash
git add src/crm/crm.service.ts src/ai/ai.service.ts src/crm/crm.service.update-contact.spec.ts
git commit -m "fix(crm): la etapa previa se guardaba solo al pasar a perdido, no a las otras tres"
```

---

## Task 6: Per-source re-entry override

**Files:**
- Modify: `src/funnel/funnel-criteria.ts`
- Modify: `src/queue/message.processor.ts:93-96`
- Modify: `src/funnel/funnel-silence.spec.ts`

**Interfaces:**
- Consumes: `SilenceShape` from Task 3.
- Produces: `resolveReentryTarget(origen: { reentryTargetId?: string | null }, candidatas:
  { id: string; enabled: boolean; isReentryTarget?: boolean }[]): string | null` — the stage id the
  lead should return to, or `null` when nothing is configured.

- [ ] **Step 1: Write the failing test**

Append to `src/funnel/funnel-silence.spec.ts`:

```ts
import { resolveReentryTarget } from './funnel-criteria';

describe('resolveReentryTarget — PRD 16 §6.3', () => {
  const global = { id: 'interesado-id', enabled: true, isReentryTarget: true };
  const nuevo = { id: 'nuevo-id', enabled: true, isReentryTarget: false };

  it('sin override usa el destino global de PRD 15', () => {
    expect(resolveReentryTarget({ reentryTargetId: null }, [global, nuevo])).toBe('interesado-id');
  });

  it('con override usa el override', () => {
    // no-calificado: el spam no vuelve como "interesado" (§6.2).
    expect(resolveReentryTarget({ reentryTargetId: 'nuevo-id' }, [global, nuevo])).toBe('nuevo-id');
  });

  it('un override que apunta a una etapa desactivada cae al global', () => {
    const apagada = { id: 'nuevo-id', enabled: false, isReentryTarget: false };
    expect(resolveReentryTarget({ reentryTargetId: 'nuevo-id' }, [global, apagada]))
      .toBe('interesado-id');
  });

  it('un override que apunta a una etapa inexistente cae al global', () => {
    expect(resolveReentryTarget({ reentryTargetId: 'fantasma' }, [global, nuevo]))
      .toBe('interesado-id');
  });

  it('sin override y sin destino global devuelve null', () => {
    // El degrade de PRD 15 §3.3: no se mueve al lead. El caller loguea.
    expect(resolveReentryTarget({ reentryTargetId: null }, [nuevo])).toBeNull();
  });

  it('ignora un destino global desactivado', () => {
    const apagado = { id: 'interesado-id', enabled: false, isReentryTarget: true };
    expect(resolveReentryTarget({ reentryTargetId: null }, [apagado, nuevo])).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/funnel/funnel-silence.spec.ts`
Expected: FAIL — `resolveReentryTarget` is not exported.

- [ ] **Step 3: Write the implementation**

Append to `src/funnel/funnel-criteria.ts`:

```ts
/**
 * PRD 16 §6.3 — a donde vuelve un lead que escribe estando fuera del flujo.
 *
 * Es un OVERRIDE con fallback, no un reemplazo: PRD 15 ya esta implementado y
 * migrado, y cambiar sus dos booleanos por una sola FK costaba una migracion
 * destructiva sobre todas las bases de tenant para beneficiar a UNA fila.
 *
 * El override que apunta a algo invalido cae al global en vez de trabar el
 * reingreso: el lead volviendo al flujo importa mas que volver al lugar exacto,
 * y la validacion de escritura (checkReentryOverride) ya impide llegar a ese
 * estado por la via normal.
 */
export function resolveReentryTarget(
  origen: { reentryTargetId?: string | null },
  candidatas: { id: string; enabled: boolean; isReentryTarget?: boolean }[],
): string | null {
  const override = origen.reentryTargetId
    ? candidatas.find((s) => s.id === origen.reentryTargetId && s.enabled)
    : null;
  if (override) return override.id;

  const global = candidatas.find((s) => s.enabled && s.isReentryTarget === true);
  return global ? global.id : null;
}
```

- [ ] **Step 4: Run the test and confirm it passes**

Run: `npm test -- src/funnel/funnel-silence.spec.ts`
Expected: PASS.

- [ ] **Step 5: Use it in the read path**

In `src/queue/message.processor.ts`, replace **only** these four lines (93–96) —

```ts
    if (currentStage && currentStage.reentersOnReply) {
      const destino = await db.funnelStage.findFirst({
        where: { isReentryTarget: true, enabled: true },
      });
```

— with this. The `if (destino) {` on the following line and everything inside it stay exactly as
they are:

```ts
    if (currentStage && currentStage.reentersOnReply) {
      // PRD 16 §6.3: el destino puede estar overrideado por la etapa de ORIGEN.
      // `no-calificado` vuelve a `nuevo` y no a `interesado`, porque afirmar que
      // un blast de promociones esta "interesado" es una afirmacion sin nada
      // atras (§6.2).
      //
      // Se traen todas las activas en vez de armar un OR: la tabla tiene nueve
      // filas y el OR necesitaba un id centinela para el caso sin override.
      const candidatas = await db.funnelStage.findMany({ where: { enabled: true } });
      const destinoId = resolveReentryTarget(currentStage, candidatas);
      const destino = candidatas.find((s: any) => s.id === destinoId) ?? null;
      if (!destino) {
        // PRD 15 §3.3 dejaba este caso en silencio; despues de PRD 16 es una
        // mala configuracion, no un estado normal.
        this.logger.warn(
          `[${displayId}] sin destino de reingreso configurado; el lead queda en ${currentStage.slug}`,
        );
      }
```

Add the import at the top of the file:

```ts
import { resolveReentryTarget } from '../funnel/funnel-criteria';
```

- [ ] **Step 6: Type-check and lint**

Run: `npx tsc --noEmit -p tsconfig.json`
Run: `npx eslint src/queue/message.processor.ts src/funnel/funnel-criteria.ts src/funnel/funnel-silence.spec.ts`
Expected: clean.

- [ ] **Step 7: Commit**

```bash
git add src/funnel/funnel-criteria.ts src/queue/message.processor.ts src/funnel/funnel-silence.spec.ts
git commit -m "feat(queue): el spam no vuelve como interesado, vuelve a nuevo"
```

---

## Task 7: The sweep decision, as a pure function

**Files:**
- Create: `src/queue/silence-sweep.ts`
- Test: `src/queue/silence-sweep.spec.ts`

**Interfaces:**
- Consumes: `businessDaysBefore` from Task 1.
- Produces:
  - `type SilenceRule = { stageId: string; stageSlug: string; silenceDays: number | null; silenceTargetId: string | null }`
  - `type SilenceAction = { kind: 'disqualify'; toStageId: string; reason: string } | { kind: 'notify'; reason: string }`
  - `silenceCutoff(rule: SilenceRule, nowMs: number): number | null`
  - `decideSilenceAction(rule: SilenceRule): SilenceAction | null`

- [ ] **Step 1: Write the failing test**

Create `src/queue/silence-sweep.spec.ts`:

```ts
import { silenceCutoff, decideSilenceAction, type SilenceRule } from './silence-sweep';

const ART = (iso: string) => new Date(`${iso}-03:00`).getTime();

const regla = (over: Partial<SilenceRule> = {}): SilenceRule => ({
  stageId: 'st-1',
  stageSlug: 'interesado',
  silenceDays: 5,
  silenceTargetId: 'out-1',
  ...over,
});

describe('silenceCutoff', () => {
  it('cinco dias habiles desde un viernes cae el viernes anterior', () => {
    expect(silenceCutoff(regla(), ART('2026-09-18T10:00:00'))).toBe(ART('2026-09-11T10:00:00'));
  });

  it('una etapa sin plazo no tiene corte', () => {
    expect(silenceCutoff(regla({ silenceDays: null }), ART('2026-09-18T10:00:00'))).toBeNull();
  });

  it('el plazo largo de cotizado usa el mismo calculo', () => {
    const r = regla({ stageSlug: 'cotizado', silenceDays: 10, silenceTargetId: null });
    expect(silenceCutoff(r, ART('2026-09-25T10:00:00'))).toBe(ART('2026-09-11T10:00:00'));
  });
});

describe('decideSilenceAction', () => {
  it('con destino, descalifica', () => {
    expect(decideSilenceAction(regla())).toEqual({
      kind: 'disqualify',
      toStageId: 'out-1',
      reason: '5 días hábiles sin respuesta en "interesado"',
    });
  });

  it('sin destino, avisa y NO mueve — el lead cotizado no lo cierra el bot', () => {
    const r = regla({ stageSlug: 'cotizado', silenceDays: 10, silenceTargetId: null });
    expect(decideSilenceAction(r)).toEqual({
      kind: 'notify',
      reason: '10 días hábiles sin respuesta en "cotizado"',
    });
  });

  it('sin plazo no hay accion, aunque haya destino', () => {
    expect(decideSilenceAction(regla({ silenceDays: null }))).toBeNull();
  });

  it('un plazo de cero no dispara: lo rechaza la validacion, pero el dato viejo puede existir', () => {
    expect(decideSilenceAction(regla({ silenceDays: 0 }))).toBeNull();
  });
});
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `npm test -- src/queue/silence-sweep.spec.ts`
Expected: FAIL — `Cannot find module './silence-sweep'`.

- [ ] **Step 3: Write the implementation**

Create `src/queue/silence-sweep.ts`:

```ts
//
// PRD 16 §4 — la decision del barrido, aislada de la base.
//
// Toda la logica que se puede equivocar vive aca. El processor queda siendo
// consultas y escrituras, que es lo que NO se puede testear bien en este repo:
// cada handle `db` es `any`, asi que una consulta con la forma equivocada pasa
// el tsc limpio.
//

import { businessDaysBefore } from './business-days';

export type SilenceRule = {
  stageId: string;
  stageSlug: string;
  silenceDays: number | null;
  silenceTargetId: string | null;
};

export type SilenceAction =
  | { kind: 'disqualify'; toStageId: string; reason: string }
  | { kind: 'notify'; reason: string };

/**
 * El instante contra el que se compara el ultimo mensaje ENTRANTE. Un contacto
 * cuyo ultimo entrante es <= a esto se quedo callado el plazo configurado.
 */
export function silenceCutoff(rule: SilenceRule, nowMs: number): number | null {
  if (rule.silenceDays === null || rule.silenceDays < 1) return null;
  return businessDaysBefore(nowMs, rule.silenceDays);
}

/**
 * Que hacer con un lead que ya cumplio el silencio de su etapa.
 *
 * `silenceTargetId` en NULL es el caso de `cotizado` (§4.3): se avisa a un
 * humano y NO se cierra. Es la unica regla donde se descarto el cierre
 * automatico a proposito — un lead cotizado es el estado mas valioso del embudo
 * y el bot no deberia ser lo que lo da por perdido.
 */
export function decideSilenceAction(rule: SilenceRule): SilenceAction | null {
  if (rule.silenceDays === null || rule.silenceDays < 1) return null;

  const reason = `${rule.silenceDays} días hábiles sin respuesta en "${rule.stageSlug}"`;

  if (rule.silenceTargetId === null) return { kind: 'notify', reason };
  return { kind: 'disqualify', toStageId: rule.silenceTargetId, reason };
}
```

- [ ] **Step 4: Run the test and confirm it passes**

Run: `npm test -- src/queue/silence-sweep.spec.ts`
Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add src/queue/silence-sweep.ts src/queue/silence-sweep.spec.ts
git commit -m "feat(queue): la decision del barrido de silencio, aislada de la base"
```

---

## Task 8: The sweep processor

First repeatable job in this codebase — there is no `@Cron` or `ScheduleModule` anywhere, so this
introduces the pattern using BullMQ's own repeat option.

**Files:**
- Create: `src/queue/silence.processor.ts`
- Modify: `src/queue/queue.constants.ts`
- Modify: `src/queue/queue.module.ts`

**Interfaces:**
- Consumes: `silenceCutoff`, `decideSilenceAction`, `SilenceRule` (Task 7);
  `ContactLifecycleService.moveStage`; `NotificationsService.createForBranch`.
- Produces: `SILENCE_QUEUE = 'silence-sweep'` exported from `queue.constants.ts`.

- [ ] **Step 1: Add the queue constant**

In `src/queue/queue.constants.ts`:

```ts
export const SILENCE_QUEUE = 'silence-sweep';
```

- [ ] **Step 2: Write the processor**

Create `src/queue/silence.processor.ts`:

```ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger, OnModuleInit } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { PrismaService } from '../prisma/prisma.service';
import { TenantPrismaFactory } from '../tenants/tenant-prisma.factory';
import { ContactLifecycleService } from '../lifecycle/contact-lifecycle.service';
import { NotificationsService } from '../notifications/notifications.service';
import { SILENCE_QUEUE } from './queue.constants';
import { silenceCutoff, decideSilenceAction, type SilenceRule } from './silence-sweep';

// §4.3: el titulo es el marcador con el que se de-duplica el aviso. Si cambia,
// se vuelve a avisar una vez por cada lead cotizado que ya estaba vencido.
export const AVISO_COTIZADO_TITULO = 'Cotización sin respuesta';

@Processor(SILENCE_QUEUE)
export class SilenceProcessor extends WorkerHost implements OnModuleInit {
  private readonly logger = new Logger(SilenceProcessor.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly tenantFactory: TenantPrismaFactory,
    private readonly lifecycle: ContactLifecycleService,
    private readonly notifications: NotificationsService,
    @InjectQueue(SILENCE_QUEUE) private readonly queue: Queue,
  ) {
    super();
  }

  // El barrido corre solo. Una hora es mucho mas seguido que el plazo mas corto
  // (5 dias habiles), asi que la precision sobra y el costo es una consulta por
  // etapa con regla, por tenant.
  async onModuleInit() {
    await this.queue.add(
      'sweep',
      {},
      { repeat: { every: 60 * 60 * 1000 }, jobId: 'silence-sweep', removeOnComplete: true },
    );
  }

  async process(): Promise<void> {
    const tenants = await this.prisma.tenant.findMany({ where: { active: true } });
    for (const tenant of tenants) {
      try {
        await this.barrerTenant(tenant);
      } catch (e: any) {
        // A diferencia del runner de migraciones, esto se loguea como ERROR: un
        // tenant que no barre deja leads abiertos para siempre y nadie lo nota.
        this.logger.error(`[${tenant.slug}] barrido de silencio fallo: ${e?.message}`);
      }
    }
  }

  private async barrerTenant(tenant: any): Promise<void> {
    const db = this.tenantFactory.getClient(tenant.database_url);
    const now = Date.now();

    // Se traen TODAS las etapas activas, no solo las que tienen regla: la etapa
    // DESTINO no tiene `silenceDays` (no-contesta no vence por silencio), asi
    // que filtrando aca no se podria resolver su slug mas abajo.
    const todas = await db.funnelStage.findMany({ where: { enabled: true } });
    const conRegla = todas.filter((s: any) => s.silenceDays !== null);

    for (const stage of conRegla) {
      const rule: SilenceRule = {
        stageId: stage.id,
        stageSlug: stage.slug,
        silenceDays: stage.silenceDays,
        silenceTargetId: stage.silenceTargetId,
      };
      const cutoffMs = silenceCutoff(rule, now);
      const accion = decideSilenceAction(rule);
      if (cutoffMs === null || !accion) continue;

      const cutoff = new Date(cutoffMs);

      // §4.1: `messages: { none: ... }` genera un NOT EXISTS, que es lo que sirve
      // el indice (contactId, role, createdAt).
      //
      // `botActive: true` es del requerimiento, no una precaucion: la regla
      // aplica "solo si la conversacion esta en modo bot". Si un vendedor tomo la
      // conversacion, el silencio es problema suyo.
      //
      // `createdAt <= cutoff` cubre al contacto SIN mensajes: sin esa condicion
      // el `none` lo matchea y un contacto recien creado se descalificaria.
      const candidatos = await db.contact.findMany({
        where: {
          stageId: stage.id,
          botActive: true,
          createdAt: { lte: cutoff },
          messages: { none: { role: 'user', createdAt: { gt: cutoff } } },
        },
        select: { id: true, branchId: true, name: true, phone: true },
      });

      for (const contacto of candidatos) {
        if (accion.kind === 'disqualify') {
          // PRD 12: 'system' — lo decide el paso del tiempo, no una persona ni el
          // modelo. moveStage no escribe evento si la etapa no cambia, asi que
          // correr el barrido dos veces produce una sola transicion.
          //
          // PRD 16 §6.1: NO se toca botActive. El reingreso esta condicionado a
          // el, y apagarlo desactiva la reactivacion en silencio.
          const destino = todas.find((s: any) => s.id === accion.toStageId);
          if (!destino) {
            this.logger.warn(`[${tenant.slug}] "${stage.slug}" apunta a un destino inexistente o desactivado`);
            break;
          }
          await this.lifecycle.moveStage(db, contacto.id, destino.id, { kind: 'system' }, {
            data: { status: destino.slug },
            reason: accion.reason,
          });
          this.logger.log(`[${tenant.slug}] ${contacto.phone ?? contacto.id} → descalificado: ${accion.reason}`);
          continue;
        }

        // accion.kind === 'notify' — §4.3, el lead cotizado.
        //
        // De-duplicacion sin tipo de evento nuevo: PRD 12 §3 tiene el conjunto de
        // `type` CERRADO, y "avise a un humano" no es una mecanica del ciclo de
        // vida del lead. Se usa la propia tabla de notificaciones: si ya hay un
        // aviso posterior al ultimo mensaje del cliente, no se repite.
        const ultimoEntrante = await db.message.findFirst({
          where: { contactId: contacto.id, role: 'user' },
          orderBy: { createdAt: 'desc' },
          select: { createdAt: true },
        });
        const desde = ultimoEntrante?.createdAt ?? new Date(0);

        const yaAvisado = await db.notification.findFirst({
          where: { contactId: contacto.id, title: AVISO_COTIZADO_TITULO, createdAt: { gt: desde } },
        });
        if (yaAvisado) continue;

        if (!contacto.branchId) {
          this.logger.warn(`[${tenant.slug}] ${contacto.id} vencido sin sucursal: no hay a quien avisar`);
          continue;
        }

        await this.notifications.createForBranch(
          contacto.branchId,
          contacto.id,
          AVISO_COTIZADO_TITULO,
          `${contacto.name ?? contacto.phone ?? 'Un lead'} no responde hace ${rule.silenceDays} días hábiles desde que se le cotizó.`,
        );
        this.logger.log(`[${tenant.slug}] ${contacto.phone ?? contacto.id} → avisado: ${accion.reason}`);
      }
    }
  }
}
```

- [ ] **Step 3: Register the queue and the processor**

In `src/queue/queue.module.ts`:

- add `SilenceProcessor` to the `import` list and to `providers`
- add `SILENCE_QUEUE` to the import from `./queue.constants`, to the re-export line, and add
  `BullModule.registerQueue({ name: SILENCE_QUEUE }),` to `imports`

- [ ] **Step 4: Type-check**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: clean. If `db.notification` is not on the tenant schema, check the model name in
`prisma/schema.prisma` and use the real one — this is exactly the class of error `tsc` cannot catch,
because `db` is `any`.

- [ ] **Step 5: Verify against a throwaway tenant database**

Start the dev stack (`powershell -ExecutionPolicy Bypass -File .\start-dev.ps1` from
`soylaika.backend/`), then in the CRM's test chat drive one contact into `interesado`, and
back-date its last inbound message past the cutoff directly in the scratch database:

```sql
UPDATE "Message" SET "createdAt" = now() - interval '20 days'
  WHERE "contactId" = '<id>' AND role = 'user';
```

Trigger the sweep without waiting an hour:

```bash
node -e "const {Queue}=require('bullmq');new Queue('silence-sweep',{connection:{host:'localhost',port:6379}}).add('sweep',{}).then(()=>process.exit(0))"
```

Expected in the backend log: `→ descalificado: 5 días hábiles sin respuesta en "interesado"`, and
the contact now sits in `no-contesta` with `previousStageId` pointing at `interesado`.

- [ ] **Step 6: Commit**

```bash
git add src/queue/silence.processor.ts src/queue/queue.constants.ts src/queue/queue.module.ts
git commit -m "feat(queue): barrido periodico que cierra los leads vencidos por silencio"
```

---

## Task 9: The 24-hour rule stops closing leads

Do this **after** Task 8 works. Between removing this and the sweep running, nothing would close a
silent lead at all.

**Files:**
- Modify: `src/queue/followup.processor.ts`
- Modify: `src/queue/message.processor.ts` (the `no-answer-24h` enqueue block)
- Modify: `src/queue/queue.constants.ts`
- Modify: `src/crm/crm.service.ts` (`testFollowup`)

**Interfaces:**
- Consumes: nothing.
- Produces: `FollowupJobData['type']` narrows to `'window-6h' | 'last-call-23h'`.

- [ ] **Step 1: Remove the stage move from the follow-up processor**

In `src/queue/followup.processor.ts`, delete the whole `if (type === 'no-answer-24h') { ... }`
block, and leave this comment where it was:

```ts
    // PRD 16 §7: aca el lead se movia a `no-contesta` a las 24hs. Las 24hs no
    // eran una regla de negocio — son la ventana de mensajeria libre de
    // WhatsApp, que es lo que hace que los recontactos de 6h y 23h esten ahi.
    // El cierre ahora lo decide el barrido de silencio con el plazo que
    // configura el negocio (silence.processor.ts). Los dos nudges NO se tocan:
    // son los "2 intentos de recontacto" del requerimiento.
```

- [ ] **Step 2: Stop enqueuing the job**

In `src/queue/message.processor.ts`, delete the `no-answer-24h` enqueue block (the
`prevNoAnswer` lookup, the `followupQueue.add('no-answer-24h', ...)` call and its
`redis.set('followup:noAnswer:...')`). Update the final log line so it no longer claims a 24h
reclassification:

```ts
    this.logger.log(`[${phone}] Follow-ups encolados (23h last-call ${new Date(lastCallAt).toISOString()})`);
```

- [ ] **Step 3: Narrow the job type**

In `src/queue/queue.constants.ts`:

```ts
export interface FollowupJobData {
  type:            'window-6h' | 'last-call-23h';
```

- [ ] **Step 4: Follow through in the demo chat**

In `src/crm/crm.service.ts`, `testFollowup`: remove `'no-answer-24h'` from the `type` union and
delete its branch, so the test chat stops demonstrating behaviour the product no longer has.

- [ ] **Step 5: Type-check — this is what proves the removal is complete**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: clean. Narrowing the union in Step 3 turns every remaining reference to `'no-answer-24h'`
into a compile error, which is the point of doing it in that order. Fix any the compiler finds,
including the frontend caller of the test-followup endpoint if one exists
(`grep -rn "no-answer-24h" ../soylaika.frontend/app ../soylaika.frontend/lib`).

- [ ] **Step 6: Run the whole backend suite**

Run: `npm test`
Expected: PASS. A follow-up test asserting the 24h stage move must be deleted, not weakened — the
behaviour is gone.

- [ ] **Step 7: Commit**

```bash
git add src/queue/followup.processor.ts src/queue/message.processor.ts src/queue/queue.constants.ts src/crm/crm.service.ts
git commit -m "refactor(queue): las 24hs eran la ventana de WhatsApp, no un plazo de negocio"
```

---

## Task 10: Frontend stage styles

Small, and the same failure this repo has already had once: `followup.processor` started writing a
status the CRM did not know about, and those contacts rendered unstyled.

**Files:**
- Modify: `soylaika.frontend/lib/stage-style.ts`
- Modify: `soylaika.frontend/app/(crm)/contacts/page.tsx`

**Interfaces:**
- Consumes: the slugs seeded in Task 2 — `no-calificado`, `no-interesado`.
- Produces: nothing.

- [ ] **Step 1: Record the lint baseline**

From `soylaika.frontend/`:

Run: `npm run lint 2>&1 | tail -5`
Write the warning count down. It starts non-zero (`react-hooks/set-state-in-effect`) and must not
grow.

- [ ] **Step 2: Find every hardcoded copy**

Run: `grep -rn "no-contesta" lib/ app/ --include=*.ts --include=*.tsx`
Expected: `lib/stage-style.ts`, `app/(crm)/contacts/page.tsx`, and per the workspace notes possibly
`app/(crm)/resultados/page.tsx` and `app/(crm)/inicio/page.tsx`. Every map that lists `no-contesta`
needs the two new slugs.

- [ ] **Step 3: Add the slugs**

In each map found, matching the colours seeded in Task 2:

```ts
  'no-calificado': { label: 'No calificado', color: '#78716c' },
  'no-interesado': { label: 'No interesado', color: '#f97316' },
```

Match the exact shape each file already uses — they are not identical. Do not refactor them into a
shared module in this task; that is a separate change with its own reasoning.

- [ ] **Step 4: Verify in the browser**

Start the dev stack and open `http://tenant-dev.localhost:3001`, go to the contacts screen, and
filter to the two new stages. `tsc` and `lint` do not catch an unstyled badge — look at it.

Expected: both stages render with a label and colour, not as raw slugs or an empty chip.

- [ ] **Step 5: Confirm the lint baseline has not grown**

Run: `npm run lint 2>&1 | tail -5`
Expected: the same count as Step 1.

- [ ] **Step 6: Commit (frontend repo — this is a separate commit in a separate repo)**

```bash
git add lib/stage-style.ts "app/(crm)/contacts/page.tsx"
git commit -m "feat(crm): las dos etapas nuevas de descalificacion tienen estilo"
```

---

## Deploy

Not a task — it is the step that has broken this product before, so it is written down.

- [ ] **Pre-apply the migration to every tenant database BEFORE deploying the backend.**
  `TenantMigrationsService.onApplicationBootstrap` applies unregistered migrations to every active
  tenant in the background at boot and swallows failures into a `logger.warn`. A tenant that misses
  this one runs a classifier prompt naming stages its database does not have.
- [ ] **The index is the slow part.** `Message` is the largest table per tenant and a plain
  `CREATE INDEX` holds a write lock for its duration. `CONCURRENTLY` cannot run inside the runner's
  transaction, so for a large tenant do this in a maintenance window, by hand, with
  `CREATE INDEX CONCURRENTLY`, before the migration runs.
- [ ] **`perdido`'s criteria changes the bot's behaviour on deploy day** (§2.1). It is the only part
  of this work that does. Tell whoever watches the funnel that early "no me interesa" now lands in
  `no-interesado`, and that their historical `perdido` counts mean something slightly different from
  the day forward.

---

## Self-Review

**Spec coverage.** PRD 16 §2 → Tasks 2, 4. §2.2 → Tasks 2, 4. §2.3 → Task 3. §3 → Tasks 2, 3.
§3.1 → Task 3. §4 → Task 8. §4.1 → Tasks 2, 8. §4.2 → Task 1. §4.3 → Tasks 7, 8. §5 → Task 5.
§6 → Task 2 (the `reentersOnReply` seeds). §6.1 → Task 8 (the processor never touches `botActive`).
§6.2/§6.3 → Task 6. §7 → Task 9. §7.1 → Task 10. §8 → Task 2 + Deploy. §9 → the tests inside each
task. §10 is the not-doing list and needs no task.

**Known gap, deliberate.** PRD 16 §5 says the classifier may write a structured reason into
`ContactEvent.reason`; the sweep does that, but the *classifier* path for `no-interesado` and
spam-`no-calificado` relies entirely on the criteria text seeded in Task 2 plus the exception line
in Task 4 — there is no new code teaching the model those categories, by design (§3 of PRD 8 owns
that surface). If the model does not pick them up in practice, that is a prompt-tuning problem, not
a missing task.
