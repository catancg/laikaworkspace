# WhatsApp audio — failure handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A WhatsApp audio whose transcription fails must still be saved, still be enqueued, and still get a reply — instead of vanishing without a trace.

**Architecture:** `case 'audio'` in `WhatsappService.handleMessage` copies the `try/catch` shape that `case 'image'` already uses in the same `switch`, just above it. Network and model calls go inside the guard; the row write and the queue push stay outside it and always run. When there is no usable text, `content` falls back to a bracketed Spanish placeholder — the same convention the image branch already uses — so the agent and the funnel analyst both read "an audio came in that we could not understand" rather than an empty string.

**Tech Stack:** NestJS + Prisma (backend, `:3000`), Jest + ts-jest (`testEnvironment: node`, Node 24), BullMQ.

**Spec:** There is no PRD for this. It is a defect fix, not a design. The evidence it argues from is in [Background](#background-what-already-exists--do-not-rebuild-it) below and in [`soylaika.backend/doc/pendientes.md`](../../soylaika.backend/doc/pendientes.md) §1. **Read Background before Task 1** — the transcription feature itself is already built and working, and rebuilding it is the main way this plan can go wrong.

**Scope:** Backend only, one repo, one branch. Instagram audio parity (`instagram.service.ts`) and rendering `transcriptRaw` in the CRM are **deliberately out** — both are recorded as pending in Task 3 instead of built.

## Global Constraints

- **One repo, one commit per task.** This plan touches `soylaika.backend` only. The plan document itself lives in the root repo (`laikaworkspace`) and is already committed — do not copy it into the backend repo.
- **Stage explicit paths.** Never `git add -A`, `git add .`, or `git stash` — the code repos carry uncommitted local `CLAUDE.md` edits that are not recoverable from the remote.
- **Check the branch before the first write.** The code repos are not always on `main`.
- **Lint with `npx eslint <paths>`, never `npm run lint`** — the backend script is `eslint --fix` and has reformatted 93 untouched files in a single run.
- **The eslint baseline is large and non-zero.** Measured 2026-09-10: `src/whatsapp/whatsapp.service.ts` reports 206 problems (197 errors) before any change here, and the existing reference test `src/whatsapp/send-manual.spec.ts` reports 19. `npx eslint` returning "no errors" is not achievable and is not the bar. The bar is: **introduce no rule categories beyond those comparable existing files already trip, and leave zero `prettier/prettier` violations on lines you wrote** — commit `8663bb4` (`style(negocio): formatear las lineas nuevas … segun prettier`) shows prettier is enforced on new code even where the rest is not.
- **Never `npm run build`** while `start-dev.ps1` is running; it competes with `nest start --watch` over `dist/` and takes the backend down. Type-check with `npx tsc --noEmit -p tsconfig.json` instead.
- **Backend comments and user-facing strings are Spanish.** Match the file being edited. The placeholder strings in this plan are user-visible-adjacent (an agent reads them) and are Spanish on purpose.
- **No migration, no schema change.** `Message.transcriptRaw` already exists (`prisma/schema.prisma:258`, migration `20260613000003_add_message_transcript_raw`). Adding a migration here would propagate to every tenant DB on the next dev-server boot for no reason.
- **Run commands from `soylaika.backend/`,** not from the workspace root.

---

## Background: what already exists — do not rebuild it

Audio transcription for WhatsApp is **implemented and working**. A previous session built it; this plan only hardens it. Before writing any code, confirm each of these is still true:

| Piece | Where | What it does |
|---|---|---|
| Transcription | `src/whatsapp/whatsapp.service.ts:258-340`, `case 'audio'` | media URL → download → Whisper (Groq `whisper-large-v3`, falling back to OpenAI `whisper-1`, `language: 'es'`) |
| Cleanup | `src/ai/ai.service.ts:190-232`, `cleanTranscript()` | conservative correction using the tenant's rubro as context; **already catches its own errors and returns the raw transcript** (`:227-229`) — do not add a guard around it |
| Conversation | `whatsapp.service.ts:316-338` | the cleaned transcript *is* `Message.content` with `type: 'audio'`, enqueued to `MESSAGE_QUEUE` with `body` set exactly like a text message |
| Qualification | `src/ai/ai.service.ts:720-724` | `analyzeConversation` builds its input from `m.content` over the last 10 messages — audio transcripts are already included, with no special-casing |
| Storage | `prisma/schema.prisma:258` | `transcriptRaw` keeps the pre-cleanup text when it differs from the cleaned one |
| Config | `.env.example:43` | `GROQ_API_KEY`; with no key at all the branch already stores `[Audio recibido — transcripción no configurada]` |

**The defect.** `case 'image'` wraps its download-and-describe in `try/catch` (`:207-224`) and degrades to `[El cliente mandó una foto que no se pudo analizar]`. `case 'audio'` has **no such guard**, and `handleMessage` is invoked fire-and-forget at `whatsapp.service.ts:75`:

```ts
void this.handleMessage(message, value?.contacts ?? [], tenant, db, channel);
```

So a Meta media-URL expiry, a Groq 429, a socket timeout, or a codec Whisper rejects produces an **unhandled promise rejection**: `message.create` never runs, `messageQueue.add` never runs, and the customer who sent a voice note gets silence. Nothing in the logs ties the silence to the audio.

**The second defect.** If Whisper returns `200` with empty or whitespace-only text (a silent or very short recording), `transcript` becomes `''`, `cleanTranscript` returns it unchanged (`:191`, `if (!text) return raw`), and the branch saves `content: ''` and enqueues `body: ''`. The agent is asked to reply to a blank message.

**Test coverage today: none.** No spec in the repo mentions audio or transcription. `src/whatsapp/whatsapp.service.spec.ts` is a `describe.skip` scaffold that documents its own uselessness. The only real test of this service is `src/whatsapp/send-manual.spec.ts` — **read it before Task 1**; this plan's spec file copies its approach exactly (construct the class with minimal doubles, no `TestingModule`).

---

## File Structure

| File | Responsibility |
|---|---|
| `src/whatsapp/audio-transcription.spec.ts` | **create** — the four assertions that keep the audio path honest |
| `src/whatsapp/whatsapp.service.ts:258-340` | **modify** — guard steps 1–3, always write and enqueue |
| `doc/agentes-e-info-del-negocio.md:141,154,160-166` | **modify** — the degradation note currently claims behaviour that did not exist |
| `doc/pendientes.md` §4 | **modify** — record the CRM gap this plan does not close |

Named after the feature, not the class (`send-manual.spec.ts` set that convention). `src/ai/ai.service.ts` is **not** modified — `cleanTranscript` already degrades correctly.

---

### Task 1: A failed transcription no longer loses the message

**Files:**
- Create: `src/whatsapp/audio-transcription.spec.ts`
- Modify: `src/whatsapp/whatsapp.service.ts:258-340`

**Interfaces:**
- Consumes: the `WhatsappService` constructor, positionally — `(config, http, prisma, tenantFactory, redis, ai, messageQueue, followupQueue)`; and `private async handleMessage(message, contacts, tenant, db, channel): Promise<void>`, reached with `(service as any)`.
- Produces: two constants inside `case 'audio'` — `SIN_STT` (`'[Audio recibido — transcripción no configurada]'`, the existing string, unchanged) and `NO_SE_PUDO` (`'[El cliente mandó un audio que no se pudo transcribir]'`, new). Task 2 reuses `NO_SE_PUDO`.

- [ ] **Step 1: Write the failing tests**

Create `src/whatsapp/audio-transcription.spec.ts` with exactly this content:

```ts
/**
 * Audio de WhatsApp: que el mensaje nunca se pierda.
 *
 * handleMessage se invoca con `void` (whatsapp.service.ts, en processChange),
 * asi que nadie atrapa una excepcion suya. Mientras el `case 'audio'` no tuvo
 * try/catch, cualquier falla de red o de Whisper hacia desaparecer el audio del
 * cliente: no se guardaba la fila, no se encolaba el job, y el bot no contestaba
 * nunca. El cliente mandaba un audio y recibia silencio. Estos tests son lo que
 * impide que vuelva.
 *
 * Mismo enfoque que send-manual.spec.ts: se instancia la clase con dobles
 * minimos en vez de armar un TestingModule (de los ocho providers del
 * constructor esta ruta toca cinco), y al metodo privado se llega por
 * `(service as any)`.
 */
import { of, throwError } from 'rxjs';
import { WhatsappService } from './whatsapp.service';

const NO_SE_PUDO = '[El cliente mandó un audio que no se pudo transcribir]';

const mensajeAudio = {
  id: 'wamid.audio1',
  from: '5491100000000',
  type: 'audio',
  audio: { id: 'media-1' },
};
const tenant = { slug: 'demo', openai_api_key: null };
const channel = { id: 'ch1', access_token: 'token-de-prueba' };

function crearServicio(
  opts: {
    get?: (url: string) => any;
    post?: () => any;
    cleanTranscript?: (raw: string) => Promise<string>;
  } = {},
) {
  // Dos GET distintos en la misma ruta: primero el metadata del media, despues
  // la descarga del archivo. Se distinguen por el host.
  const get =
    opts.get ??
    ((url: string) =>
      url.includes('graph.facebook.com')
        ? of({ data: { url: 'https://media.example/audio.ogg' } })
        : of({ data: Buffer.from('ogg-de-mentira') }));
  const post = opts.post ?? (() => of({ data: { text: 'ola kiero el kombo de 2' } }));

  // Los mocks se tipan con (...args: any[]) para que `mock.calls[0]` no quede
  // como tupla vacia y se pueda leer lo que se guardo y lo que se encolo.
  const http = {
    get: jest.fn((url: string, ..._rest: any[]) => get(url)),
    post: jest.fn((..._args: any[]) => post()),
  };
  const config = {
    get: jest.fn((key: string) => (key === 'GROQ_API_KEY' ? 'groq-key' : 'valor-de-prueba')),
  };
  const ai = {
    cleanTranscript: jest.fn(opts.cleanTranscript ?? (async (raw: string) => raw)),
  };
  const db = {
    message: {
      findUnique: jest.fn(async (..._args: any[]) => null), // no es duplicado
      create: jest.fn(async (args: any) => ({ id: 'm1', ...args.data })),
    },
    contact: {
      upsert: jest.fn(async (..._args: any[]) => ({
        id: 'c1',
        name: 'Ana',
        phone: '5491100000000',
        botActive: true,
      })),
    },
  };
  const redis = {
    get: jest.fn(async (..._args: any[]) => null),
    del: jest.fn(async (..._args: any[]) => 1),
  };
  const messageQueue = { add: jest.fn(async (..._args: any[]) => ({ id: 'job-1' })) };
  const followupQueue = { getJob: jest.fn(async (..._args: any[]) => null) };

  const service = new WhatsappService(
    config as any,
    http as any,
    {} as any, // prisma: no se toca, el db entra por parametro
    {} as any, // tenantFactory
    redis as any,
    ai as any,
    messageQueue as any,
    followupQueue as any,
  );
  return { service, http, db, ai, messageQueue };
}

// handleMessage es privado.
const procesar = (service: WhatsappService, db: any) =>
  (service as any).handleMessage(mensajeAudio, [], tenant, db, channel);

describe('WhatsappService — audio', () => {
  it('guarda y encola el audio aunque falle la descarga del media', async () => {
    const { service, db, messageQueue } = crearServicio({
      get: () => throwError(() => new Error('media URL expirada')),
    });

    // Antes esto rechazaba, y como arriba se llama con `void`, el rechazo no lo
    // atrapaba nadie.
    await expect(procesar(service, db)).resolves.toBeUndefined();

    expect(db.message.create).toHaveBeenCalledTimes(1);
    const fila = db.message.create.mock.calls[0][0].data;
    expect(fila.content).toBe(NO_SE_PUDO);
    expect(fila.type).toBe('audio');
    expect(fila.transcriptRaw).toBeNull();

    // Lo que este test existe para afirmar: el bot igual contesta.
    expect(messageQueue.add).toHaveBeenCalledTimes(1);
    expect(messageQueue.add.mock.calls[0][1].body).toBe(NO_SE_PUDO);
  });

  it('guarda y encola el audio aunque Whisper devuelva error', async () => {
    const { service, db, messageQueue } = crearServicio({
      post: () =>
        throwError(() => ({
          message: 'Rate limit reached',
          response: { status: 429, data: { error: { message: 'Rate limit reached' } } },
        })),
    });

    await expect(procesar(service, db)).resolves.toBeUndefined();

    expect(db.message.create.mock.calls[0][0].data.content).toBe(NO_SE_PUDO);
    expect(messageQueue.add).toHaveBeenCalledTimes(1);
  });

  it('en el camino feliz guarda la transcripcion limpia y la cruda', async () => {
    const { service, db, messageQueue } = crearServicio({
      post: () => of({ data: { text: 'ola kiero el kombo de 2' } }),
      cleanTranscript: async () => 'Hola, quiero el combo de 2',
    });

    await procesar(service, db);

    // El control del placeholder: si este test se rompe porque ahora tambien
    // devuelve NO_SE_PUDO, el fallback se volvio incondicional y no cubre nada.
    const fila = db.message.create.mock.calls[0][0].data;
    expect(fila.content).toBe('Hola, quiero el combo de 2');
    expect(fila.transcriptRaw).toBe('ola kiero el kombo de 2');
    expect(messageQueue.add.mock.calls[0][1].body).toBe('Hola, quiero el combo de 2');
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test -- src/whatsapp/audio-transcription.spec.ts`

Expected: the first two FAIL, the third PASSES.
- Test 1: `Received promise rejected instead of resolved` — rejected value `Error: media URL expirada`.
- Test 2: the same, rejected with the `429` object.
- Test 3 already passes — the happy path works today; it is the control.

If test 3 fails too, stop: the doubles are wrong, not the code. Do not change `whatsapp.service.ts` to make it pass.

- [ ] **Step 3: Guard steps 1–3 in the audio branch**

In `src/whatsapp/whatsapp.service.ts`, replace the whole `case 'audio': {` block (currently lines 258–340, ending at its `break;` and closing brace) with:

```ts
      case 'audio': {
        const mediaId    = message.audio?.id;
        const accessToken = channel?.access_token ?? this.config.get<string>('WHATSAPP_ACCESS_TOKEN');

        // El `content` de la fila es lo que LEE el agente para contestar y lo que
        // lee el analista del embudo para clasificar la etapa. Si el audio no se
        // pudo transcribir tiene que decirlo en texto: el agente necesita saber
        // que entro un audio que no entendio.
        const SIN_STT    = '[Audio recibido — transcripción no configurada]';
        const NO_SE_PUDO = '[El cliente mandó un audio que no se pudo transcribir]';

        let cleaned: string | null = null;      // limpia (la que ve el agente y el cliente)
        let transcriptRaw: string | null = null;

        // Los pasos 1-3 son red y modelos, y cualquiera de los tres puede fallar:
        // la URL del media de Meta vence, Groq tira 429, el socket se corta.
        // Van envueltos igual que en `case 'image'` porque handleMessage se
        // invoca con `void` (ver processChange): una excepcion aca no la atrapa
        // nadie y el audio del cliente desaparecia entero — no se guardaba la
        // fila, no se encolaba el job, y el cliente recibia silencio.
        try {
          // 1. Obtener URL del media
          const mediaRes = await firstValueFrom(
            this.http.get(`https://graph.facebook.com/v22.0/${mediaId}`, {
              headers: { Authorization: `Bearer ${accessToken}` },
            })
          );
          const audioUrl = mediaRes.data.url;

          // 2. Descargar el archivo
          const audioRes = await firstValueFrom(
            this.http.get(audioUrl, {
              headers: { Authorization: `Bearer ${accessToken}` },
              responseType: 'arraybuffer',
            })
          );
          const audioBuffer = Buffer.from(audioRes.data);

          // 3. Transcribir con Whisper. Preferimos Groq (whisper-large-v3: más rápido,
          // barato y mejor modelo); si no hay key de Groq, caemos a OpenAI. Ambos exponen
          // la misma API compatible con OpenAI: solo cambian URL, modelo y key.
          const groqKey = this.config.get<string>('GROQ_API_KEY');
          const openaiKey = tenant?.openai_api_key ?? this.config.get<string>('OPENAI_API_KEY');
          const stt = groqKey
            ? { url: 'https://api.groq.com/openai/v1/audio/transcriptions', model: 'whisper-large-v3', key: groqKey, provider: 'Groq' }
            : openaiKey
              ? { url: 'https://api.openai.com/v1/audio/transcriptions', model: 'whisper-1', key: openaiKey, provider: 'OpenAI' }
              : null;

          if (stt) {
            const formData = new FormData();
            formData.append('file', new Blob([audioBuffer], { type: 'audio/ogg' }), 'audio.ogg');
            formData.append('model', stt.model);
            formData.append('language', 'es');
            const whisperRes = await firstValueFrom(
              this.http.post(stt.url, formData, {
                headers: { Authorization: `Bearer ${stt.key}` },
              })
            );
            const transcript = whisperRes.data.text ?? '';   // cruda de Whisper

            // 3b. Limpieza conservadora de la transcripción con IA. `cleanTranscript`
            // atrapa sus propios errores y devuelve la cruda: no hace falta guardarla acá.
            cleaned = await this.ai.cleanTranscript(transcript, tenant, resolvedDb);
            // Guardar la cruda solo si difiere de la limpia (para mostrar "dijo → interpretamos")
            transcriptRaw = cleaned.trim() !== transcript.trim() ? transcript : null;
            this.logger.log(`[${contact.name ?? message.from}] Audio (${stt.provider}): "${transcript.slice(0, 60)}" → "${cleaned.slice(0, 60)}"`);
          } else {
            // Falta de configuracion, no falla transitoria: se distingue a proposito
            // del placeholder de abajo, porque en los logs se leen distinto.
            cleaned = SIN_STT;
            this.logger.warn('Sin GROQ_API_KEY ni OPENAI_API_KEY: audio no transcripto');
          }
        } catch (err: any) {
          // Se descarta cualquier estado parcial: si entramos aca no confiamos en
          // lo que haya quedado a medio escribir.
          cleaned = null;
          transcriptRaw = null;
          this.logger.warn(`No se pudo transcribir el audio ${mediaId}: ${err.message}`);
        }

        // 4. Guardar mensaje: content = versión limpia, transcriptRaw = cruda (si difiere).
        // Fuera del try: esto pasa siempre, falle lo que falle arriba.
        const content = cleaned ?? NO_SE_PUDO;
        const saved = await resolvedDb.message.create({
          data: { contactId: contact.id, role: 'user', content, transcriptRaw, type: 'audio', whatsappMsgId },
        });

        // 5. Encolar para la IA igual que texto (con la transcripción limpia, o con
        // el placeholder: que el bot conteste "no pude escuchar el audio" es mejor
        // que no contestar nada).
        if (contact.botActive) {
          await this.messageQueue.add(
            'process-message',
            {
              contactId: contact.id,
              phone:     message.from,
              body:      content,
              whatsappMsgId,
              jobId:      whatsappMsgId,
              userMsgDbId: saved.id,
              tenantSlug:  tenant?.slug ?? null,
              channelId:   channel?.id  ?? null,
              platform:    'whatsapp',
            } satisfies MessageJobData,
            { jobId: whatsappMsgId, delay: 12000, attempts: 5, backoff: { type: 'exponential', delay: 2000 } },
          );
        }
        break;
      }
