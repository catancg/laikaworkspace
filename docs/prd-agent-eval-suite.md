# PRD 6 — Agent evaluation suite

**Status:** proposed
**Depends on:** [PRD 1 — RAG knowledge layer](prd-rag-knowledge-layer.md), [PRD 5 — full test of the RAG system](prd-rag-system-test.md)
**Related:** [briefing-contexto-agentes-ES.md](../briefing-contexto-agentes-ES.md) — the agents, their invariants, and the known failure modes

## 1. Why this exists

Every lever that decides what the bot says is non-deterministic and edited without a safety net: agent prompts live in a database table and are changed from the superadmin panel; the knowledge base is loaded by the customer; the retrieval threshold is an env var; the model can be swapped per tenant. Change any of them and the only way to find out what happened is to talk to the bot yourself and form an impression.

That is not a small gap. `0.78` is a threshold nobody has validated. A prompt edit that fixes one conversation can break three others, and nothing would say so. **Today, "did that change help?" is answered by vibes.**

This document specifies the instrument that answers it with evidence.

**Boundary with PRD 5.** PRD 5 tests the *plumbing*: is the retrieval query correct, does the invariant hold, can an unapproved chunk reach the bot. Deterministic, and either right or wrong. This document tests *behaviour*: given the plumbing works, does the bot answer well. Different question, different instrument, and conflating them produces a suite that is bad at both.

## 2. Goal

Make it possible to answer, before shipping a change to prompts, knowledge base, retrieval settings or model:

> **Is variant B better than variant A, on the conversations that actually matter, and where exactly did it get worse?**

The last clause matters as much as the first. An aggregate that says "B is 4% better" and cannot say *which conversations regressed* is not usable for a decision.

## 3. Non-goals

- **Not a correctness oracle.** The judge is another LLM. It is a measuring instrument with its own error, and §6.4 treats it as such rather than as a source of truth.
- **Not a replacement for reading transcripts.** The suite decides where to look; a human still reads the conversations it flags. PRD 5 §4 called this rung 5, and it does not scale — which is why the suite exists to point at the right 5 conversations instead of 100.
- **Not load or latency testing.** Latency appears only where it changes behaviour (the 800 ms embedding timeout).
- **Not testing the model provider.**

## 4. What gets measured, and by what

The single most important design decision: **most of what "does the agent follow its prompt?" means is not a judgment call.** Routing it through an LLM judge makes it slower, costlier, and less reliable than a regular expression.

### 4.1 Deterministic checks — free, instant, no variance

These are assertions over the reply text and the turn's metadata. They never need a judge:

| Check | Why it is deterministic |
|---|---|
| Invented a price or discount | The `precio` lint rule already exists (`faq-lint.ts`) and is tuned for Argentine price formats |
| WhatsApp formatting | `**markdown**`, `[text](url)`, `* ` bullets — `normalizeWhatsappText` already defines the contract |
| Leaked internal routing | The reply must never name an agent, `[[DERIVAR]]`, or the retrieved-knowledge markers |
| Said "diseñador de interiores" instead of "asesor" | A named business rule from the briefing |
| Greeted twice | Second turn onward must not re-greet |
| Escalation actually escalated | `handToHuman` is returned by `chat()` |
| Retrieval fired when it should | `RetrievalOutcome.fired` + `chunkIds` (needs §9) |
| Answer grounded in the retrieved chunk | `faq-support.ts` already computes exactly this |

**These run on every turn of every scenario, in both variants, at zero marginal cost.** A regression here is a hard failure, not a preference — it does not go to the judge and it does not get outvoted by "but B sounded nicer".

### 4.2 Judged comparisons — for what genuinely needs judgment

What survives after §4.1 is the part that requires reading: did it understand what the customer asked, was the answer useful, did it advance the sale, was escalating the right call. These go to the judge, **pairwise**.

## 5. The run model

Five nouns, and the whole system is these:

- **Variant** — a named, frozen configuration: agent prompts, knowledge-base snapshot, retrieval settings, models. What A and B *are*.
- **Scenario** — an ordered list of customer messages, plus its deterministic expectations. The 28 in `scripts/test-bot.js` are the starting corpus.
- **Turn** — one customer message and the bot's reply, with everything observable about it: `agentType`, `handToHuman`, `RetrievalOutcome`, tokens, cost, latency.
- **Run** — one variant executed over a set of scenarios, N times each.
- **Comparison** — two runs of the same scenarios, judged pairwise.

A run is reproducible in its *inputs*, never in its outputs (§6.3).

## 6. The judge

### 6.1 Pairwise, not scored

The judge sees the same conversation prefix and two candidate replies, and answers: **which is better, and why in one sentence?** Options: A, B, or tie.

Not 1–5 rubric scores. LLM judges are substantially more reliable choosing between two concrete outputs than assigning an absolute number, and absolute scores drift silently when the judge model changes — so a tracked "4.1 → 3.9" tells you nothing about whether the bot got worse or the judge did. Pairwise also needs no reference answers, which removes the largest authoring cost.

