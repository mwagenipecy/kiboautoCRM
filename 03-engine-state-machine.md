# Part 3: Engine and state machine

**Part 3 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

Stage 3. What happens between a validated `InboundMessage` and an `OrchestrationResult`. This is the largest part of the specification because it is the part with logic in it: everything else is a mapping.

Shapes are in [Part 1](01-schemas.md), checks are in [Part 2](02-validation.md), the journey format the engine interprets is in [Part 4](04-flow-config.md), and the model and knowledge interfaces the engine calls into are in [Part 8](08-knowledge-and-model.md).

---

## 1. Component interactions

### 1.1 Structure

Who calls whom. An arrow means "calls and waits for a result". There are no arrows back up the page, and that is the point.

```
                      ┌──────────────────────────────┐
   InboundMessage ───▶│        ORCHESTRATOR          │───▶ OrchestrationResult
                      │  owns the turn, owns order   │
                      └───┬─────┬──────┬──────┬──────┘
                          │     │      │      │
          ┌───────────────┘     │      │      └────────────────┐
          ▼                     ▼      ▼                       ▼
  ┌───────────────┐   ┌──────────────────┐   ┌──────────────┐  ┌──────────────┐
  │ STATE MANAGER │   │  STATE MACHINE   │   │  EXECUTOR    │  │ CLASSIFIER   │
  │ load · save   │   │  pure decision   │   │ runs actions │  │ intent · lang│
  │ context       │   │  no writes       │   │              │  │ turn class   │
  └───────┬───────┘   └────────┬─────────┘   └──────┬───────┘  └──────┬───────┘
          │                    │                    │                 │
          ▼                    ▼                    ▼                 ▼
  ┌───────────────┐   ┌──────────────────┐   ┌──────────────┐  ┌──────────────┐
  │ Redis + store │   │ TRANSITION       │   │ TOOL         │  │ MODEL        │
  │               │   │ HANDLER          │   │ REGISTRY     │  │ INTERFACE    │
  └───────────────┘   └────────┬─────────┘   └──────┬───────┘  └──────┬───────┘
                               │                    │                 │
                               ▼                    ▼                 ▼
                        ┌──────────────┐     ┌──────────────┐  ┌──────────────┐
                        │  EVALUATOR   │     │ KiboAuto     │  │ OpenAI       │
                        │  expressions │     │ APIs · kb    │  │ (Part 8)     │
                        └──────────────┘     └──────────────┘  └──────────────┘
```

### 1.2 Call rules (MUST)