```

Three things to check you did not lose while replacing:

1. `body:` in the queue payload is now `content`, **not** `cleaned`. Leaving it as `cleaned` enqueues `null` on the failure path.
2. `message.create` and `messageQueue.add` are **outside** the `try`. Pulling them inside re-creates the exact bug this task fixes.
3. `let transcript` is gone from the outer scope; it is now `const transcript` inside the `if (stt)` block, its only use.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test -- src/whatsapp/audio-transcription.spec.ts`
Expected: 3 passed.

- [ ] **Step 5: Type-check and lint**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no output.

Run: `npx eslint src/whatsapp/whatsapp.service.ts src/whatsapp/audio-transcription.spec.ts`
Expected: a non-zero count — see the baseline constraint above. What you check is that **no `prettier/prettier` violation lands on a line you wrote**, and that the rule categories reported match the ones `send-manual.spec.ts` and the untouched parts of `whatsapp.service.ts` already trip. (`npm run lint` would rewrite 93 unrelated files — do not run it.)

- [ ] **Step 6: Confirm nothing else broke**

Run: `npm test`
Expected: the suite is green and the audio file adds 3 passing tests. `whatsapp.service.spec.ts` stays skipped — leave it alone, its comment explains why it exists.

- [ ] **Step 7: Commit**