The cost: no absolute number to graph over time. §12 addresses this with deterministic pass rates, which *are* absolute and *are* stable.

### 6.2 Position bias is real and must be controlled

LLM judges systematically favour whichever candidate appears first. A suite that ignores this produces confident, wrong verdicts.

**Every pair is judged twice, with the order swapped.** If the two judgements disagree, the result is a **tie**, not a coin flip. The rate of order-disagreement is itself a metric: if it is high, the judge is not discriminating and the comparison is noise.

### 6.3 The system under test is non-deterministic

The same variant, run twice on the same scenario, produces different replies. So:

- Each scenario runs **N times per variant** (default 3), and a scenario's verdict is the aggregate, not a single sample.
- A comparison reports **win / loss / tie counts**, never a single "B is better".
- **A within-variant control is mandatory:** run A against A. Any apparent "win rate" there is pure noise, and it calibrates how large a real difference has to be before it means anything. A suite without this control will confidently report improvements that are variance.

### 6.4 The judge is itself under test

Keep a small set (~20 pairs) with a **human verdict recorded**. Every time the judge model or prompt changes, re-run it and report agreement with the humans. A judge that agrees 60% of the time is a random number generator with good grammar.

This set is also the answer to "why should I believe the suite?" — without it, there is no reason to.

### 6.5 The judge must not be the agent

Use a different, stronger model than the one under test, and never the tenant's own key silently — judge cost is platform cost, not the customer's. Self-judging inflates scores.

## 7. Test corpus

Four groups. The first already exists.

**Common conversations (28, existing).** `apertura`, `venta`, `objeciones`, `cierre`, `envio`, `desordenado`, `revendedor`, `arquitecto`, `sincatalogo`… These were curated against a real business and are the most valuable asset here. They move from `scripts/test-bot.js` into versioned fixtures, gaining deterministic expectations.

**RAG usage.** Questions with an approved answer in the knowledge base (must fire, must be grounded); questions *near* one but not covered (must not fire, must not invent); greetings and one-word replies (prefilter must skip, zero embedding calls); questions matching a chunk targeted at a different agent.

**Edge cases.** Prompt injection inside a customer message *and* inside a knowledge chunk; contradictory instructions; a customer asking for a price the catalog does not have; abusive input; a language switch mid-conversation; empty and emoji-only messages.

**Prompt adherence.** One scenario per invariant in the briefing — voseo, WhatsApp formatting, never inventing business data, never exposing routing, escalation criteria.

Every scenario states **what it is for**, so a reviewer can tell when it stopped testing that.

## 8. Isolation, and the cost of the drift run

Running the suite creates conversations. `AiService.chat()` takes a `contactId` and reads history from the database — it cannot be called with a bare message list — which is exactly why `/api/test-chat` persists `Contact` and `Message` rows, by design ("deja la conversación en /conversations como un lead real").

**Decision: a dedicated evaluation tenant**, seeded from a snapshot of the real tenant's prompts, knowledge base and catalog. The customer's CRM never sees a synthetic lead.

**The consequence, stated plainly:** the decision to *also* run occasionally against live production (§9 of the questions, "both baselines") cannot use that tenant. Two ways, and neither is free:

1. **A non-persisting execution path** — separate "produce a reply" from "record the conversation" in `AiService`/`CrmService`, which today are one thing. Clean, reusable, and a real refactor of the most load-bearing service in the system.
2. **Run against production and clean up** — rejected. A run that dies halfway leaves fake leads in a customer's CRM, and a cleanup filter with an off-by-one deletes a real one.

Option 1 is the answer, and it is the largest single engineering cost in this document. It is worth being explicit that the "compare against production too" requirement is what buys it — without that requirement, the eval tenant alone would do.

**Snapshot drift is the risk this creates.** A pinned snapshot that no longer resembles production means you are evaluating a system that does not exist. Mitigation: the snapshot records when it was taken and against which tenant; the drift run compares snapshot vs production on the same scenarios and reports divergence as a first-class number.

## 9. Instrumentation needed

One change, and everything about RAG measurement depends on it.

`AiService.chat()` returns `{ reply, statusChange, agentType, handToHuman, attachments }`. The `RetrievalOutcome` — `fired`, `topScore`, `chunkIds`, `prefilterHit`, `cacheHit`, `embedMs` — is computed inside and written to a log line, then discarded.

**Without surfacing it, "did the RAG behave correctly?" is only answerable by scraping logs**, which is fragile and untestable. Add it to the return as an optional field (or a debug-scoped one), so the eval records per turn which chunks were retrieved and at what score.

This also makes the retrieval preview in the panel and the eval share one source of truth.

## 10. Data model

Stored in the **master** database, not per tenant — a comparison spans variants and must outlive any one tenant snapshot.

