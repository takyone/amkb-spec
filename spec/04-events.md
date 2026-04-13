# 04 — Events

This chapter specifies the normative semantics of AMKB events: their
role, durability, ordering, retention, and replay. Event shape and
reserved kinds are defined in [02 — Types §2.6](02-types.md); the
`events` operation is defined in [03 — Operations §3.7](03-operations.md).
This chapter unifies the cross-cutting rules that govern how events
flow through a store and how observers consume them.

## 4.1 Role of the Event Log

Events are the externalized record of committed transactions. They
serve four distinct purposes, each with different caller expectations:

1. **Audit.** Given an event stream, an observer MUST be able to
   reconstruct *what changed, by whom, when, and under which
   transaction* without inspecting store state. Audit observers
   typically read past events and never subscribe to new ones.
2. **Replication.** A second store can be brought to the same state
   as a primary store by replaying its event stream. Replication
   observers subscribe to the live stream and apply events in order.
3. **Observation.** Dashboards, visualizers, and compliance tools
   subscribe to the live stream to react to changes. Observation
   observers care about low latency and tolerate at-least-once
   delivery.
4. **Revert.** The `revert` operation consults the event log for a
   target transaction to construct its inverse. Revert is an
   internal caller but imposes the strictest retention guarantee on
   the log.

The protocol treats the event log as the authoritative record of
mutations. Store state MAY be rebuilt from events; events MAY NOT be
rebuilt from store state alone (they carry intent — `actor`, `tag` —
that state does not).

See [99 — Rationale §R7](99-rationale.md) for the argument against
letting observers watch store state directly.

## 4.2 Emission

### 4.2.1 Atomicity

Exactly one `ChangeSet` MUST be emitted per committed transaction. A
ChangeSet is itself atomic: either all of its Events become visible
to the event stream together, or none does. Implementations MUST NOT
partially emit a ChangeSet to observers.

### 4.2.2 Aborted transactions

Aborted transactions MUST NOT produce a ChangeSet. Operations that
were issued inside an aborted transaction MUST NOT appear in any
event.

### 4.2.3 Idempotent operations

When an operation is a no-op (see 03-operations for the cases —
retiring an already-retired Node, unlinking an already-retired
Edge), the implementation MUST NOT emit an Event for it. Events
represent mutations that took effect, not invocations that were
received.

### 4.2.4 Event count per ChangeSet

A ChangeSet MUST contain exactly one Event for each mutation that
took effect within its transaction. A transaction containing `n`
effective mutations produces a ChangeSet with `n` Events, regardless
of whether the caller invoked one or many operations (for example,
a single `merge` call on `k` Nodes produces `k` retirement events
plus one creation event).

## 4.3 Ordering

Three ordering guarantees apply at different scopes.

### 4.3.1 Within a ChangeSet

Events within a single ChangeSet MUST be totally ordered, consistent
with the order in which the operations that produced them were
issued in the transaction. Implementations MUST NOT reorder events
within a ChangeSet, even when the operations are semantically
independent.

Consumers MAY rely on this order to reconstruct the intended
sequence of mutations.

### 4.3.2 Across ChangeSets

ChangeSets MUST be ordered in at least a **causal order** consistent
with commit time. If ChangeSet `A` was committed before ChangeSet
`B`, the event stream MUST yield `A`'s events entirely before any of
`B`'s events.

Implementations MAY provide a **total order** across all ChangeSets.
If they do, the total order MUST be consistent with commit time and
MUST be stable across restarts and replays.

### 4.3.3 Implications for observers

Because causal order is the minimum, observers MUST NOT assume that
two ChangeSets committed at the same wall-clock time will appear in
any particular order. Observers requiring a deterministic total
order MUST use an implementation that provides one.

## 4.4 Durability Levels

AMKB defines three durability levels for the event log. Level B is
the minimum required by L1 conformance; Levels A and C are
informative descriptions of weaker and stronger guarantees
respectively.

### 4.4.1 Level A — In-memory (not conformant)

Events live only in process memory and are lost on restart. A store
at this level does **not** conform to AMKB L1. Level A is mentioned
only to name the failure mode; no conformant implementation may stop
here.

### 4.4.2 Level B — Recoverable after restart (MUST, L1 minimum)

Committed ChangeSets and their Events MUST be recoverable after
store restart. An implementation MAY persist events in the same
physical store as state (a SQLite file, a Postgres database) as long
as events survive normal restart and recover the same ordering
guarantees as in §4.3.

Level B permits:
- Events stored in the same database as state
- Events derived from a write-ahead log
- Events flushed to disk on commit

Level B forbids:
- Events held only in memory
- Event retention conditional on store state survival (if the state
  table is truncated but the log file is not, events MUST still be
  recoverable from the log or from the combination)

### 4.4.3 Level C — Independent append-only log (SHOULD)

Events SHOULD be persisted to an append-only log that is
independent of the primary store state. "Independent" means the
log's failure modes are not correlated with the state store's
failure modes: corruption or truncation of the state store MUST NOT
imply corruption or truncation of the log.