```bash
git add src/whatsapp/whatsapp.service.ts src/whatsapp/audio-transcription.spec.ts
git commit -m "fix(audio): un audio que no se puede transcribir ya no desaparece

handleMessage se llama con void, asi que la excepcion del case 'audio' no la
atrapaba nadie: no se guardaba el mensaje, no se encolaba, y el cliente que
mando un audio recibia silencio. Ahora degrada como el case 'image'.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01STkQx9AHknJ1mEmteQeAWo"
```

---

### Task 2: An empty transcription is not an empty message

A silent recording, a half-second tap, or a codec Whisper decodes to nothing returns `200` with `text: ''`. `cleanTranscript` passes empty input straight through (`ai.service.ts:191`). Task 1 leaves that as `content: ''` — a blank message the agent is asked to answer, and a blank line in the funnel analyst's conversation transcript.

Separable from Task 1 on purpose: Task 1 fixes a crash path, this fixes a success path that produces garbage. A reviewer can take one without the other.

**Files:**
- Modify: `src/whatsapp/audio-transcription.spec.ts` (append one test)
- Modify: `src/whatsapp/whatsapp.service.ts` — one line, the `content` fallback from Task 1

**Interfaces:**
- Consumes: `NO_SE_PUDO` and the `crearServicio` / `procesar` helpers, all from Task 1.
- Produces: nothing new.

