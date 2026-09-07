# FAQ Content Ingestion — Phase 2 (Vertical Starter Packs) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make a new tenant's FAQ index useful without anyone authoring it from scratch — a curated Q&A set per business vertical, stored once in the control plane, copied into a tenant on demand with the tenant's own business details filled in, landing in the review queue phase 1 built.

**Architecture:** Two new models in the shared `schema.prisma` — `FaqTemplate` / `FaqTemplateEntry` — whose rows live **only** in the control-plane (master) database. `faq-placeholders.ts` is a pure module resolving `{{horario}}`-style tokens from a tenant's `BusinessProfile`. `FaqTemplateService` reads templates from the control plane and applies one to a tenant by copying its entries through phase 1's `FaqIngestionService.upsertBatch()` as `source_type: TEMPLATE`, `review_status: PENDING_REVIEW`. A seed script loads the first vertical. **Copy, never reference** — once applied, chunks belong to the tenant and template updates do not propagate.

**Tech Stack:** NestJS 11, Prisma 7, Jest. No new dependencies.

**Spec:** `docs/prd-faq-content-ingestion.md` (workspace root, one level above this repo). §4.1 and §8 are this phase; §9 phase 2. Read `docs/superpowers/plans/2026-09-02-faq-content-ingestion-phase1.md` too — this plan calls the services it built.

## Global Constraints

- Comments and log messages in Spanish, matching the rest of `soylaika.backend`.
- Services take `tenantDb?: any` as a trailing parameter, resolved through a private `db(tenantDb)` helper — the established pattern.
- **Template rows live in the control plane only.** This codebase runs one shared `schema.prisma` across the master database and every tenant database, so the two new tables will physically materialise in every tenant DB as well (exactly as the `Tenant` table already does). That is harmless and expected: `FaqTemplateService` always reads them through `this.prisma` (the master client), never through `req.tenantDb`. Do not "fix" the empty tables in tenant databases.
- **Applied entries land `PENDING_REVIEW`, never `APPROVED`** — §4.1 is explicit that even curated template content needs a pass, because a template says "consultá con un asesor por plazos de envío" and this particular client might actually publish theirs.
- **`source_ref` is `template:<templateId>:v<version>`** and `source_ordinal` is the entry's ordinal. Phase 1's `@@unique([source_ref, source_ordinal])` then makes re-applying the same template version idempotent — the second apply updates rather than duplicating, and unchanged content hashes are skipped entirely.
- **Unresolved placeholders are deliberately left visible.** §8 requires that an unresolved `{{...}}` never reach `APPROVED`. Phase 1's lint already enforces exactly this — placeholders are a `warning` at `intake` (so a template carrying one still lands in the queue) and an `error` at `approval` (so it cannot be approved until a human resolves it). **Write no new enforcement for this**; verify the existing behaviour instead.
- **Deferred, deliberately:**
  - **Superadmin CRUD for authoring templates.** §8 argues for it ("requiring a deploy per edit means they stop being edited") and it is the right end state, but phase 2's value is the copy mechanism plus one real vertical. This plan ships read + apply endpoints and a seed script; authoring lands in a later slice. Recorded because §9's "seeded from repo files" and §8's "superadmin CRUD" contradict each other, and this plan takes §8's storage decision with §9's initial-population approach.
  - **Auto-applying a template at tenant creation.** `TenantsService.create` seeds funnel and agents; adding templates there would drop 40 unreviewed entries into every new tenant's queue unasked. Applying stays an explicit action.
  - **Diff-and-reapply across versions.** §8 mentions it as a possible future; `source_ref` carrying the version keeps it answerable later. Not built.
  - **Cross-vertical template inheritance / composition.** Not in the spec, not built.

---

## File Structure

```
soylaika.backend/
  prisma/
    schema.prisma                                        [MODIFY] +FaqTemplate, +FaqTemplateEntry
    migrations/20260902120000_add_faq_templates/migration.sql   [CREATE]
  src/faq/
    faq-placeholders.ts            [CREATE] pure {{token}} → BusinessProfile resolution
    faq-placeholders.spec.ts       [CREATE]
    faq-template.service.ts        [CREATE] control-plane reads + apply-to-tenant
    faq-template.service.spec.ts   [CREATE]
    faq.controller.ts              [MODIFY] +GET templates, +POST templates/:id/apply
    faq.controller.spec.ts         [MODIFY]
    faq.module.ts                  [MODIFY] register FaqTemplateService
  prisma/
    seed-faq-templates.ts          [CREATE] loads the first vertical into the control plane
  package.json                     [MODIFY] +faq:seed-templates script
```

---

### Task 1: Template schema + migration

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/20260902120000_add_faq_templates/migration.sql`

**Interfaces:**
- Produces: tables `"FaqTemplate"` and `"FaqTemplateEntry"`, consumed by Tasks 3–5.

- [ ] **Step 1: Add the models to `prisma/schema.prisma`**

Append at the end of the file:

```prisma
// Paquetes de FAQs por rubro, para arrancar un tenant nuevo sin escribir 40
// preguntas a mano (PRD 2 §4.1, §8). Viven SOLO en la base de control (master):
// el schema es compartido, asi que las tablas existen fisicamente en todas las
// bases de tenant igual que "Tenant", pero vacias — FaqTemplateService siempre
// las lee por el cliente master, nunca por req.tenantDb.
//
// Se COPIAN, no se referencian: una vez aplicadas, los chunks son del tenant y
// una edicion posterior de la plantilla no se propaga. Propagar cambios a
// tenants que ya editaron su copia es un problema mucho mas caro y no vale la
// pena resolverlo aca.
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
  @@index([vertical, active])
}

