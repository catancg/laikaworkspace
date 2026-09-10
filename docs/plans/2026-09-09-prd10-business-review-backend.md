# PRD 10 Phase 2 — Business info review gate (backend) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make a tenant admin's edits to business info a *proposal* that SoyLaika must approve, without changing anything the bot reads.

**Architecture:** Draft/live split. `BusinessProfile` stays exactly as it is and remains the only thing `AiService` reads. A new `BusinessProfileDraft` table holds the customer's pending submission. Approving copies the draft's fields onto the live row and deletes the draft; rejecting leaves live untouched and records a note the customer can see.

**Tech Stack:** NestJS + Prisma + PostgreSQL, Jest. One hand-written SQL migration.

**Spec:** [docs/prd-customer-agent-config-boundary.md](../prd-customer-agent-config-boundary.md) — §2.2, §2.3, §2.4, §2.5, §3.1, §3.3, §3.4, §4.2, §4.3, §4.4, §5

**Scope:** Backend only. No frontend work — Phase 3 owns every screen. Rules are already locked down by Phase 1 and are not touched here.

## Global Constraints

- **Stage explicit paths.** Never `git add -A`, `git add .`, or `git stash` — this repo carries an uncommitted local `CLAUDE.md` edit that is not recoverable from the remote.
- **Backend lint is `npx eslint <paths>`, never `npm run lint`** — that script is `eslint --fix` and has reformatted 93 untouched files in one run. The repo has heavy pre-existing lint debt; the gate is **no new error in a file this task changed**, not a clean run. Report remaining errors grouped by originating file.
- **Never `npm run build`** while the dev server is running; type-check with `npx tsc --noEmit -p tsconfig.json`.
- **Comments, log messages and error strings are Spanish.**
- **The migration directory name is both ordering and identity.** `TenantMigrationsService` sorts `prisma/migrations/` lexicographically and records applied names in each tenant's `_tenant_migrations`. Renaming a pushed migration makes it run again. Once committed, the name is frozen.
- **Never run migrations or destructive SQL against an existing tenant or production database.** Create a throwaway tenant, use it, drop it.
- **`AiService` must not learn what a draft is.** No file under `src/ai/` may reference `businessProfileDraft`. This is the property that makes the change incapable of blanking the business block in a live conversation.
- Request bodies are **not validated** — there is no global `ValidationPipe`. Every write path needs a **named allowlist**. Do not add `app.useGlobalPipes(new ValidationPipe(...))`; almost no endpoint here has a DTO.

## File Structure

| File | Responsibility |
|---|---|
| `prisma/schema.prisma` | add the `BusinessProfileDraft` model |
| `prisma/migrations/20260909120000_business_profile_draft/migration.sql` | create the table |
| `src/business/business.service.ts` | draft read/write, approve, reject; named allowlist for both live and draft writes |
| `src/business/business.service.spec.ts` | create — allowlist, draft lifecycle, approve/reject |
| `src/business/business.controller.ts` | `PUT` routes draft-vs-live by role; new `GET /api/business/draft` |
| `src/ai/ai.service.ts` | add `invalidateBusinessCache` beside `invalidateAgentCache` |
| `src/ai/ai.service.business-isolation.spec.ts` | create — proves the bot never reads the draft |
| `src/tenants/tenants.service.ts` | superadmin review methods |
| `src/tenants/tenants.controller.ts` | superadmin review routes |
| `src/tenants/tenants.service.business.spec.ts` | create — delegation, cache invalidation, pending count |

`BusinessProfile`'s columns and every `businessService.get()` call site are **unchanged**.

---