- [ ] **Step 1: Write the failing test**

Append inside the existing `describe('WhatsappService — audio', ...)` block in `src/whatsapp/audio-transcription.spec.ts`:

```ts
  it('usa el placeholder si Whisper devuelve una transcripcion vacia', async () => {
    const { service, db, messageQueue } = crearServicio({
      post: () => of({ data: { text: '   ' } }),
    });

    await procesar(service, db);

    // Sin esto se guardaba content: '' y se encolaba body: '' — al agente le
    // llegaba un mensaje en blanco y contestaba cualquier cosa, y el analista
    // del embudo veia una linea vacia atribuida al cliente.
    expect(db.message.create.mock.calls[0][0].data.content).toBe(NO_SE_PUDO);
    expect(db.message.create.mock.calls[0][0].data.transcriptRaw).toBeNull();
    expect(messageQueue.add.mock.calls[0][1].body).toBe(NO_SE_PUDO);
  });
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npm test -- src/whatsapp/audio-transcription.spec.ts -t "transcripcion vacia"`
Expected: FAIL — `Expected: "[El cliente mandó un audio que no se pudo transcribir]"`, `Received: "   "`.

- [ ] **Step 3: Tighten the fallback**

In `src/whatsapp/whatsapp.service.ts`, in the `case 'audio'` block from Task 1, change the single line:

