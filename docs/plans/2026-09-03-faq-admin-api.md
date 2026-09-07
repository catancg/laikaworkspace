# FAQ admin API (PRD 3 phase 0) — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a superadmin read and edit any tenant's FAQ knowledge base from the platform panel, turn retrieval on per tenant, and leave a trace of who changed what.

**Architecture:** No new FAQ logic. Four contract-level changes to existing code: an `updated_by` column so edits are attributable; a `list` method extracted from the controller so it can be shared; a set of superadmin-only `/tenants/:slug/faq*` routes that resolve the tenant database from the path and delegate to the existing `Faq*` services; and two small body-field additions.

**Tech Stack:** NestJS 11, Prisma 7 (raw tagged templates for vector I/O), PostgreSQL + pgvector, Jest/ts-jest.

**Spec:** `f:\docs\Laika\docs\prd-faq-admin-ui.md` — §4.1, §4.2, and the manual-create note under §4.1's route table. This plan is that document's phase 0 in full.

## Global Constraints

- Comments and user-facing strings in **español rioplatense (voseo)**. Match the surrounding file.
- **The load-bearing invariant: only `review_status = 'APPROVED'` AND `active = true` chunks are retrievable by the bot.** Nothing here may create a path that publishes content without a human, or that silently unpublishes approved content.
- **Cross-tenant access is the point of this plan and its main hazard.** Every new route must sit on `TenantsController`, which carries class-level `@UseGuards(JwtAuthGuard, RolesGuard)` and `@Roles('superadmin')`. Never add a tenant-slug-in-path route to `FaqController` — that controller is for a tenant acting on itself.
- Migrations are **hand-written SQL** in `prisma/migrations/<timestamp>_<nombre>/`, idempotent (`IF NOT EXISTS`), because `TenantMigrationsService` applies them to every tenant database at boot.
- **`tsc` cannot see query shapes.** All DB access goes through `private db(tenantDb?: any)`, which returns `any`. Wrong column names and wrong `where` keys are invisible to `tsc` and to `ts-jest`. Verify column names against `prisma/schema.prisma` by hand, and prefer running SQL against the local `pgvector/pg16` container over reasoning about it.
- Both gates on every task, reported separately: `npm test -- src/faq/ src/tenants/` and `npx tsc --noEmit -p tsconfig.json`.
- Never run `git stash`. Stage only files you edited, by explicit path. **`CLAUDE.md` has uncommitted user changes — do not touch it.**

---

## File structure

| File | Responsibility |
|---|---|
| `prisma/migrations/<ts>_faq_chunk_updated_by/migration.sql` | new — `updated_by` column |
| `prisma/schema.prisma` | modify — mirror it |
| `src/faq/faq-ingestion.service.ts` | modify — record `updated_by`; extract `list()` |
| `src/faq/faq-review.service.ts` | modify — pass the reviewer through as editor |
| `src/faq/faq.controller.ts` | modify — use the extracted `list()`; pass the actor; accept `reviewStatus` on create |
| `src/tenants/tenants.service.ts` | modify — per-slug FAQ delegations |
| `src/tenants/tenants.controller.ts` | modify — the superadmin FAQ routes |

---

### Task 1: `updated_by` — make an edit attributable

Today `updateOne` records no actor. `reviewed_by` is set only by approve and reject, so a superadmin editing another tenant's answer leaves no trace — and because the edit overwrites in place with no version, a wrong edit is both untraceable and unrecoverable (PRD 3 §12).

**Files:**
- Create: `prisma/migrations/20260903120000_faq_chunk_updated_by/migration.sql`
- Modify: `prisma/schema.prisma`, `src/faq/faq-ingestion.service.ts`, `src/faq/faq-review.service.ts`, `src/faq/faq.controller.ts`
- Test: `src/faq/faq-ingestion.service.spec.ts`

**Interfaces:**
- Produces: `updateOne(id, patch, tenantDb?, tenant?, updatedBy?)` — a fifth optional parameter. Consumed by Task 3.

- [ ] **Step 1: Write the migration**

Create `prisma/migrations/20260903120000_faq_chunk_updated_by/migration.sql`:

