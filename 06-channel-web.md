# Part 6: Web channel (Socket.IO)

**Part 6 of the** [KiboAuto AI Engine guide and specification](KiboAuto-AI-Engine.md) · Neurotech Africa for KiboAuto Tanzania Ltd
**Convention:** MUST · SHOULD · NOTE, as defined in [How to read this pack](KiboAuto-AI-Engine.md#how-to-read-this-pack).

Both directions of the web channel: stage 2 inbound and stage 5 outbound. This channel is being built from scratch, so every name is given explicitly. Event names and payload shapes here are normative, because a browser client and a gateway have to agree on them exactly.

---

## 1. Transport (MUST)

The web channel runs on **Socket.IO**, on both server and browser. This is a decision, not a default, and the event contract in section 3 is written against it.

| Relied on | Used for |
|---|---|
| Namespaces | Isolating chat traffic on `/chat` from any other realtime feature on the site |
| Rooms | Addressing one conversation, `conv:{conversation_id}`, so a human agent and the engine can both write to it |
| Acknowledgements | `message:send` returns `{ accepted, provider_message_id }` on the send itself, which is what lets the client show a sent state honestly |
| Automatic reconnection with backoff | A customer on mobile data reconnects without losing the conversation |
| Handshake `auth` | Carrying `session_token` before any event is accepted |
| Pub/sub adapter | Broadcasting to a room from any gateway instance |

**MUST** the browser client uses the Socket.IO client, not a bare WebSocket. The two are not wire compatible, and the reconnection, acknowledgement and room semantics above are the parts of the contract that make [replay](#43-rules-must) and delivery state work.

**NOTE** the reason for choosing it over plain WebSocket or server-sent events: this channel needs bidirectional events, per-conversation addressing, acknowledgements and reconnection. Building those on a bare socket is rebuilding Socket.IO, less well, on the critical path of a phase-one deadline. The engine itself remains transport agnostic: everything Socket.IO-specific stops at this file.

---

## 2. Connection

| Item | Value |
|---|---|
| Namespace | `/chat` |
| Room | `conv:{conversation_id}` |
| Handshake auth | `{ session_token }`, short-lived and signed |
| Session | HttpOnly, Secure cookie |
| Token issuance | `POST /api/chat/session` returns `{ session_token, expires_at }` |
| Multi-instance | Shared pub/sub adapter, so any gateway instance can broadcast to a room |

**MUST** the token authenticates the *connection*, the cookie carries the *session*. A connection without a valid token is refused before any event is accepted.

---

## 3. Parsing a web event (MUST)

1. Validate the handshake session token. Reject unauthenticated connections.
2. Resolve `conversation_id` from the session, **never** from client-supplied data.
3. Parse the event to [InboundMessage](01-schemas.md#1-inboundmessage).

| Source | To `InboundMessage` |
|---|---|
| session subject | `user_id` |
| session | `conversation_id` |
| `"web_" + client_msg_id` | `provider_message_id` |
| time of receipt | `received_at` |
| `text` | `text`, with `message_type` = `TEXT` |
| `selection` | `selection`, with `message_type` = `POSTBACK` |
| `media[]` | `media`, with `message_type` = `MEDIA` |
| `session:start` payload | `context`: `listing_id`, `utm`, `landing_page`, `referrer` |
| `session:start.locale` | `locale_hint`, the browser or site language, or `null` |
| entire event | `raw` |
| not from the payload | `channel` = `web` |

**MUST** `provider_message_id` is derived from the client-supplied `client_msg_id`, prefixed `web_`. This gives web the same durable deduplication key as WhatsApp, so **one deduplication rule covers both channels**. A client that retries a send with the same `client_msg_id` is deduplicated exactly as a retried webhook is.

**MUST** the client never sends conversation history, and the server never reads history from the client. This closes both the data-loss defect on the current site and a prompt-injection hole.

---

## 4. Events

### 4.1 Client to server

| Event | Payload | Ack |
|---|---|---|
| `session:start` | `{ locale?, context: { listing_id?, utm?, landing_page?, referrer? } }` | `{ conversation_id, resumed, bot_mode }` |
| `session:resume` | `{ conversation_id, last_event_id }` | `{ events: [ ... ] }` |
| `message:send` | `{ client_msg_id, text?, selection?: { id }, media?: [{ id }] }` | `{ accepted, provider_message_id }` |
| `typing` | `{ state: "start" \| "stop" }` | none |

**MUST** `message:send` carries exactly one of `text`, `selection` or `media`. `client_msg_id` is a client-generated UUID, required, and is what makes the send idempotent.

### 4.2 Server to client

| Event | Payload | Purpose |
|---|---|---|
| `bot:typing` | `{ on: boolean }` | Emitted on receipt, before any engine work |
| `message:start` | `{ message_id, type }` | A message is beginning |
| `message:delta` | `{ message_id, seq, chunk }` | Incremental text |
| `message:end` | `{ message_id }` | Stream complete |
| `message` | `{ event_id, message_id, message: GenericMessage, meta }` | **Authoritative** final message |
| `state` | `{ flow_id, state, requires }` | Flow position, for UI affordances |
| `bot_mode` | `{ mode: "active" \| "paused" \| "handoff" }` | Who is speaking |
| `agent:joined` | `{ agent_name }` | Human takeover became visible |
| `error` | `{ code, message, recoverable }` | Specific, actionable failure |

### 4.3 Rules (MUST)

- **`message` is authoritative.** `message:delta` is a preview. On receiving `message`, the client discards any accumulated preview for that `message_id` and renders the message content instead.
- **`event_id` is monotonic per conversation**, carried on every `message`. It is the cursor for `session:resume`.
- **The client deduplicates on `message_id`.** Replay after reconnect will resend events the client already has.
- **`bot:typing` fires on receipt**, before classification or any model call. Perceived latency is what the customer experiences.
- Deltas are emitted **only** for `TEXT`. Structured types (`BUTTONS`, `LIST`, `CARDS`) arrive whole, because a half-rendered option set is worse than a brief pause.

### 4.4 Error codes

| `code` | `recoverable` | Meaning | Client behaviour |
|---|:---:|---|---|
| `AUTH_INVALID` | no | Token invalid or expired | Re-issue token, reconnect |
| `SESSION_UNKNOWN` | no | `conversation_id` not found or not owned | Start a new session |
| `RATE_LIMITED` | yes | Too many messages | Back off, show a notice |
| `MESSAGE_INVALID` | no | Payload failed validation | Log, do not retry unchanged |
| `ENGINE_TIMEOUT` | yes | Turn exceeded the budget | Offer retry, offer a human |
| `ENGINE_ERROR` | yes | Unhandled failure | Show the fallback message and the human route |
| `BOT_PAUSED` | yes | Sent while a human owns the conversation | Accept and queue, do not error visibly |

---

## 5. Web rendering (MUST)

How one [GenericMessage](01-schemas.md#22-genericmessage-must) becomes web events.

| `GenericMessage.type` | Rendered as |
|---|---|
| `TEXT` | `message:start`, then `message:delta` zero or more times, then `message:end`, then `message` |
| `BUTTONS` · `LIST` | `message` with the full structure. The client renders quick replies or a list |
| `CARDS` | `message` with `cards[]`. The client renders them natively |
| `IMAGE` · `DOCUMENT` | `message` with `media` |
| `LOCATION_REQUEST` | `message`. The client renders a location prompt |
| `CTA_URL` | `message`. The client renders a link button |
| `TEMPLATE` | **Not applicable.** Templates are a WhatsApp construct. If a flow produces one for web, the transport renders `template_variables` into `text` and sends `TEXT` |

**MUST** the web transport applies **no** button or row limits. Those exist for WhatsApp and must not constrain web rendering.

---

## 6. Connection lifecycle

```
connect (auth: session_token)
   │
   ├── invalid ──▶  error: AUTH_INVALID  ──▶ disconnect
   │
   └── valid
        │
        ├── session:start  ──ack──▶ { conversation_id, resumed: false, bot_mode }
        │      └── join room conv:{id}
        │
        ├── session:resume ──ack──▶ { events: [...] }        (replay from last_event_id)
        │
        ├── message:send   ──ack──▶ { accepted, provider_message_id }
        │      └── bot:typing(on) → [engine] → message:start → message:delta* → message:end → message
        │
        └── disconnect ──▶ room membership dropped, conversation state untouched
```

**MUST** disconnection never changes conversation state. A customer who closes the tab and returns resumes exactly where they were.

---

**Next:** [Part 7, Conformance](07-conformance.md) · **Up:** [Guide and spec](KiboAuto-AI-Engine.md)