```ts
        const content = cleaned ?? NO_SE_PUDO;
```

to:

```ts
        // `?.trim()` y no `??`: Whisper devuelve 200 con texto vacio para un audio
        // mudo o de medio segundo, y `cleanTranscript` deja pasar el vacio tal cual.
        // Guardar '' le manda al agente un mensaje en blanco para contestar.
        const content = cleaned?.trim() ? cleaned : NO_SE_PUDO;
```

Keep `cleaned` (untrimmed) as the stored value when it is non-blank — trimming it here would silently change the text the agent reads on the happy path, which is not this task's business.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test -- src/whatsapp/audio-transcription.spec.ts`
Expected: 4 passed. The happy-path control from Task 1 must still pass — if it now returns the placeholder, the condition is inverted.

- [ ] **Step 5: Type-check and lint**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no output.

Run: `npx eslint src/whatsapp/whatsapp.service.ts src/whatsapp/audio-transcription.spec.ts`
Expected: a non-zero count — see the baseline constraint above. Check only that no `prettier/prettier` violation lands on a line you wrote.

- [ ] **Step 6: Commit**

```bash
git add src/whatsapp/whatsapp.service.ts src/whatsapp/audio-transcription.spec.ts
git commit -m "fix(audio): una transcripcion vacia ya no se guarda como mensaje en blanco

Whisper devuelve 200 con texto vacio para un audio mudo. Se guardaba
content: '' y se encolaba body: '', asi que el agente contestaba a la nada.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01STkQx9AHknJ1mEmteQeAWo"
```