| Rule | Reason |
|---|---|
| Only the orchestrator writes state | One writer per turn makes the turn atomic and the turn log truthful |
| The state machine is pure | Same context plus same input gives the same decision, every time. This is what makes flows testable without a database |
| Executors never read Redis directly | They receive what they need and return what they produced. An executor that reads state can produce a different result on replay |
| The classifier never resolves a transition | It returns a proposal, the state machine disposes. See [section 2.4](#24-the-state-machine-step-in-detail) |
| The evaluator has no side effects | It resolves expressions. An evaluator that can call a tool turns a condition into an action with no audit trail |
| Nothing here knows a channel exists | The dependency rule from the index |

### 1.3 ExecutionResult

The shape an executor returns is defined in [Part 1, section 5](01-schemas.md#5-executionresult). The orchestrator merges results in declaration order: messages append, variables merge last-write-wins, and the first `transition` returned by any action in the state wins.

---

## 2. The turn

### 2.1 Ordering (MUST)

```
 1  parse                    raw payload to InboundMessage
 2  deduplicate              on provider_message_id, against the durable store
 3  acquire lock             per conversation, at conv:lock:{conversation_id}
 4  load context             conv:ctx:{conversation_id}, state plus pinned flow version
 5  classify                 intent, language, turn class. May be a stub in phase one
 6  state machine step       resolve transition, choose next state
 7  execute actions          in declared order, 3 per state at most
 8  dispatch tools           where an action calls one
 9  build messages           GenericMessage[]
10  guardrail checks         on the built messages
11  persist                  context, messages and turn log, atomically
12  release lock
13  hand to transport        stage 5
```

**MUST** steps 3 and 12 bracket everything. The lock is acquired **before** the context is loaded and released **after** persistence completes. Any other arrangement allows two concurrent messages to interleave writes and corrupt the conversation.

**MUST** step 2 happens **before** any side effect. Deduplicating after creating a lead or an order defeats the purpose.

**MUST** step 11 is atomic. Context, messages and the turn log commit together or not at all. A half-written turn leaves a conversation in a state its history does not explain.

### 2.2 The same turn, as a flowchart

```
   stage 2  InboundMessage
                 │
                 ▼
        ╭─────────────────╮   seen before
        │  duplicate?     │──────────────▶  acknowledge · stop · no side effects
        ╰────────┬────────╯
                 │ new
                 ▼
        ╭─────────────────╮   held elsewhere
        │  acquire lock   │──────────────▶  queue for retry · never proceed unlocked
        ╰────────┬────────╯
                 │ acquired
                 ▼
        ┌─────────────────┐
        │  load context   │   see 5.4 for a cache miss
        └────────┬────────┘
                 ▼
        ╭─────────────────╮   paused · handoff
        │   bot_mode?     │──────────────▶  persist inbound · emit nothing ──────┐
        ╰────────┬────────╯                                                      │
                 │ active                                                        │
                 ▼                                                               │
        ┌─────────────────┐                                                      │
        │    classify     │   intent · language · turn class                     │
        └────────┬────────┘                                                      │
                 ▼                                                               │
        ┌─────────────────┐                                                      │
        │  state machine  │   see 2.4                                            │
        └────────┬────────┘                                                      │
                 ▼                                                               │
        ┌─────────────────┐                                                      │
        │ execute actions │   tools dispatched here · see 8                      │
        └────────┬────────┘                                                      │
                 ▼                                                               │
        ┌─────────────────┐                                                      │
        │ build messages  │   GenericMessage[]                                   │
        └────────┬────────┘                                                      │
                 ▼                                                               │
        ╭─────────────────╮   blocked                                            │
        │   guardrails?   │──────────────▶  fallback or escalate ──────┐         │
        ╰────────┬────────╯                                            │         │
                 │ pass                                                │         │
                 ▼                                                     ▼         ▼
        ┌────────────────────────────────────────────────────────────────────────┐
        │  PERSIST  ·  context + messages + turn log  ·  atomic, or not at all    │
        └────────┬───────────────────────────────────────────────────────────────┘
                 │ fails ──▶ release lock · send nothing · retry the whole turn
                 ▼
        ┌─────────────────┐
        │  release lock   │
        └────────┬────────┘
                 ▼
        stage 5  transport
```

**Read the diamonds as the whole of the engine's control flow.** Everything else is a straight line, and every branch in it is one of five questions: is this new, is it mine to process, is anyone else speaking, does the flow accept it, and is it safe to send.

### 2.3 Invariants (MUST)

| Invariant | Consequence if violated |
|---|---|
| A state with a `slot` cannot advance until that slot validates | Flows skip required fields |
| The classifier may **propose** a transition, only the state machine **performs** one | Model output becomes control flow |
| While `bot_mode` is `paused` or `handoff`, the engine produces **no** messages | Bot and human answer the same customer |
| A conversation keeps its pinned `flow_version` until it terminates | Mid-journey publishes strand customers |
| Every turn writes exactly one turn-log row | Behaviour becomes unexplainable after the fact |
| An unresolvable transition goes to the fallback state and logs an error | Conversations silently go nowhere |
| No message is sent before persistence commits | A customer is answered by an engine with no memory of it |

### 2.4 The state machine step, in detail

Step 6 of the ordering, expanded. This is the only place where a flow decides anything, and every branch is deterministic.

```
   InboundMessage  +  ConversationContext  +  the pinned flow version
                            │
                            ▼
              ╭──────────────────────────╮  yes
              │  universal exit match?   │─────▶  __menu · __back · __restart · __agent
              ╰────────────┬─────────────╯        resolve and stop · flows cannot disable these
                           │ no
                           ▼
              ╭──────────────────────────╮  no
              │  does this state have    │─────────────────────────┐
              │  a slot?                 │                         │
              ╰────────────┬─────────────╯                         │
                           │ yes                                   │
                           ▼                                       │
              ╭──────────────────────────╮  invalid                │
              │  value validates?        │─────▶  the slot loop    │
              ╰────────────┬─────────────╯        see 2.5          │
                           │ valid                                 │
                           ▼                                       │
                 store the slot value                              │
                           │                                       │
                           ├───────────────────────────────────────┘
                           ▼
              ╭──────────────────────────╮  first match wins, highest priority first
              │  conditional_transitions │─────▶  target_state
              ╰────────────┬─────────────╯
                           │ none match
                           ▼
              ╭──────────────────────────╮  hit
              │  transitions[key]        │─────▶  target
              ╰────────────┬─────────────╯
                           │ miss
                           ▼
              ╭──────────────────────────╮  hit
              │  a state name in this    │─────▶  target
              │  flow? a flow intent?    │
              ╰────────────┬─────────────╯
                           │ miss
                           ▼
              ╭──────────────────────────╮  hit
              │  default · * · any       │─────▶  target
              ╰────────────┬─────────────╯
                           │ miss
                           ▼
                 fallback_state  +  logged error
                 (never a silent no-op)
```

**MUST** the classifier may propose a transition value. This diagram, and nothing else, turns a value into a state. The full resolution order is normative in [Part 4, section 5](04-flow-config.md#5-transition-resolution-order-must).

**MUST** the state machine returns a *decision*, never a write: `{ next_state, transition_used, actions_to_run, slot_written }`. The orchestrator applies it. This is what makes a flow replayable in a test with no storage attached.

### 2.5 The slot filling loop (MUST)

The single most common conversation pattern, specified exactly because implementations diverge here and the divergence is invisible until a customer is stuck.

```
        enter state with slot S
                 │
                 ▼
        ┌──────────────────┐
        │ send S.prompt    │   attempts = 0
        └────────┬─────────┘
                 ▼
          ( wait for input )
                 │
                 ▼
        ╭──────────────────╮  yes
        │ universal exit?  │────────▶ leave the loop entirely
        ╰────────┬─────────╯
                 │ no
                 ▼
        ╭──────────────────╮  yes
        │ turn class is    │────────▶ park the flow, slots intact
        │ switches / risk? │          answer or escalate
        ╰────────┬─────────╯
                 │ no
                 ▼
        ╭──────────────────╮  valid
        │ validates?       │────────▶ store · attempts = 0 · resolve transition
        ╰────────┬─────────╯
                 │ invalid
                 ▼
           attempts = attempts + 1
                 │
                 ▼
        ╭──────────────────╮  attempts = 1
        │ how many?        │────────▶ send S.reprompt ──────┐
        ╰────────┬─────────╯                                │
                 │ attempts = 2                             │
                 ▼                                          │
           send S.error_message                             │
           + the human route                                │
                 │                                          │
                 ▼                                          │
        ╭──────────────────╮  attempts = 3                  │
        │ still failing?   │────────▶ escalate · bot_mode    │
        ╰──────────────────╯          becomes paused         │
                 ▲                                          │
                 └──────────────────────────────────────────┘
```

**MUST** the attempt counter lives on the context, resets on success, and resets when the flow leaves the state. Three failed attempts on one slot escalates to a human. A loop with no ceiling is the single worst experience a bot can produce, and the customer will not tell you it is happening.

**MUST** a `corrects` turn class overwrites the slot and re-confirms without restarting the flow, and does not increment the counter. Correcting yourself is not failing.

### 2.6 Turn classification (SHOULD)

Before the state machine runs, classify what the message *is* relative to the current state:

| Class | Meaning | Engine behaviour |
|---|---|---|
| `continues` | The expected answer | Fill the slot, advance |
| `corrects` | Revises a previously given value | Update that slot, re-confirm, **do not restart** |
| `switches` | Different topic mid-flow | Park the flow with its slots intact, answer, offer to resume |
| `adds` | Two intents in one message | Handle the first, acknowledge the second |
| `unclear` | Ambiguous or empty | Re-prompt with the slot's `reprompt` and options |
| `risk` | Complaint, fraud, urgency | Leave the flow, escalate |

**NOTE** the state machine decides what a valid next step is, the classifier decides whether the message is a next step at all. Without this split, "actually, do you have the 2015?" is recorded as the answer to whatever was last asked. In phase one the classifier may be a deterministic stub behind the real interface. The interface is what matters now, and it is specified in [Part 8, section 4](08-knowledge-and-model.md#4-classification).

### 2.7 Budgets and timeouts (MUST)

Every wait has a ceiling, and every ceiling has a defined behaviour when it is hit. Values are the starting set, and are tuned with evidence rather than by feel.

| Wait | Budget | On expiry |
|---|---|---|
| Webhook acknowledgement | 200 ms | Ghala retries, which the deduplication rule absorbs |
| Lock acquisition | 5 s | Queue the message for retry, never proceed unlocked |
| Lock hold (TTL) | 30 s | Lock expires. The holder token stops a late worker releasing someone else's lock |
| Single tool call | 6 s | `ToolResult{ success: false }`, route to `on_error` |
| Classification call | 3 s | Treat the turn as `unclear`, do not block |
| Knowledge retrieval | 3 s | Answer without a snippet, which means saying so and offering a human |
| Whole turn, engine side | 20 s | `ENGINE_TIMEOUT` to the client, offer retry and a human |

**MUST** the sum of the per-step budgets on the critical path is smaller than the whole-turn budget. If it is not, the turn budget is decoration.

**NOTE** the 30 second lock TTL and the 20 second turn budget are deliberately different. The TTL is a crash guard, not a turn timer: it must outlast the slowest legitimate turn or a healthy worker loses its lock mid-turn.

### 2.8 When something fails

The failure behaviour table is in [Part 2, section 8.2](02-validation.md#82-failure-behaviour-must). Two rules from it govern the ordering above: persistence precedes sending, and a failed tool routes to its `on_error` state rather than producing an invented result.

---

## 3. Conversation lifecycle

Two things are often confused and must not be: **`bot_mode` is who is speaking, the flow state is where the conversation is.** They move independently.

```
                      ┌──────────┐
     first message ──▶│   NEW    │   no flow yet, no state
                      └────┬─────┘
                           │ intent matched, flow entered, version pinned
                           ▼
                      ┌──────────┐         slot to fill
                      │  ACTIVE  │◀──────────────────────────┐
                      └────┬─────┘                           │
                           │                                 │
            ┌──────────────┼──────────────┬──────────────────┤
            │              │              │                  │
            ▼              ▼              ▼                  │
      ┌───────────┐  ┌───────────┐  ┌───────────┐            │
      │ COLLECTING│  │  WAITING  │  │ CONFIRMING│            │
      │  a slot   │  │  a tool   │  │  an action│            │
      └─────┬─────┘  └─────┬─────┘  └─────┬─────┘            │
            │              │              │                  │
            └──────────────┴──────────────┴──────────────────┘
                           │
                           │ terminal state reached
                           ▼
                      ┌──────────┐
                      │ TERMINAL │   flow ends · pin released · slots archived
                      └────┬─────┘
                           │ new intent
                           ▼
                      back to NEW, same conversation, new flow


   bot_mode, independently, at any point above:

      active ──escalation triggered──▶ paused ──agent claims──▶ handoff
         ▲                                                        │
         └──────────── explicit agent action only ────────────────┘
```

**MUST** the version pin is set on entering a flow and released on reaching a `TERMINAL` state. A conversation that runs three flows over a week may be pinned to three different versions in sequence, and that is correct.

**MUST** while `bot_mode` is not `active`, every stage of the turn still runs except message production. The inbound message is parsed, deduplicated, persisted and mirrored to the CRM. Only the sending stops.

**NOTE** the `switches` turn class parks a flow rather than terminating it. A parked flow keeps its slots and its pin, so "actually, what are your opening hours?" mid-order does not cost the customer the six answers they already gave.

---

## 4. Expressions and conditions

`conditional_transitions` carry a `condition`, and actions carry `${...}` references. Both are read by the evaluator, and both need a defined grammar or every implementation invents a different one.

### 4.1 References (MUST)

| Root | Resolves to | Example |
|---|---|---|
| `${slots.*}` | A collected slot value | `${slots.part_name}` |
| `${tool.result.*}` | The last tool result in this state | `${tool.result.available}` |
| `${context.*}` | Web entry context | `${context.listing_id}` |
| `${customer.*}` | Identity and locale | `${customer.locale}` |
| `${message.*}` | The current inbound message | `${message.text}` |

**MUST** an unresolvable reference is `null`, never an empty string and never the literal text. A flow that renders `${slots.part_name}` to a customer has silently failed and looks broken.

### 4.2 Conditions (MUST)

Deliberately small. Comparison, membership, presence, and boolean combination. Nothing else.

```
condition   := term (("&&" | "||") term)*
term        := reference operator literal
             | "exists(" reference ")"
             | "empty(" reference ")"
operator    := "==" | "!=" | ">" | ">=" | "<" | "<=" | "in"
literal     := string | number | boolean | null | [ literal, ... ]
```

Examples that must evaluate:

```
${tool.result.available} == true
${slots.quantity} > 0 && ${slots.quantity} <= 10
${slots.delivery_method} in ["delivery","collection"]
exists(${context.listing_id})
```

**MUST** no arithmetic, no function calls, no regular expressions, no assignment. A condition language that grows becomes a programming language living in a JSON file, unreviewable and untestable. If a decision needs more than this grammar, it needs a tool.

**MUST** comparison against `null` is explicit. `${slots.x} == null` is true only when the slot was never filled.

---

## 5. State storage (Redis)

### 5.1 Keys (MUST)

| Key | Type | TTL | Holds |
|---|---|---|---|
| `conv:ctx:{conversation_id}` | hash | 30d sliding | The [ConversationContext](01-schemas.md#3-conversationcontext) |
| `conv:lock:{conversation_id}` | string, set-if-absent | 30 s | Per-conversation mutex. Value is a holder token |
| `conv:opts:{conversation_id}` | string | 24 h | The option set last presented, for the [selection rule](05-channel-whatsapp.md#4-the-selection-rule-for-plain-text-must) |
| `cache:tool:{tool}:{args_hash}` | string | per-tool | Read-tool results, **discover-class tools only** |
| `cfg:active` | string | none | Active configuration version |
| `cfg:{version}` | string | none | Compiled configuration for that version |
| `cfg:reload` | pub/sub channel | | Version-change announcements |
| `kb:active` | string | none | Active knowledge version |
| `kb:{version}` | string | none | The compiled knowledge pack for that version, per locale. See [Part 8](08-knowledge-and-model.md#3-publishing-and-versioning) |

### 5.2 Rules (MUST)

**The lock is not optional.** Two messages from the same customer arriving together will otherwise interleave slot writes.

**Redis is a cache, not the record.** The durable store holds conversations, messages, leads and the audit trail. Losing Redis must cost a session resume, never data. Design the context so it can be rebuilt from the durable store.

**Deduplication belongs in the durable store.** See [Part 2, section 3](02-validation.md#3-deduplication-must). A Redis check is fast and forgettable, a unique constraint is durable and correct.

**Only discover-class tools are cached**, per [section 8.1](#81-tool-classes-must). A stale order status shown to a customer is worse than a slow one.

### 5.3 The lock protocol

Two messages, one conversation, arriving together. This is what correct looks like.

```
   Worker A                     Redis                      Worker B
      │                           │                           │
      │──SET lock NX EX 30 tokA──▶│                           │
      │◀────────── OK ────────────│                           │
      │                           │◀──SET lock NX EX 30 tokB──│
      │                           │─────── nil (held) ───────▶│
      │  load ctx                 │                           │  wait 250 ms
      │  run turn                 │                           │  retry (up to 5 s)
      │  persist                  │                           │
      │──DEL lock IF value=tokA──▶│                           │
      │◀────────── 1 ─────────────│                           │
      │                           │◀──SET lock NX EX 30 tokB──│
      │                           │◀───────── OK ─────────────│
      │                           │                           │  loads ctx *after* A persisted
      │                           │                           │  sees A's slot values
```

**MUST** release is conditional on the holder token matching. An unconditional delete lets a worker whose lock already expired release the lock a second worker legitimately holds, and the resulting corruption looks random.

**MUST** a worker that cannot acquire within the 5 second budget queues the message rather than proceeding. Never process unlocked, and never drop.

**NOTE** the ordering at the bottom of the diagram is the whole reason for the lock: B loads context *after* A persisted, so B sees A's answer rather than overwriting it.

### 5.4 Context load, and rebuild on a miss

```
        need ConversationContext
                 │
                 ▼
        ╭──────────────────╮  hit
        │ conv:ctx present?│───────▶ use it · refresh the sliding TTL
        ╰────────┬─────────╯
                 │ miss (evicted, or Redis was lost)
                 ▼
        ┌────────────────────────────────────────────┐
        │ rebuild from the durable store:             │
        │   last conversation row  → flow_id,         │
        │     pinned flow_version, current_state      │
        │   collected slot rows    → variables        │
        │   last inbound message   → last_inbound_at  │
        │   conversation record    → bot_mode, locale │
        └────────┬───────────────────────────────────┘
                 ▼
        ╭──────────────────╮  nothing found
        │ rebuilt?         │───────▶ treat as a NEW conversation
        ╰────────┬─────────╯          and say so gracefully
                 │ yes
                 ▼
          write back to Redis · continue the turn
```

**MUST** a cache miss is a normal event, not an error. It happens on eviction, on failover and on deploy. A conversation that cannot survive one is not resumable, and the customer experience of that is a bot that forgets an order halfway through.

---

## 6. Persistence and the turn log

Step 11. One commit, three things written.

| Written | Contains |
|---|---|
| Conversation context | The full [ConversationContext](01-schemas.md#3-conversationcontext) after the turn |
| Messages | The inbound message, and each outbound `GenericMessage` with its delivery state |
| Turn log, one row | See below |

The turn-log row, which is what makes behaviour explainable after the fact:

| Field | Why it is there |
|---|---|
| `trace_id`, `conversation_id`, `turn_index` | Correlation |
| `received_at`, `completed_at` | Latency, and the budgets in [2.7](#27-budgets-and-timeouts-must) |
| `channel`, `locale` | Segmentation |
| `flow_id`, `flow_version`, `previous_state`, `current_state`, `transition_used` | Exactly which path was taken, against exactly which configuration |
| `intent_detected`, `turn_class`, `classifier_confidence` | Whether the classifier was right, reviewable weekly |
| `slot_written`, `attempts` | Where customers struggle |
| `tools_called[]` with class, latency, cache hit, success | Which tool is slow, which is failing |
| `knowledge_used[]` with document ids and scores | Which answer came from which source, see [Part 8](08-knowledge-and-model.md) |
| `model_calls[]` with model, effort, tokens, latency | Cost and quality evidence, per turn |
| `guardrail_codes[]`, `escalated`, `escalation_reason` | Safety review |

**MUST** one row per turn, written in the same commit as the context. Two rows or none both make the log unusable for the weekly review that the design guide depends on.

**MUST** the log records ids and scores for knowledge, never the retrieved text. The text is in the knowledge store and it is versioned there. Copying it into every turn row makes the log enormous and the source ambiguous.

---

## 7. Configuration lifecycle

### 7.1 Publishing

```
 author edits flow JSON
        │
        ▼
  ┌───────────┐   fails    ┌──────────────────────────────┐
  │ VALIDATE  │───────────▶│ reject · previous stays live │
  └─────┬─────┘            └──────────────────────────────┘
        │ passes
        ▼
  ┌───────────┐
  │  COMPILE  │  resolve references · build transition index · precompute copy per locale
  └─────┬─────┘
        ▼
  ┌───────────┐
  │   STORE   │  cfg:{new_version}
  └─────┬─────┘
        ▼
  ┌───────────┐
  │   SWAP    │  cfg:active → new_version   (atomic)
  └─────┬─────┘
        ▼
  ┌───────────┐
  │ ANNOUNCE  │  publish on cfg:reload → workers load and swap in memory
  └───────────┘
```

### 7.2 What VALIDATE checks

The nine lint checks in [Part 2, section 7](02-validation.md#7-configuration-lint-must). They run before COMPILE, so a rejected configuration is never compiled, stored or announced.

### 7.3 Rules (MUST)

**Fail closed.** Configuration that fails validation is rejected whole. The previous version stays active. Never partially apply.

**Version pinning.** A conversation records `flow_version` when it starts and keeps it until it terminates. New conversations get the new version, in-flight ones do not move.

**NOTE** pinning is the subtlest requirement here. Without it, publishing a configuration that renames or removes a state strands every customer currently sitting in it: they reply, the engine looks up a state that no longer exists, and the conversation dies. Pinning makes publishing safe at any time of day, which is the entire point of hot reload.

**Retention.** Keep every version referenced by at least one open conversation. A version may be removed only when no conversation is pinned to it.

---

## 8. Tool dispatch

Shapes are in [Part 1, section 4](01-schemas.md#4-tool-interface-shapes). This section is the policy around them.

### 8.1 Tool classes (MUST)

| Class | Purpose | Cacheable | Writes |
|---|---|:---:|:---:|
| `discover` | Find candidates, such as search, browse, or knowledge retrieval | ✅ per-tool TTL, at `cache:tool:{tool}:{args_hash}` | ✗ |
| `verify` | Confirm a fact before stating it, such as availability or status | **✗ never** | ✗ |
| `transact` | Create or change something real | ✗ | ✅ idempotent |
| `record` | Capture an outcome for humans | ✗ | ✅ idempotent |

**MUST** every registered tool declares its class. A tool with no class is a configuration error, because the cache has no safe default: caching a verify tool tells customers stale facts, and never caching a discover tool makes browsing slow enough to abandon.

### 8.2 Idempotency (MUST)

Every `transact` and `record` invocation carries an idempotency key derived from:

```
{conversation_id} + {flow_id} + {state_name} + hash(arguments)
```

Replaying the same call returns the original result rather than creating a second record. This is what enforces the rule that a customer has exactly one active lead per intent.

**NOTE** deriving the key from conversation and state, rather than from a random value, means a retried turn produces the same key. A random key would make every retry a new order, which is the failure this prevents.

### 8.3 Dispatch, as a flowchart

```
        CALL_TOOL action
                │
                ▼
        ╭────────────────────╮  unknown
        │  registered?       │───────▶  configuration error
        ╰─────────┬──────────╯          (the lint catches this before publish)
                  │ yes
                  ▼
        ╭────────────────────╮  verify · transact · record
        │  class?            │─────────────────────────────┐
        ╰─────────┬──────────╯                             │
                  │ discover                               │
                  ▼                                        │
        ╭────────────────────╮  hit                        │
        │  cached?           │───────▶  return cached      │
        ╰─────────┬──────────╯                             │
                  │ miss                                   │
                  ▼                                        ▼
        ┌───────────────────────────────────────────────────────────┐
        │  invoke  ·  transact and record carry an idempotency key   │
        └─────────┬─────────────────────────────────────────────────┘
                  ▼
        ╭────────────────────╮  success: false, or timeout
        │  success?          │───────▶  the action's on_error state
        ╰─────────┬──────────╯          never an invented result
                  │ true
                  ▼
        bind to ${tool.result}  ·  cache only if class is discover
```

---

**Next:** [Part 4, Flow configuration](04-flow-config.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
