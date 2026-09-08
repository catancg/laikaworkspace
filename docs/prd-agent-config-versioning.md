# PRD 7 — Versioning and revert for agent configuration

**Status:** proposed
**Pairs with:** [PRD 6 — agent evaluation suite](prd-agent-eval-suite.md). PRD 6 measures what the bot does; this one owns how its configuration is changed and undone. Neither depends on the other to ship, but together they close the loop.
**Related:** [PRD 2 — FAQ content ingestion](prd-faq-content-ingestion.md) — where this governance already exists, for the other half of the system

## 1. Why this exists

`Agent` has a single `prompt` column. `AgentsService.update()` overwrites it in place:

```ts
return this.db(tenantDb).agent.update({ where: { id }, data: { ...data, ... } });
```

There is no draft, no history, no previous value. A superadmin edits the `ventas` prompt in the panel; it is live on the next customer message; the text that was there is **gone**. If the new one is worse, there is nothing to go back to except somebody's memory of what they replaced.

`BotRule` and `BusinessProfile` — the owner's rules and the business facts, both injected into every prompt — are the same: `updatedAt` and nothing else.

### 1.1 The inversion worth naming

Compare that with the other half of the system:

| | Governance |
|---|---|
| `FaqChunk` — content the **customer** uploads | `version`, `superseded_by`, `updated_by`, `reviewed_by`, `reviewed_at`, an approval gate, a review queue, batch withdraw, and a lint |
| `Agent` / `BotRule` / `BusinessProfile` — the **prompts and rules** that decide how the bot behaves | `updatedAt` |

PRDs 2 and 4 built an entire governance apparatus around customer-supplied content, on the correct reasoning that it reaches real customers and nobody should be able to publish unreviewed.

**Every word of that reasoning applies harder to the prompts**, which are edited more often, by less technical people, and have wider blast radius: a bad chunk affects the answers that retrieve it, a bad prompt affects *every conversation the bot has*.

The asymmetry is not a decision anyone made. It is an accident of the order things were built in.

## 2. Goal

Make a configuration change **reversible in one click**, and make it obvious what changed, when, and who did it.

Explicitly *not* "make it impossible to break things". Prompt editing is iterative work and gating it behind approval would stop the people doing it. The target is **fast recovery**, not prevention.

## 3. Non-goals

- **No draft or staging state.** Considered and declined: it would require threading prompt overrides through `AiService`'s prompt assembly, the hottest path in the product. The chosen workflow saves first and evaluates after (§4), accepting the exposure window that creates.
- **No approval queue for prompts.** `FaqChunk`'s gate exists because customer-supplied content reaches the bot unreviewed. Prompts are edited by the platform side, and a queue would make iteration unusable.
- **No branching or merging.** Linear history, one live version, and a way back.
- **Not a diff tool for prose.** Show two versions side by side; a person reads them.

## 4. The workflow this enables

```
edit  →  save  →  [PRD 6 runs the suite]  →  keep, or revert in one click
```

**The change is live while it is being evaluated.** A run takes minutes, and during those minutes real customers can receive the version under test. That is the accepted cost of not building a draft state (§3), and two things bound it:

- **A quick-check mode** — deterministic checks only, a subset of scenarios, no judge (PRD 6 §4.1). Seconds rather than minutes, and it catches the categorical failures: invented price, broken escalation, leaked routing. This should be the *default* after a prompt edit, with the full judged run as a deliberate second step.
- **Revert is one click and takes effect immediately** — recovery time is what matters here, precisely because prevention is what was not bought.

If this trade ever proves wrong — a bad prompt reaches enough customers to matter — the answer is the override path, and that is the moment to pay for it. Not before.

## 5. What gets versioned

**`Agent.prompt`** is the primary target: edited most, riskiest, and the reason this document exists. Also `displayName`, `tools` and `active`, since disabling an agent changes routing.

**`BotRule`** — rules are injected into every system prompt, and they are added and removed casually.

**`BusinessProfile`** — the business facts. Changed rarely, but a wrong shipping policy is a wrong answer in every conversation that touches shipping.

Out of scope: `FaqChunk` already has all of this, and `Product` is a catalog with its own lifecycle.

## 6. Shape