---

### Task 3: Make the docs say what the code does

`doc/agentes-e-info-del-negocio.md` currently promises graceful degradation that did not exist until Task 1, names a model the code no longer prefers, and describes a CRM display that was never built. A doc that claims a control nobody implemented is worse than no doc — the next session reads it and assumes the case is covered.

**Files:**
- Modify: `doc/agentes-e-info-del-negocio.md` (lines 141, 154, 160–166)
- Modify: `doc/pendientes.md` §4 (Frontend / UX)

**Interfaces:** none — documentation only.

- [ ] **Step 1: Correct the model in the routing table**

In `doc/agentes-e-info-del-negocio.md`, line 141 reads:

```
| **Transcribir** (audios) | `whatsapp.service.ts` | Whisper (`whisper-1`) + limpieza con el modelo del orquestador | Whisper fijo; limpieza usa `orchestrator_model` |
```

Replace with:

```
| **Transcribir** (audios) | `whatsapp.service.ts` | Whisper por Groq (`whisper-large-v3`), con fallback a OpenAI (`whisper-1`) + limpieza con el modelo del orquestador | STT fijo; limpieza usa `orchestrator_model` |
```

- [ ] **Step 2: Correct step 1 of the transcription section**

Line 154 reads:

```
1. Se descarga el audio y se transcribe con **Whisper** (`whisper-1`, español).
```

