# PRD 6 — Agent evaluation suite

**Status:** proposed
**Depends on:** [PRD 1 — RAG knowledge layer](prd-rag-knowledge-layer.md), [PRD 5 — full test of the RAG system](prd-rag-system-test.md)
**Related:** [briefing-contexto-agentes-ES.md](../briefing-contexto-agentes-ES.md) — the agents, their invariants, and the known failure modes

## 1. Why this exists

Every lever that decides what the bot says is non-deterministic and edited without a safety net: agent prompts live in a database table and are changed from the superadmin panel; the knowledge base is loaded by the customer; the retrieval threshold is an env var; the model can be swapped per tenant. Change any of them and the only way to find out what happened is to talk to the bot and form an impression.

`0.78` is a threshold nobody has validated. A prompt edit that fixes one conversation can break three others, and nothing would say so. **Today, "is the bot getting better or worse?" is answered by vibes, and only for the conversations someone happened to read.**

**Boundary with PRD 5.** PRD 5 tests the *plumbing* — is the retrieval query correct, can an unapproved chunk reach the bot. Deterministic, right or wrong. This tests *behaviour*: given the plumbing works, does the bot answer well. Different question, different instrument.

## 2. Goal

A suite that can be run against a tenant — locally or in production — and produces a **durable, comparable record** of how the bot behaved.

Two uses, in priority order:

1. **Track production over time.** Run it periodically against the live tenant and see whether behaviour is drifting up or down.
2. **Check a change before and after.** Run it, change a prompt, run it again, read the two records side by side. **Comparison is manual for now** — the suite produces the evidence; a person draws the conclusion.

Automated A/B judging is explicitly out (§3). That decision shapes §6.

### 2.1 Who this is for

**The primary user is a non-technical superadmin**, not an engineer. The people who edit agent prompts and load knowledge-base content do it from the panel; they have no local environment and never will. A suite that only runs from a terminal would be built for the wrong audience — it would serve the people who *don't* make these changes.

Two consequences that run through the whole document:

- **The panel is the primary interface**, not a later convenience. The CLI is the engineering path, and it exists because engineers also change these things — but it is the secondary one.
- **The output has to be legible to someone who is not reading JSON.** "El bot inventó un precio en 2 conversaciones" is actionable. "La media de utilidad bajó 0.2" is not. This is a second, independent reason the deterministic checks (§4.1) matter more than the judged scores: they are both the more trustworthy signal *and* the more readable one.

## 3. Non-goals

- **No separate eval tenant.** Runs happen against whatever tenant is named — production or local.
- **Not a CLI-only tool.** The people who edit prompts work in the panel and have no local environment (§2.1).
- **No automated A/B verdict.** No "variant B wins". Two runs, read by a person.
- **No promotion pipeline.** Nothing moves configuration between tenants.
- **Not a correctness oracle.** The judge is another LLM with its own error (§6, §7).
- **Not a replacement for reading transcripts.** The suite decides *where to look*.
- **Not load or latency testing.**

## 4. What gets measured, and by what

The most important design decision: **most of what "does the agent follow its prompt?" means is not a judgment call.** Routing it through an LLM makes it slower, costlier and less reliable than a regular expression.

### 4.1 Deterministic checks — the backbone

Assertions over the reply text and the turn's metadata. No judge, no cost, no variance:

| Check | Why it is deterministic |
|---|---|
| Invented a price or discount | `faq-lint.ts` already implements this, tuned for Argentine price formats |
| ~~WhatsApp formatting~~ | **Not measurable as specified** — see §4.2 |
| Leaked internal routing | Must never name an agent, `[[DERIVAR]]`, or the retrieved-knowledge markers |
| Said "diseñador de interiores" instead of "asesor" | A named rule from the briefing |
| Greeted twice | Second turn onward must not re-greet |
| Escalation actually escalated | `handToHuman` is returned by `chat()` |
| Retrieval fired when it should | `RetrievalOutcome.fired` + `chunkIds` (needs §10) |
| Answer grounded in its chunk | `faq-support.ts` already computes exactly this |

**These are also the only metric in this document that is trustworthy across months.** A pass rate is absolute, stable, and unaffected by which model is judging — which is precisely what §7 shows the judge scores are not. If only half of this PRD gets built, build this half.

### 4.2 The formatting check that cannot be done

