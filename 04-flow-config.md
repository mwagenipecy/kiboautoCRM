# Part 4: Flow configuration, the format and how to design one

**Part 4 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

A journey is JSON. The engine is an interpreter of this format and does not change when a journey does.

This part has two halves. **Sections 1 to 9 are the reference**, the shapes the engine reads. **Sections 10 to 18 are the design guide**, which exists because knowing the format does not tell you how to arrive at a good flow. Read the guide first if you are about to author a journey, and treat the reference as the thing you return to.

---

# A. Reference

## 0. Structure

| Object | Purpose |
|---|---|
| `ConversationFlow` | One journey: id, version, locales, states, entry conditions |
| `State` | One step: type, actions, transitions, optional slot |
| `Action` | Something the state does: send a message, collect input, call a tool, hand off |
| `Transition` | Where to go next, by key or by condition |
| `Slot` | A named value being collected, with validation |

## 1. ConversationFlow (MUST)

| Field | Type | Required | Notes |
|---|---|:---:|---|
| `id` | string | ✅ | Stable identifier, for example `spare_parts_order` |
| `version` | integer | ✅ | Increments on every publish. Used for [pinning](03-engine-state-machine.md#73-rules-must) |
| `locales` | array | ✅ | `["en","sw"]`. Every copy object must cover all listed locales |
| `entry` | object | ✅ | `{ intents: [...], keywords: { en: [...], sw: [...] } }` |
| `initial_state` | string | ✅ | Must name a state with `type: "ENTRY"` |
| `states` | array | ✅ | One or more `State` objects |
| `slots` | array | | `Slot` definitions referenced by states |
| `fallback_state` | string | | Where unresolvable input goes |

## 2. State (MUST)

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique within the flow |
| `type` | enum | `ENTRY` \| `STANDARD` \| `FALLBACK` \| `TERMINAL` |
| `actions` | array | 3 at most. Executed in order |
| `transitions` | object | Map of transition key to target |
| `conditional_transitions` | array | `{ condition, target_state, intent?, priority }` |
| `slot` | string \| null | The slot this state collects, if any |

## 3. Slot (MUST)

| Field | Type | Notes |
|---|---|---|
| `name` | string | Referenced as `${slots.name}` |
| `type` | enum | `text` \| `number` \| `phone` \| `email` \| `choice` \| `media` \| `location` |
| `required` | boolean | |
| `validation` | object | `{ pattern? , min? , max? , options? }` |
| `prompt` | copy object | Asked when entering the state |
| `reprompt` | copy object | Asked when validation fails |
| `error_message` | copy object | Shown on repeated failure |

What each `type` accepts, and where validation stops and a tool starts, is in [Part 2, section 5](02-validation.md#5-slot-validation-must).

## 4. Actions

Phase one supports the message-producing set. Everything else is phase two.

| `type` | Purpose | Key fields |
|---|---|---|
| `SEND_TEXT` | Say something | `text` (copy object) |
| `SEND_BUTTONS` | Offer up to 3 choices | `text`, `buttons[] { id, title }` |
| `SEND_LIST` | Offer up to 10 choices | `text`, `button_text`, `sections[]` |
| `COLLECT_INPUT` | Wait for and validate a slot value | `slot` |
| `CALL_TOOL` | Invoke a registered tool | `tool`, `arguments`, `on_error` |
| `ANSWER_FROM_KB` | Answer a "how does it work" question from the knowledge base | `query`, `category`, `top_k`, `min_score`, `on_no_answer` |
| `HANDOFF` | Move the conversation to a human | `queue`, `priority` |

**MUST** `CALL_TOOL` always names an `on_error` target state. A tool call with no error path is a flow that can stop dead.

**MUST** actions execute in declared order, and a state has 3 at most. A state needing four actions is two states.

**MUST** `ANSWER_FROM_KB` always names `on_no_answer`. Knowledge retrieval that finds nothing must route somewhere explicit, never improvise. The retrieval contract, the scoring threshold and the grounding checks are in [Part 8, section 5](08-knowledge-and-model.md#5-retrieval).

## 5. Transition resolution order (MUST)

Given a transition value, resolve in this order and stop at the first hit:

1. A [universal exit](#7-universal-exits-must) key
2. A key in the state's own `transitions` map
3. The name of a state in this flow
4. The name of a flow-level intent, whose target is used

Reserved catch-all keys, tried in order: `default`, `*`, `any`.

**MUST** `conditional_transitions` are evaluated **before** any of the four steps above, highest `priority` first, and the first condition that matches wins. Keys are only consulted when no condition matched. The full decision, including slot validation, is drawn in [Part 3, section 2.4](03-engine-state-machine.md#24-the-state-machine-step-in-detail).

**MUST** an unresolvable transition is an **error**, not a silent no-op. It surfaces at configuration-lint time as an unreachable target, and at runtime as an engine error routed to the fallback state.

**NOTE** the failure mode this prevents: a flow that validates but silently goes nowhere at runtime, because validation and the runtime disagreed about what a transition value meant. Both must use the same resolution logic.

## 6. Copy objects (MUST)

Every customer-visible string is an object keyed by locale:

```json
{ "en": "Which part do you need?", "sw": "Ni sehemu gani unahitaji?" }
```

Coverage is enforced at lint time, see [Part 2, section 6](02-validation.md#6-copy-coverage-must).

## 7. Universal exits (MUST)

Every non-terminal state accepts four escapes, whatever the flow author wrote. The engine resolves them **before** any flow-defined transition key.

| Reserved key | Customer input that triggers it | Engine behaviour |
|---|---|---|
| `__menu` | `menu`, `menyu`, `mwanzo` | Leave the flow, show the main menu |
| `__back` | `back`, `rudi` | Return to the previous state, keeping slots already filled |
| `__restart` | `0`, `restart`, `anza upya` | Clear the flow's slots, return to `initial_state` |
| `__agent` | `agent`, `human`, `mtu`, "talk to a person", "ongea na mtu" | Escalate, `bot_mode` becomes `paused` |

**MUST** a flow may override the *target* of a reserved key, by declaring it in `transitions`. A flow may **not** disable one. There is no configuration that traps a customer.

**MUST** matching is case insensitive, whitespace trimmed, and applied to the whole message only. A message that merely contains the word "menu" inside a sentence is not an exit, because "I saw it on the menu page" is not a request to leave.

**NOTE** this is already promised in the design guide and already true of the current WhatsApp implementation. It is specified here as a reserved key rather than left to authors because it is exactly the kind of rule that is remembered in the first three flows and forgotten in the fourth.

## 8. Rendering hints (SHOULD)

A state may carry per-channel hints. They are advisory, and the transport limits in [Part 5](05-channel-whatsapp.md) and [Part 6](06-channel-web.md) always win.

```json
"rendering": {
  "whatsapp": { "prefer": "LIST", "button_text": { "en": "View parts", "sw": "Ona vipuri" } },
  "web":      { "prefer": "CARDS" }
}
```

**MUST** a hint may never change *what* is said, only *how it is presented*. Copy lives in the action, not the hint.

## 9. Annotated example

```json
{
  "id": "spare_parts_order",
  "version": 3,
  "locales": ["en", "sw"],
  "initial_state": "start",
  "fallback_state": "unclear",

  "entry": {
    "intents": ["spare_parts"],
    "keywords": { "en": ["spare part", "brake", "battery"],
                  "sw": ["kipuri", "breki", "betri"] }
  },

  "slots": [
    {
      "name": "part_name",
      "type": "text",
      "required": true,
      "validation": { "min": 2, "max": 120 },
      "prompt":   { "en": "Which part do you need?",
                    "sw": "Ni sehemu gani unahitaji?" },
      "reprompt": { "en": "Please give the part name, for example: front brake pads.",
                    "sw": "Tafadhali andika jina la sehemu, mfano: brake pads za mbele." }
    }
  ],

  "states": [
    {
      "name": "start",
      "type": "ENTRY",
      "actions": [
        { "type": "SEND_BUTTONS",
          "text":    { "en": "I can help with spare parts. Shall we start?",
                       "sw": "Naweza kukusaidia na vipuri. Tuanze?" },
          "buttons": [ { "id": "yes", "title": { "en": "Yes", "sw": "Ndiyo" } },
                       { "id": "agent", "title": { "en": "Talk to a person",
                                                   "sw": "Ongea na mtu" } } ] }
      ],
      "transitions": { "yes": "ask_part_name", "agent": "handoff", "default": "unclear" }
    },
    {
      "name": "ask_part_name",
      "type": "STANDARD",
      "slot": "part_name",
      "actions": [ { "type": "COLLECT_INPUT", "slot": "part_name" } ],
      "transitions": { "collected": "check_availability" }
    },
    {
      "name": "check_availability",
      "type": "STANDARD",
      "actions": [ { "type": "CALL_TOOL",
                     "tool": "parts.availability",
                     "arguments": { "part_name": "${slots.part_name}" },
                     "on_error": "tool_failed" } ],
      "conditional_transitions": [
        { "condition": "${tool.result.available} == true",
          "target_state": "confirm_order", "priority": 10 }
      ],
      "transitions": { "default": "no_stock" }
    },
    { "name": "handoff", "type": "TERMINAL",
      "actions": [ { "type": "HANDOFF", "queue": "parts", "priority": "normal" } ],
      "transitions": {} }
  ]
}
```

---

# B. Design guide

## 10. Before you write any JSON

Answer these five questions in a paragraph each. A journey that cannot answer them is not ready to be configured, and writing the JSON first will hide that rather than fix it.

| Question | What a good answer looks like |
|---|---|
| **What is the customer trying to finish?** | One sentence, in the customer's words. "Order a part and know when I can collect it." Not "handle parts enquiries." |
| **What is the minimum set of facts needed to finish it?** | A list. If a fact does not change what happens next or what a human receives, it is not needed. |
| **Which facts come from the customer, and which from a tool?** | Split the list in two. Customer facts become slots, system facts become tool calls. Asking a customer something KiboAuto already knows is the most common flow defect. |
| **What does done look like?** | A named terminal state and a record written somewhere a human can act on. "The bot answered" is not done. |
| **What are the exits?** | Where does an unclear answer go, where does a failed tool go, where does an angry customer go. |

**SHOULD** write these answers into the flow file as a comment block or a companion note. Six months later the answer to "why does this flow ask for the plate number" is worth more than the JSON.

## 11. The shape of a good state

Every transactional journey is a variation of one skeleton:

```
   ┌─────────┐   ┌──────────┐   ┌──────────────┐   ┌─────────┐   ┌────────┐
   │ COLLECT │──▶│ VALIDATE │──▶│ VERIFY(tool) │──▶│ CONFIRM │──▶│ RECORD │
   └────┬────┘   └────┬─────┘   └──────┬───────┘   └────┬────┘   └───┬────┘
        │             │ fails          │ unavailable    │ no         │
        │             ▼                ▼                ▼            ▼
        │          reprompt        on_error         back to      TERMINAL
        │                          state            collect
        └───────────── universal exits available at every step ──────┘
```

**MUST** one state asks one thing. A state that collects two facts cannot reprompt precisely, cannot be corrected precisely, and cannot be reported on.

**SHOULD** confirm before you record, whenever the record costs someone something: an order, a finance application, a complaint that opens a ticket. Do not confirm what is free to redo.

**NOTE** the `VERIFY` step is what keeps the bot honest. The customer says they want brake pads for a Harrier, the flow validates that the answer is well formed, and only a tool can say whether that part is actually in stock today. Skipping verify produces a bot that confidently promises stock it does not have.

## 12. Journey inventory (SHOULD)

The four transactional journeys named in the design guide, as a starting shape. KiboAuto confirms the exact field lists against their own CRM and catalogue before authoring.

| Journey | Customer slots | Tools | Terminal states |
|---|---|---|---|
| **Spare parts order** | part name, vehicle make and model, year, quantity, delivery or collection, location | `parts.availability` (verify), `parts.price` (verify), `order.create` (transact) | order placed · no stock, lead recorded · handed to a person |
| **Finance application** | vehicle of interest, deposit available, employment type, monthly income band, contact preference | `finance.eligibility` (discover), `lead.create` (record) | application referred to finance team · not eligible, alternatives offered · handed to a person |
| **Complaint intake** | what happened, when, order or vehicle reference, what outcome is wanted | `order.lookup` (verify), `ticket.create` (record) | ticket raised with reference · handed to a person immediately if flagged as risk |
| **Partner registration** | business name, region, category, contact person, phone | `partner.create` (record) | registered, next step explained · handed to a person |

**Build spare parts first.** It is the most mature flow, it is already state-machine shaped on WhatsApp, and every other journey is a simplification of it.

**NOTE** complaint intake is the journey where the `risk` classification in [Part 3, section 2.6](03-engine-state-machine.md#26-turn-classification-should) matters most. A complaint that mentions injury, fraud or legal action leaves the flow immediately. Design it to escalate early rather than to complete.

## 13. Slot design rules

**Choose the `type` before writing a `pattern`.** A `phone` slot normalizes to E.164 and rejects what cannot be normalized. Writing a regular expression on a `text` slot to do the same job means every flow re-implements it slightly differently.

**Prefer `choice` over free text on WhatsApp.** A `choice` slot renders as buttons or a list, arrives as a `selection.id`, needs no interpretation, and cannot be misspelled. Free text is right when the answer space is genuinely open, such as a part name, and wrong when it is not, such as "delivery or collection".

**Validation is local, verification is remote.** See [Part 2, section 5](02-validation.md#5-slot-validation-must). If the check needs an API call, it is a `verify` tool in the next state, not a `validation` rule.

**Every optional slot costs a turn.** On WhatsApp a turn is a round trip through a customer's day. Mark a slot optional only when the journey genuinely completes without it, and put optional slots last so that abandonment still leaves a usable lead.

**SHOULD** a slot that a tool can fill should be filled by the tool. If the customer arrived from a listing page, `context.listing_id` already names the vehicle, so do not ask.

## 14. Copy rules for authors

- **Both locales or it does not publish.** English and Swahili are peers here, not original and translation.
- **One question per message.** Two questions in one message reliably get one answer, and you cannot tell which.
- **The reprompt says what good input looks like**, it does not repeat the question. "Please give the part name, for example: front brake pads" is a reprompt. "Which part do you need?" asked twice is a dead end.
- **Keep option titles short.** WhatsApp button titles cap at 20 characters, and the numbered-text fallback in [Part 5](05-channel-whatsapp.md#3-the-numbered-text-fallback) puts every title on its own line. A title that wraps reads badly on a phone.
- **Never put a phone number, price or stock figure in copy.** Those come from tools. Copy that states a fact goes stale the day it is written.

**SHOULD** write the Swahili first for journeys where most customers write Swahili. Translated-from-English Swahili reads as translated, and code-switching customers notice.

## 15. Sizing limits, and why

| Limit | Cause |
|---|---|
| 3 buttons per message | WhatsApp interactive reply buttons cap at 3 |
| 20 characters per button title | WhatsApp truncates beyond it |
| 10 rows per list | WhatsApp interactive lists cap at 10 rows in total across sections |
| 3 actions per state | Engine rule, not a channel rule: it keeps a state explainable and its turn log readable |

Design inside these from the start. A journey that needs to show fifteen options needs a narrowing question first, not a bigger list, and discovering that at lint time means redesigning states rather than editing copy.

## 16. Naming conventions (SHOULD)

| Thing | Convention | Good | Avoid |
|---|---|---|---|
| Flow id | `snake_case`, names the journey | `spare_parts_order` | `flow1`, `SparePartsV2` |
| State name | `snake_case`, names what the customer is doing | `ask_part_name`, `confirm_order` | `state_3`, `step_b` |
| Slot name | `snake_case` noun | `part_name`, `delivery_method` | `input1`, `tmp` |
| Selection id | Stable identifier, never display text | `delivery`, `make.toyota` | `Delivery to my home` |
| Tool name | `domain.action` | `parts.availability` | `getStock` |

**MUST** selection ids are stable identifiers and never display text. The parser matches replies against them, the turn log records them, and analytics groups by them. Changing an id changes history, and translating one breaks matching in the other locale.

## 17. Anti-patterns

| Pattern | Why it fails |
|---|---|
| Letting the model decide the sequence | The sequence is the product. Models skip steps under pressure, and the skipped step is usually the confirmation |
| One state collecting two facts | Cannot reprompt or correct precisely |
| A confirmation state nothing reaches | Lint catches unreachable states, but only after someone believed the flow confirmed orders |
| Copy inside `rendering` hints | Copy stops being reviewable in one place, and one locale silently drifts |
| The same target spelled two ways | `check_availability` and `check_avail` both lint clean if both states exist, and the flow quietly splits in two |
| No path to a human | Every journey needs one. A customer with no exit leaves the brand, not just the conversation |
| Asking for something already known | `context.listing_id`, the phone number, a prior order: all already available |
| A tool call with no `on_error` | The flow stops dead the first time an upstream API is slow |

## 18. Authoring checklist

Before publishing, walk this list. Items marked with a check are enforced by the [lint](02-validation.md#7-configuration-lint-must), the rest are judgement.

- [x] Schema valid, all copy objects cover all locales
- [x] Every state reachable, every path can terminate
- [x] Every transition target resolves, every `${slots.x}` names a real slot
- [x] Every `CALL_TOOL` names a registered tool and an `on_error` target
- [x] Within the sizing limits
- [x] Universal exits resolve from every non-terminal state
- [ ] The five questions in [section 10](#10-before-you-write-any-json) have written answers
- [ ] One question per state, one fact per slot
- [ ] Every fact the flow asks for is a fact no tool can supply
- [ ] Confirmation precedes anything that costs the customer or KiboAuto something
- [ ] The Swahili reads as Swahili, not as a translation
- [ ] Dry-run against the [conformance fixtures](07-conformance.md) that touch flows: `engine.slot-blocks-advance`, `engine.paused-is-silent`, `flow.universal-exits`, `config.version-pinning`, `config.fail-closed`

---

**Next:** [Part 5, WhatsApp channel](05-channel-whatsapp.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