Example: events appended to a separate log file or an external log
service while state is kept in a SQL database. Corruption of the
SQL file leaves the log intact, and the store can be rebuilt from
the log.

Level C is a SHOULD, not a MUST, because some deployment contexts
(single-file SQLite, embedded stores) legitimately prefer a single
physical artifact.

### 4.4.4 Level C+ — External streaming (MAY)

Implementations MAY stream events to external sinks (message
queues, object stores, webhooks) in near-real-time. This is a
convenience for observers; it is neither required nor credited
toward any conformance level.

## 4.5 Retention

### 4.5.1 Minimum retention

An implementation MUST retain all events from all committed
transactions for as long as any of the following is true:

- A revert of the originating transaction is possible.
- The implementation claims L2 conformance and any Node produced by
  the originating transaction is still live.
- The implementation has not explicitly pruned the event log
  according to §4.5.2.

In practice, retention is unbounded unless an operator opts into
pruning.

### 4.5.2 Pruning

Implementations MAY offer an out-of-protocol pruning mechanism that
deletes events older than a threshold. Pruning is outside protocol
scope — no AMKB operation triggers it, and the spec does not define
its interface. If pruning is offered:

1. It MUST NOT delete events belonging to transactions that are
   still revert-eligible.
2. It SHOULD record a pruning checkpoint so that later `events`
   calls with `since` before the checkpoint can return
   `E_INVALID` (or equivalent) rather than silently omitting
   events.
3. It MUST NOT silently drop events visible to a live
   `events(follow=true)` subscription.

## 4.6 Subscription Semantics

The `events(since, follow)` operation in 03-operations §3.7 is the
single subscription primitive.

### 4.6.1 At-least-once delivery

Subscription delivery is **at-least-once**. An event MUST eventually
be delivered to every live subscription whose `since` range includes
it. Implementations MAY redeliver events if a subscription drops and
reconnects; consumers MUST be prepared to deduplicate by event
identity (typically the ChangeSet ref plus the event index).

### 4.6.2 Backpressure

When a subscriber cannot keep up, the implementation MAY:
- Buffer events up to an implementation-defined limit.
- Slow the producer (if the backend supports it).
- Terminate the subscription and let the consumer reconnect with a
  later `since`.

The protocol does not specify which strategy. Consumers SHOULD be
prepared for either slow delivery or subscription termination.

### 4.6.3 Replay

A consumer that wishes to replay the full event history calls
`events(since=None, follow=false)`, which returns every retained
event in order and terminates. Combining replay with a subsequent
`events(since=last_seen, follow=true)` call yields a complete
catch-up + live-follow pattern; implementations MUST ensure no event
is lost between the two calls (transition SHOULD be atomic or overlap
by a small window).

### 4.6.4 Starting point

A `since=None` subscription starts from the earliest retained event,
which is:
- The first event ever committed, if no pruning has occurred.
- The pruning checkpoint, otherwise.

A consumer that needs to know whether it received a complete history
MUST compare its first observed event against the store's pruning
checkpoint (if pruning is in use).

## 4.7 Event Content

### 4.7.1 Before/after state

For mutations that change Node or Edge state, Events MUST carry
`before` and `after` snapshots sufficient to reconstruct the
mutation. "Sufficient" means:

- `node.created`: `after` carries the full Node; `before` is `None`.
- `node.rewritten`: `before` and `after` carry the Node state before
  and after the rewrite.
- `node.retired`: `before` carries the live Node state; `after`
  carries the tombstoned state.
- `node.merged`: one event per source Node with `before` = live,
  `after` = tombstoned; one additional `node.created` event for the
  merged Node. The merge relationship is expressed in the `meta`
  field, which MUST include the refs of all inputs and the resulting
  merged Node.
- `edge.created` / `edge.retired`: symmetric with Node events.

### 4.7.2 Content hashes

When an Event changes `Node.content`, the Event's `after` (and,
where possible, `before`) MUST include a `content_hash` field per
02-types §2.6.5. Observers rely on this to detect staleness in
external indexes without re-reading content.

### 4.7.3 Actor and reason

Every Event inherits the `actor` from its ChangeSet. Events MAY
carry an additional `reason` string in `meta` if the calling
operation provided one. `reason` is non-normative prose intended
for human audit trails.

## 4.8 Conformance

L1 implementations MUST:
- Emit a ChangeSet for every committed transaction (§4.2).
- Maintain ordering within ChangeSets and causal order across them
  (§4.3).
- Meet Level B durability (§4.4.2).
- Support the `events` operation with both `follow=true` and
  `follow=false` modes (§4.6).
- Retain events at least as long as their transaction is
  revert-eligible (§4.5).

L1 implementations SHOULD:
- Meet Level C durability (§4.4.3).
- Provide a total order across ChangeSets (§4.3.2).

L1 implementations MAY:
- Stream events to external sinks (§4.4.4).
- Offer out-of-protocol pruning (§4.5.2).
- Provide rich subscription filters beyond `since`.

Higher conformance levels (L2–L4b) inherit these requirements
unchanged. Level 5 (Policy, optional) MAY add authorization on
`events` subscription; the rest of this chapter is unaffected by
that extension.