### Task 1: Schema and migration

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/20260909120000_business_profile_draft/migration.sql`

**Interfaces:**
- Produces: the `businessProfileDraft` Prisma delegate with the twelve content columns plus `review_status`, `submitted_by`, `submitted_at`, `reviewed_by`, `reviewed_at`, `reject_note`. Every later task depends on these exact names.

- [ ] **Step 1: Add the model**

Append to `prisma/schema.prisma`, next to `BusinessProfile`:

```prisma
// PRD 10 §3.1. La propuesta del cliente vive aca hasta que SoyLaika la aprueba.
// BusinessProfile sigue siendo lo unico que lee el bot: por eso el split.
// Como maximo una fila por tenant (singleton, igual que BusinessProfile).
model BusinessProfileDraft {
  id             String   @id @default(uuid())
  name           String?
  about          String?
  hours          String?
  address        String?
  branches       String?
  phone          String?
  email          String?
  website        String?
  paymentMethods String?
  shippingInfo   String?
  returnPolicy   String?
  extra          String?

  review_status ReviewStatus @default(PENDING_REVIEW)
  submitted_by  String?
  submitted_at  DateTime     @default(now())
  reviewed_by   String?
  reviewed_at   DateTime?
  reject_note   String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([review_status])
}
```

- [ ] **Step 2: Write the migration**

Create `prisma/migrations/20260909120000_business_profile_draft/migration.sql`:

```sql
-- PRD 10 fase 2: la info del negocio que carga el cliente pasa a ser una
-- propuesta que SoyLaika aprueba. Esta migracion SOLO agrega una tabla: no
-- toca BusinessProfile ni una sola fila existente. Por eso el grandfathering
-- de §2.4 sale gratis — lo que ya esta cargado ES el registro aprobado — y por
-- eso es segura aunque TenantMigrationsService la aplique sola al arranque:
-- el peor caso es un tenant con una tabla vacia.

-- El enum ya existe en toda base que tenga FaqChunk. El guard es por si esta
-- migracion llega a una base donde el FAQ nunca se aplico.
DO $$ BEGIN
  CREATE TYPE "ReviewStatus" AS ENUM ('DRAFT', 'PENDING_REVIEW', 'APPROVED', 'ARCHIVED');
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

CREATE TABLE IF NOT EXISTS "BusinessProfileDraft" (
  "id"             TEXT NOT NULL,
  "name"           TEXT,
  "about"          TEXT,
  "hours"          TEXT,
  "address"        TEXT,
  "branches"       TEXT,
  "phone"          TEXT,
  "email"          TEXT,
  "website"        TEXT,
  "paymentMethods" TEXT,
  "shippingInfo"   TEXT,
  "returnPolicy"   TEXT,
  "extra"          TEXT,
  "review_status"  "ReviewStatus" NOT NULL DEFAULT 'PENDING_REVIEW',
  "submitted_by"   TEXT,
  "submitted_at"   TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "reviewed_by"    TEXT,
  "reviewed_at"    TIMESTAMP(3),
  "reject_note"    TEXT,
  "createdAt"      TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "updatedAt"      TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT "BusinessProfileDraft_pkey" PRIMARY KEY ("id")
);

CREATE INDEX IF NOT EXISTS "BusinessProfileDraft_review_status_idx"
  ON "BusinessProfileDraft" ("review_status");
```

- [ ] **Step 3: Regenerate the client and type-check**

Run: `npx prisma generate`
Then: `npx tsc --noEmit -p tsconfig.json`
Expected: clean. If `businessProfileDraft` is missing from the generated client, the model was not saved — fix before continuing.

- [ ] **Step 4: Apply it to a throwaway database, not to a real tenant**

Create a throwaway tenant through the API (superadmin), confirm the table exists in its database, then delete the tenant. Do **not** point this migration at `tenant-dev` by hand — the dev server applies it on its own at boot, and hand-running SQL against an existing tenant is what the root CLAUDE.md forbids.

```bash
docker exec <postgres-container> psql -U postgres -d <throwaway-db> -c '\d "BusinessProfileDraft"'
```
Expected: the table with all nineteen columns.

- [ ] **Step 5: Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260909120000_business_profile_draft/
git commit -m "feat(negocio): tabla de borrador para la info del negocio (PRD 10 fase 2)"
```

---

### Task 2: Draft write path

**Files:**
- Modify: `src/business/business.service.ts`
- Modify: `src/business/business.controller.ts`
- Test: `src/business/business.service.spec.ts`

**Interfaces:**
- Consumes: the `businessProfileDraft` delegate from Task 1.
- Produces, for Task 3 and Phase 3:
  - `BusinessService.getDraft(tenantDb?) → draft | null`
  - `BusinessService.upsertDraft(data, submittedBy?, tenantDb?) → draft`
  - `BusinessService.approveDraft(reviewerId?, tenantDb?) → live profile`
  - `BusinessService.rejectDraft(note, reviewerId?, tenantDb?) → draft`
  - `PUT /api/business` — draft for `admin`, live for `superadmin`
  - `GET /api/business/draft` → draft | null

- [ ] **Step 1: Write the failing test**

Create `src/business/business.service.spec.ts`:

```ts
import { NotFoundException } from '@nestjs/common';
import { BusinessService } from './business.service';

// PRD 10 §4.2: el control es que un PUT del cliente NO toque BusinessProfile.
// Todo lo demas es plomeria.
function makeDb(live: any = null, draft: any = null) {
  return {
    businessProfile: {
      findFirst: jest.fn().mockResolvedValue(live),
      update: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'live-1', ...data })),
      create: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'live-new', ...data })),
    },
    businessProfileDraft: {
      findFirst: jest.fn().mockResolvedValue(draft),
      update: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'draft-1', ...data })),
      create: jest.fn().mockImplementation(({ data }) => Promise.resolve({ id: 'draft-new', ...data })),
      delete: jest.fn().mockResolvedValue({}),
    },
  };
}

describe('BusinessService', () => {
  const svc = new BusinessService({} as any);

  it('upsertDraft no escribe BusinessProfile', async () => {
    const db = makeDb({ id: 'live-1', name: 'Original' });
    await svc.upsertDraft({ name: 'Propuesto' }, 'user-1', db);
    expect(db.businessProfile.update).not.toHaveBeenCalled();
    expect(db.businessProfile.create).not.toHaveBeenCalled();
    expect(db.businessProfileDraft.create).toHaveBeenCalled();
  });

  it('upsertDraft aplica allowlist: descarta campos que no son del negocio', async () => {
    const db = makeDb();
    await svc.upsertDraft({ name: 'X', id: 'otro', review_status: 'APPROVED', evil: 1 } as any, 'u', db);
    const data = db.businessProfileDraft.create.mock.calls[0][0].data;
    expect(data.name).toBe('X');
    expect(data.id).toBeUndefined();
    expect(data.review_status).toBe('PENDING_REVIEW');
    expect((data as any).evil).toBeUndefined();
  });

  it('upsert (live) tambien aplica allowlist', async () => {
    const db = makeDb({ id: 'live-1' });
    await svc.upsert({ name: 'X', id: 'otro', evil: 1 } as any, db);
    const data = db.businessProfile.update.mock.calls[0][0].data;
    expect(data).toEqual({ name: 'X' });
  });

  it('editar despues de un rechazo vuelve a PENDING_REVIEW y limpia la nota', async () => {
    const db = makeDb(null, { id: 'draft-1', review_status: 'ARCHIVED', reject_note: 'faltan horarios' });
    await svc.upsertDraft({ name: 'Corregido' }, 'u', db);
    const data = db.businessProfileDraft.update.mock.calls[0][0].data;
    expect(data.review_status).toBe('PENDING_REVIEW');
    expect(data.reject_note).toBeNull();
  });

  it('approveDraft copia los campos a live y borra el borrador', async () => {
    const db = makeDb({ id: 'live-1', name: 'Viejo' }, { id: 'draft-1', name: 'Nuevo', hours: '9 a 18' });
    await svc.approveDraft('rev-1', db);
    const data = db.businessProfile.update.mock.calls[0][0].data;
    expect(data.name).toBe('Nuevo');
    expect(data.hours).toBe('9 a 18');
    expect(db.businessProfileDraft.delete).toHaveBeenCalledWith({ where: { id: 'draft-1' } });
  });

  it('approveDraft CREA el perfil si el tenant todavia no tiene uno', async () => {
    const db = makeDb(null, { id: 'draft-1', name: 'Nuevo' });
    await svc.approveDraft('rev-1', db);
    expect(db.businessProfile.create).toHaveBeenCalled();
    expect(db.businessProfile.update).not.toHaveBeenCalled();
  });

  it('approveDraft sin borrador tira NotFound', async () => {
    await expect(svc.approveDraft('rev-1', makeDb())).rejects.toThrow(NotFoundException);
  });

  it('rejectDraft deja live intacto y guarda la nota', async () => {
    const db = makeDb({ id: 'live-1', name: 'Original' }, { id: 'draft-1', name: 'Propuesto' });
    await svc.rejectDraft('faltan horarios', 'rev-1', db);
    expect(db.businessProfile.update).not.toHaveBeenCalled();
    const data = db.businessProfileDraft.update.mock.calls[0][0].data;
    expect(data.review_status).toBe('ARCHIVED');
    expect(data.reject_note).toBe('faltan horarios');
    expect(data.reviewed_by).toBe('rev-1');
  });

  it('getDraft devuelve null si la tabla todavia no existe en esa base', async () => {
    const db = makeDb();
    db.businessProfileDraft.findFirst = jest.fn().mockRejectedValue(new Error('relation does not exist'));
    await expect(svc.getDraft(db)).resolves.toBeNull();
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npm test -- src/business/business.service.spec.ts`
Expected: FAIL — `svc.upsertDraft is not a function`.

