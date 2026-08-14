# Part 7: Conformance

**Part 7 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

So an implementation can be proven correct without matching anyone's internals. These are acceptance fixtures: given the input, the output must match exactly. Any implementation that passes them is correct, in any language.

**Each fixture has a name, not a number.** Use the name verbatim as the test name in whatever framework KiboAuto runs, so a failure in CI reads as `fanout.buttons-overflow` rather than as `C6`. Names are stable and normative, and new fixtures are added without renumbering anything.

| Fixture | Proves | Part |
|---|---|---|
| [`parse.whatsapp-text`](#parsewhatsapp-text) | Plain text maps cleanly | [5](05-channel-whatsapp.md) |
| [`parse.whatsapp-numbered-reply`](#parsewhatsapp-numbered-reply) | A numeric reply becomes a selection | [5](05-channel-whatsapp.md) |
| [`parse.web-send`](#parseweb-send) | The session owns identity, not the payload | [6](06-channel-web.md) |
| [`dedupe.repeat-delivery`](#deduperepeat-delivery) | One message, one side effect | [2](02-validation.md) |
| [`fanout.buttons-fit`](#fanoutbuttons-fit) | One message, two correct renderings | [5](05-channel-whatsapp.md), [6](06-channel-web.md) |
| [`fanout.buttons-overflow`](#fanoutbuttons-overflow) | WhatsApp limits do not reach web | [5](05-channel-whatsapp.md), [6](06-channel-web.md) |
| [`fanout.window-expired`](#fanoutwindow-expired) | The 24-hour rule is enforced in the transport | [5](05-channel-whatsapp.md) |
| [`engine.slot-blocks-advance`](#engineslot-blocks-advance) | A required field cannot be skipped | [3](03-engine-state-machine.md), [4](04-flow-config.md) |
| [`engine.paused-is-silent`](#enginepaused-is-silent) | The bot does not talk over a human | [3](03-engine-state-machine.md) |
| [`engine.concurrent-writes`](#engineconcurrent-writes) | The lock actually serializes | [3](03-engine-state-machine.md) |
| [`flow.universal-exits`](#flowuniversal-exits) | No customer is ever trapped | [4](04-flow-config.md) |
| [`config.version-pinning`](#configversion-pinning) | Publishing mid-conversation is safe | [3](03-engine-state-machine.md) |
| [`config.fail-closed`](#configfail-closed) | Bad configuration never goes live | [2](02-validation.md) |
| [`kb.below-threshold`](#kbbelow-threshold) | A weak match produces no answer | [8](08-knowledge-and-model.md) |
| [`kb.uncited-answer`](#kbuncited-answer) | Ungrounded text is never sent | [8](08-knowledge-and-model.md) |
| [`model.provider-down`](#modelprovider-down) | Transactions survive a model outage | [8](08-knowledge-and-model.md) |

**Write these four first.** `parse.whatsapp-numbered-reply`, `fanout.buttons-fit`, `fanout.buttons-overflow` and `engine.concurrent-writes` are the four places an implementation most commonly diverges while still appearing to work. Each one passes the eye test in a demo and fails in production.

---

## parse.whatsapp-text

**Given** a verified webhook whose message is `{ from: "255712345678", id: "wamid.X", timestamp: "1786...", type: "text", text: { body: "nataka breki" } }` and no option set stored,
**then** the `InboundMessage` must have `channel: "whatsapp"`, `user_id: "+255712345678"`, `provider_message_id: "wamid.X"`, `message_type: "TEXT"`, `text: "nataka breki"`, `selection: null`.

## parse.whatsapp-numbered-reply

**Given** the last presented option set was `[{id:"quote"},{id:"finance"},{id:"agent"}]` and the customer sends the text `"2"`,
**then** `message_type` must be `POSTBACK` and `selection.id` must be `"finance"`.

## parse.web-send

**Given** an authenticated `message:send` with `{ client_msg_id: "abc-123", text: "hello" }`,
**then** `channel` must be `web`, `provider_message_id` must be `"web_abc-123"`, and `conversation_id` must come from the session, **not** from the payload.

## dedupe.repeat-delivery

**Given** the same `provider_message_id` delivered three times,
**then** exactly one turn is processed, one set of messages sent, and any `transact` tool called exactly once.

## fanout.buttons-fit

**Given** `GenericMessage{ type: "BUTTONS", text: T, buttons: [b1,b2,b3] }`,
**then** WhatsApp must render interactive reply buttons, and web must emit one `message` event carrying all three, from the same input, with no flow change.

## fanout.buttons-overflow

**Given** the same message with **five** buttons,
**then** WhatsApp must render a `LIST`, and web must still render all five as quick replies. Web applies no WhatsApp limit.

## fanout.window-expired

**Given** a non-`TEMPLATE` message and a last inbound message more than 24 hours old,
**then** the WhatsApp transport must not send, and must return a failure with reason `WINDOW_EXPIRED`.

## engine.slot-blocks-advance

**Given** a state collecting a slot with `min: 2`, and the customer sends `"a"`,
**then** the state must not change, and the message sent must be the slot's `reprompt` in the conversation's locale.

## engine.paused-is-silent

**Given** `bot_mode` is `paused` or `handoff`,
**then** `messages` must be empty and the transport must send nothing, while the inbound message is still persisted.

## engine.concurrent-writes

**Given** two messages for the same conversation arriving concurrently,
**then** they must be processed sequentially, and the resulting slot values must reflect both, never one overwriting the other.

## flow.universal-exits

**Given** any non-terminal state and the customer sends `rudi`,
**then** the conversation must return to the previous state with already-filled slots intact, whether or not the flow author declared a `__back` transition.

## config.version-pinning

**Given** a conversation pinned to version 3 and version 4 published mid-conversation,
**then** that conversation must continue on version 3 to termination, and a new conversation must start on version 4.

## config.fail-closed

**Given** a configuration missing a `sw` translation for one copy object,
**then** publication must be rejected and the previously active version must remain active.

## kb.below-threshold

**Given** an `ANSWER_FROM_KB` action whose best snippet scores below `min_score`,
**then** no knowledge answer is produced, the conversation moves to `on_no_answer`, and the query is logged as a knowledge gap.

## kb.uncited-answer

**Given** a model reply with no `[sources: ...]` marker, or one citing a `document_id` that was not in the prompt,
**then** the reply must not be sent, the turn must record a grounding violation, and the customer must get the offer-a-human message.

## model.provider-down

**Given** the model interface is unavailable,
**then** a spare-parts order must still run to its terminal state on the deterministic path, and only knowledge answering is degraded.

---

## The proof

Not a fixture so much as the acceptance test for the whole phase. The same `GenericMessage`:

```json
{ "type": "BUTTONS", "text": "Ungependa kufanya nini?",
  "buttons": [ { "id": "quote", "title": "Pata bei" },
               { "id": "finance", "title": "Mikopo" },
               { "id": "agent", "title": "Ongea na mtu" } ] }
```

renders on **WhatsApp** as an interactive three-button message, and on **web** as a single `message` event the client draws as three quick replies, with no flow, state or configuration change between them.

If a flow has to know which channel it is on, the architecture has not been achieved.

---

**Next:** [Part 8, Knowledge and the model](08-knowledge-and-model.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
