# PRD 9 — Line breaks in manual replies

**Status:** propuesto
**Repo:** `soylaika.frontend` only — no backend change, no migration
**Screen:** `/contacts/[id]`, the manual reply box ([page.tsx:282](../soylaika.frontend/app/(crm)/contacts/[id]/page.tsx))

---

## 1. Problem

When a seller takes a conversation off the bot and answers by hand, they cannot write a
message with more than one line. Everything they type collapses into a single paragraph.

That matters because the messages a seller sends by hand are exactly the ones that need
structure: a price list, an address plus a schedule, three payment options. On WhatsApp those
are written as separate lines. Sent as one run-on paragraph they are harder to read than the
bot's own replies — which *do* arrive multi-line, because the model emits `\n` and every layer
below preserves it.

So the seller is the only participant in the conversation who cannot use the Enter key.

### 1.1 Why Shift+Enter does nothing today

The reply box is a single-line `<Input>`:

```tsx
<Input
  value={message}
  onChange={(e) => setMessage(e.target.value)}
  onKeyDown={(e) => e.key === "Enter" && !e.shiftKey && handleSend()}
  disabled={botActive || sending}
  className="h-11 rounded-full ..."
/>
```

The `!e.shiftKey` guard reads as if Shift+Enter were already supported. It is not, and never
was: an `<input>` element cannot hold a newline character, so Shift+Enter falls through to a
default action that has nothing to do. The guard is inert.

**This is the whole bug.** The fix is an element change, not a protocol change.

## 2. What already works — verified against the code, not assumed

Before designing anything, each layer between the textarea and the customer's phone was
checked for something that would strip or mangle a `\n`. Nothing does.

| Layer | Behaviour | Where |
|---|---|---|
| `handleSend` | `message.trim()` — drops leading/trailing whitespace, keeps interior newlines | [page.tsx:142-151](../soylaika.frontend/app/(crm)/contacts/[id]/page.tsx) |
| API client | `JSON.stringify({ text })` — `\n` is escaped and restored, not lost | [lib/api.ts:616](../soylaika.frontend/lib/api.ts) |
| Controller | forwards `body.text` verbatim to `whatsapp.sendManual` | [crm.controller.ts:62-69](../soylaika.backend/src/crm/crm.controller.ts) |
| Persistence | `message.create({ content: text })` — stored as typed | [whatsapp.service.ts:107](../soylaika.backend/src/whatsapp/whatsapp.service.ts) |
| Graph API | posted as `text: { body: text }`; WhatsApp renders `\n` as a line break | [whatsapp.service.ts:112-130](../soylaika.backend/src/whatsapp/whatsapp.service.ts) |
| Bubble render | `whitespace-pre-line` — newlines already display, for both roles | [page.tsx:247](../soylaika.frontend/app/(crm)/contacts/[id]/page.tsx) |

Two consequences worth stating plainly, because they set the size of this work:

- **The feature is one file.** No endpoint, no DTO, no allowlist, no tenant migration. Nothing
  here needs pre-applying before a deploy, and nothing here can reach a tenant database.
- **The rendering half is already built.** `whitespace-pre-line` is on the shared bubble, which
  is why the bot's multi-line replies look right today. A multi-line manual reply will display
  correctly the moment one can be typed.

## 3. Decision: Enter sends, Shift+Enter breaks the line

The alternative — Enter inserts a newline, Ctrl+Enter sends — is safer against half-typed
messages going out to a real customer, and was rejected anyway:

- Every seller using this screen today sends with Enter. Changing that silently converts a
  reflex into a mistake, on the one screen where a mistake is a message a customer receives.
- It is the WhatsApp Web and Instagram convention, which is the tool these sellers switch to
  and from all day.
- It is what the existing code already claims to do. The `!e.shiftKey` guard becomes true
  instead of decorative.

The ➤ button keeps working as a second send path, unchanged.

## 4. The change

### 4.1 The `<Input>` becomes a `<textarea>`

There is no `components/ui/textarea` primitive in this repo; the notes field and the chat-test
input are both hand-styled elements. Follow that. The textarea keeps the current visual: 44px
tall at one row, pill-shaped (`rounded-[22px]`, not `rounded-full` — a full radius on a grown
box bulges), `resize-none`, same hex light/dark class pairs as the `<Input>` it replaces.

The row wrapping it (`flex gap-3`) needs `items-end` so the buttons stay aligned to the bottom
of the box as it grows upward, rather than centering against a tall textarea.

### 4.2 Key handling

```tsx
onKeyDown={(e) => {
  if (e.key === "Enter" && !e.shiftKey && !e.nativeEvent.isComposing) {
    e.preventDefault();
    handleSend();
  }
}}
```

Three details, each of which is a bug if omitted:

- **`e.preventDefault()` is new and mandatory.** Today's handler omits it, which is harmless on
  an `<input>`. On a textarea, without it the browser inserts the newline *and* the send fires —
  the message goes out with a trailing break and the box is left holding a stray line.