- [ ] **Step 3: Implement the service**

Replace the body of `src/business/business.service.ts` with:

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

// Los unicos campos de contenido que se pueden escribir, en live o en borrador.
// Allowlist explicito porque en runtime `body` es lo que mande el cliente y no
// hay ValidationPipe global: sin esto, un PUT podia pisar `id`.
const CAMPOS = [
  'name', 'about', 'hours', 'address', 'branches', 'phone',
  'email', 'website', 'paymentMethods', 'shippingInfo', 'returnPolicy', 'extra',
] as const;

type BusinessData = Partial<Record<(typeof CAMPOS)[number], string>>;

function soloCampos(input: any): BusinessData {
  const out: any = {};
  for (const k of CAMPOS) if (input?.[k] !== undefined) out[k] = input[k];
  return out;
}

@Injectable()
export class BusinessService {
  constructor(private readonly prisma: PrismaService) {}

  private db(tenantDb?: any) { return tenantDb ?? this.prisma; }

  // Perfil aprobado (singleton por base de tenant) o null. ESTO es lo unico que
  // lee el bot; el borrador no entra nunca aca (PRD 10 §3.4).
  async get(tenantDb?: any) {
    return this.db(tenantDb).businessProfile.findFirst();
  }

  // Escribe el perfil aprobado. Solo lo llama un superadmin (§2.5) o approveDraft.
  async upsert(data: BusinessData, tenantDb?: any) {
    const db = this.db(tenantDb);
    const limpio = soloCampos(data);
    const existing = await db.businessProfile.findFirst();
    if (existing) {
      return db.businessProfile.update({ where: { id: existing.id }, data: limpio });
    }
    return db.businessProfile.create({ data: limpio });
  }

  // Borrador pendiente del tenant, o null. Si la base todavia no tomo la
  // migracion, no rompe el CRM: devuelve null (mismo criterio que
  // RulesService.listActive).
  async getDraft(tenantDb?: any) {
    return this.db(tenantDb).businessProfileDraft.findFirst().catch(() => null);
  }

  // Propuesta del cliente. Un borrador por tenant; editar despues de un rechazo
  // lo reabre como pendiente y limpia la nota (§5).
  async upsertDraft(data: BusinessData, submittedBy?: string, tenantDb?: any) {
    const db = this.db(tenantDb);
    const payload = {
      ...soloCampos(data),
      review_status: 'PENDING_REVIEW' as const,
      submitted_by: submittedBy ?? null,
      submitted_at: new Date(),
      reviewed_by: null,
      reviewed_at: null,
      reject_note: null,
    };
    const existing = await db.businessProfileDraft.findFirst();
    if (existing) {
      return db.businessProfileDraft.update({ where: { id: existing.id }, data: payload });
    }
    return db.businessProfileDraft.create({ data: payload });
  }

  // Aprobar: los campos del borrador pasan a live y el borrador se borra.
  // upsert crea el perfil si el tenant todavia no tenia uno (§5).
  async approveDraft(reviewerId?: string, tenantDb?: any) {
    const db = this.db(tenantDb);
    const draft = await db.businessProfileDraft.findFirst();
    if (!draft) throw new NotFoundException('No hay cambios pendientes de aprobación');
    const live = await this.upsert(soloCampos(draft), tenantDb);
    await db.businessProfileDraft.delete({ where: { id: draft.id } });
    return live;
  }