`normalizeWhatsappText` is private in `AiService` **and runs on the reply before it is returned**: it converts `**markdown**` to `*bold*`, strips `[text](url)`, fixes bullets. By the time the runner sees `reply`, the violation has already been repaired.

So a formatting check over the returned reply **always passes** — not because the bot behaves, but because it is looking after the cleanup. It is exactly the kind of green test that discriminates nothing, and it was specified in an earlier version of this document.

What *is* measurable, and is better signal: **how often the normalizer had to intervene.** A prompt producing markdown on 40% of turns has a problem, even though the customer never sees it. That needs the raw model output exposed alongside the normalized one — one more field, the same size as the finding in §10.

Until that exists, this check is out. A check that cannot fail is worse than no check.

### 4.3 Judged scores — for what needs reading

What survives §4.1 requires judgment: did it understand the customer, was the answer useful, did it advance the sale, was escalating right. Those get an LLM score (§6).

## 5. Running it

> **Editing and reverting the configuration under test is [PRD 7](prd-agent-config-versioning.md), not this document.** This one runs the suite and records what happened; that one owns how a change is made and undone. They meet in one workflow — save, evaluate, keep or revert — and PRD 7 §4 describes it.

### 5.1 One engine, two front doors

The same engine, pointed at a tenant by slug, reachable two ways:

- **From the panel** — a superadmin picks scenarios and presses run. This is the path that matters (§2.1), and it is the one the business actually uses.
- **From the command line** — engineers, locally or from the Railway console.

Same code, same scenarios, same storage shape. If the two diverge, runs stop being comparable and the historical record is worthless.

### 5.2 Running against production creates conversations

`AiService.chat()` takes a `contactId` and reads history from the database — it cannot be called with a bare message list. That is why `/api/test-chat` persists `Contact` and `Message` rows, deliberately ("deja la conversación en /conversations como un lead real").

So a production run **will** create contacts. With no separate tenant, that has to be handled rather than avoided.

**Decision: tag them.** `Contact.source = 'eval'`, which already exists on the model **and is already indexed** (`@@index([source])`). Every list, count and metric in the CRM excludes that source.

- Cost: ~13 call sites (12 in `crm.service.ts`, 1 in `funnel.service.ts`). Bounded and mechanical.
- The alternative — splitting "produce a reply" from "record the conversation" inside `AiService` — is architecturally cleaner and far more invasive, in the most load-bearing service in the product. **Not worth it at this scope.** If a future requirement needs a truly read-only run, that is when to pay for it.

**A fresh contact per scenario per run**, never reused: `chat()` reads history, so a reused contact would let run N see run N-1's conversation. Eval contacts are purged on a retention window (§8).

**A run costs real money** against the tenant's own model budget. The runner estimates before starting and records actual spend (`AiUsage` already tracks model, kind, tokens and cost).

## 6. The judge

### 6.1 Absolute scores, and why that is a downgrade taken deliberately

The judge scores each reply against a fixed rubric. Not pairwise comparison.

This is the weaker mode, and the reason is worth writing down: pairwise judging is substantially more reliable — LLMs are better at "which of these two is better" than at "rate this 1–5" — but a pairwise result is **only meaningful about the pair**. "B won 60/40" cannot be plotted against next month's comparison of two different things. Longitudinal tracking (§2, use 1) requires an absolute number, so absolute is what this produces.

The consequence is that the scores are **soft**. §7 is entirely about keeping them honest, and §4.1 exists because the deterministic checks are the part that stays solid.

### 6.2 The rubric

Few criteria, each with concrete anchors describing what a 1, a 3 and a 5 look like — not adjectives. Anchors are what stop a judge from drifting between runs, and they are versioned with the prompt (§7).

Starting set: **understanding** (did it grasp what was asked), **usefulness** (did it move the conversation forward), **grounding** (is every business claim supported), **escalation** (was handing off, or not, the right call).

### 6.3 The judge must not be the agent

A different, stronger model than the one under test, on the platform key rather than the tenant's — judge cost is ours. Self-judging inflates scores.

### 6.4 The judge is itself under test

Keep ~20 turns with a **human score recorded**. Whenever the judge model or prompt changes, re-score them and report agreement. A judge that agrees 60% of the time is a random number generator with good grammar, and every trend built on it is decoration.

## 7. Keeping scores comparable over time

**This is the part that makes the historical record worth keeping, and the part most likely to be skipped.**

A score is meaningless without knowing what produced it. Three things silently rewrite history:

- **The judge model changes.** A provider deprecates one, or someone upgrades. Same reply, different number.
- **The judge prompt or rubric changes.** Adding a criterion shifts every score.
- **The scenario corpus changes.** Adding harder scenarios lowers the average with no change in the bot.

So **every stored score carries the judge model, the judge-prompt version, and the corpus version.** A chart that mixes them is lying, and the UI must refuse to draw a single line across a boundary — it shows a break instead.

### 7.1 Calibration runs

When any of those three changes, the old numbers do not become wrong — they become *incomparable*, which is worse because it is invisible.

The fix: keep the stored transcripts (§8), and when the judge changes, **re-score a sample of historical runs with the new judge**. That produces a delta — "the new judge scores 0.4 lower on the same replies" — which turns an invisible break into a measured step. Then either the history is re-scored in full, or the chart shows two segments with the offset stated.

**This is why §8 stores full replies and not just aggregates.** Without the transcripts, a judge change destroys every prior number permanently.

## 8. The record

Stored in the master database of wherever the run happened. Production runs are the tracked series; local runs are for iteration.

- `EvalRun` — tenant slug, environment, corpus version, judge model, judge-prompt version, started/finished, cost, who or what triggered it
- `EvalTurn` — run, scenario, repetition, turn index, **the full reply text**, `agentType`, `handToHuman`, retrieval outcome, tokens, cost, latency
- `EvalCheck` — turn, check name, pass/fail, detail
- `EvalScore` — turn, criterion, score, the judge's one-line reason

Two rules:

- **`EvalTurn` keeps the full reply.** Aggregates cannot be acted on, cannot be re-judged, and cannot be audited. This is the single most important storage decision here (§7.1).
- **Retention is asymmetric.** Eval *contacts* in the tenant DB are purged on a short window — they are CRM clutter. Eval *records* in master are kept indefinitely — they are the history the whole feature exists for. Do not let a cleanup job confuse the two.

## 9. Corpus

Four groups. The first already exists.

**Common conversations (28, existing).** `apertura`, `venta`, `objeciones`, `cierre`, `envio`, `desordenado`, `revendedor`, `arquitecto`, `sincatalogo`… in `scripts/test-bot.js`. They move into versioned fixtures, gaining deterministic expectations. **This is the expensive part and it is already done.**

Two things to know before treating them as "the corpus":

- **They are ITT's, not generic.** The messages say "me gusta el Nexery", "vinilo autoadhesivo", "empapelados", "almohadones". They serve the only real tenant perfectly well and cannot be reused for a customer in another trade without rewriting. That is fine; what is not fine is planning as though the corpus were portable.
- **They are coupled to the catalog.** "Me gusta el Nexery" depends on that product existing in `Product`. Change the catalog and scenarios unrelated to the change start failing. Each scenario should declare which business data it depends on, so that failure reads as "the catalog changed" rather than "the bot got worse".

**RAG usage.** Questions with an approved answer (must fire, must be grounded); questions *near* one but uncovered (must not fire, must not invent); greetings and one-word replies (prefilter must skip, zero embedding calls); a chunk targeted at a different agent.

**Edge cases.** Prompt injection in a customer message *and* in a knowledge chunk; contradictory instructions; a price the catalog does not have; abusive input; a mid-conversation language switch; empty and emoji-only messages.

**Prompt adherence.** One scenario per invariant in the briefing — voseo, WhatsApp formatting, never inventing business data, never exposing routing, escalation criteria.

Every scenario states **what it is for**, so a reviewer can tell when it stopped testing that. The corpus is versioned, and the version is stamped on every run (§7).

## 10. Instrumentation needed

One change, and every RAG measurement depends on it.

`AiService.chat()` returns `{ reply, statusChange, agentType, handToHuman, attachments }`. The `RetrievalOutcome` — `fired`, `topScore`, `chunkIds`, `prefilterHit`, `cacheHit`, `embedMs` — is computed inside and written to a log line, then discarded.

**Without surfacing it, "did the RAG behave correctly?" is only answerable by scraping logs.** Add it to the return as an optional field.

## 11. Metrics

- **Deterministic pass rate per check.** The trustworthy trend line (§4.1).
- **Mean score per rubric criterion**, always displayed with its judge model / prompt version / corpus version, and never drawn across a boundary (§7).
- **Judge–human agreement** on the labelled set (§6.4).
- **Retrieval precision on scenarios that should fire** — fired-and-correct vs fired-and-wrong. The number PRD 5 §8 wanted and could not produce.
- **Cost and latency per run.**

