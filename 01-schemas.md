# Part 1: Data contracts

**Part 1 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

Every shape that crosses a component boundary is defined here, and only here. Rules *about* these shapes live elsewhere: validation in [Part 2](02-validation.md), engine policy in [Part 3](03-engine-state-machine.md), channel rendering in [Part 5](05-channel-whatsapp.md) and [Part 6](06-channel-web.md).

---

## 1. InboundMessage

The single normalized shape every parser produces, at stage 2. This is what makes the engine channel agnostic.

### 1.1 Field definition (MUST)

| Field | Type | Required | Notes |
|---|---|:---:|---|
| `conversation_id` | string | ✅ | Stable per customer per channel. Format `cnv_` + ULID. Resolved by the parser, created if absent. |
| `channel` | enum | ✅ | `whatsapp` \| `web` |
| `user_id` | string | ✅ | Channel-native identity. E.164 phone for WhatsApp, opaque session subject for web. |
| `provider_message_id` | string | ✅ | Globally unique. **The deduplication key.** Synthesised for web, see [Part 6](06-channel-web.md#3-parsing-a-web-event-must). |
| `received_at` | string | ✅ | ISO 8601 UTC, from the provider where available, otherwise time of receipt |
| `message_type` | enum | ✅ | `TEXT` \| `INTERACTIVE` \| `MEDIA` \| `LOCATION` \| `POSTBACK` |
| `text` | string \| null | | Body text, or the caption of a media message |
| `selection` | object \| null | | `{ id, title }`, set when the customer chose a presented option |
| `media` | array | | `[{ id, mime_type, url, caption, size_bytes }]`, **references only** |
| `location` | object \| null | | `{ latitude, longitude, name, address }` |
| `locale_hint` | string \| null | | BCP-47, `en` or `sw`. A hint only: the engine may override it after classification. |
| `context` | object | ✅ | `{ listing_id, utm, landing_page, referrer }`, all nullable. Present but empty on WhatsApp. |
| `reply_to_message_id` | string \| null | | The `provider_message_id` being replied to |
| `raw` | object | ✅ | The untouched source payload, for audit and debugging |

### 1.2 Example

```json
{
  "conversation_id": "cnv_01J8XQZ4M7T2K9V3",
  "channel": "whatsapp",
  "user_id": "+255712345678",
  "provider_message_id": "wamid.HBgMMjU1NzEyMzQ1Njc4",
  "received_at": "2026-08-10T09:14:22Z",
  "message_type": "TEXT",
  "text": "nataka brake pads za Harrier",
  "selection": null,
  "media": [],
  "location": null,
  "locale_hint": "sw",
  "context": { "listing_id": null, "utm": {}, "landing_page": null, "referrer": null },
  "reply_to_message_id": null,
  "raw": { }
}
```

### 1.3 JSON Schema

Authoritative. Where prose and this block disagree, this block wins.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://kiboauto.co.tz/schema/inbound-message.json",
  "title": "InboundMessage",
  "type": "object",
  "required": ["conversation_id","channel","user_id","provider_message_id",
               "received_at","message_type","context","raw"],
  "additionalProperties": false,
  "properties": {
    "conversation_id":     { "type": "string", "pattern": "^cnv_[0-9A-HJKMNP-TV-Z]{16,26}$" },
    "channel":             { "enum": ["whatsapp","web"] },
    "user_id":             { "type": "string", "minLength": 1 },
    "provider_message_id": { "type": "string", "minLength": 1 },
    "received_at":         { "type": "string", "format": "date-time" },
    "message_type":        { "enum": ["TEXT","INTERACTIVE","MEDIA","LOCATION","POSTBACK"] },
    "text":                { "type": ["string","null"] },
    "selection": {
      "type": ["object","null"], "required": ["id"], "additionalProperties": false,
      "properties": { "id": { "type": "string" }, "title": { "type": ["string","null"] } }
    },
    "media": {
      "type": "array",
      "items": {
        "type": "object", "required": ["id","mime_type"], "additionalProperties": false,
        "properties": {
          "id":         { "type": "string" },
          "mime_type":  { "type": "string" },
          "url":        { "type": ["string","null"] },
          "caption":    { "type": ["string","null"] },
          "size_bytes": { "type": ["integer","null"] }
        }
      }
    },
    "location": {
      "type": ["object","null"], "required": ["latitude","longitude"],
      "properties": {
        "latitude":  { "type": "number" },
        "longitude": { "type": "number" },
        "name":      { "type": ["string","null"] },
        "address":   { "type": ["string","null"] }
      }
    },
    "locale_hint": { "type": ["string","null"], "enum": ["en","sw",null] },
    "context": {
      "type": "object", "additionalProperties": false,
      "properties": {
        "listing_id":   { "type": ["string","null"] },
        "utm":          { "type": "object" },
        "landing_page": { "type": ["string","null"] },
        "referrer":     { "type": ["string","null"] }
      }
    },
    "reply_to_message_id": { "type": ["string","null"] },
    "raw": { "type": "object" }
  }
}
```

### 1.4 The rule that makes one configuration serve both channels (MUST)

A WhatsApp numbered text reply (`"2"`), a WhatsApp interactive list selection, and a web quick-reply click **all normalize to the same `selection.id`**.

The parser is responsible for that resolution, not the engine. Concretely: when the previous outbound message offered options, the parser matches the customer's reply against those options and populates `selection`. Only when no option matches does the message stay `TEXT`.

**NOTE** this single rule is why flows never branch on channel. Without it, every state in every flow would need channel-specific handling, and the configuration-driven design collapses.

### 1.5 Media (MUST)

`media[]` carries **references, never bytes**. Downloading, virus scanning and re-hosting are the responsibility of a separate media pipeline, which is phase two. Provider-hosted URLs expire: treat `url` as a short-lived fetch hint and `id` as the durable handle.

---

## 2. The standard meta response

What the engine returns at stage 4, always, regardless of channel. Transports translate it. **The engine never emits channel syntax.**

### 2.1 MessageType (MUST)

| Value | Meaning |
|---|---|
| `TEXT` | Plain text |
| `BUTTONS` | Text plus a small set of discrete choices |
| `LIST` | Text plus a larger set of choices, optionally grouped |
| `CARDS` | Rich items with image, title, subtitle and actions |
| `TEMPLATE` | A pre-approved WhatsApp template with variables |
| `IMAGE` · `DOCUMENT` | Media with optional caption |
| `LOCATION_REQUEST` | Ask the customer to share location |
| `CTA_URL` | Text plus a single outbound link action |

### 2.2 GenericMessage (MUST)

| Field | Type | Applies to |
|---|---|---|
| `type` | `MessageType` | all |
| `text` | string | all except pure media |
| `header` · `footer` | string \| null | `BUTTONS`, `LIST`, `CARDS` |
| `buttons[]` | `{ id, title }` | `BUTTONS`, **max 3**, `title` ≤ 20 chars |
| `button_text` | string | `LIST`, the label that opens the list |
| `sections[]` | `{ title, rows: [{ id, title, description }] }` | `LIST`, **max 10 rows total** |
| `cards[]` | `{ id, title, subtitle, image_url, actions: [{ id, title, url }] }` | `CARDS` |
| `media` | `{ id, mime_type, url, caption }` | `IMAGE`, `DOCUMENT` |
| `template_name` · `template_language` · `template_variables` | string · string · object | `TEMPLATE` |

The per-type required-field matrix is a validation concern and lives in [Part 2, section 4](02-validation.md#4-genericmessage-validity-must).

**NOTE** the button and row limits come from WhatsApp. They are enforced in the schema rather than in the WhatsApp transport deliberately: a flow author gets an error at configuration-lint time instead of a silently degraded message in production.

### 2.3 OrchestrationResult (MUST)

What the orchestrator returns for one inbound message.

```json
{
  "success": true,
  "conversation_id": "cnv_01J8XQZ4M7T2K9V3",
  "user_id": "+255712345678",
  "current_state": "ask_part_name",
  "previous_state": "ask_vehicle",
  "intent_detected": "spare_parts",
  "messages": [ { "type": "TEXT", "text": "Ni sehemu gani unahitaji?" } ],
  "meta": {
    "flow_id": "spare_parts_order",
    "flow_version": 3,
    "locale": "sw",
    "bot_mode": "active",
    "requires": "user_input",
    "escalate": null,
    "guardrails": { "passed": true, "codes": [] },
    "trace_id": "trc_01J8XQZ55P",
    "turn_index": 4
  }
}
```

| `meta` field | Values | Used by |
|---|---|---|
| `flow_id` · `flow_version` | string · integer | Turn log, and [version pinning](03-engine-state-machine.md#73-rules-must) |
| `locale` | `en` \| `sw` | Transport, analytics |
| `bot_mode` | `active` \| `paused` \| `handoff` | Transport, CRM, web `bot_mode` event |
| `requires` | `user_input` \| `tool_result` \| `nothing` | Orchestrator loop, web `state` event |
| `escalate` | `null` \| `{ reason, queue, priority }` | CRM handoff |
| `guardrails` | `{ passed, codes[] }` | Turn log, quality review |
| `trace_id` | string | Correlates all logs for one turn |

**MUST** `messages` may be empty. An empty array with `bot_mode: "handoff"` is the correct, expected result when a human owns the conversation. It is not an error, and the transport must send nothing.

**NOTE** `meta` is the operational half of the contract. The turn log, the CRM event, the analytics pipeline and the future evaluation harness all read it. Treat it as stable: adding fields is safe, renaming or removing them is a breaking change.

---

## 3. ConversationContext

The working state of one conversation. Held in Redis at `conv:ctx:{conversation_id}`, see [Part 3, section 5](03-engine-state-machine.md#5-state-storage-redis), and rebuildable from the durable store.

### 3.1 Field definition (MUST)

| Field | Type | Notes |
|---|---|---|
| `conversation_id` | string | Matches `InboundMessage.conversation_id` |
| `user_id` | string | Channel-native identity |
| `channel` | enum | `whatsapp` \| `web` |
| `flow_id` | string \| null | The active journey, null when no flow is running |
| `flow_version` | integer \| null | **Pinned** at flow start, held until the flow terminates |
| `current_state` | string \| null | State name within `flow_id` |
| `previous_state` | string \| null | For corrections and for the turn log |
| `variables` | object | Collected slot values, keyed by slot name |
| `locale` | enum | `en` \| `sw`, the engine's decision, not the parser's hint |
| `bot_mode` | enum | `active` \| `paused` \| `handoff` |
| `turn_index` | integer | Increments once per processed turn |
| `last_activity_at` | string | ISO 8601 UTC, drives the sliding TTL |
| `last_inbound_at` | string | ISO 8601 UTC, drives the WhatsApp 24-hour window check |

**MUST** `variables` holds only what the flow declared as slots. It is not a scratch pad, and nothing outside the flow configuration may write to it. A context that accumulates undeclared keys cannot be validated, migrated or reasoned about.

**NOTE** `locale` and `InboundMessage.locale_hint` are deliberately different fields. The hint is what the channel claims, the context value is what the engine decided. A customer whose WhatsApp profile is English but who writes in Swahili gets Swahili.

---

## 4. Tool interface shapes

Shapes only. The dispatch policy, caching rules and idempotency derivation are in [Part 3, section 8](03-engine-state-machine.md#8-tool-dispatch). The tools themselves are phase two.

### 4.1 ToolContext (MUST)

Passed to every tool invocation.

| Field | Type | Notes |
|---|---|---|
| `conversation_id` | string | |
| `user_id` | string | |
| `locale` | string | For any customer-visible strings the tool returns |
| `flow_id` · `flow_version` | string · integer | For tracing |
| `variables` | object | Current slot values |
| `trace_id` | string | Correlates with the turn log |

### 4.2 ToolResult (MUST)

Returned by every tool, on success and on failure.

| Field | Type | Notes |
|---|---|---|
| `success` | boolean | |
| `data` | object \| null | **Shaped** result, never a raw upstream payload |
| `error` | object \| null | `{ code, message }`, where `code` maps to configuration copy |
| `metadata` | object | Latency, cache hit, upstream status |

**MUST** `data` is shaped for the conversation: the handful of fields the flow needs, named for this domain. Passing an upstream response through unchanged couples every flow to an API's internal shape.

**MUST** a tool never returns customer-facing prose. It returns data and an error `code`, and the *configuration* supplies the wording. This keeps copy bilingual and reviewable in one place.

---

## 5. ExecutionResult

What an executor returns for one action. The orchestrator merges these in declaration order.

| Field | Type | Notes |
|---|---|---|
| `success` | boolean | A failed action routes per its `on_error`, it does not throw the turn away |
| `messages` | array | Zero or more `GenericMessage`. Appended in order |
| `variables` | object | Slot or working values to merge into the context |
| `transition` | string \| null | A proposed transition value. The **first** non-null across the state's actions wins |
| `tool_result` | object \| null | The `ToolResult`, bound to `${tool.result}` for later actions in the same state |
| `error` | object \| null | `{ code, message }` |

**MUST** an executor returns this shape and writes nothing. Persistence belongs to the orchestrator, which is what makes a turn atomic. See [Part 3, section 1.2](03-engine-state-machine.md#12-call-rules-must).

---

## 6. Knowledge shapes

`KnowledgeDocument` and `KnowledgeSnippet` cross the same boundaries as the shapes above, and are defined with the retrieval rules that give them meaning, in [Part 8](08-knowledge-and-model.md#2-the-document-contract).

---

**Next:** [Part 2, Validation and integrity](02-validation.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