  // Rechazar: live no se toca. El borrador queda con la nota para que el cliente
  // vea por que no se aplico (si no, lo vuelve a tipear y abre un ticket).
  async rejectDraft(note: string | null, reviewerId?: string, tenantDb?: any) {
    const db = this.db(tenantDb);
    const draft = await db.businessProfileDraft.findFirst();
    if (!draft) throw new NotFoundException('No hay cambios pendientes de aprobación');
    return db.businessProfileDraft.update({
      where: { id: draft.id },
      data: {
        review_status: 'ARCHIVED',
        reviewed_by: reviewerId ?? null,
        reviewed_at: new Date(),
        reject_note: note,
      },
    });
  }
}
```

`ARCHIVED` is the rejected state, matching what `faq-review.service.ts:170` already does for a rejected chunk — same vocabulary, not a second one.

- [ ] **Step 4: Route the controller by role**

In `src/business/business.controller.ts`, replace the `@Put()` handler and add the draft read. `@Get('draft')` must be declared before any parameterised GET (there is none today, but keep it above `@Get()` for safety):

```ts
  // Borrador propio del tenant: el cliente ve que mando y, si se lo rechazaron,
  // por que. Cualquier usuario autenticado del tenant puede leerlo.
  @Get('draft')
  getDraft(@Req() req: any) {
    return this.business.getDraft(req.tenantDb);
  }

  // El admin del tenant PROPONE; el superadmin escribe live directo (PRD 10 §2.5:
  // hacer que SoyLaika se apruebe a si misma es ceremonia sin control).
  @Put()
  @UseGuards(RolesGuard)
  @Roles('admin', 'superadmin')
  upsert(
    @Body() body: {
      name?: string; about?: string; hours?: string; address?: string;
      branches?: string; phone?: string; email?: string; website?: string;
      paymentMethods?: string; shippingInfo?: string; returnPolicy?: string; extra?: string;
    },
    @Req() req: any,
  ) {
    if (req.user?.role === 'superadmin') return this.business.upsert(body, req.tenantDb);
    return this.business.upsertDraft(body, req.user?.id, req.tenantDb);
  }
```

Add `Get` and `Req` to the `@nestjs/common` import if they are not already there.

**A deliberate gap, do not "fix" it:** a superadmin's direct live write does not invalidate the prompt cache, so it can take up to 10s (`businessCaches` TTL) to apply. `BusinessService` has no `AiService` and injecting one risks a cycle (`AiService` already depends on `BusinessService`). The approve path — the one customers' changes actually travel through — *does* invalidate, in Task 3.

- [ ] **Step 5: Run the tests and gates**

Run: `npm test -- src/business/business.service.spec.ts` → PASS, 9 tests.
Run: `npm test` → no previously-passing test now failing.
Run: `npx tsc --noEmit -p tsconfig.json` → clean.
Run: `npx eslint src/business` → no new error in a file you changed; report the rest by file.

- [ ] **Step 6: Commit**

```bash
git add src/business/business.service.ts src/business/business.controller.ts src/business/business.service.spec.ts
git commit -m "feat(negocio): el cliente propone, no escribe (PRD 10 fase 2)"
```

---

### Task 3: Superadmin review endpoints

**Files:**
- Modify: `src/ai/ai.service.ts`
- Modify: `src/tenants/tenants.service.ts`
- Modify: `src/tenants/tenants.controller.ts`
- Test: `src/tenants/tenants.service.business.spec.ts`

**Interfaces:**
- Consumes: `BusinessService.{getDraft,approveDraft,rejectDraft}` from Task 2; `AiService.invalidateBusinessCache` from this task's Step 1.
- Produces, for Phase 3:
  - `GET /tenants/:slug/business` → `{ live, draft }`
  - `POST /tenants/:slug/business/approve` → live profile
  - `POST /tenants/:slug/business/reject` body `{ note?: string }` → draft
  - `GET /tenants/business/pending-count` → `Record<slug, number>`

- [ ] **Step 1: Add cache invalidation**

In `src/ai/ai.service.ts`, next to `invalidateRulesCache`:

```ts
  // Igual que invalidateRulesCache: al aprobar un cambio queremos que el bot lo
  // use ya, no cuando venza el TTL de 10s.
  invalidateBusinessCache(db?: any) {
    this.businessCaches.delete(db ?? this.prisma);
  }
```

- [ ] **Step 2: Write the failing test**

Create `src/tenants/tenants.service.business.spec.ts`:

```ts
import { TenantsService } from './tenants.service';