Replace with:

```
1. Se descarga el audio y se transcribe con **Whisper** en español: se prefiere
   Groq (`whisper-large-v3`) y, si no hay `GROQ_API_KEY`, se cae a OpenAI
   (`whisper-1`).
```

- [ ] **Step 3: Stop claiming a CRM display that does not exist**

Lines 160–162 read:

```
   - `content` = transcripción **limpia** (lo que ve el agente y se procesa).
   - `transcriptRaw` = transcripción **cruda** (solo si difiere), para mostrar en el
     CRM el formato "lo que dijo → lo que interpretamos".
```

Replace with:

```
   - `content` = transcripción **limpia** (lo que ve el agente y se procesa).
   - `transcriptRaw` = transcripción **cruda** (solo si difiere). Se guarda para
     poder mostrar "lo que dijo → lo que interpretamos", pero **el CRM todavía no
     la muestra**: la conversación solo marca la burbuja como audio. Está anotado
     en `pendientes.md` §4.
```

- [ ] **Step 4: Replace the degradation note with what actually happens**

Lines 165–166 read:

```
> Si no hay `OPENAI_API_KEY` (Whisper) o falla la limpieza, se degrada con gracia:
> sin Whisper guarda un placeholder; si la limpieza falla, usa la transcripción cruda.
```