model FaqTemplateEntry {
  id          String      @id @default(uuid())
  templateId  String
  template    FaqTemplate @relation(fields: [templateId], references: [id], onDelete: Cascade)
  ordinal     Int
  question    String
  answer      String                    // puede tener {{placeholders}}
  agents      String[]    @default([])
  tags        String[]    @default([])

  @@unique([templateId, ordinal])
  @@index([templateId])
}
```

- [ ] **Step 2: Write the migration**

Create `prisma/migrations/20260902120000_add_faq_templates/migration.sql`:

```sql
-- Plantillas de FAQ por rubro (PRD 2 §8). Las filas viven solo en la base de
-- control; el schema es compartido asi que las tablas se crean en todas las
-- bases, igual que "Tenant". IF NOT EXISTS por el runner idempotente de tenants.
CREATE TABLE IF NOT EXISTS "FaqTemplate" (
    "id" TEXT NOT NULL,
    "vertical" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "version" INTEGER NOT NULL DEFAULT 1,
    "active" BOOLEAN NOT NULL DEFAULT true,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "FaqTemplate_pkey" PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "FaqTemplateEntry" (
    "id" TEXT NOT NULL,
    "templateId" TEXT NOT NULL,
    "ordinal" INTEGER NOT NULL,
    "question" TEXT NOT NULL,
    "answer" TEXT NOT NULL,
    "agents" TEXT[] NOT NULL DEFAULT '{}',
    "tags" TEXT[] NOT NULL DEFAULT '{}',

    CONSTRAINT "FaqTemplateEntry_pkey" PRIMARY KEY ("id")
);

CREATE UNIQUE INDEX IF NOT EXISTS "FaqTemplate_vertical_name_version_key" ON "FaqTemplate"("vertical", "name", "version");
CREATE INDEX IF NOT EXISTS "FaqTemplate_vertical_active_idx" ON "FaqTemplate"("vertical", "active");
CREATE UNIQUE INDEX IF NOT EXISTS "FaqTemplateEntry_templateId_ordinal_key" ON "FaqTemplateEntry"("templateId", "ordinal");
CREATE INDEX IF NOT EXISTS "FaqTemplateEntry_templateId_idx" ON "FaqTemplateEntry"("templateId");

DO $$ BEGIN
  ALTER TABLE "FaqTemplateEntry"
    ADD CONSTRAINT "FaqTemplateEntry_templateId_fkey"
    FOREIGN KEY ("templateId") REFERENCES "FaqTemplate"("id") ON DELETE CASCADE ON UPDATE CASCADE;
EXCEPTION WHEN duplicate_object THEN NULL; END $$;
```

- [ ] **Step 3: Apply and verify**

```bash
npx prisma generate
npx prisma migrate deploy
```
```bash
node -e "const {Client}=require('pg');(async()=>{const c=new Client({connectionString:process.env.DATABASE_URL||'postgresql://postgres:1234@localhost:5432/asap_crm'});await c.connect();console.log((await c.query(\"SELECT table_name FROM information_schema.tables WHERE table_name LIKE 'FaqTemplate%'\")).rows);await c.end();})()"
```
Expected: both `FaqTemplate` and `FaqTemplateEntry`.

- [ ] **Step 4: Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260902120000_add_faq_templates
git commit -m "feat(faq): add FaqTemplate/FaqTemplateEntry control-plane tables"
```

---

### Task 2: Placeholder resolution

**Files:**
- Create: `src/faq/faq-placeholders.ts`
- Test: `src/faq/faq-placeholders.spec.ts`

**Interfaces:**
- Produces: `resolvePlaceholders(text: string, profile: any | null): { text: string; unresolved: string[] }` and the exported `PLACEHOLDER_FIELDS` map. Consumed by Task 3.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-placeholders.spec.ts`:

```ts
import { resolvePlaceholders, PLACEHOLDER_FIELDS } from './faq-placeholders';

const profile = {
  name: 'Ideas Todo Terreno',
  hours: 'Lunes a viernes de 9 a 18',
  address: 'Av Siempreviva 742',
  phone: '011 4555-1234',
  returnPolicy: 'Cambios dentro de los 30 dias con ticket',
  shippingInfo: 'Enviamos a todo el pais',
  paymentMethods: 'Efectivo, transferencia y tarjeta',
  website: 'ideastodoterreno.com',
  email: 'hola@itt.com.ar',
  branches: 'Palermo y Caballito',
};

describe('resolvePlaceholders', () => {
  it('replaces a known token with the business profile value', () => {
    const out = resolvePlaceholders('Atendemos {{horario}}.', profile);
    expect(out.text).toBe('Atendemos Lunes a viernes de 9 a 18.');
    expect(out.unresolved).toEqual([]);
  });

  it('replaces several tokens in one string', () => {
    const out = resolvePlaceholders('Estamos en {{direccion}} y atendemos {{horario}}.', profile);
    expect(out.text).toBe('Estamos en Av Siempreviva 742 y atendemos Lunes a viernes de 9 a 18.');
    expect(out.unresolved).toEqual([]);
  });

  it('tolerates whitespace inside the braces', () => {
    expect(resolvePlaceholders('Atendemos {{ horario }}.', profile).text)
      .toBe('Atendemos Lunes a viernes de 9 a 18.');
  });

  it('leaves an unknown token untouched and reports it', () => {
    const out = resolvePlaceholders('Consultá por {{garantia_extendida}}.', profile);
    expect(out.text).toBe('Consultá por {{garantia_extendida}}.');
    expect(out.unresolved).toEqual(['garantia_extendida']);
  });

  // Un campo vacio en el perfil NO es una resolucion: dejar "" borraria la frase
  // y nadie se enteraria. Se deja el token para que la cola lo muestre.
  it('leaves a known token unresolved when the profile field is empty', () => {
    const out = resolvePlaceholders('Atendemos {{horario}}.', { ...profile, hours: '   ' });
    expect(out.text).toBe('Atendemos {{horario}}.');
    expect(out.unresolved).toEqual(['horario']);
  });

  it('leaves every token unresolved when there is no profile at all', () => {
    const out = resolvePlaceholders('{{negocio}} atiende {{horario}}.', null);
    expect(out.text).toBe('{{negocio}} atiende {{horario}}.');
    expect(out.unresolved).toEqual(['negocio', 'horario']);
  });

  it('reports each distinct unresolved token once', () => {
    const out = resolvePlaceholders('{{xx}} y {{xx}} y {{yy}}', profile);
    expect(out.unresolved).toEqual(['xx', 'yy']);
  });

  it('returns text unchanged when there are no placeholders', () => {
    const out = resolvePlaceholders('Sin tokens aca.', profile);
    expect(out.text).toBe('Sin tokens aca.');
    expect(out.unresolved).toEqual([]);
  });

  it('maps every documented token to a real BusinessProfile field', () => {
    for (const field of Object.values(PLACEHOLDER_FIELDS)) {
      expect(Object.keys(profile)).toContain(field);
    }
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-placeholders.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-placeholders.ts`**

```ts
// Resolucion de {{tokens}} de una plantilla contra el BusinessProfile del
// tenant, en el momento de copiarla (PRD 2 §8).
//
// Regla central: si un token no se puede resolver, se DEJA COMO ESTA. No se
// reemplaza por vacio. Una respuesta que dice "Atendemos ." leida rapido en la
// cola pasa por buena; una que dice "Atendemos {{horario}}." es obviamente algo
// que falta completar — y ademas el lint de aprobacion la bloquea, asi que no
// puede llegar al cliente sin que alguien la resuelva.

// Token → campo de BusinessProfile. Los alias existen porque quien escribe una
// plantilla no deberia tener que recordar el nombre exacto de la columna.
export const PLACEHOLDER_FIELDS: Record<string, string> = {
  negocio: 'name',
  nombre_negocio: 'name',
  sobre_el_negocio: 'about',
  horario: 'hours',
  horarios: 'hours',
  direccion: 'address',
  sucursales: 'branches',
  telefono: 'phone',
  email: 'email',
  web: 'website',
  sitio: 'website',
  medios_de_pago: 'paymentMethods',
  envios: 'shippingInfo',
  politica_cambios: 'returnPolicy',
  devoluciones: 'returnPolicy',
};

const TOKEN_RE = /\{\{\s*([\w.-]+)\s*\}\}/g;

export function resolvePlaceholders(
  text: string,
  profile: any | null,
): { text: string; unresolved: string[] } {
  const unresolved: string[] = [];

  const out = (text ?? '').replace(TOKEN_RE, (match, rawToken: string) => {
    const token = String(rawToken).trim();
    const field = PLACEHOLDER_FIELDS[token.toLowerCase()];
    const value = field && profile ? profile[field] : undefined;

    if (typeof value === 'string' && value.trim()) return value.trim();

    if (!unresolved.includes(token)) unresolved.push(token);
    return match; // se deja el token visible a proposito
  });

  return { text: out, unresolved };
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-placeholders.spec.ts
```
Expected: PASS (9 tests).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-placeholders.ts src/faq/faq-placeholders.spec.ts
git commit -m "feat(faq): resolve template placeholders from the business profile"
```

---

### Task 3: Template service

**Files:**
- Create: `src/faq/faq-template.service.ts`
- Test: `src/faq/faq-template.service.spec.ts`

**Interfaces:**
- Consumes: `resolvePlaceholders` (Task 2); `FaqIngestionService.upsertBatch(chunks, opts, tenantDb?, tenant?)` and `BusinessService.get(tenantDb?)` (both pre-existing).
- Produces: `FaqTemplateService.list(vertical?)`, `.get(id)`, `.apply(id, tenantDb?, tenant?)`. Consumed by Task 4.

- [ ] **Step 1: Write the failing test**

Create `src/faq/faq-template.service.spec.ts`:

```ts
import { NotFoundException } from '@nestjs/common';
import { FaqTemplateService } from './faq-template.service';

const template = {
  id: 'tpl-1',
  vertical: 'decoracion',
  name: 'Decoracion basico',
  version: 2,
  active: true,
  entries: [
    { id: 'e1', ordinal: 1, question: '¿Hacen envios?', answer: '{{envios}}', agents: [], tags: ['envios'] },
    { id: 'e2', ordinal: 2, question: '¿Que horario tienen?', answer: 'Atendemos {{horario}}.', agents: ['default'], tags: [] },
  ],
};

// `alreadyCopied` simula los chunks que el tenant ya tiene de esta version.
function makeService(
  tpl: any = template,
  profile: any = { shippingInfo: 'Enviamos a todo el pais', hours: '9 a 18' },
  alreadyCopied: any[] = [],
) {
  const prisma = {
    faqTemplate: { findMany: jest.fn().mockResolvedValue([tpl]), findUnique: jest.fn().mockResolvedValue(tpl) },
    faqChunk: { findMany: jest.fn().mockResolvedValue(alreadyCopied) },
  } as any;
  const ingestion = { upsertBatch: jest.fn().mockResolvedValue({ created: 2, updated: 0, unchanged: 0, chunks: [] }) } as any;
  const business = { get: jest.fn().mockResolvedValue(profile) } as any;
  const tenantDb = { faqChunk: { findMany: jest.fn().mockResolvedValue(alreadyCopied) } } as any;
  return { service: new FaqTemplateService(prisma, ingestion, business), prisma, ingestion, business, tenantDb };
}

describe('FaqTemplateService.list', () => {
  it('lists active templates from the control plane, newest version first', async () => {
    const { service, prisma } = makeService();
    await service.list();
    expect(prisma.faqTemplate.findMany).toHaveBeenCalledWith(
      expect.objectContaining({ where: { active: true }, orderBy: [{ vertical: 'asc' }, { version: 'desc' }] }),
    );
  });

  it('filters by vertical when given', async () => {
    const { service, prisma } = makeService();
    await service.list('decoracion');
    expect(prisma.faqTemplate.findMany).toHaveBeenCalledWith(
      expect.objectContaining({ where: { active: true, vertical: 'decoracion' } }),
    );
  });
});

describe('FaqTemplateService.apply', () => {
  it('copies entries as TEMPLATE / PENDING_REVIEW with a versioned source_ref', async () => {
    const { service, ingestion } = makeService();
    const tenantDb = {};
    await service.apply('tpl-1', tenantDb, { slug: 'itt' });

    const [chunks, opts, passedDb, passedTenant] = ingestion.upsertBatch.mock.calls[0];
    expect(opts).toEqual({
      sourceType: 'TEMPLATE',
      sourceRef: 'template:tpl-1:v2',
      reviewStatus: 'PENDING_REVIEW',
    });
    expect(passedDb).toBe(tenantDb);
    expect(passedTenant).toEqual({ slug: 'itt' });
    expect(chunks).toHaveLength(2);
    expect(chunks[0].sourceOrdinal).toBe(1);
    expect(chunks[1].sourceOrdinal).toBe(2);
  });

  it('resolves placeholders from the tenant business profile', async () => {
    const { service, ingestion } = makeService();
    await service.apply('tpl-1', {});
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].answer).toBe('Enviamos a todo el pais');
    expect(chunks[1].answer).toBe('Atendemos 9 a 18.');
  });

  // Un placeholder que no se pudo resolver viaja tal cual. El lint de intake lo
  // deja pasar como advertencia (entra a la cola) y el de aprobacion lo bloquea,
  // asi que no puede llegar al cliente sin que un humano lo resuelva.
  it('keeps unresolved placeholders visible and reports them', async () => {
    const { service, ingestion } = makeService(template, { hours: '9 a 18' }); // sin shippingInfo
    const result = await service.apply('tpl-1', {});
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].answer).toBe('{{envios}}');
    expect(result.unresolved).toEqual([{ ordinal: 1, tokens: ['envios'] }]);
  });

  it('reports what the copy produced', async () => {
    const { service } = makeService();
    const result = await service.apply('tpl-1', {});
    expect(result).toMatchObject({
      templateId: 'tpl-1',
      version: 2,
      sourceRef: 'template:tpl-1:v2',
      created: 2,
    });
  });

  it('throws NotFound for an unknown template', async () => {
    const { service, prisma } = makeService();
    prisma.faqTemplate.findUnique.mockResolvedValue(null);
    await expect(service.apply('nope', {})).rejects.toBeInstanceOf(NotFoundException);
  });

  it('throws NotFound for a template with no entries', async () => {
    const { service } = makeService({ ...template, entries: [] });
    await expect(service.apply('tpl-1', {})).rejects.toBeInstanceOf(NotFoundException);
  });

  it('copies without a business profile at all, leaving every token visible', async () => {
    const { service, ingestion } = makeService(template, null);
    const result = await service.apply('tpl-1', {});
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks[0].answer).toBe('{{envios}}');
    expect(chunks[1].answer).toBe('Atendemos {{horario}}.');
    expect(result.unresolved).toEqual([
      { ordinal: 1, tokens: ['envios'] },
      { ordinal: 2, tokens: ['horario'] },
    ]);
  });

  it('proceeds when the profile lookup fails outright', async () => {
    const { service, business, ingestion } = makeService();
    business.get.mockRejectedValue(new Error('tenant db unreachable'));
    const result = await service.apply('tpl-1', {});
    expect(ingestion.upsertBatch).toHaveBeenCalled();
    expect(result.unresolved).toHaveLength(2);
  });

  // Lo central: una vez copiada, la fila es del tenant. Reaplicar la misma
  // version NO puede pisar lo que un humano ya corrigio y aprobo.
  it('preserves an entry the tenant edited after the first apply', async () => {
    const edited = [
      { source_ordinal: 1, question: '¿Hacen envios?', answer: 'Si, y son gratis arriba de cierto monto.' },
    ];
    const { service, ingestion, tenantDb } = makeService(template, undefined, edited);
    const result = await service.apply('tpl-1', tenantDb);

    expect(result.preserved).toEqual([1]);
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks).toHaveLength(1);
    expect(chunks[0].sourceOrdinal).toBe(2);
  });

  it('still refreshes an entry the tenant left untouched', async () => {
    const untouched = [
      { source_ordinal: 1, question: '¿Hacen envios?', answer: 'Enviamos a todo el pais' },
    ];
    const { service, ingestion, tenantDb } = makeService(template, undefined, untouched);
    const result = await service.apply('tpl-1', tenantDb);

    expect(result.preserved).toEqual([]);
    const [chunks] = ingestion.upsertBatch.mock.calls[0];
    expect(chunks).toHaveLength(2);
  });

  it('does not call upsertBatch when every entry was preserved', async () => {
    const allEdited = [
      { source_ordinal: 1, question: '¿Hacen envios?', answer: 'editado' },
      { source_ordinal: 2, question: '¿Que horario tienen?', answer: 'editado' },
    ];
    const { service, ingestion, tenantDb } = makeService(template, undefined, allEdited);
    const result = await service.apply('tpl-1', tenantDb);

    expect(result.preserved).toEqual([1, 2]);
    expect(result.created).toBe(0);
    expect(ingestion.upsertBatch).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-template.service.spec.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `faq-template.service.ts`**

```ts
import { Injectable, Logger, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { BusinessService } from '../business/business.service';
import { FaqIngestionService } from './faq-ingestion.service';
import { resolvePlaceholders } from './faq-placeholders';

export interface ApplyResult {
  templateId: string;
  version: number;
  sourceRef: string;
  created: number;
  updated: number;
  unchanged: number;
  /** Entradas que quedaron con algun {{token}} sin resolver, para avisar en el panel. */
  unresolved: { ordinal: number; tokens: string[] }[];
  /**
   * Ordinales que NO se tocaron porque la copia del tenant ya no coincide con
   * la plantilla — es decir, alguien la edito despues de aplicarla.
   */
  preserved: number[];
}

@Injectable()
export class FaqTemplateService {
  private readonly logger = new Logger(FaqTemplateService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly ingestion: FaqIngestionService,
    private readonly business: BusinessService,
  ) {}

  // Solo para leer los chunks YA copiados al tenant. Las plantillas nunca
  // pasan por aca — esas siempre salen de this.prisma (control plane).
  private db(tenantDb?: any) { return tenantDb ?? this.prisma; }

  // Las plantillas viven en la base de control: SIEMPRE this.prisma, nunca
  // req.tenantDb. Las tablas existen vacias en las bases de tenant porque el
  // schema es compartido; leerlas desde ahi devolveria siempre cero filas.
  list(vertical?: string) {
    return this.prisma.faqTemplate.findMany({
      where: { active: true, ...(vertical ? { vertical } : {}) },
      orderBy: [{ vertical: 'asc' }, { version: 'desc' }],
      select: { id: true, vertical: true, name: true, version: true, created_at: true,
                _count: { select: { entries: true } } },
    });
  }

  get(id: string) {
    return this.prisma.faqTemplate.findUnique({
      where: { id },
      include: { entries: { orderBy: { ordinal: 'asc' } } },
    });
  }

  // Copia (no referencia) las entradas de la plantilla al tenant, resolviendo
  // los {{tokens}} contra su BusinessProfile. Entra todo como PENDING_REVIEW:
  // §4.1 es explicito en que hasta el contenido curado necesita una pasada,
  // porque una plantilla dice "consulta con un asesor por plazos de envio" y
  // este cliente en particular capaz si los publica.
  async apply(id: string, tenantDb?: any, tenant?: any): Promise<ApplyResult> {
    const template = await this.get(id);
    if (!template) throw new NotFoundException('Plantilla no encontrada');
    if (!template.entries.length) throw new NotFoundException('La plantilla no tiene entradas');

    // Si esto falla de verdad (base del tenant caida, no "no hay perfil"), se
    // sigue igual pero se LOGUEA: sin el warning, un error de infra se ve
    // identico a un perfil vacio y alguien va a preguntarse por que la
    // plantilla entro con todos los tokens sin resolver.
    const profile = await this.business.get(tenantDb).catch((err: any) => {
      this.logger.warn(`No se pudo leer el BusinessProfile del tenant: ${err?.message ?? err}`);
      return null;
    });

    const sourceRef = `template:${template.id}:v${template.version}`;

    // Lo ya copiado de ESTA misma version. El source_ref lleva la version y el
    // contenido de una version es inmutable (para cambiarla se sube el numero),
    // asi que si lo guardado difiere de lo que la plantilla produce ahora, lo
    // edito el tenant. Esas filas no se tocan: la plantilla se copia una vez y
    // despues el contenido es del tenant. Sin esto, reaplicar pisaba el texto
    // que un humano ya habia corregido Y lo devolvia a PENDING_REVIEW,
    // tirando la revision a la basura.
    const already = await this.db(tenantDb)
      .faqChunk.findMany({
        where: { source_ref: sourceRef },
        select: { source_ordinal: true, question: true, answer: true },
      })
      .catch(() => [] as any[]);
    const byOrdinal = new Map<number, any>(already.map((r: any) => [r.source_ordinal, r]));

    const unresolved: { ordinal: number; tokens: string[] }[] = [];
    const preserved: number[] = [];
    const chunks: any[] = [];

    for (const entry of template.entries as any[]) {
      const q = resolvePlaceholders(entry.question, profile);
      const a = resolvePlaceholders(entry.answer, profile);

      const prev = byOrdinal.get(entry.ordinal);
      if (prev && (prev.question !== q.text || prev.answer !== a.text)) {
        preserved.push(entry.ordinal);
        continue;
      }

      const tokens = [...new Set([...q.unresolved, ...a.unresolved])];
      if (tokens.length) unresolved.push({ ordinal: entry.ordinal, tokens });

      chunks.push({
        question: q.text,
        answer: a.text,
        agents: entry.agents ?? [],
        tags: entry.tags ?? [],
        sourceOrdinal: entry.ordinal,
      });
    }

    // El sourceRef lleva la version: reaplicar la MISMA version es idempotente
    // (upsertBatch matchea por source_ref + source_ordinal y saltea lo que no
    // cambio), mientras que una version nueva entra como contenido aparte.
    const result = chunks.length
      ? await this.ingestion.upsertBatch(
          chunks,
          { sourceType: 'TEMPLATE', sourceRef, reviewStatus: 'PENDING_REVIEW' },
          tenantDb,
          tenant,
        )
      : { created: 0, updated: 0, unchanged: 0, chunks: [] };

    this.logger.log(
      `Plantilla ${sourceRef} aplicada a ${tenant?.slug ?? 'master'}: ` +
      `${result.created} nuevas, ${result.updated} actualizadas, ${result.unchanged} sin cambios, ` +
      `${preserved.length} preservadas (editadas por el tenant), ` +
      `${unresolved.length} con placeholders sin resolver`,
    );

    return {
      templateId: template.id,
      version: template.version,
      sourceRef,
      created: result.created,
      updated: result.updated,
      unchanged: result.unchanged,
      unresolved,
      preserved,
    };
  }
}
```

- [ ] **Step 4: Run the test, confirm it passes**

```bash
npm test -- src/faq/faq-template.service.spec.ts
```
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add src/faq/faq-template.service.ts src/faq/faq-template.service.spec.ts
git commit -m "feat(faq): add template service that copies a vertical pack into a tenant"
```

---

### Task 4: Template endpoints

**Files:**
- Modify: `src/faq/faq.controller.ts`
- Modify: `src/faq/faq.module.ts`
- Modify: `src/faq/faq.controller.spec.ts`

**Interfaces:**
- Consumes: `FaqTemplateService` (Task 3).
- Produces: `GET /api/faq/templates`, `POST /api/faq/templates/:id/apply`.

- [ ] **Step 1: Write the failing tests**

Append to the `FaqController — import and review` describe block's sibling in `src/faq/faq.controller.spec.ts` (a new describe at the end of the file):

```ts
describe('FaqController — templates', () => {
  function makeController() {
    const templates = {
      list: jest.fn().mockResolvedValue([{ id: 'tpl-1', vertical: 'decoracion' }]),
      apply: jest.fn().mockResolvedValue({ templateId: 'tpl-1', created: 40, unresolved: [] }),
    } as any;
    const controller = new (require('./faq.controller').FaqController)(
      {} as any, {} as any, {} as any, {} as any, {} as any, templates,
    );
    return { controller, templates };
  }

  const req = { tenantDb: {}, tenant: { slug: 'itt' }, user: { id: 'user-7' } } as any;

  it('templates() forwards the vertical filter', async () => {
    const { controller, templates } = makeController();
    await controller.templates('decoracion');
    expect(templates.list).toHaveBeenCalledWith('decoracion');
  });

  it('applyTemplate() applies to the calling tenant', async () => {
    const { controller, templates } = makeController();
    const result = await controller.applyTemplate('tpl-1', req);
    expect(templates.apply).toHaveBeenCalledWith('tpl-1', req.tenantDb, req.tenant);
    expect(result.created).toBe(40);
  });
});
```

Also update the three `makeController()` helpers already in this file to pass a sixth `{} as any` argument, and the existing `new FaqController(...)` calls to pass six arguments total — the constructor grows again. (Same reason as phase 1 Task 4: `tsc` checks arity even though `ts-jest` does not.)

- [ ] **Step 2: Run, confirm failure**

```bash
npm test -- src/faq/faq.controller.spec.ts
```
Expected: FAIL — the new handlers don't exist.

- [ ] **Step 3: Add the handlers**

In `src/faq/faq.controller.ts`, import the service and add it as a sixth constructor parameter (plainly required, like the others):

```ts
import { FaqTemplateService } from './faq-template.service';
```
```ts
    private readonly templates: FaqTemplateService,
```

Add these handlers **before** the `:id` routes, alongside the other static-segment routes:

```ts
  // Plantillas disponibles. Lectura del control plane: no depende del tenant,
  // pero pedimos sesion igual porque expone el catalogo de contenido curado.
  @Get('templates')
  templates(@Query('vertical') vertical?: string) {
    return this.templates.list(vertical);
  }

  // Copia una plantilla al tenant que hace el request. Entra todo a la cola de
  // revision, asi que es seguro: nada llega al cliente sin que alguien lo mire.
  @Post('templates/:id/apply')
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  applyTemplate(@Param('id') id: string, @Req() req: any) {
    return this.templates.apply(id, req.tenantDb, req.tenant);
  }
```

- [ ] **Step 4: Register in `faq.module.ts`**

```ts
import { FaqTemplateService } from './faq-template.service';
```
Add `FaqTemplateService` to both `providers` and `exports`.

- [ ] **Step 5: Run tests and type-check**

```bash
npm test -- src/faq/faq.controller.spec.ts
npx tsc --noEmit -p tsconfig.json
```
Both must be clean. **Run both** — `ts-jest` does not full-type-check, so a green suite can hide a constructor-arity error.

- [ ] **Step 6: Commit**

```bash
git add src/faq/faq.controller.ts src/faq/faq.module.ts src/faq/faq.controller.spec.ts
git commit -m "feat(faq): expose template listing and apply endpoints"
```

---

### Task 5: Seed the first vertical

**Files:**
- Create: `prisma/seed-faq-templates.ts`
- Modify: `package.json`
- Modify: `doc/api-frontend.md`

- [ ] **Step 1: Write the seed script**

Create `prisma/seed-faq-templates.ts`. It is idempotent on `(vertical, name, version)` so re-running is safe, and it writes to the **control plane** (`DATABASE_URL`), not to a tenant:

```ts
// Carga los paquetes de FAQ por rubro en la base de control (PRD 2 §4.1).
// Idempotente: matchea por (vertical, name, version), asi que correrlo dos
// veces no duplica. Para publicar cambios, subir `version` — los tenants que
// ya aplicaron la anterior conservan su copia intacta.
//
// Uso: npm run faq:seed-templates
import 'dotenv/config';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

type Entry = { question: string; answer: string; agents?: string[]; tags?: string[] };

// Rubro decoracion / empapelados. Las respuestas evitan precios a proposito
// (el catalogo es la unica fuente) y usan {{tokens}} donde el dato es propio de
// cada negocio, para que se resuelvan al copiar.
const DECORACION: Entry[] = [
  { question: '¿Hacen envios?', answer: '{{envios}}', tags: ['envios'] },
  { question: '¿Cuanto tarda el envio?', answer: 'El plazo depende de la zona. Te lo confirma un asesor cuando cerramos el pedido.', tags: ['envios'] },
  { question: '¿Que horario de atencion tienen?', answer: 'Atendemos {{horario}}.', tags: ['institucional'] },
  { question: '¿Donde estan ubicados?', answer: 'Estamos en {{direccion}}.', tags: ['institucional'] },
  { question: '¿Tienen local para ir a ver?', answer: 'Si, podes visitarnos en {{direccion}}, {{horario}}.', tags: ['institucional'] },
  { question: '¿Que medios de pago aceptan?', answer: '{{medios_de_pago}}', tags: ['pagos'] },
  { question: '¿Puedo cambiar un producto?', answer: '{{politica_cambios}}', agents: ['devolucion'], tags: ['cambios'] },
  { question: '¿Que pasa si llega dañado?', answer: 'Si llega con una falla, escribinos con fotos del producto y del embalaje y lo resolvemos.', agents: ['soporte', 'devolucion'], tags: ['postventa'] },
  { question: '¿Como se coloca el empapelado?', answer: 'Se coloca sobre pared lisa, limpia y seca. Cada rollo viene con instrucciones y podemos pasarte la guia de colocacion.', agents: ['ventas', 'soporte'], tags: ['colocacion'] },
  { question: '¿Necesito contratar un colocador?', answer: 'No es obligatorio: muchos clientes lo colocan por su cuenta siguiendo la guia. Si preferis un profesional, un asesor te orienta.', agents: ['ventas'], tags: ['colocacion'] },
  { question: '¿Sirve para baños o cocinas?', answer: 'Si, siempre que la pared no tenga humedad activa. En ambientes muy humedos conviene el vinilo antes que el papel.', agents: ['ventas'], tags: ['materialidad'] },
  { question: '¿Cual es la diferencia entre papel mural y vinilo?', answer: 'El papel mural tiene mejor terminacion visual y va en ambientes secos. El vinilo resiste humedad y se limpia, ideal para cocinas, baños y zonas de mucho uso.', agents: ['ventas'], tags: ['materialidad'] },
  { question: '¿Como mido cuanto material necesito?', answer: 'Necesitas el ancho y el alto de cada pared a empapelar. Con esas medidas un asesor te calcula los rollos.', agents: ['ventas'], tags: ['medidas'] },
  { question: '¿Se puede colocar sobre azulejos?', answer: 'Sobre azulejo liso y bien adherido si, pero la junta puede marcarse. Contanos como esta la pared y te decimos si conviene.', agents: ['ventas', 'soporte'], tags: ['colocacion'] },
  { question: '¿Hacen diseños personalizados?', answer: 'Si, trabajamos diseños a medida. Un asesor te explica como es el proceso segun lo que necesites.', agents: ['ventas'], tags: ['personalizado'] },
  { question: '¿Puedo pedir una muestra antes de comprar?', answer: 'Escribinos que diseño te interesa y un asesor te cuenta como conseguir una muestra.', agents: ['ventas'], tags: ['muestras'] },
  { question: '¿Como sigo mi pedido?', answer: 'Pasanos tu numero de pedido o el mail con el que compraste y te decimos como viene.', agents: ['tracking'], tags: ['seguimiento'] },
  { question: '¿Puedo retirar por el local?', answer: 'Si, podes retirar por {{direccion}} dentro del horario de atencion.', tags: ['envios'] },
  { question: '¿Tienen sucursales?', answer: '{{sucursales}}', tags: ['institucional'] },
  { question: '¿Como los contacto?', answer: 'Podes escribirnos por aca mismo, o al {{telefono}}.', tags: ['institucional'] },
];

async function seedTemplate(vertical: string, name: string, version: number, entries: Entry[]) {
  const existing = await prisma.faqTemplate.findUnique({
    where: { vertical_name_version: { vertical, name, version } },
  });
  if (existing) {
    console.log(`= ${vertical}/${name} v${version} ya existe (${existing.id}), no se toca`);
    return;
  }

  const created = await prisma.faqTemplate.create({
    data: {
      vertical,
      name,
      version,
      entries: {
        create: entries.map((e, i) => ({
          ordinal: i + 1,
          question: e.question,
          answer: e.answer,
          agents: e.agents ?? [],
          tags: e.tags ?? [],
        })),
      },
    },
  });
  console.log(`+ ${vertical}/${name} v${version} creada (${created.id}) con ${entries.length} entradas`);
}

async function main() {
  await seedTemplate('decoracion', 'Decoracion basico', 1, DECORACION);
}

main()
  .catch((e) => { console.error(e); process.exit(1); })
  .finally(() => prisma.$disconnect());
```

- [ ] **Step 2: Add the npm script**

In `package.json`, next to the existing `seed` script:

```json
    "faq:seed-templates": "ts-node -r tsconfig-paths/register prisma/seed-faq-templates.ts",
```

- [ ] **Step 3: Run it, twice**

```bash
npm run faq:seed-templates
npm run faq:seed-templates
```
Expected: the first run prints `+ decoracion/Decoracion basico v1 creada (...) con 20 entradas`; the second prints `= ... ya existe ..., no se toca`. That second run is the point — it proves idempotency.

- [ ] **Step 4: Verify the entries landed**

```bash
node -e "const {Client}=require('pg');(async()=>{const c=new Client({connectionString:process.env.DATABASE_URL||'postgresql://postgres:1234@localhost:5432/asap_crm'});await c.connect();console.log((await c.query('SELECT t.vertical, t.name, t.version, count(e.id) AS entradas FROM \"FaqTemplate\" t LEFT JOIN \"FaqTemplateEntry\" e ON e.\"templateId\"=t.id GROUP BY t.id, t.vertical, t.name, t.version')).rows);await c.end();})()"
```
Expected: one row, `entradas` = 20.

- [ ] **Step 5: Document the endpoints**

In `doc/api-frontend.md`, extend the FAQ section with `GET /api/faq/templates` (optional `vertical` filter, returns id/vertical/name/version/entry count) and `POST /api/faq/templates/:id/apply` (admin only; copies into the calling tenant as `PENDING_REVIEW`; returns `created`/`updated`/`unchanged` plus `unresolved`, the per-entry list of `{{tokens}}` that had no matching `BusinessProfile` field). Note explicitly that applying the same template version twice is idempotent, and that unresolved placeholders block approval until a human resolves them.

- [ ] **Step 6: Commit**

```bash
git add prisma/seed-faq-templates.ts package.json doc/api-frontend.md
git commit -m "feat(faq): seed the decoracion starter pack and document template endpoints"
```

---

## Spec Coverage Check (self-review)

| PRD 2 section | Covered by |
|---|---|
| §4.1 starter packs stored in the control plane | Tasks 1, 3 |
| §4.1 copied not referenced, no propagation | Task 3 (`source_ref` carries the version; no back-reference stored) |
| §4.1 seeded as `TEMPLATE` / `PENDING_REVIEW` | Task 3 |
| §4.1 placeholders resolved from `BusinessProfile`, unresolved left visible | Tasks 2, 3 — and blocked from approval by phase 1's lint, verified not rebuilt |
| §8 `FaqTemplate` / `FaqTemplateEntry` schema | Task 1 |
| §8 `source_ref: template:<id>:v<version>` | Task 3 |
| §8 versioning answers "which tenants got which version" | Task 1 (`version` column) + Task 3 (`source_ref`) |
| §8 superadmin CRUD authoring | **Deferred** — see Global Constraints; read + apply + seed ship here |
| §9 phase 2 | This plan |
| §11 "template content wrong for a client" → lands `PENDING_REVIEW` | Task 3 |
| §11 "placeholders reach a customer unresolved" | Phase 1's approval-stage lint, unchanged |

## Execution

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks.
2. **Inline Execution** — batch through in this session with checkpoints.