## 12. Phasing

| Phase | Scope | Why in this order |
|---|---|---|
| **A** | Fixtures: move the 28 scenarios out of `test-bot.js`, add deterministic expectations and their data dependencies | The corpus exists; this makes it addressable. No new infrastructure |
| **B** | Engine + deterministic checks + `EvalRun`/`EvalTurn`/`EvalCheck` storage; CLI front door | The measurement core. Runs locally, zero judge cost, and measures its own variance for free (§13 q3) |
| **C** | Judge: rubric, scores, version stamping (§7), the labelled agreement set (§6.4) | The interpretation layer. Built on B rather than instead of it — the deterministic pass rate is what tells you whether the judge is worth believing |
| **D** | `source: 'eval'` tagging and CRM filtering | Must land **before** the panel: the panel runs against production, and without this it fills the customer's CRM with fake leads |
| **E** | Panel: run, quick-check mode, results, run history | **The phase that serves the actual user** (§2.1). Everything before it is plumbing for engineers |
| **F** | `RetrievalOutcome` surfaced from `chat()` (§10), and raw model output for §4.2 | Two small returns that unblock the RAG and formatting measurements |

**B is useful on its own**, before any judge exists: it already catches an invented price, a broken escalation or a leaked routing marker, at zero cost per run.

**E is where the feature becomes real for the business.** An earlier draft put the panel last; that was wrong, because it would leave the people who make these changes waiting behind six phases of tooling built for someone else.

## 13. Open questions

1. **How often does production run?** With the cost measured (question 2), nightly is affordable at roughly $15/month. It is a product decision now, not a budget one.
2. ~~**Cost per run.**~~ **Measured, from `AiUsage` in the local environment.** A turn is **three** model calls, not two — there is a `classifier` alongside the orchestrator and the agent:

   | kind | costo promedio |
   |---|---|
   | `orchestrator` | $0.000148 |
   | `classifier` | $0.000156 |
   | `agent` | $0.000214 |
   | `faq_query` (embedding) | ~$0.000000 |

   ≈ **$0.0005 per turn**. A full run (28 scenarios × ~4 turns × N=3) is ~336 turns: **~$0.17 without the judge**, and with a more expensive judge per turn, on the order of **$0.50 total**.

   Cents, not dollars. That answers question 1: running it nightly costs about **$15 a month**, and cost stops being a reason to measure infrequently.

   **Caveat:** the number comes from `tenant-dev`, which may use cheaper models than ITT in production. Re-run the same query against production's `AiUsage` before fixing a cadence. The order of magnitude — cents per run — is unlikely to move.

   Who pays: judging is platform cost; the conversation runs on the tenant's key by construction.
3. **How many repetitions?** The system is non-deterministic; a single run of a scenario is one sample. N=3 is a guess until the variance is measured — which phase C can do for free by running the same scenario repeatedly and looking at the spread of deterministic results.
4. **Retention for eval contacts** in the tenant database (§8). Days, probably.
5. **Per-tenant retrieval settings.** `FAQ_RETRIEVAL_THRESHOLD`, `TOP_K`, `MAX_ANSWER_CHARS` and `EMBED_TIMEOUT_MS` are read once in `FaqRetrievalService`'s constructor and apply **process-wide**. Testing a different threshold therefore requires a separate deployment, even locally. The code's own comment says the intent was "retocarlo por tenant/vertical sin deploy". Four nullable columns on `Tenant` with env fallback would fix it — small, and it unlocks experimenting on what PRD 1 calls "el dial mas importante".

## 14. Risks

- **The trend line becomes a number nobody trusts.** The defences are §4.1's deterministic backbone, §6.4's agreement set, and §7's versioning. Skip them and this is decoration with a chart.
- **A judge change silently invalidates history.** The specific, likely failure. §7.1 is the whole answer, and it only works because §8 stores transcripts.
- **Optimising for the judge** rather than for customers. The deterministic checks and real transcripts are the counterweight.
- **Eval contacts leak into business metrics.** One missed call site and the customer's lead count is wrong. The filter needs a test, not just a code review.
- **A bad prompt reaches customers while it is being evaluated** (§5.3). The accepted cost of not building the override path. Mitigated by the quick-check mode and one-click revert, not eliminated — if it bites, the answer is the override path.
- **Cost surprises.** Bounded by phase C being free and by estimating before every judged run.