- **`isComposing`** guards IME and dead-key composition. Cheap insurance on a Spanish-language
  product where accented characters may arrive through a compose sequence.
- **Enter on an empty box does nothing at all** — `preventDefault` runs, `handleSend` returns
  early on `!message.trim()`. This is the intended behaviour, not an oversight: an empty box
  should not accumulate blank lines.

### 4.3 Growing and resetting

The box grows from 1 row (44px) to a ceiling of 6 rows (144px), then scrolls internally. Growth
is done by setting `style.height = "auto"` and then `scrollHeight`, clamped in JS between those
two constants, called from `onChange` — not from a `useEffect`, to avoid adding to the repo's
non-zero `react-hooks/set-state-in-effect` lint baseline. The clamp lives in JS rather than in a
`max-h-*` class because the inline height set by the same function would otherwise fight it.

`scrollHeight` excludes the borders under `box-sizing: border-box`, so the measurement adds
`offsetHeight - clientHeight` back. Without it the box lands 2px short and jitters.

`handleSend` clears `message` by setting state, which fires no `onChange`, so **the height must
be reset explicitly after a send** or the box stays six rows tall over an empty value.

**Do not reset it by measuring.** Calling the grow function right after `setMessage("")` reads a
DOM that still holds the old text — React has not re-rendered yet — so it measures the long
value and the box stays tall. The reset assigns the one-row height directly, which is
deterministic and needs no measurement.

### 4.4 What does not change

`handleSend`'s `.trim()` stays exactly as it is — it strips the trailing newlines a seller
leaves behind and preserves the interior ones, which is the correct behaviour for both. The
`disabled={botActive || sending}` condition, the send button's `!message.trim()` guard, the
scroll-to-bottom and new-message-count logic: all untouched.

The manual placeholder gains a hint — "Escribí un mensaje manual (Shift+Enter para saltar de
línea)..." — because a key combination nobody is told about is a key combination nobody uses.
The bot-mode placeholder is unchanged.

## 5. Mobile is a known gap, accepted

A phone's soft keyboard has no Shift. Under §3, a seller on a phone can send but cannot insert
a line break — the same limitation they have today, neither better nor worse.

This is accepted rather than solved, because the alternative costs more than it returns right
now: branching on a coarse-pointer / touch check to make Enter insert a newline on mobile and
send only via the ➤ button adds a device-detection path that guesses wrong on hybrids
(touchscreen laptops, tablets with keyboards) and that no test here can cover.

**If it bites, that touch branch is the named remedy** — the decision is recorded here so the
next person does not re-derive it. The signal to watch for: sellers reporting that they compose
in WhatsApp Web on their phone and paste into the CRM.

## 6. Out of scope

- **The bot chat-test panel** ([chat-test/page.tsx:451](../soylaika.frontend/app/(crm)/admin/chat-test/page.tsx))
  has the identical single-line input and the identical inert `!e.shiftKey` guard. It stays as
  it is: it simulates an *inbound customer* message to an admin testing prompts, not a reply to
  a real person. Recorded here so it is not rediscovered as a bug — if it is ever changed, it is
  a copy of §4, not a new design.
- **Template/quick-reply insertion, drafts, formatting toolbars, attachments.** Unrelated.
- **Splitting a multi-line message into several WhatsApp messages.** The bot does something like
  this for its own replies; a manual reply is one message with line breaks in it, deliberately.
- **Instagram.** `sendManual` posts through the WhatsApp Graph endpoint only. Nothing here
  changes that, for better or worse.

## 7. Verification

There is no test suite in `soylaika.frontend`, so this is a manual checklist. It is not
optional: every item below is a behaviour that a plausible implementation gets wrong.

1. Shift+Enter inserts a line break; the box grows by one row.
2. Enter sends, and the sent bubble shows the line breaks — no trailing blank line.
3. **The message arrives multi-line on a real WhatsApp client**, not just in the CRM bubble.
   This is the only step that proves the end-to-end claim in §2.
4. After sending, the box collapses back to one row (§4.3).
5. Past ~6 rows the box stops growing and scrolls internally; the message list is not pushed
   off-screen.
6. Enter on an empty box does nothing — no send, no newline.
7. In bot mode the box is disabled and the placeholder still reads
   "El bot está respondiendo automáticamente...".
8. `npm run lint`: record the count before the change, and confirm it did not grow.

## 8. Risks

| Risk | Mitigation |
|---|---|
| A seller mid-sentence hits Enter and a partial message reaches a customer | Unchanged from today — Enter already sends. This PRD adds no new way to send. |
| Missing `e.preventDefault()` ships a trailing newline on every message | §4.2; checklist item 2. |
| The grown box eats the conversation view on a short screen | Hard ceiling at ~6 rows; checklist item 5. |
| Height not reset after send | §4.3; checklist item 4. |

## 9. Decisions taken

- Enter sends, Shift+Enter breaks the line (§3) — chosen over Ctrl+Enter-to-send.
- Mobile line breaks are out of scope, with the touch branch recorded as the remedy (§5).
- The chat-test panel is not touched (§6).

No open questions.