- `EvalVariant` — name, description, snapshot ref, model settings, created_by
- `EvalScenario` — key, group, messages, deterministic expectations, what it is for
- `EvalRun` — variant, scenario set, N, started/finished, cost, status
- `EvalTurn` — run, scenario, repetition, turn index, reply, `agentType`, retrieval outcome, deterministic check results, tokens, cost, latency
- `EvalComparison` — run A, run B, per-scenario verdicts, order-disagreement rate, judge model
- `EvalJudgement` — comparison, turn pair, verdict, one-line reason, order presented

`EvalTurn` keeps the full reply text. Aggregates without transcripts cannot be acted on.

## 11. Panel

Superadmin, because prompt editing is already there and the person doing it is not necessarily technical.

- **Variants** — create one from the current state of a tenant ("snapshot ITT now"), name it, describe what changed.
- **Run** — pick variant, scenario groups, N. Runs in the background (BullMQ, like every other long job here); progress visible; costs money and says so with an estimate before starting.
- **Compare** — pick two runs. Headline: wins / losses / ties, and the within-variant control beside it so the number can be read honestly. Below: **the regressions first**, each expandable to the full transcript of both variants side by side with the judge's reason.
- **Deterministic failures are a separate, non-negotiable list.** A price invented in variant B is not a "loss" to be averaged — it is a blocker.

## 12. Metrics

- **Deterministic pass rate per check.** Absolute, stable, comparable across months. This is what §6.1 gives up by choosing pairwise, recovered where it is trustworthy.
- **Win / loss / tie** vs the named baseline, with the A-vs-A control alongside.
- **Order-disagreement rate.** Judge health. Rising means the comparisons are noise.
- **Judge–human agreement** on the labelled set (§6.4).
- **Retrieval precision on scenarios that should fire** — fired-and-correct vs fired-and-wrong. This is the number PRD 5 §8 wanted and could not produce without this suite.
- **Cost per run**, tracked from the start. See §14 q2.

## 13. Phasing

| Phase | Scope | Why in this order |
|---|---|---|
| **A** | Fixtures: move the 28 scenarios out of `test-bot.js`, add deterministic expectations | The corpus already exists; this makes it addressable. No new infrastructure |
| **B** | Deterministic checks + a CLI runner against the eval tenant | Catches real regressions with zero judge cost. Useful alone |
| **C** | `RetrievalOutcome` surfaced from `chat()` (§9) | Unblocks every RAG-behaviour measurement |
| **D** | Pairwise judge, order-swapping, A-vs-A control | The comparison engine. Only meaningful once B and C exist |
| **E** | Data model + panel | Puts it in the hands of whoever edits prompts |
| **F** | Non-persisting execution path (§8) + production drift run | The largest refactor; deliberately last, and its cost should be re-confirmed before starting |
| **G** | Judge–human labelled set, RAG and edge-case scenarios | Ongoing rather than one-off |

**Phase B is worth shipping alone.** Deterministic checks over 28 real conversations, with no judge and no panel, would already catch an invented price or a broken escalation before a customer sees it.

## 14. Open questions

1. **How large a difference is real?** Cannot be answered in advance — the A-vs-A control in the first runs measures the noise floor, and only then can a threshold be set. Until that number exists, no comparison should be treated as decisive.
2. **Cost per run, and who pays.** A full run is 28 scenarios × ~4 turns × N=3 × 2 variants × (orchestrator + agent) calls, plus 2 judge calls per pair for order-swapping. Order-of-magnitude hundreds of model calls per comparison. Needs a real estimate before phase D, and a decision on whether judging is platform cost (recommended) or tenant cost.
3. **Does the eval tenant use the real tenant's OpenRouter key?** Isolation says no; fidelity says the model routing should match production. Probably: same models, platform key.
4. **Who curates the corpus?** New scenarios come from real failures. That requires someone reading production conversations and adding the ones that went wrong — the same person-shaped bottleneck as PRD 4's approval question, and the same answer is likely.
5. **Do scenarios stay static?** A fixed corpus is comparable over time and slowly stops representing reality. Versioning the corpus and reporting which version a run used is the minimum.
6. **Snapshot refresh cadence** (§8). Too rare and the pinned baseline becomes fiction; too often and nothing is comparable across time.

## 15. Risks

- **The suite becomes a number nobody trusts.** The defences are the A-vs-A control, order-swapping, and judge–human agreement. If those three are skipped for speed, the output is decoration.
- **Optimising for the judge.** Prompts get tuned to what the judge likes rather than what customers need. The deterministic checks and real transcripts are the counterweight; the labelled set is the alarm.
- **Cost surprises.** Bounded by starting with deterministic-only runs (free) and estimating before every judged run.
- **Snapshot drift** (§8) — evaluating a system that no longer exists, and not knowing it.
- **The refactor in phase F destabilises `AiService`.** It is the most load-bearing service in the product, and this splits its core path. It is last for that reason, and PRD 5's rung 2/3 harnesses should cover the message path before it starts.
