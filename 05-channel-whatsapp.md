# Part 5: WhatsApp channel (Ghala)

**Part 5 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

Both directions of one channel, in one file: stage 2 inbound and stage 5 outbound. All WhatsApp-specific knowledge lives here and nowhere else. If a channel name appears anywhere outside this file and [Part 6](06-channel-web.md), it is a defect.

---

## 1. Order of operations (MUST)

1. **Verify the signature** on the raw body before parsing anything. Reject invalid, stale or replayed requests with `401`. Never parse an unverified payload.
2. **Classify the payload**: an inbound *message* or a *status callback*. These take different paths.
3. **Acknowledge `2xx` within 200 ms**, then process. A slow webhook is retried, and a retry you process is a duplicate order.
4. Parse to [InboundMessage](01-schemas.md#1-inboundmessage).

The parser is otherwise a pure function: raw payload in, `InboundMessage` out. It makes no engine calls and has no side effects. Its one exception is reading the last presented option set, see [section 4](#4-the-selection-rule-for-plain-text-must).

---

## 2. Inbound mapping

### 2.1 Field mapping

| Source | To `InboundMessage` |
|---|---|
| `messages[0].from` | `user_id`, normalized to E.164 with `+` |
| `messages[0].id` | `provider_message_id` |
| `messages[0].timestamp` | `received_at`, epoch seconds to ISO 8601 UTC |
| `messages[0].context.id` | `reply_to_message_id` |
| entire payload | `raw` |
| not from the payload | `channel` = `whatsapp`, `context` = an all-null object |

### 2.2 Per-subtype mapping

| Source subtype | `message_type` | `text` | `selection` | `media` |
|---|---|---|---|---|
| `text.body` | `TEXT` | the body | see [section 4](#4-the-selection-rule-for-plain-text-must) | |
| `interactive.button_reply` | `POSTBACK` | `.title` | `{ id: .id, title: .title }` | |
| `interactive.list_reply` | `POSTBACK` | `.title` | `{ id: .id, title: .title }` | |
| `button` (template quick reply) | `POSTBACK` | `.text` | `{ id: .payload, title: .text }` | |
| `image` · `document` · `audio` · `video` | `MEDIA` | `.caption` | | one entry from `.id`, `.mime_type` |
| `location` | `LOCATION` | | | populates `location` instead |

### 2.3 Locale hint

Derive from the WhatsApp profile language where present, otherwise `null`. Do not guess from message content at parse time: language classification belongs to the engine.

---

## 3. The numbered-text fallback

When an option set cannot be rendered as an interactive message, the transport sends it as text:

```
{text}

1. {option 1 title}
2. {option 2 title}
3. {option 3 title}

{footer}
```

The transport records the option set at `conv:opts:{conversation_id}` so the parser can resolve a numeric reply back to a `selection.id`.

---

## 4. The selection rule for plain text (MUST)

When the previous outbound message for this conversation offered options (`BUTTONS`, `LIST`, or the numbered-text fallback), the parser attempts to resolve the incoming text against them, in this order:

1. Exact match on a presented option id
2. Numeric match on the numbered fallback position, so `"2"` is the second option
3. Case-insensitive, whitespace-trimmed match on an option title

On a match, set `message_type: "POSTBACK"` and populate `selection`. On no match, leave it as `TEXT` and let the engine treat it as free input.

**NOTE** this requires the parser to read the last presented option set, stored at `conv:opts:{conversation_id}`, see [Part 3, section 5.1](03-engine-state-machine.md#51-keys-must). It is the one place a parser touches stored state, and it is deliberate: doing it here keeps the *engine* free of channel-shaped input handling. Implementations may pass the option set in rather than reading storage, as long as the observable result is the same.

---

## 5. Outbound rendering (MUST)

How one [GenericMessage](01-schemas.md#22-genericmessage-must) becomes a WhatsApp payload.

| `GenericMessage.type` | Condition | Rendered as |
|---|---|---|
| `TEXT` | | text message |
| `BUTTONS` | 3 buttons or fewer | interactive reply buttons |
| `BUTTONS` | more than 3 buttons | **convert to `LIST`**, then apply the `LIST` rules |
| `LIST` | 10 rows or fewer in total | interactive list |
| `LIST` | more than 10 rows | numbered-text fallback, [section 3](#3-the-numbered-text-fallback) |
| `CARDS` | | text summary plus one link per card, or a media message per card where an image exists |
| `IMAGE` · `DOCUMENT` | | media message with caption |
| `LOCATION_REQUEST` | | interactive location request |
| `CTA_URL` | | interactive CTA-URL message |
| `TEMPLATE` | | template send with stored name, language and variables |
| *any* | **outside the 24-hour window** | **`TEMPLATE` only**, see [section 6](#6-the-24-hour-window-must) |

The universal exits in [Part 4, section 7](04-flow-config.md#7-universal-exits-must) are never rendered as options. They are always available and never take up one of the three button slots.

---

## 6. The 24-hour window (MUST)

Before sending, the transport checks the time since the customer's last inbound message, held as `last_inbound_at` on the [ConversationContext](01-schemas.md#3-conversationcontext).

- **Inside 24 hours:** send as rendered above.
- **Outside 24 hours:** only a pre-approved `TEMPLATE` may be sent. Any other `type` **must not** be sent. The transport returns a delivery failure with reason `WINDOW_EXPIRED`, the engine records it, and the CRM surfaces it.

**NOTE** this is a WhatsApp platform rule, not a Ghala rule, and it cannot be worked around. It is enforced in the transport because it is a channel fact. The flow does not know about it, and must not.

**NOTE** the window applies to human agent replies too. An agent answering this morning a message from yesterday morning needs a template. This has to be visible in the CRM composer, or agents will write replies that silently fail to deliver.

---

## 7. Status callbacks (MUST)

Delivery callbacks (`sent`, `delivered`, `read`, `failed`) **never enter the flow**. They update message delivery state and are mirrored to the CRM.

A delivery receipt is not a customer turn. Treating one as a turn advances flows spuriously, and the symptom, a conversation that moves on its own while the customer is asleep, is confusing enough to be worth this much emphasis.

---

**Next:** [Part 6, Web channel](06-channel-web.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
