# Part 2: Validation and integrity

**Part 2 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

Everything that decides whether data is allowed to proceed. These rules are gathered here because they are one concern, even though they fire at five different moments: at the edge, at parse, at build, at publish, and at send.

| Moment | Checked | Section |
|---|---|---|
| Request arrives | Authenticity of the caller | [1](#1-inbound-authenticity-must) |
| Payload parsed | Shape of the `InboundMessage` | [2](#2-inbound-shape-must) |
| Before any side effect | Not a duplicate | [3](#3-deduplication-must) |
| Messages built | Shape of each `GenericMessage` | [4](#4-genericmessage-validity-must) |
| Customer answers | Slot value acceptable | [5](#5-slot-validation-must) |
| Configuration published | Whole flow coherent | [7](#7-configuration-lint-must) |
| Before sending | Guardrails | [8](#8-guardrails-and-failure-behaviour) |

```
   request                                                     publish
      │                                                            │
      ▼                                                            ▼
 ╭─────────────╮ fail  401                                 ╭───────────────╮ fail
 │ authentic?  │──────▶ never parsed                       │ flow lints?   │──────▶ reject whole
 ╰──────┬──────╯                                           ╰───────┬───────╯        previous stays live
        │ ok                                                       │ pass
        ▼                                                          ▼
 ╭─────────────╮ fail  4xx                                    compile · store · swap
 │ shape ok?   │──────▶ no state change
 ╰──────┬──────╯
        │ ok
        ▼
 ╭─────────────╮ seen
 │ new?        │──────▶ acknowledge · zero side effects
 ╰──────┬──────╯
        │ new
        ▼
    the engine runs
        │
        ▼
 ╭─────────────╮ invalid
 │ slot ok?    │──────▶ reprompt · state unchanged
 ╰──────┬──────╯
        │ ok
        ▼
 ╭─────────────╮ invalid or blocked
 │ message ok? │──────▶ do not send · fallback or escalate · log the code
 ╰──────┬──────╯
        │ ok
        ▼
     persist, then send
```

**Every gate fails towards silence, never towards a guess.** That is the single principle behind this part: an engine that cannot be sure says nothing and offers a person.


---

## 1. Inbound authenticity (MUST)

**WhatsApp.** Verify the signature on the raw request body **before parsing anything**. Reject invalid, stale or replayed requests with `401`. Never parse an unverified payload. Full ordering in [Part 5, section 1](05-channel-whatsapp.md#1-order-of-operations-must).

**Web.** Validate the handshake session token before any event is accepted, and resolve `conversation_id` from the session, **never** from client-supplied data. Full ordering in [Part 6, section 2](06-channel-web.md#2-connection).

**MUST** the client never sends conversation history, and the server never reads history from the client. This closes both a data-loss defect and a prompt-injection hole.

**NOTE** these two checks look like plumbing and are the entire security boundary of stage 1. Everything downstream trusts that the sender is who they claim to be, so nothing downstream re-checks it.

---

## 2. Inbound shape (MUST)

Every parser output validates against the [InboundMessage JSON Schema](01-schemas.md#13-json-schema) before it reaches the engine. `additionalProperties` is `false`: an unknown field is a parser defect, not a payload to pass through.

A payload that fails validation produces a `4xx`, no state change, and a log entry. It never reaches the engine, and it never advances a flow.

---

## 3. Deduplication (MUST)

Keyed on `provider_message_id`, which is globally unique for both channels: the provider's id on WhatsApp, and `web_` + `client_msg_id` on web. **One rule covers both channels.**

**MUST** the check happens **before any side effect**. Deduplicating after creating a lead or an order defeats the purpose.

**MUST** deduplication lives in the **durable store**, as a unique constraint on `provider_message_id`. A Redis check is fast and forgettable, a unique constraint is durable and correct. This is the one place where being slower is being right.

A repeated delivery is acknowledged normally and processed zero further times. Retried webhooks and client resends are both ordinary, not exceptional.

---

## 4. GenericMessage validity (MUST)

Checked when messages are built, at step 9 of the [engine ordering](03-engine-state-machine.md#21-ordering-must), and again by the configuration lint for any message a flow can produce.

| `type` | Must set | Must not set |
|---|---|---|
| `TEXT` | `text` | `buttons`, `sections`, `cards` |
| `BUTTONS` | `text`, `buttons` (1 to 3) | `sections`, `cards` |
| `LIST` | `text`, `button_text`, `sections` (1 to 10 rows) | `buttons`, `cards` |
| `CARDS` | `cards` (1 or more) | `buttons`, `sections` |
| `TEMPLATE` | `template_name`, `template_language` | `buttons`, `sections`, `cards` |
| `IMAGE` · `DOCUMENT` | `media` | `buttons`, `sections`, `cards` |
| `LOCATION_REQUEST` | `text` | `buttons`, `sections`, `cards` |
| `CTA_URL` | `text`, one action with `url` | `sections`, `cards` |

Limits: `buttons` max 3, `title` max 20 characters, `sections` max 10 rows in total.

**NOTE** these limits are WhatsApp facts, enforced here rather than in the WhatsApp transport so that an author sees the error at lint time. The web transport applies none of them at render time, see [Part 6](06-channel-web.md#5-web-rendering-must).

---

## 5. Slot validation (MUST)

A slot declares a `type` and an optional `validation` object, defined in [Part 4, section 3](04-flow-config.md#3-slot-must).

| Slot `type` | Accepts | Typical `validation` |
|---|---|---|
| `text` | Free text | `min`, `max` |
| `number` | Numeric value | `min`, `max` |
| `phone` | E.164 after normalization | `pattern` |
| `email` | Mailbox address | `pattern` |
| `choice` | One of a fixed option set | `options` |
| `media` | One or more media references | |
| `location` | A coordinate pair | |

**MUST** a state carrying a slot cannot advance until that slot validates. On failure the engine sends the slot's `reprompt` in the conversation's locale and stays in the same state. On repeated failure it sends `error_message` and offers the human route.

**MUST** validation is deterministic and local. Anything requiring a live check against KiboAuto systems, such as whether a part exists or an order number is real, is a `verify` tool call, not slot validation. Slot validation answers "is this a well-formed answer", a verify tool answers "is this true".

---

## 6. Copy coverage (MUST)

Every customer-visible string is a copy object keyed by locale:

```json
{ "en": "Which part do you need?", "sw": "Ni sehemu gani unahitaji?" }
```

**MUST** configuration validation fails if any copy object is missing a locale listed in `flow.locales`. A missing translation is a lint error, never a runtime fallback to English.

**NOTE** the failure this prevents is silent and embarrassing: a Swahili conversation that switches to English for one message because nobody translated a reprompt.

---

## 7. Configuration lint (MUST)

Run on publish, before compile. Configuration is rejected whole if any check fails.

| Check | Rule |
|---|---|
| Schema | Validates against the flow JSON Schema |
| Locales | Every copy object covers every locale in `flow.locales` |
| Reachability | Every state is reachable from `initial_state` |
| Termination | Every path can reach a `TERMINAL` state |
| Transitions | Every transition target resolves, using the [resolution order](04-flow-config.md#5-transition-resolution-order-must) |
| Slots | Every `${slots.x}` reference names a defined slot |
| Tools | Every `CALL_TOOL` action names a registered tool |
| Limits | 3 actions per state or fewer, 3 buttons or fewer, 10 list rows or fewer |
| Exits | Every non-terminal state resolves the [universal exits](04-flow-config.md#7-universal-exits-must) |

**MUST** fail closed. A configuration that fails any check is rejected whole and the previous version stays active. Never partially apply.

**MUST** the lint and the runtime use the **same** transition resolution logic. If they disagree about what a transition value means, a flow validates and then goes nowhere in production, which is the hardest class of bug to see.

---

## 8. Guardrails and failure behaviour

### 8.1 Hook points

Phase one wires the hooks and keeps the checks minimal. The full chain is phase two, and it attaches here without re-architecting.

| Hook | Fires | Phase one behaviour |
|---|---|---|
| Pre-classification | On the inbound text | Length and rate sanity only |
| Pre-tool | Before a `transact` or `record` call | Confirm the flow reached the state legitimately |
| Pre-send | On each built `GenericMessage` | Shape validity, section 4, plus banned-content stub |

A blocked message is **not sent**. The turn routes to the fallback state or escalates, and the guardrail code is recorded in `OrchestrationResult.meta.guardrails.codes`.

**MUST** codes come from a fixed vocabulary, prefixed `GRD_`: `GRD_UNGROUNDED_PRICE`, `GRD_COMMITMENT`, `GRD_CREDENTIAL_SOLICIT`, `GRD_PII_LEAK`, `GRD_LANG_MISMATCH`, `GRD_INJECTION_SUSPECTED`, `GRD_MODE_VIOLATION`. Each is argued in the guide, section 5. A free-text reason is not a code and cannot be counted.

### 8.2 Failure behaviour (MUST)

| Failure | Behaviour |
|---|---|
| Parser rejects the payload | `4xx`, no state change, log |
| Lock not acquired within timeout | Queue for retry, never proceed unlocked |
| Tool call fails | Route to the action's `on_error` state, never invent a result |
| Classifier unavailable | Treat as `unclear`, do not block the turn |
| Guardrail blocks a message | Do not send, route to fallback or escalate, log the code |
| Persistence fails | Do not send. Release the lock and retry the whole turn |

**MUST** messages are sent **after** persistence, never before. Sending first and failing to persist produces a customer who has been answered by an engine with no memory of it.

**MUST** the engine never fabricates a result to keep a conversation moving. A tool that cannot answer means the engine says so and offers a person.

---

**Next:** [Part 3, Engine and state machine](03-engine-state-machine.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