Replace with:

```
> **Degradación.** Los tres modos de falla terminan con el mensaje guardado y
> encolado — el bot siempre contesta algo:
>
> - Sin `GROQ_API_KEY` ni `OPENAI_API_KEY`: `content` queda como
>   `[Audio recibido — transcripción no configurada]`.
> - Falla la descarga del media o la llamada a Whisper (URL de Meta vencida, 429,
>   timeout), o Whisper devuelve texto vacío: `content` queda como
>   `[El cliente mandó un audio que no se pudo transcribir]`.
> - Falla solo la limpieza: se usa la transcripción cruda (`cleanTranscript`
>   atrapa su propio error).
>
> Hasta 2026-09-10 el segundo caso no estaba cubierto: `handleMessage` se invoca
> con `void`, así que la excepción no la atrapaba nadie y el audio desaparecía sin
> guardarse ni encolarse. Lo sostienen los tests de
> `src/whatsapp/audio-transcription.spec.ts`; si se tocan esas ramas, se tocan
> esos tests.
```

- [ ] **Step 5: Record the gap this plan does not close**

`doc/pendientes.md` §1 already carries the Instagram audio item (line 22) — leave it exactly as it is. Append to §4 (Frontend / UX):

```
- 🟢 **`Message.transcriptRaw` no se muestra en el CRM.** El backend lo guarda
  desde la migración `20260613000003` para poder mostrar "lo que dijo → lo que
  interpretamos" en los audios, pero la conversación solo pone un `title` en la
  burbuja (`app/(crm)/contacts/[id]/page.tsx`) y nunca renderiza el valor.
  Requiere exponerlo en `lib/api.ts`. *Esfuerzo: bajo.*
```

- [ ] **Step 6: Verify the doc matches the code**

Every placeholder string in the doc must match the constants in the code character for character, accents included — `[Audio recibido — transcripción no configurada]` uses an em dash, not a hyphen.

Run: `grep -n "no se pudo transcribir\|transcripción no configurada" src/whatsapp/whatsapp.service.ts doc/agentes-e-info-del-negocio.md`

Expected: four hits — two in the service (the two constants) and two in the doc.

- [ ] **Step 7: Commit**

```bash
git add doc/agentes-e-info-del-negocio.md doc/pendientes.md
git commit -m "docs(audio): la degradacion documentada no existia, y el CRM no muestra transcriptRaw

El doc prometia degradar con gracia ante una falla de Whisper; esa rama recien
quedo cubierta ahora. Tambien nombraba whisper-1 cuando se prefiere Groq, y
describia una pantalla del CRM que nunca se construyo.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01STkQx9AHknJ1mEmteQeAWo"
```

---

## Verification before calling this done

Not a substitute for the per-task steps — this is the end-to-end check.

- [ ] `npm test` is green from `soylaika.backend/`, with 4 passing tests in `audio-transcription.spec.ts`.
- [ ] `npx tsc --noEmit -p tsconfig.json` produces no output.
- [ ] `git log --oneline -3` shows three commits, all in `soylaika.backend`, none touching the frontend or the root repo.
- [ ] `git status --short` shows no stray staged files — in particular `CLAUDE.md` must not appear.
- [ ] **Optional, needs a live tenant.** With `start-dev.ps1` running, set `GROQ_API_KEY` to a syntactically valid but wrong key, send a voice note from a test number, and confirm the conversation shows a bubble reading `[El cliente mandó un audio que no se pudo transcribir]` and that the bot replies. Before this plan, that produced nothing at all — no row, no reply, only an unhandled rejection in the logs.