// PRD 10 §3.3/§3.4. Lo que importa: TenantsService delega en BusinessService y
// invalida la cache del prompt al aprobar; rechazar NO la invalida porque live
// no cambio.
describe('TenantsService — revision de info del negocio', () => {
  const db = { marker: 'tenant-db' };
  let business: any;
  let ai: any;
  let service: TenantsService;

  beforeEach(() => {
    business = {
      get: jest.fn().mockResolvedValue({ id: 'live-1', name: 'Vivo' }),
      getDraft: jest.fn().mockResolvedValue({ id: 'draft-1', name: 'Propuesto' }),
      approveDraft: jest.fn().mockResolvedValue({ id: 'live-1', name: 'Propuesto' }),
      rejectDraft: jest.fn().mockResolvedValue({ id: 'draft-1', review_status: 'ARCHIVED' }),
    };
    ai = { invalidateBusinessCache: jest.fn() };

    service = Object.create(TenantsService.prototype);
    (service as any).business = business;
    (service as any).ai = ai;
    (service as any).logger = { log: jest.fn() };
    (service as any).factory = { getClient: jest.fn().mockReturnValue(db) };
    (service as any).findBySlug = jest.fn().mockResolvedValue({ database_url: 'postgres://x' });
  });

  it('devuelve live y draft juntos para poder diffear', async () => {
    await expect(service.getTenantBusiness('acme')).resolves.toEqual({
      live: { id: 'live-1', name: 'Vivo' },
      draft: { id: 'draft-1', name: 'Propuesto' },
    });
  });

  it('aprobar delega e invalida la cache del prompt', async () => {
    await service.approveTenantBusiness('acme', 'rev-1');
    expect(business.approveDraft).toHaveBeenCalledWith('rev-1', db);
    expect(ai.invalidateBusinessCache).toHaveBeenCalledWith(db);
  });

  it('rechazar delega y NO invalida la cache (live no cambio)', async () => {
    await service.rejectTenantBusiness('acme', 'faltan horarios', 'rev-1');
    expect(business.rejectDraft).toHaveBeenCalledWith('faltan horarios', 'rev-1', db);
    expect(ai.invalidateBusinessCache).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 3: Run it to verify it fails**

Run: `npm test -- src/tenants/tenants.service.business.spec.ts`
Expected: FAIL — `service.getTenantBusiness is not a function`.

- [ ] **Step 4: Add the service methods**

In `src/tenants/tenants.service.ts`, after the rules block added in Phase 1:

```ts
  // ─── Info del negocio: revision (DB del tenant; solo superadmin) ───────────
  // PRD 10 §3.3. El cliente propone en su CRM; aca se aprueba o se rechaza.

  async getTenantBusiness(slug: string) {
    const tenant = await this.findBySlug(slug);
    const db = this.factory.getClient(tenant.database_url);
    const [live, draft] = await Promise.all([
      this.business.get(db),
      this.business.getDraft(db),
    ]);
    return { live, draft };
  }

  async approveTenantBusiness(slug: string, reviewerId?: string) {
    const tenant = await this.findBySlug(slug);
    const db = this.factory.getClient(tenant.database_url);
    const live = await this.business.approveDraft(reviewerId, db);
    this.ai.invalidateBusinessCache(db);
    this.logger.log(`[${slug}] Info del negocio aprobada`);
    return live;
  }

  // Rechazar no toca live, asi que no hay cache que invalidar.
  async rejectTenantBusiness(slug: string, note: string | null, reviewerId?: string) {
    const tenant = await this.findBySlug(slug);
    const db = this.factory.getClient(tenant.database_url);
    const draft = await this.business.rejectDraft(note, reviewerId, db);
    this.logger.log(`[${slug}] Info del negocio rechazada`);
    return draft;
  }

  // Cuantos tenants tienen cambios esperando. Alimenta el badge del listado.
  // Una base que todavia no tomo la migracion no rompe el listado entero.
  async businessPendingCount(): Promise<Record<string, number>> {
    const tenants = await this.prisma.tenant.findMany({ where: { active: true } });
    const out: Record<string, number> = {};
    for (const t of tenants) {
      try {
        const db = this.factory.getClient(t.database_url);
        const n = await db.businessProfileDraft.count({ where: { review_status: 'PENDING_REVIEW' } });
        if (n > 0) out[t.slug] = n;
      } catch {
        // base vieja sin la tabla, o inalcanzable: se omite, no se propaga.
      }
    }
    return out;
  }
```

Add `BusinessService` to the constructor beside the Phase 1 `rules` parameter:

```ts
    private readonly business: BusinessService,
```

and the import:

```ts
import { BusinessService } from '../business/business.service';
```

`BusinessModule` is `@Global()` and exports `BusinessService`, and it already imports `TenantsModule` — so **do not add `BusinessModule` to `TenantsModule`**; that would create a cycle. This is the same wiring Phase 1 used for `RulesService`. If Nest raises a resolution error at boot, use `@Inject(forwardRef(() => BusinessService))` rather than a module import.

- [ ] **Step 5: Add the controller routes**

In `src/tenants/tenants.controller.ts`, after the Phase 1 rules routes:

```ts
  // ─── Info del negocio: revision (DB del tenant) ─────────────────────────────
  // Solo superadmin (clase con @Roles). El cliente carga; aca se aprueba.

  @Get('business/pending-count')
  businessPendingCount() {
    return this.tenants.businessPendingCount();
  }

  @Get(':slug/business')
  getBusiness(@Param('slug') slug: string) {
    return this.tenants.getTenantBusiness(slug);
  }

  @Post(':slug/business/approve')
  approveBusiness(@Param('slug') slug: string, @Req() req: any) {
    return this.tenants.approveTenantBusiness(slug, req.user?.id);
  }

  @Post(':slug/business/reject')
  rejectBusiness(@Param('slug') slug: string, @Body() body: { note?: string }, @Req() req: any) {
    return this.tenants.rejectTenantBusiness(slug, (body?.note ?? '').trim() || null, req.user?.id);
  }
```

`business/pending-count` and `:slug/business` are both two-segment routes but are disjoint — the second segment is a literal in each and they differ — so ordering between them is not load-bearing. Declaring the literal one first anyway matches how `@Get('expiring')` sits above the parameterised routes in this file.

- [ ] **Step 6: Add the role assertion**

The class-level `@Roles('superadmin')` is the only thing gating these. As Phase 1 established, a method-level `@Roles` would *widen* it silently. Extend `src/tenants/tenants.controller.rules.spec.ts` (created in Phase 1) with the four new handlers, or add an equivalent block asserting:

```ts
for (const h of ['businessPendingCount', 'getBusiness', 'approveBusiness', 'rejectBusiness'] as const) {
  expect(reflector.getAllAndOverride('roles',
    [TenantsController.prototype[h], TenantsController])).toEqual(['superadmin']);
}
```

- [ ] **Step 7: Run tests and gates**

Run: `npm test -- src/tenants` → PASS.
Run: `npm test` → no regression.
Run: `npx tsc --noEmit -p tsconfig.json` → clean.
Run: `npx eslint src/tenants src/ai src/business` → no new error in a changed file.

Then confirm the app booted: the new routes should answer **401** unauthenticated (route registered) rather than **404** (not registered).

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/tenants/foo/business
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/tenants/foo/nope
```
Expected: `401` then `404`.

- [ ] **Step 8: Commit**

```bash
git add src/ai/ai.service.ts src/tenants/tenants.service.ts src/tenants/tenants.controller.ts src/tenants/tenants.service.business.spec.ts src/tenants/tenants.controller.rules.spec.ts
git commit -m "feat(negocio): endpoints de revision para superadmin (PRD 10 fase 2)"
```

---

### Task 4: Prove the bot never reads the draft

**Files:**
- Test: `src/ai/ai.service.business-isolation.spec.ts`

This task is the acceptance criterion (§4.3). Everything else is plumbing; this is the property that makes the design safe.

**Interfaces:**
- Consumes: `AiService`, `BusinessService`.

- [ ] **Step 1: Write the test**

Create `src/ai/ai.service.business-isolation.spec.ts`:

```ts
import { BusinessService } from '../business/business.service';

// PRD 10 §3.4/§4.3. La razon por la que el split draft/live es seguro es que el
// camino de lectura del bot NO cambio. Si alguien alguna vez hace que AiService
// (o BusinessService.get) mire el borrador, este test falla: la base de prueba
// explota si alguien toca businessProfileDraft.
describe('El bot no lee el borrador de la info del negocio', () => {
  const svc = new BusinessService({} as any);

  const db = {
    businessProfile: {
      findFirst: jest.fn().mockResolvedValue({ id: 'live-1', name: 'Nombre Aprobado', hours: '9 a 18' }),
    },
    businessProfileDraft: {
      findFirst: jest.fn(() => { throw new Error('el bot no debe leer el borrador'); }),
      count: jest.fn(() => { throw new Error('el bot no debe leer el borrador'); }),
    },
  };

  it('get() devuelve el perfil aprobado y nunca toca el borrador', async () => {
    await expect(svc.get(db)).resolves.toMatchObject({ name: 'Nombre Aprobado' });
    expect(db.businessProfileDraft.findFirst).not.toHaveBeenCalled();
  });

  it('con un borrador pendiente, get() sigue devolviendo lo aprobado', async () => {
    // upsertDraft escribe el borrador; get() no debe verse afectado.
    const dbConBorrador = {
      ...db,
      businessProfileDraft: {
        findFirst: jest.fn().mockResolvedValue(null),
        create: jest.fn().mockResolvedValue({ id: 'draft-1', name: 'Propuesto' }),
      },
    };
    await svc.upsertDraft({ name: 'Propuesto' }, 'u1', dbConBorrador);
    await expect(svc.get(dbConBorrador)).resolves.toMatchObject({ name: 'Nombre Aprobado' });
  });
});
```

- [ ] **Step 2: Run it**

Run: `npm test -- src/ai/ai.service.business-isolation.spec.ts`
Expected: PASS. If either test fails, something in the read path now consults the draft — that is the defect this task exists to catch, not a test to adjust.

- [ ] **Step 3: Assert it statically too**

The three prompt read sites are `ai.service.ts:189`, `:239` and `:852`, all calling `businessService.get()`. Confirm no file under `src/ai/` references the draft at all:

```bash
grep -rn "businessProfileDraft\|getDraft" src/ai/ || echo "limpio"
```
Expected: `limpio`. Any hit is a Critical defect — the whole design rests on this.

- [ ] **Step 4: Commit**

```bash
git add src/ai/ai.service.business-isolation.spec.ts
git commit -m "test(negocio): el bot lee lo aprobado, nunca el borrador (PRD 10 §4.3)"
```

---

### Task 5: End-to-end verification

**Files:** none — this task changes no code. If it finds a defect, fix it and re-run the affected task's tests.

Removing a write path from a service proves nothing about the route in front of it. Verify against the running server.

- [ ] **Step 1: Create a throwaway tenant**

Log in as superadmin, then create a tenant with an admin password you choose, so you have a real tenant-admin token without touching `tenant-dev`:

```bash
curl -s -X POST http://localhost:3000/tenants \
  -H "Authorization: Bearer $SA" -H "Content-Type: application/json" \
  -d '{"name":"PRD10 F2","slug":"prd10-f2","admin_email":"f2@example.com","admin_username":"f2admin","admin_password":"<elegida>"}'
```

- [ ] **Step 2: Confirm the core control**

As the tenant admin (with `X-Tenant-Slug: prd10-f2`), read live, then `PUT` a change, then read live again:

```bash
curl -s http://localhost:3000/api/business -H "Authorization: Bearer $TA" -H "X-Tenant-Slug: prd10-f2"
curl -s -X PUT http://localhost:3000/api/business -H "Authorization: Bearer $TA" -H "X-Tenant-Slug: prd10-f2" \
  -H "Content-Type: application/json" -d '{"name":"Propuesto por el cliente","hours":"9 a 18"}'
curl -s http://localhost:3000/api/business -H "Authorization: Bearer $TA" -H "X-Tenant-Slug: prd10-f2"
```

Expected: the live profile is **byte-identical** before and after the `PUT` (§4.2). `GET /api/business/draft` returns the submission.

- [ ] **Step 3: Approve and confirm it lands**

As superadmin: `GET /tenants/prd10-f2/business` returns both; `POST /tenants/prd10-f2/business/approve` returns the updated live profile; `GET /api/business` as the tenant admin now shows the new values and `GET /api/business/draft` returns null.

- [ ] **Step 4: Reject and confirm live is untouched**

Submit another `PUT` as the tenant admin, then `POST /tenants/prd10-f2/business/reject` with `{"note":"faltan horarios"}`. Live must be unchanged, and `GET /api/business/draft` must show `review_status: ARCHIVED` with the note.

- [ ] **Step 5: Confirm the badge count**

`GET /tenants/business/pending-count` as superadmin returns `{}` when nothing is pending and `{"prd10-f2": 1}` while a submission waits.

- [ ] **Step 6: Delete the throwaway tenant and confirm its database is gone**

```bash
curl -s -X DELETE http://localhost:3000/tenants/prd10-f2 -H "Authorization: Bearer $SA"
docker exec <postgres-container> psql -U postgres -tAc "SELECT datname FROM pg_database WHERE datname LIKE 'client_%';"
```
Expected: the throwaway database is absent.

- [ ] **Step 7: Final gates**

Run: `npm test` → PASS, no regression against the pre-branch count.
Run: `npx tsc --noEmit -p tsconfig.json` → clean.

Report the actual output. Step 2 is the acceptance criterion; a green suite alone does not close this phase.