A single revision table per versioned entity, or one shared table with a discriminator — an implementation call, not a product one. What matters is what a revision records:

- **what it was** — the full previous value, not a diff. Diffs need their base to be reconstructable; storing the whole text removes a class of bug and prompts are kilobytes
- **who** changed it, and **when**
- **why**, optionally — a one-line note the editor can leave. Cheap, and it is the difference between a history you can read and a list of timestamps
- **the eval run active at the time**, if any (PRD 6 `EvalRun.id`) — this is what connects "we changed this" to "and here is what it did"

Two rules:

- **The revision is written in the same transaction as the update.** A history that can silently miss an entry is worse than none, because you would trust it.
- **Revert is a new revision, not a deletion.** Reverting from v3 to v2 creates v4 with v2's content. History is append-only; "we reverted" is itself a fact worth keeping.

`invalidateAgentCache()` already exists in `AiService` and must be called on both save and revert — prompts are cached, and a revert that leaves the cache warm does nothing visible.

## 7. Panel

In the agent editor, next to the prompt:

- **"Historial"** — the revision list: when, who, the note. Selecting one shows it beside the current text.
- **"Volver a esta versión"** — with a confirmation naming what will change, since it takes effect on the next customer message.
- **The active eval run**, if one is in flight, shown inline: *"Se está evaluando este cambio"* with a link to the results.

For `BotRule` and `BusinessProfile`, the same list without the side-by-side — those are short enough to read whole.

## 8. Retention

Prompts are kilobytes and edits are infrequent. **Keep every revision indefinitely.** Trimming would save nothing measurable and would delete exactly the old version somebody eventually needs.

The one bound worth having: a cap on revisions per entity per day, to stop an accidental loop from writing thousands of rows.

## 9. Metrics

- **Reverts per month**, and time from save to revert. A short median means the loop is working; a long one means changes are being noticed by customers rather than by a run.
- **Share of prompt edits with an eval run attached.** If it is low, PRD 6 exists and nobody is using it.
- **Edits with no note.** Not a target to enforce, just a signal about whether the history is going to be readable in six months.

## 10. Phasing

| Phase | Scope | Why in this order |
|---|---|---|
| **A** | Revision table + write-on-save for `Agent`, in the same transaction | The whole risk-reduction is here. Independent of PRD 6, and shippable on its own |
| **B** | Panel: history list, side-by-side, revert with cache invalidation | Makes A usable by the person who needs it |
| **C** | Same for `BotRule` and `BusinessProfile` | Mechanical once A and B exist |
| **D** | Link revisions to `EvalRun` (PRD 6) and surface the connection in both panels | Only meaningful once PRD 6 phase B exists |

**A and B together are the point of this document** and are small: one table, one write, one screen. They are also the only thing here that reduces risk *before* any measurement exists — which is why, of everything across PRD 6 and 7, this is what I would build first.

## 11. Open questions

1. **One shared revision table or one per entity?** Shared is fewer migrations and a uniform panel; per-entity is simpler to query and to type. Leaning shared, with a `entity_type` discriminator.
2. **Does `active: false` on an agent need a revision?** It changes routing as surely as editing the text does. Probably yes, and it makes the history noisier.
3. **Should revert be restricted to superadmin?** Prompt *editing* is already superadmin-only. Revert is strictly safer than editing, so the same permission is the obvious answer unless tenant admins ever get prompt access.
4. **Is a note mandatory?** Forcing one produces "asdf". Leaving it optional produces none. Prefilling it with what changed ("prompt de ventas, 3 líneas") is probably better than either.

## 12. Risks

- **The history is trusted and incomplete.** The failure mode of writing the revision outside the update's transaction. §6 makes it one transaction for exactly this reason, and it needs a test that kills the write mid-way.
- **Revert without cache invalidation.** Silent no-op: the panel says it reverted, the bot keeps using the cached prompt, and the next eval blames the wrong thing.
- **A bad prompt reaches customers during evaluation** (§4). Accepted, bounded by quick-check and one-click revert, not eliminated.
- **History nobody reads.** Mitigated by notes and by linking revisions to eval runs — a revision that says *"this is the change that dropped the pass rate to 21/28"* gets read.
