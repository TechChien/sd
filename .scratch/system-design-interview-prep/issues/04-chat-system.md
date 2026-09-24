# Mock Interview #4 — Design a Chat / Messaging System

Status: resolved
Type: mock-interview
Date: 2026-09-24 (Week 3, Day 3)
Format: AI mock interview, full English, 45-min style

---

## Prompt

Design a 1:1 chat system (WhatsApp / Messenger-like). Group chat and read receipts are out of
scope for this session (reserved for Mock #5).

Constraints are withheld and revealed only when the candidate asks clarifying questions:

- 50M DAU, each user sends ~40 messages/day on average; peak traffic ≈ 3× average.
- Average message size ~100 bytes of text (no media this session).
- Messages must be persisted; retention = forever. A user can scroll back through history.
- Users can be offline; they must receive missed messages when they reconnect.
- Single device per user (multi-device is out of scope).
- Delivery status shown to the sender: `sent` / `delivered` (no `read` this session).
- Latency target: message delivered to an online recipient within ~500ms (p99).
- Online presence indicator ("last seen / online") is nice-to-have.

## Session focus (carry-over from spec.md)

- Paraphrase compound questions and answer **every** sub-part.
- Every design choice: "cost is ___, mitigation is ___", and check the two actually connect.
- Keep delivery guarantee / ordering / consistency model as three separate axes (Day 14).
- Back-of-envelope estimation: sanity-check the order of magnitude.
- Reason from the mechanism just established; don't reach for a memorized term.

## Clarifying questions asked (candidate)

1. Messages per day, and peak traffic → answered: 50M DAU × 40 msgs/day, peak ≈ 3× average.
   - English: "how many messages users sent per day? and in the condition of peak" → "How many
     messages do users send per day? And what about at peak?"
2. Does the system retain messages to replay/restore conversation history → answered: yes,
   persisted forever; users can scroll back through history; ~100 bytes text per message.
   - English: phrasing was clear and natural. Minor: "replay" usually implies re-processing
     events; for a user-facing feature, "load / scroll back through history" is more precise.
3. "How many replicas does the system need?" → ⚠️ asked the interviewer for a **design decision**
   instead of a **requirement**. Redirected: "what would you need to know to decide that?" The
   underlying requirements are durability (can an acknowledged message be lost?) and
   availability target — replica count should be derived from those by the candidate.
4. (Implicit, later) "3 replicas is my assumption, users across EU/US/Asia" → stated as an
   assumption rather than asked; interviewer confirmed users are global.

Did **not** ask about: latency target, online/offline behaviour, delivery-status requirements,
single vs multi-device, presence. Offline delivery and `sent`/`delivered` had to be introduced by
the interviewer in the design prompt. Narrower clarifying set than Mock #2.

---

## Timeline & Findings

### 1. Back-of-envelope — ⚠️ started unprompted (good), but peak unit error + stopped at per-day
- Candidate did not answer the redirect from Q3 (what requirement decides replica count) —
  moved on to estimation instead. The question was dropped rather than closed.
- ✅ 50M × 40 = **2B messages/day** — correct.
- ❌ "**6 billion during peak hours**" — peak 3× applies to the **rate** (QPS), not the daily total.
  The day still has 2B messages; correct form: 2B / 86,400s ≈ **~23k msg/s avg → ~70k msg/s peak**.
  Never converted to per-second at all, and QPS is the number that sizes servers/brokers. Same
  shape as spec.md's "估算單位/量級混亂" weakness.
- ✅ 100B × 2B = **200GB/day** — correct.
- ⚠️ Stopped at per-day even though the requirement was "retain forever": did not extend to
  per-year (~73TB/yr) or a multi-year horizon, which is what actually drives the storage choice.
- ⚠️ 100B is text only; no allowance for per-row metadata (message_id, sender, receiver,
  timestamp, status) — same gap as Mock #3 "per-key metadata overhead".
- ⚠️ "users across regions → enlarge 3–5×" — an **assumption** (multi-region never asked/confirmed)
  and conflates **replication factor** (durability/availability) with **region count** (latency/
  geo); the 3–5 range isn't derived from anything.
- Interviewer probed with a compound question (peak derivation + QPS; total storage for
  "forever" + where 3–5× comes from) to test paraphrase-first.

### 2. Follow-up on estimation — ✅ fast QPS self-correction; ⚠️ one sub-question still unanswered
- ✅ Acknowledged the rate-vs-total confusion by name and corrected in one pass:
  2B / 86,400 ≈ **23k QPS avg → 69k QPS peak**. Clean recovery, no digging in.
- ✅ **Unprompted storage tiering**: keep 3 months hot → 3 × 30 × 200GB = **18TB** — correct math and
  a sensible cost-aware move nobody asked for.
- ⚠️ Tiering left half-open: nothing said about where messages **older than 3 months** go or how
  big that tier gets (~73TB/yr raw, growing forever) — the "forever" requirement was answered only
  for the hot slice.
- ❌ **"Where does the multiplier come from / which requirement drives it" — not answered.** Replaced
  3–5× with "3 replicas in different regions" as an assertion, with no requirement behind it and
  no cost. Second time this session the replica-count justification was dropped (see Q3).
  Compound question: no paraphrase before answering, and sub-part 2b was skipped — same pattern
  as Day 14 "compound 問題只答一半".
- ⚠️ Still conflating replica count with region count ("3 replicas in different regions").

### 3. Replicas / regions / archive — ✅ decomposed the compound question into 5 explicit sub-parts
- ✅ **Best compound-question handling so far**: after the coaching note, split the question into
  5 numbered sub-questions, restated each, and answered every one. The Day 14 "只答一半" pattern
  did not recur on this turn.
- ✅ Honest about the basis: "3 replicas is my assumption" — but should have turned that into a
  clarifying question ("are users global?") rather than an assumption. Interviewer then confirmed
  it: users are global (US / EU / Asia).
- ⚠️ **Replica count vs region count still fused**: RF=3 is a durability/availability decision
  (usually across AZs *within* a region); region count is a latency/DR decision. They are separate
  knobs. "3 because 3 regions" ties them together for no reason.
- ✅ Cost #1: "**stale data temporarily because of replication latency**" — correctly placed on the
  **consistency-model** axis (visibility across replicas), not confused with loss or ordering.
  Positive vs Day 14's three-axis mix-up.
- ✅ Cost #2: storage multiplies with region count — correct.
- ❌ **No mitigation given for either cost** — cost/mitigation pairing is still half-done
  (spec.md long-standing weakness: "代價是 ___，緩解是 ___").
- ✅ Archive tier for > 3 months, cheaper storage — correct direction. ⚠️ Did not state the archive's
  own cost (slow retrieval when a user scrolls back past 3 months) or size it.

### 4. High-level flow (online / offline) — ⚠️ pipeline shape right, `delivered` semantics wrong, routing missing
- ✅ Both sub-cases (a)/(b) answered, and the pipeline shape (A → MQ → consumer → WebSocket → B)
  matches Day 13/14 material.
- ✅ **Offline sync by sequence number on reconnect** ("fetch everything after my last seq") —
  correct application of Day 14's monotonic cursor, unprompted.
- ✅ **Batching acks by max sequence number** (a cumulative ack: "update rows with seq > last
  acknowledged") — a genuinely good idea, and it applies equally to `delivered` acks.
- ❌ **`delivered` is set by the consumer before/without B receiving the message.** In (b) B is
  offline, yet the status becomes `delivered` → A sees "delivered" for a message B never got.
  `delivered` must mean **B's device acknowledged receipt** (client ACK over the WebSocket), not
  "a consumer picked it up".
- ❌ **Scope not respected**: read receipts / `read` status were explicitly out of scope for this
  session, but both (a) and (b) spent a step on them. Similar to not listening to the stated
  constraints.
- ❌ **No connection routing**: "consumer sends via WebSocket to B" skips the key mechanism. With
  ~millions of concurrent connections spread over many gateway servers, how does the consumer find
  *which* server holds B's socket? (Needs a session/connection registry: user_id → gateway server.)
- ⚠️ Missing pieces: who **persists** the message and when `sent` is set (the server, after a durable
  write/broker ack, not the client "changing status"); clients shouldn't publish to MQ directly
  (they go through a chat/gateway server); how A is **notified** of `delivered`.
- Interviewer probed: (1) what A sees in case (b); (2) how the consumer locates B's connection.

### 5. `delivered` fix + connection registry — ✅ quick correction; ⚠️ "who decides" skipped, no cost again
- ✅ Admitted the mistake directly ("it is wrong on my previous answer") and fixed it: A sees `sent`;
  `delivered` only once B actually receives the message.
- ⚠️ **"Who decides that?" — not answered.** The mechanism is a **client ACK** from B's device
  (sent back over the WebSocket, or implied by the reconnect sync) that the server turns into
  `delivered` and pushes to A. Answer stayed at "when B gets it" without naming the trigger.
  Paraphrase/decomposition fired again (numbered sub-parts), but one sub-part still went missing.
- ✅ **Connection registry in Redis: user_id → connection**, queryable by any consumer — correct
  core mechanism.
  - ⚠️ Precision: the consumer needs **user_id → gateway server** (which box to route to), not just a
    connection_id; then it forwards to that gateway (RPC / per-server queue).
  - ❌ No cost/failure mode stated for the registry (e.g. a gateway crashes → stale entries point to
    a dead server). Third design choice this session with no cost/mitigation volunteered.

### 6. `delivered` trigger + registry failure modes — ✅ volunteered failure/mitigation pairs
- ✅ Named the trigger and the notification path: B's fetch hits the API handler → publish
  `delivered` event → consumer → look up A's connection → push to A over WebSocket. Complete path.
- ⚠️ Subtle: "API handler served the response" ≠ "B's device received it" (the response can be lost
  mid-flight on a flaky mobile network). The robust trigger is an explicit **client ACK** after B's
  device stores the message. Also only the offline path was covered; the online path needs the
  same ACK over the WebSocket.
- ✅ **Split failure analysis into two paths unprompted** (Redis crash vs WebSocket server crash)
  and gave a **mitigation for each** — Sentinel failover; client reconnect with exponential
  backoff then re-register. First time this session cost→mitigation came out paired (after an
  explicit prompt to do so without being asked separately).
- ⚠️ Gaps in the pairing:
  - Stale window between the crash and the reconnect: registry still points at the dead server →
    consumer's push fails. Needs: treat a failed push as "offline" (message is already persisted,
    B syncs on reconnect) + heartbeat/TTL on registry entries.
  - Mass reconnect after a server crash → thundering herd; exponential backoff helps, **jitter** is
    the standard complement.
  - Sentinel failover is async → recent registry writes can be lost; acceptable because the
    registry is soft state that clients re-register. Not stated, but not wrong.

### 7. Lost ACK + ordering — ✅ axes kept separate; ❌ misread the lost-ACK scenario
- ✅ **Delivery vs ordering kept on separate axes**: Q1 answered in delivery terms, Q2 answered with a
  sequence number — **no consistency-model vocabulary borrowed** into either answer. Direct
  improvement over Day 14's three-axis mix-up.
- ✅ Keeping the message as `sent` in storage when no ACK arrives — correct (never mark it
  `delivered` without an ACK).
- ❌ **Delivery guarantee not named.** Answered with behavior ("B gets all unread messages when back")
  instead of the term + definition: this is **at-least-once** (keep/resend until ACKed).
- ❌ **Side effect answered for the wrong scenario.** The scenario was "B *received* the message but
  the ACK was lost" → the message is re-sent → **B gets a duplicate** → fix with client-side
  **dedup by message_id / seq** (idempotent receiver). Candidate answered "too many messages at once
  → render only the most recent" — a real but *adjacent* problem (a long offline backlog). This is
  the spec.md "答成相鄰但不同的主題" pattern (Mock #1) resurfacing.
- ⚠️ Ordering: "sequence number generated when the producer publishes" — right tool, but
  **who generates it** is unclear. With many chat servers acting as producers, per-producer counters
  don't give a single order per conversation. Needs a single authority per conversation (per-
  conversation counter, or partitioning by conversation_id so one partition/consumer owns it).

### 8. Duplicate handling + sequence authority — ✅ both mechanisms correct after one correction
- ✅ Corrected the scenario: no ACK → consumer re-sends → B may see the same message twice →
  attach an **idempotency id** to each message and dedup on the client. Correct pairing of cost
  (duplicate) and mitigation (idempotent receiver) — they logically connect.
- ❌ **Name of the guarantee still not given** ("at-least-once"). Asked twice across two turns; both
  times the sub-part was skipped. Compound-question completeness is still not reliable when one
  sub-part is a "just name it" question.
- ✅ **Sequence authority = the broker**, messages partitioned by `hash(conversation_id)` → one
  partition owns the whole conversation → the partition offset is strictly increasing within it;
  client sorts by seq. This is a **valid, concrete answer** (Kafka-style per-partition offsets) and
  matches Day 14 Part 2.3's "single authority keyed by conversation_id".
- Caveats not raised (time ran out, not probed):
  - Offsets are monotonic but **not contiguous** per conversation (other conversations share the
    partition) → the client can't detect a missing message from a gap; `seq > last_seen` sync still
    works.
  - **Changing the partition count** re-maps `hash(conversation_id)` → ordering can break during a
    repartition.
  - Producer retries can append the same message twice with two different offsets → needs an
    idempotent producer or the message-level idempotency id (which the candidate already has).

---

## Communication / English notes

- Recurring grammar: third-person -s missing/misplaced ("user A see", "User B will gets", "publishs"),
  missing plurals ("2 billion message", "3 replica", "one of consumer"), double comparative ("more
  cheaper"), noun used as verb ("storage them"). None blocked understanding, but the density is
  higher than Mock #3 (which had almost none) — longer answers this session.
- Word choice: "until" used where "when/once" was meant; "in the condition of peak" → "at peak";
  "void" → "avoid"; "conversion id" → "conversation id"; "the connection crashes" → "the server
  crashes / the connection drops".
- Typos in technical terms (webscoket, seqence, reconneciton, nofity) — fine in chat, but in a
  live interview spell them out loud clearly.
- Positive: numbered sub-answers made long answers easy to follow; "let me correct" is a good,
  natural recovery phrase and was used well.

---

## Overall Assessment

**Strengths**
- **Compound-question decomposition clearly improved**: from the replicas/regions question onward,
  the candidate split each multi-part question into numbered sub-parts and answered them in order.
  Biggest behavioural step since Mock #3's first paraphrase.
- **Day 14's top weakness did not recur**: delivery guarantee, ordering and consistency stayed on
  separate axes all session (stale data → replication lag; duplicates → idempotency; order →
  sequence number). No borrowed vocabulary.
- **Fast, clean self-correction** on every flagged error (peak total vs QPS, `delivered` semantics,
  lost-ACK scenario) — one pass each, with an explicit "let me correct".
- **Unprompted good moves**: storage tiering (hot 3 months → archive), sequence-number cursor sync on
  reconnect, cumulative ack by max seq, partition by `hash(conversation_id)` for ordering.
- Once prompted to volunteer cost/mitigation, did it for **two failure paths at once** (Redis crash /
  WebSocket server crash), each with a mitigation that logically connects.

**Weaknesses to carry into next mock (priority order)**
1. **Sub-parts that ask for a name or a "who" get skipped** — "which requirement?", "who decides
   `delivered`?", "what's the name of the guarantee?" (×2). Decomposition now happens, but the
   checklist isn't verified before finishing. Drill: after answering, re-read the numbered list and
   tick each item.
2. **Cost/mitigation still not volunteered by default** — the first three design choices (replicas,
   archive tier, Redis registry) came with no cost or no mitigation until explicitly asked.
   Improved *after* the prompt; needs to fire *before* it.
3. **Semantics of a state before designing the flow**: `delivered` was set by a consumer, not by the
   recipient's ACK; read receipts (out of scope) were designed in. Define what each status *means*
   and who can assert it, then build the pipeline.
4. **Replica count ≠ region count** — conflated twice. RF comes from durability/availability
   (across AZs); regions come from latency/DR. Separate knobs.
5. **Clarifying questions**: asked for a design decision ("how many replicas?") instead of a
   requirement; missed latency, offline behaviour and delivery-status requirements. Turned an
   assumption (global users) into a design input instead of a question.
6. **Estimation**: peak multiplier applied to the daily total instead of the rate (self-corrected);
   per-row metadata still omitted (same as Mock #3); "forever" sized only for the hot tier.
7. Answering an adjacent scenario (lost ACK → "backlog rendering") — Mock #1 pattern resurfaced once,
   corrected after the scenario was restated precisely.

**Recommendation**: proceed to Week 3 Day 4 (debrief + 弱點補強). Priorities: (1) "tick every
sub-part" verification step, (2) cost/mitigation *before* being asked, (3) define state semantics
first. Mock #5 (group chat + read receipts) is a good test for #3: read receipts in a group force a
precise definition of `delivered`/`read` per member.

## Comments
