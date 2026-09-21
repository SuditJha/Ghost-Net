# Feature List

> **Core idea:** GhostNet is a conditional/stateful messaging system. A message may be visible immediately, remain locked until a condition is satisfied, reveal later in its original conversation position, and optionally expire after reveal.
> 

## Core message model

- Every message follows one common model rather than separate implementations for normal, scheduled, quiz, presence, or location messages.
- A message conceptually has:
    - content
    - conversation position / index
    - current state
    - optional reveal condition
    - optional expiry rule
- Typical lifecycle:
    - `CREATED → LOCKED / WAITING → REVEALED → EXPIRED`
    - Normal messages can go directly `CREATED → REVEALED`
    - Cancellation/deletion can terminate appropriate states.
- Reveal is normally irreversible. Once a locked message becomes `REVEALED`, it does not become locked again.

## SDE-required features

- **Normal 1:1 chat**
    - Basic user-to-user conversations.
    - Immediate messages behave like ordinary chat.
        
        
    - Offline recipients can retrieve the current conversation state after reconnecting.
- **Ordered / indexed messages**
    - Messages retain their original position in the conversation.
    - A locked message that reveals later appears at its original index rather than as a new message at the bottom.
    - Keep message identity separate from conversation ordering.
- **Scheduled message**
    - Sender can define a future reveal time.
    - Message stays locked until the configured time.
    - When due, transition `LOCKED → REVEALED`.
    - Recipient may see a locked placeholder before reveal.
- **Answer-before-reveal message**
    - Sender defines a question, expected answer, hidden message, and optionally a hint.
    - Recipient sees the question but not the hidden content.
    - Wrong answer causes no state transition.
    - Correct answer satisfies the condition and reveals the message.
    - Hidden content must not be sent to the client before reveal.
- **Hint**
    - Support one optional hint initially.
    - Keep advanced multi-stage hints for later.
- **Time-based expiry**
    - Message may expire after a configured duration, especially after reveal.
    - Example: reveal at 8 PM, expire 10 minutes after reveal.
    - Prefer leaving an expired tombstone in the ordered chat while deleting/hiding the message content.
- **Presence-based reveal**
    - Message can reveal when both participants are considered online.
    - Presence only determines when the transition is allowed.
    - Once revealed, disconnecting does not relock the message.
- **Explicit message state machine**
    - Valid transitions must be enforced centrally.
    - Invalid transitions such as `CANCELLED → REVEALED` must be rejected.
- **Cancellation**
    - Sender can cancel a waiting/locked message when allowed.
    - Late scheduler/condition events must not revive a cancelled message.
- **Durable server-side state**
    - Server/database remains the authoritative source of message state.
    - Client memory/cache is not the source of truth.
    - Refresh/reconnect should reconstruct current conversation state from the server.
- **Realtime WebSocket delivery**
    - WebSocket communicates state changes quickly to connected users.
    - WebSocket delivery is notification, not the source of truth.
    - If a state transition commits but a WebSocket push fails, reconnecting must still show the correct state.
- **Idempotent event/condition processing**
    - Duplicate scheduler or condition events must not reveal/deliver a message twice.
    - Reprocessing an already completed transition should safely become a no-op.
- **Concurrency-safe transitions**
    - If two workers/events attempt the same transition simultaneously, only one should succeed.
    - Use versioning/locking and transaction boundaries to protect state changes.
- **Late-event handling**
    - Scheduler/event processing must check the current state before changing anything.
    - Example: if a timer fires after cancellation or expiry, no invalid reveal occurs.
- **Tests**
    - Unit tests for state-transition rules.
    - Integration tests for scheduled, answer-locked, presence, expiry and reconnect behavior.
    - Concurrency tests for duplicate/racing reveal events.

## Useful extra

- **Location-locked message**
    - Reveal when the recipient reports being within an allowed radius of a preset location.
    - Treat this as game/location mechanics, not cryptographic proof of physical presence.
- **Treasure Hunt mode**
    - Chain ordinary GhostNet message primitives:
        - answer clue
        - timed clue
        - location clue
        - another answer clue
        - final expiring message
    - Avoid building a generic workflow engine initially.
- **Multiple / chained conditions**
    - Example: `TIME AND PRESENCE`.
    - Later allow dependency on a previously revealed message.
- **Common-message dashboard**
    - Reusable quick-message templates such as:
        - “Reached home?”
        - “Call me when free.”
        - birthday message
        - recurring hint/template
    - Templates simply create ordinary GhostNet messages with selected rules.
- **Redis-backed presence**
    - Store ephemeral online/presence information.
    - Define online semantics using authenticated WebSocket sessions and heartbeat freshness.
- **Kafka + transactional outbox**
    - Add only after the synchronous/modular-monolith flow works.
    - Use it to deliberately explore asynchronous events, duplicates, retries, ordering and eventual consistency.
- **Multi-instance WebSocket delivery**
- **Message-state history / audit trail**
- **Delivery/read receipts**
- **Rate limiting**
- **Observability**
- **Load/concurrency testing**

## Skip for now

- Strict “online-only” messaging.
- “Zero permanent storage” as the primary architecture goal.
- Guaranteed prevention of screenshots, copying or airplane-mode tricks.
- Complex end-to-end/client-side cryptography.
- AI-generated conditions.
- Automatically mutating message content.
- General-purpose workflow engine.
- Threshold cryptography / cryptographic deletion proofs.
- Multi-region global architecture.
- Kubernetes/service mesh.
- Large microservice split before the core state machine works.

## Product boundaries

- 1:1 chat first.
- Limited number of chats is acceptable for scope control.
- No groups, channels, communities, stories, calls, voice/video, forwarding, reactions or rich social features initially.
- GhostNet's differentiator is **conditional message lifecycle**, not feature parity with WhatsApp.

## Key behavioral rules

1. A locked message keeps its original conversation position after reveal.
2. Protected content is not sent to the client before its condition is satisfied.
3. Reveal is normally irreversible.
4. Database/state machine is authoritative; WebSocket only communicates changes.
5. Duplicate and late events must be harmless.
6. Expiry should be determined from durable message state/timestamps, not solely from a transient Redis notification.
7. Message identity and conversation order are separate concepts.
8. Once information has been shown to a recipient, GhostNet cannot guarantee they did not copy or record it.