```sql
-- Quien toco el contenido por ultima vez. `reviewed_by` solo lo escriben approve()
-- y reject(), asi que una edicion manual no dejaba rastro de autor — y como
-- updateOne pisa en el lugar sin crear version, una edicion equivocada no se
-- podia ni atribuir ni deshacer (PRD 3 §12).
ALTER TABLE "FaqChunk" ADD COLUMN IF NOT EXISTS "updated_by" TEXT;
```

- [ ] **Step 2: Mirror it in `schema.prisma`**

In the `FaqChunk` model, beside `reviewed_by`:

```prisma
  updated_by     String?
```

Then `npx prisma generate`.

- [ ] **Step 3: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService.updateOne attribution', () => {
  function makeService() {
    const embeddingClient = { embed: jest.fn().mockResolvedValue({ vectors: [[0.1]], usage: {}, model: 'm' }) } as any;
    return new FaqIngestionService({} as any, embeddingClient);
  }

  const existing = {
    id: 'c1', question: '¿Cuanto tarda?', answer: 'Tarda 48 horas.',
    agents: [], tags: [], active: true, review_status: 'APPROVED', content_hash: 'viejo',
  };

  it('records who edited the content when the text changes', async () => {
    const service = makeService();
    const tenantDb = {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(existing), update: jest.fn() },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };

    await service.updateOne('c1', { answer: 'Tarda 72 horas.' }, tenantDb, undefined, 'user-7');

    const sql = tenantDb.$executeRaw.mock.calls[0][0].join('?');
    expect(sql).toContain('updated_by');
    expect(tenantDb.$executeRaw.mock.calls[0].slice(1)).toContain('user-7');
  });

  // El camino barato (solo agents/tags/active) tambien es una edicion y tambien
  // tiene autor: si no, cambiar el targeting de un chunk queda sin rastro.
  it('records the editor even when no re-embedding happens', async () => {
    const service = makeService();
    const tenantDb = {
      faqChunk: {
        findUnique: jest.fn().mockResolvedValue({ ...existing, content_hash: undefined }),
        update: jest.fn(),
      },
      $executeRaw: jest.fn(),
      aiUsage: { create: jest.fn() },
    };
    // mismo texto => mismo hash => rama sin re-embed
    const same = await service.updateOne('c1', { question: existing.question, answer: existing.answer }, tenantDb);
    void same;

    const service2 = makeService();
    const db2 = {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(existing), update: jest.fn() },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };
    await service2.updateOne('c1', { agents: ['ventas'] }, db2, undefined, 'user-9');
    // el hash no cambia => update() del cliente, no $executeRaw
    expect(db2.faqChunk.update).toHaveBeenCalled();
    expect(db2.faqChunk.update.mock.calls[0][0].data.updated_by).toBe('user-9');
  });

  it('leaves updated_by untouched when no actor is supplied', async () => {
    const service = makeService();
    const tenantDb = {
      faqChunk: { findUnique: jest.fn().mockResolvedValue(existing), update: jest.fn() },
      $executeRaw: jest.fn().mockResolvedValue(1),
      aiUsage: { create: jest.fn() },
    };

    await service.updateOne('c1', { answer: 'Tarda 72 horas.' }, tenantDb);

    expect(tenantDb.$executeRaw.mock.calls[0].slice(1)).toContain(null);
  });
});
```

- [ ] **Step 4: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — `updateOne` takes four parameters and writes no `updated_by`.

- [ ] **Step 5: Implement**

In `faq-ingestion.service.ts`, change `updateOne`'s signature and both write branches:

```ts
  async updateOne(
    id: string,
    patch: { question?: string; answer?: string; agents?: string[]; tags?: string[]; active?: boolean },
    tenantDb?: any,
    tenant?: any,
    updatedBy?: string,
  ) {
```

In the re-embedding branch, add `updated_by` to the `SET` list (keep every existing assignment):

```ts
      await db.$executeRaw`
        UPDATE "FaqChunk" SET question=${question}, answer=${answer}, agents=${agents}, tags=${tags},
          active=${active}, content_hash=${hash}, embedding=${vectorLiteral}::vector,
          updated_by=${updatedBy ?? null},
          embedded_at=NOW(), updated_at=NOW()
        WHERE id=${id}`;
```

And in the cheap branch:

```ts
      await db.faqChunk.update({ where: { id }, data: { agents, tags, active, updated_by: updatedBy ?? null } });
```

- [ ] **Step 6: Pass the actor through from the two callers**

In `faq-review.service.ts`, `approve()` already receives `reviewerId` and calls `updateOne` when the patch has edits. Pass it:

```ts
      chunk = await this.ingestion.updateOne(id, patch, tenantDb, tenant, reviewerId);
```

In `faq.controller.ts`, the `@Patch(':id')` handler passes `req.user?.id` as the new fifth argument. Read the handler first and keep its existing argument order intact.

- [ ] **Step 7: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 8: Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260903120000_faq_chunk_updated_by/migration.sql src/faq/faq-ingestion.service.ts src/faq/faq-review.service.ts src/faq/faq.controller.ts src/faq/faq-ingestion.service.spec.ts
git commit -m "feat(faq): record who edited a chunk, not just who approved it"
```

---

### Task 2: extract `FaqIngestionService.list()`

`GET /api/faq` builds its `where` and runs `findMany` inline in the controller. The superadmin route needs the same query, and duplicating a `select` list in a codebase where `tsc` cannot check column names is how the two drift apart silently.

**Files:**
- Modify: `src/faq/faq-ingestion.service.ts`, `src/faq/faq.controller.ts`
- Test: `src/faq/faq-ingestion.service.spec.ts`

**Interfaces:**
- Produces: `list(filters: FaqListFilters, tenantDb?: any)`. Consumed by Task 3 and by `FaqController`.

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq-ingestion.service.spec.ts`:

```ts
describe('FaqIngestionService.list', () => {
  function makeService() {
    return new FaqIngestionService({} as any, { embed: jest.fn() } as any);
  }

  it('returns everything when no filters are given', async () => {
    const service = makeService();
    const tenantDb = { faqChunk: { findMany: jest.fn().mockResolvedValue([]) } };
    await service.list({}, tenantDb);
    expect(tenantDb.faqChunk.findMany.mock.calls[0][0].where).toEqual({});
  });

  it('maps each filter onto the query', async () => {
    const service = makeService();
    const tenantDb = { faqChunk: { findMany: jest.fn().mockResolvedValue([]) } };
    await service.list(
      { active: true, agent: 'ventas', reviewStatus: 'PENDING_REVIEW', sourceType: 'DOCUMENT' },
      tenantDb,
    );
    expect(tenantDb.faqChunk.findMany.mock.calls[0][0].where).toEqual({
      active: true,
      agents: { has: 'ventas' },
      review_status: 'PENDING_REVIEW',
      source_type: 'DOCUMENT',
    });
  });

  // El panel de superadmin muestra origen y version; si el select no los trae,
  // esas columnas quedan vacias y nadie se entera hasta verlo en pantalla.
  it('selects the fields the admin UI needs, including provenance and version', async () => {
    const service = makeService();
    const tenantDb = { faqChunk: { findMany: jest.fn().mockResolvedValue([]) } };
    await service.list({}, tenantDb);
    const select = tenantDb.faqChunk.findMany.mock.calls[0][0].select;
    for (const field of ['id', 'question', 'answer', 'agents', 'tags', 'active',
                         'review_status', 'source_type', 'source_ref', 'version',
                         'updated_by', 'embedded_at', 'updated_at']) {
      expect(select[field]).toBe(true);
    }
  });

  it('returns the newest first', async () => {
    const service = makeService();
    const tenantDb = { faqChunk: { findMany: jest.fn().mockResolvedValue([]) } };
    await service.list({}, tenantDb);
    expect(tenantDb.faqChunk.findMany.mock.calls[0][0].orderBy).toEqual({ created_at: 'desc' });
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq-ingestion.service.spec.ts
```
Expected: FAIL — `service.list is not a function`.

- [ ] **Step 3: Implement**

Add to `faq-ingestion.service.ts`:

```ts
export interface FaqListFilters {
  active?: boolean;
  agent?: string;
  reviewStatus?: string;
  sourceType?: string;
}

/** Campos que ve el panel. `embedding` y `content_hash` no salen nunca: uno es
 *  enorme y el otro es interno. `source_ref` y `version` SI van — el panel
 *  muestra origen y version, y sin ellos esas columnas quedan vacias. */
const LIST_SELECT = {
  id: true, question: true, answer: true, agents: true, tags: true, active: true,
  review_status: true, source_type: true, source_ref: true, source_ordinal: true,
  source_span: true, version: true, reviewed_by: true, reviewed_at: true,
  updated_by: true, embedded_at: true, created_at: true, updated_at: true,
} as const;
```

and the method:

```ts
  // Listado para el panel. Antes vivia inline en el controller; lo comparten el
  // endpoint del tenant y el de superadmin, y duplicar el `select` en un codigo
  // donde tsc no chequea nombres de columna es como se desincronizan sin avisar.
  async list(filters: FaqListFilters, tenantDb?: any) {
    const where: any = {};
    if (filters.active !== undefined) where.active = filters.active;
    if (filters.agent) where.agents = { has: filters.agent };
    if (filters.reviewStatus) where.review_status = filters.reviewStatus;
    if (filters.sourceType) where.source_type = filters.sourceType;

    return this.db(tenantDb).faqChunk.findMany({
      where,
      orderBy: { created_at: 'desc' },
      select: LIST_SELECT,
    });
  }
```

- [ ] **Step 4: Use it from the controller**

Replace the body of `FaqController`'s `@Get()` handler so it keeps its existing `assertEnumParam` validation and then delegates:

```ts
    assertEnumParam(reviewStatus, Object.values(ReviewStatus), 'review_status');
    assertEnumParam(sourceType, Object.values(SourceType), 'source_type');

    return this.ingestion.list(
      {
        active: active === undefined ? undefined : active === 'true',
        agent,
        reviewStatus,
        sourceType,
      },
      req.tenantDb,
    );
```

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq/
npx tsc --noEmit -p tsconfig.json
```
The existing controller tests must still pass unchanged — if one asserted on the inline `findMany`, update the test to assert on `ingestion.list` rather than weakening it.

- [ ] **Step 6: Commit**

```bash
git add src/faq/faq-ingestion.service.ts src/faq/faq.controller.ts src/faq/faq-ingestion.service.spec.ts src/faq/faq.controller.spec.ts
git commit -m "refactor(faq): share one list query between the tenant and admin endpoints"
```

---

### Task 3: superadmin FAQ routes

The reason nothing in PRD 3 can work today: `/api/faq/*` resolves its tenant from request context, a superadmin has no tenant context, and forcing one via `X-Tenant-Slug` makes `JwtStrategy` look the superadmin up in the tenant's database, where they do not exist — a 401.

**Files:**
- Modify: `src/tenants/tenants.service.ts`, `src/tenants/tenants.controller.ts`
- Test: `src/tenants/tenants.service.spec.ts` (create if absent)

**Interfaces:**
- Consumes: `FaqIngestionService.list/updateOne/remove`, `FaqReviewService.listQueue/counts/approve/reject`, `FaqRetrievalService.retrieve` (Task 1 and 2 signatures).
- Produces: seven `/tenants/:slug/faq*` routes, consumed by the frontend plan.

- [ ] **Step 1: Write the failing test**

Create or append to `src/tenants/tenants.service.spec.ts`:

```ts
import { TenantsService } from './tenants.service';

describe('TenantsService FAQ delegation', () => {
  function makeService() {
    const tenantDb = { marker: 'tenant-db' } as any;
    const factory = { getClient: jest.fn().mockReturnValue(tenantDb) } as any;
    const ingestion = {
      list: jest.fn().mockResolvedValue([]),
      updateOne: jest.fn().mockResolvedValue({ id: 'c1' }),
      remove: jest.fn().mockResolvedValue({ id: 'c1' }),
    } as any;
    const review = {
      listQueue: jest.fn().mockResolvedValue([]),
      counts: jest.fn().mockResolvedValue({}),
      approve: jest.fn().mockResolvedValue({ id: 'c1' }),
      reject: jest.fn().mockResolvedValue({ id: 'c1' }),
    } as any;
    const prisma = {
      tenant: { findUnique: jest.fn().mockResolvedValue({ slug: 'itt', database_url: 'postgres://x', active: true }) },
    } as any;
    // Orden real del constructor: prisma, railway, funnel, agents, factory, ai,
    // y recien despues los tres de FAQ. Si esto queda desalineado el test pasa
    // igual (todo es `any`) pero inyecta el mock equivocado en cada slot.
    const service = new TenantsService(
      prisma, {} as any, {} as any, {} as any, factory, {} as any,
      ingestion, review, {} as any,
    );
    return { service, factory, ingestion, review, tenantDb };
  }

  it('resolves the tenant database from the slug and lists through it', async () => {
    const { service, factory, ingestion, tenantDb } = makeService();
    await service.getTenantFaq('itt', {});
    expect(factory.getClient).toHaveBeenCalledWith('postgres://x');
    expect(ingestion.list).toHaveBeenCalledWith({}, tenantDb);
  });

  it('passes the reviewer id through on approve', async () => {
    const { service, review, tenantDb } = makeService();
    await service.approveTenantFaq('itt', 'c1', { answer: 'nueva' }, 'user-7');
    expect(review.approve).toHaveBeenCalledWith('c1', { answer: 'nueva' }, tenantDb, expect.anything(), 'user-7');
  });

  // La atribucion es el unico rastro que deja una edicion (Task 1): si el slug
  // resuelve la base pero el actor se pierde, queda un cambio anonimo en los
  // datos de otro tenant.
  it('passes the editor id through on update', async () => {
    const { service, ingestion, tenantDb } = makeService();
    await service.updateTenantFaq('itt', 'c1', { answer: 'nueva' }, 'user-7');
    expect(ingestion.updateOne).toHaveBeenCalledWith('c1', { answer: 'nueva' }, tenantDb, expect.anything(), 'user-7');
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/tenants/
```
Expected: FAIL — the methods do not exist.

- [ ] **Step 3: Implement the service methods**

`TenantsService`'s constructor is, in this exact order:

```ts
  constructor(
    private readonly prisma: PrismaService,
    private readonly railway: RailwayService,
    private readonly funnel: FunnelService,
    private readonly agents: AgentsService,
    private readonly factory: TenantPrismaFactory,
    private readonly ai: AiService,
  ) {}
```

Append the three FAQ services **after** `ai`, keeping every existing parameter in place.

**Do not import `FaqModule` into `TenantsModule`.** `FaqModule` already imports `TenantsModule`, so that would be a circular import. It is unnecessary anyway: `FaqModule` is declared `@Global()` and exports these services, so they inject anywhere without the importing module declaring them — which is precisely how `AiService` already injects `FaqRetrievalService` without `AiModule` importing `FaqModule` (see the comment at the top of `faq.module.ts`). No module file changes in this task.

```ts
  // ─── FAQ del tenant (panel de superadmin) ──────────────────────────────────
  // Mismo patron que getTenantAgents: el slug resuelve la base, y de ahi para
  // abajo son los servicios de FAQ de siempre, sin logica nueva. La diferencia
  // con /api/faq es de donde sale el tenant — aca del path, alla del contexto
  // del request, que un superadmin no tiene (PRD 3 §4.1).
  private async faqDb(slug: string) {
    const tenant = await this.findBySlug(slug);
    return { tenant, db: this.factory.getClient(tenant.database_url) };
  }

  async getTenantFaq(slug: string, filters: FaqListFilters) {
    const { db } = await this.faqDb(slug);
    return this.faqIngestion.list(filters, db);
  }

  async updateTenantFaq(slug: string, id: string, patch: any, updatedBy?: string) {
    const { tenant, db } = await this.faqDb(slug);
    return this.faqIngestion.updateOne(id, patch, db, tenant, updatedBy);
  }

  async removeTenantFaq(slug: string, id: string) {
    const { db } = await this.faqDb(slug);
    return this.faqIngestion.remove(id, db);
  }

  async getTenantFaqQueue(slug: string, filters: any) {
    const { db } = await this.faqDb(slug);
    return this.faqReview.listQueue(filters, db);
  }

  async getTenantFaqCounts(slug: string) {
    const { db } = await this.faqDb(slug);
    return this.faqReview.counts(db);
  }

  async approveTenantFaq(slug: string, id: string, patch: any, reviewerId?: string) {
    const { tenant, db } = await this.faqDb(slug);
    return this.faqReview.approve(id, patch, db, tenant, reviewerId);
  }

  async rejectTenantFaq(slug: string, id: string, reviewerId?: string) {
    const { db } = await this.faqDb(slug);
    return this.faqReview.reject(id, db, reviewerId);
  }

  async testTenantFaq(slug: string, message: string, agentType?: string) {
    const { tenant, db } = await this.faqDb(slug);
    return this.faqRetrieval.retrieve(message, agentType ?? 'default', db, tenant);
  }
```

Use the existing constructor property names for the factory and prisma — read the file rather than assuming `this.factory`.

- [ ] **Step 4: Implement the routes**

In `tenants.controller.ts`, below the agents block. **Do not add any guard decorators** — the class already carries `@UseGuards(JwtAuthGuard, RolesGuard)` and `@Roles('superadmin')`, and re-declaring them per method is how one gets accidentally weakened later.

```ts
  // ─── FAQ / base de conocimiento del tenant ─────────────────────────────────
  // Solo superadmin (clase con @Roles). Es el unico camino por el que el panel
  // puede tocar la base de conocimiento de otro tenant.

  @Get(':slug/faq')
  getFaq(
    @Param('slug') slug: string,
    @Query('active') active: string | undefined,
    @Query('agent') agent: string | undefined,
    @Query('review_status') reviewStatus: string | undefined,
    @Query('source_type') sourceType: string | undefined,
  ) {
    return this.tenants.getTenantFaq(slug, {
      active: active === undefined ? undefined : active === 'true',
      agent,
      reviewStatus,
      sourceType,
    });
  }

  @Patch(':slug/faq/:id')
  updateFaq(
    @Param('slug') slug: string,
    @Param('id') id: string,
    @Body() body: { question?: string; answer?: string; agents?: string[]; tags?: string[]; active?: boolean },
    @Req() req: any,
  ) {
    // Allowlist explicito: `body` en runtime es lo que mande el cliente, y
    // updateOne escribe `active` si se lo pasan.
    const patch = {
      question: body?.question,
      answer: body?.answer,
      agents: body?.agents,
      tags: body?.tags,
      active: body?.active,
    };
    return this.tenants.updateTenantFaq(slug, id, patch, req.user?.id);
  }

  @Delete(':slug/faq/:id')
  removeFaq(@Param('slug') slug: string, @Param('id') id: string) {
    return this.tenants.removeTenantFaq(slug, id);
  }

  @Get(':slug/faq/review')
  getFaqQueue(
    @Param('slug') slug: string,
    @Query('review_status') reviewStatus: string | undefined,
    @Query('source_type') sourceType: string | undefined,
    @Query('source_ref') sourceRef: string | undefined,
  ) {
    return this.tenants.getTenantFaqQueue(slug, { reviewStatus, sourceType, sourceRef });
  }

  @Get(':slug/faq/review/counts')
  getFaqCounts(@Param('slug') slug: string) {
    return this.tenants.getTenantFaqCounts(slug);
  }

  @Post(':slug/faq/:id/approve')
  approveFaq(
    @Param('slug') slug: string,
    @Param('id') id: string,
    @Body() body: { question?: string; answer?: string; agents?: string[]; tags?: string[] },
    @Req() req: any,
  ) {
    // Mismo allowlist que /api/faq/:id/approve: sin esto un {"active": false}
    // llegaria hasta updateOne y desactivaria la fila al aprobarla.
    const patch = {
      question: body?.question,
      answer: body?.answer,
      agents: body?.agents,
      tags: body?.tags,
    };
    return this.tenants.approveTenantFaq(slug, id, patch, req.user?.id);
  }

  @Post(':slug/faq/:id/reject')
  rejectFaq(@Param('slug') slug: string, @Param('id') id: string, @Req() req: any) {
    return this.tenants.rejectTenantFaq(slug, id, req.user?.id);
  }

  @Post(':slug/faq/test')
  testFaq(@Param('slug') slug: string, @Body() body: { message: string; agentType?: string }) {
    return this.tenants.testTenantFaq(slug, body?.message ?? '', body?.agentType);
  }
```

**Route ordering matters:** `:slug/faq/review` and `:slug/faq/review/counts` must be declared *before* any `:slug/faq/:id` route that could shadow them, and `:slug/faq/test` likewise. As written above, `@Get(':slug/faq/review')` precedes no conflicting `@Get(':slug/faq/:id')` (there is none), and `@Post(':slug/faq/test')` does not collide with `@Post(':slug/faq/:id/approve')` because the path depth differs. Verify with a request to `/tenants/x/faq/review` after wiring.

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 6: Commit**

```bash
git add src/tenants/tenants.service.ts src/tenants/tenants.controller.ts src/tenants/tenants.service.spec.ts
git commit -m "feat(tenants): let a superadmin read and edit a tenant's FAQ from the panel"
```

---

### Task 4: turn retrieval on, and let manual entry go to the queue

Two small body-field changes with outsized effect. `faq_rag_enabled` defaults to `false` and is settable nowhere, so today **no tenant consults its FAQ at all**. And manual creation is born `APPROVED`, which is the only intake path that skips review.

**Files:**
- Modify: `src/tenants/tenants.controller.ts`, `src/tenants/tenants.service.ts`, `src/faq/faq.controller.ts`
- Test: `src/faq/faq.controller.spec.ts`

- [ ] **Step 1: Write the failing test**

Append to `src/faq/faq.controller.spec.ts`, following that file's existing `makeController` helper:

```ts
describe('manual creation review status', () => {
  it('creates as PENDING_REVIEW when asked to', async () => {
    const { controller, ingestion } = makeController();
    await controller.create({ question: '¿Q?', answer: 'A.', reviewStatus: 'PENDING_REVIEW' } as any, req);
    expect(ingestion.upsertBatch.mock.calls[0][1].reviewStatus).toBe('PENDING_REVIEW');
  });

  // Compatibilidad: quien no manda nada sigue creando aprobado, como hoy.
  it('still creates approved when nothing is asked for', async () => {
    const { controller, ingestion } = makeController();
    await controller.create({ question: '¿Q?', answer: 'A.' } as any, req);
    expect(ingestion.upsertBatch.mock.calls[0][1].reviewStatus).toBeUndefined();
  });

  it('rejects a review status that is not a real one', async () => {
    const { controller } = makeController();
    await expect(
      controller.create({ question: '¿Q?', answer: 'A.', reviewStatus: 'INVENTADO' } as any, req),
    ).rejects.toThrow();
  });
});
```

`makeController` may need `ingestion.upsertBatch` mocked to return `{ chunks: [{ id: 'x' }] }` — extend the existing helper rather than duplicating it.

- [ ] **Step 2: Run it, confirm it fails**

```bash
npm test -- src/faq/faq.controller.spec.ts
```

- [ ] **Step 3: Implement**

In `faq.controller.ts`'s `create`, accept and validate the field:

```ts
  async create(
    @Body() body: { question: string; answer: string; agents?: string[]; tags?: string[]; reviewStatus?: string },
    @Req() req: any,
  ) {
    const question = (body?.question ?? '').trim();
    const answer = (body?.answer ?? '').trim();
    if (!question || !answer) throw new BadRequestException('question y answer son obligatorios');
    assertEnumParam(body?.reviewStatus, Object.values(ReviewStatus), 'reviewStatus');
    const result = await this.ingestion.upsertBatch(
      [{ question, answer, agents: body.agents, tags: body.tags }],
      // Sin reviewStatus explicito upsertBatch cae en APPROVED, que es el
      // comportamiento historico de este endpoint. El panel manda
      // PENDING_REVIEW para que lo cargado a mano pase por la misma cola que
      // todo lo demas (PRD 3 §4.1).
      { sourceType: 'MANUAL', reviewStatus: body?.reviewStatus as any },
      req.tenantDb,
      req.tenant,
    );
    return result.chunks[0];
  }
```

Confirm by reading `upsertBatch` that passing `reviewStatus: undefined` still yields the `?? 'APPROVED'` default — the second test above depends on it.

- [ ] **Step 4: Let the flag be set — and stop the handler writing whatever it is sent**

`PATCH /tenants/:slug` currently does this:

```ts
  update(@Param('slug') slug: string, @Body() body: { name?: string; /* … */ }) {
    return this.tenants.update(slug, body);          // ← body goes straight to prisma
  }
```

and the service does `this.prisma.tenant.update({ where: { slug }, data })`. **Both "allowlists" are TypeScript parameter types, which do not exist at runtime**, so any field a caller sends is written to the `Tenant` row — including `database_url`, `verify_token`, `slug` and `active`.

It is a superadmin-only endpoint, so this is not privilege escalation. The realistic failure is the UI this PRD is about to build: a screen that loads a tenant, flips one boolean and `PATCH`es the object back would rewrite `database_url` from stale client state, silently repointing a tenant at the wrong database. Adding a field to this handler without fixing the pass-through would be building directly on top of that.

Build the data explicitly, the same way the FAQ routes in Task 3 do:

```ts
  @Patch(':slug')
  update(
    @Param('slug') slug: string,
    @Body() body: {
      name?: string;
      openrouter_api_key?: string;
      openrouter_model?: string;
      orchestrator_model?: string;
      faq_rag_enabled?: boolean;
    },
  ) {
    // El tipo de `body` es solo de compilacion: en runtime llega lo que el
    // cliente mande, y esto iba derecho a prisma.tenant.update. Un campo de mas
    // —`database_url`, `slug`, `active`— se escribia igual. Se pasan solo las
    // claves conocidas. Prisma ignora las `undefined`, asi que omitir una en el
    // body sigue significando "no la toques".
    return this.tenants.update(slug, {
      name: body?.name,
      openrouter_api_key: body?.openrouter_api_key,
      openrouter_model: body?.openrouter_model,
      orchestrator_model: body?.orchestrator_model,
      faq_rag_enabled: body?.faq_rag_enabled,
    });
  }
```

Then add `faq_rag_enabled?: boolean` to `TenantsService.update`'s `data` parameter type so the new field type-checks.

- [ ] **Step 4b: Test the allowlist**

Add to `src/tenants/tenants.service.spec.ts` (the file Task 3 created), testing the controller:

```ts
describe('TenantsController.update allowlist', () => {
  it('passes the known fields through', () => {
    const tenants = { update: jest.fn() } as any;
    const controller = new TenantsController(tenants);
    controller.update('itt', { name: 'Nuevo', faq_rag_enabled: true } as any);
    const data = tenants.update.mock.calls[0][1];
    expect(data.name).toBe('Nuevo');
    expect(data.faq_rag_enabled).toBe(true);
  });

  // El caso que importa: el tipo de `body` no existe en runtime, y esto iba
  // derecho a prisma.tenant.update. Un `database_url` de mas repuntaba al
  // tenant a otra base, en silencio.
  it('drops fields the caller must not be able to write', () => {
    const tenants = { update: jest.fn() } as any;
    const controller = new TenantsController(tenants);
    controller.update('itt', {
      name: 'Nuevo',
      database_url: 'postgres://otro',
      slug: 'otro-tenant',
      active: false,
      verify_token: 'robado',
    } as any);
    const data = tenants.update.mock.calls[0][1];
    expect('database_url' in data).toBe(false);
    expect('slug' in data).toBe(false);
    expect('active' in data).toBe(false);
    expect('verify_token' in data).toBe(false);
  });
});
```

`TenantsController`'s constructor takes only `TenantsService` — check it before writing the test rather than assuming.

- [ ] **Step 5: Both gates**

```bash
npm test -- src/faq/ src/tenants/
npx tsc --noEmit -p tsconfig.json
```

- [ ] **Step 6: Commit**

```bash
git add src/faq/faq.controller.ts src/tenants/tenants.controller.ts src/tenants/tenants.service.ts src/faq/faq.controller.spec.ts
git commit -m "feat(faq): let the panel enable retrieval and send manual entries to review"
```

---

## Self-review notes

**Spec coverage:** PRD 3 §4.1 is Tasks 2 and 3 (the routes plus the shared list they need); §4.2 is Task 4; the `updated_by` mitigation recommended in §12 and folded into phase 0 is Task 1; the manual-create decision under §4.1's route table is Task 4.

**Deliberately not here:**
- Nothing changes about retrieval, thresholds, or prompt assembly (PRD 3 §3).
- No pagination. PRD 3 §6.5 accepts unbounded lists for phase 1 and explains the coupling to client-side search; adding `take`/`skip` now would silently break that.
- The duplicated 16-column INSERTs in `faq-ingestion.service.ts` stay untouched — a parked item from the previous branch, verified benign twice, and not something to fold into a plan that is already changing this file.

**Known risk:** Task 3 adds three service dependencies to `TenantsService`, which means a constructor arity change. `ts-jest` will not catch a stale `new TenantsService(...)` in a spec — only `tsc` will. Both gates run on every task for this reason.
