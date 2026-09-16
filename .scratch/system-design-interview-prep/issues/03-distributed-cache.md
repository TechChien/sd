# Mock Interview #3 — Design a Distributed Cache

Status: resolved
Type: mock-interview
Date: 2026-09-16 (Week 2, Day 5)
Format: AI mock interview, full English, 45-min style

---

## Prompt

Design a general-purpose distributed cache (Redis/Memcached-like) sitting in front of a
primary database, shared by ~a dozen backend microservices. Given constraints:

- Read-heavy: 20:1 read:write ratio. Peak read 300k req/s, write 15k req/s.
- Simple KV interface: `GET`/`SET`/`DELETE`.
- Dataset: 50M distinct keys, average value size 2KB.
- Latency budget: p99 < 10ms for a `GET`.

## Clarifying questions asked (candidate)

1. Read-heavy vs write-heavy
2. RPS
3. Number of APIs the system provides (ambiguous phrasing — clarified to mean the cache's own
   interface surface, not client count)
4. Data size and latency budget (asked together after some prompting)

Did **not** proactively ask about: failure/availability requirements, consistency requirements
between primary DB and cache, hot-key skew, key expiration semantics — reasonable first pass but
narrower than Mock #2's clarifying set.

---

## Timeline & Findings

### 1. Memory sizing — ✅ correct, used the numbers immediately
- 50M keys × 2KB = 100GB → 5 shards × 20GB, unprompted. This is exactly the reflex Mock #2 was
  missing (Day 7/Day 9 checklist #3) — **transferred successfully this session**.
- Minor gap: didn't account for per-key metadata overhead (Redis ~50-80B/key on top of value
  size) — not challenged on it this session, worth tightening later.

### 2. Per-node throughput ceiling — ⚠️ initially conflated latency with throughput, self-corrected under pushing
- First pass: derived "1 request / 10ms → 100 req/sec" from the p99 latency budget, then
  immediately used 100,000 (10^5) instead — a 1000x unexplained jump, using a number that didn't
  come from his own math.
- When challenged to reconcile, correctly separated the two axes: **throughput is bounded by
  concurrent-request handling capacity; latency is bounded by per-request processing speed** —
  these are different axes, and a low per-request latency does not by itself determine throughput
  ceiling.
- Made a units error along the way (1/100k = 10ns, should be 10μs — off by 1000x) but the
  conceptual correction was the important part and it landed.
- **Compare to Mock #2 finding #8** (back-of-envelope essentially absent): this session he *did*
  attempt the estimation unprompted, and *did* recover when the numbers didn't add up, rather
  than just asserting a plausible-sounding figure and moving on. Real improvement, even though
  the first attempt had errors.

### 3. Physical node count from throughput — ✅ correct once units-error was fixed
- 300k / 100k = 3 nodes as a baseline capacity floor — correct application of his (corrected)
  ceiling estimate.

### 4. Failure-headroom reasoning — ✅ strong, unprompted, chained correctly
- Volunteered the concern himself: "if one of 3 goes down, the others can't handle the traffic" —
  proactively connected sizing-for-capacity to sizing-for-failure, without being asked. This is
  new; neither Mock #1 nor Mock #2 showed this kind of forward-looking risk framing unprompted.
- When asked to trace the cascade explicitly: correctly walked through **basic consistent hashing
  concentrates a failed node's entire load onto exactly one clockwise neighbor** (not spread) →
  that neighbor's capacity is blown → triggers its own eviction → hot-key miss → backend traffic
  flood → and correctly extended it one more step: if the neighbor also can't absorb it, it
  crashes too, cascading further. This is a genuinely correct and fairly deep failure-mode chain,
  produced with only the "what happens to a failed node's traffic" prompt.

### 5. Virtual nodes vs physical node count — ❌ conflated the two, needed direct correction
- After correctly reasoning toward needing headroom, jumped to "**100 nodes**, cost is virtual
  node memory, mitigate by adding the proper number of virtual nodes" — this doesn't follow from
  anything calculated, and conflates **physical node count** (a capacity/failure-headroom
  decision) with **virtual nodes** (a load-evenness mechanism on the hash ring, orthogonal to
  physical count). Needed an explicit correction before recovering.
- After correction, recovered cleanly to **6 physical nodes** (50k/node baseline, one failure →
  neighbor at 50k+50k=100k, exactly at ceiling with zero margin but not over) — correct use of
  his own numbers.
- Cost/mitigation for 6 nodes was still garbled: said cost = "metadata overhead," mitigation =
  "virtual nodes so I can use fewer physical nodes" — internally inconsistent (virtual nodes add
  metadata, don't reduce physical count) and was flagged but not fully resolved before time to
  move on.

### 6. Eviction policy selection — ⚠️ mostly right, one clear mix-up, self-corrected on challenge
- Correct default: `allkeys-lru` for general-purpose mixed workload.
- First pairing was wrong: "even access pattern → `volatile-ttl`". Correct pairing is **even
  access → `allkeys-random`**; `volatile-ttl` is for the *different* case where the app can
  predict which keys will go stale soon and assigns them short TTLs. When asked to restate both
  mappings side by side, corrected both **cleanly and completely** in one pass — better recovery
  speed than the token-bucket/Retry-After corrections in Mock #1/#2, which took multiple rounds.

### 7. Compound question — ✅ paraphrased this time, first success across all 3 mocks
- This is the **#1 carry-over item since Mock #1** — never applied unprompted before.
- This session: paraphrased before answering, unprompted. Paraphrase was slightly imprecise on
  part (b) (restated as "impact on the replica" rather than the intended "does the eviction
  *mechanism* change") and had to be corrected once — but the *habit* of restating before
  answering finally fired. This is the single most important behavioral change this session.

### 8. Master-promotion eviction cost — ❌ reached for a memorized term again (recurring pattern)
- Correctly identified that the promoted node starts doing its own eviction (Day 10 material,
  applied correctly).
- But then named the cost as "stale data" and offered `WAIT` as mitigation — `WAIT` addresses
  *lost writes during failover*, unrelated to eviction. Same failure shape as Mock #2's `CAS`
  reach: grabbing a plausible-sounding term and attaching it to a problem it doesn't solve,
  instead of reasoning from what was just established.
- When asked "what does the promoted node's independent eviction actually cost, given what it
  had been ignoring as a replica," corrected to the right shape: sudden hot-key eviction burst →
  backend traffic flood. Missing piece even after correction: didn't connect it back to "the
  replica had been **ignoring `maxmemory`** the whole time it was a replica" (Day 10 §2.5) — that
  specific mechanism had to be supplied by the interviewer.

---

## Communication / English notes

- No notable grammar/vocabulary corrections needed this session — a first, compared to
  consistent notes in Mock #1 and #2. Sentences were shorter and more direct, which may have
  helped avoid the garbled-phrasing pattern seen before.
- Pattern carried over from Mock #2: reaching for a memorized term (`WAIT`) and attaching it to
  the wrong problem, rather than reasoning from the specific mechanism just established. This is
  now confirmed as a *recurring* pattern across two sessions (Mock #2: `CAS` for cross-region
  overshoot; Mock #3: `WAIT` for promotion-eviction cost), not a one-off.

---

## Overall Assessment

**Strengths**
- **Back-of-envelope estimation transferred into the mock** — memory sizing was immediate and
  correct; throughput/node-count estimation was attempted unprompted and self-corrected when the
  math didn't reconcile. Direct progress on Mock #2's #3 carry-over.
- **Compound-question paraphrasing fired for the first time**, unprompted. Direct progress on the
  #1 carry-over item since Mock #1.
- **Unprompted failure-headroom reasoning** and a genuinely correct, multi-step cascade-failure
  chain (neighbor overload → eviction → hot-key miss → backend flood → further cascade) — this is
  new depth, not seen in Mock #1 or #2.
- Fast, clean recovery on the eviction-policy mix-up (`volatile-ttl` vs `allkeys-random`) — single
  correction pass, no digging in.

**Weaknesses to carry into next mock (priority order)**
1. **Reaching for a memorized term instead of reasoning from the established mechanism** — now a
   *confirmed recurring pattern* across Mock #2 (`CAS`) and Mock #3 (`WAIT`). This should be the
   top priority for Week 2 Day 6 debrief: needs a concrete trigger-condition drill, similar to the
   compound-question paraphrase drill that just started working.
2. **Virtual nodes vs physical node count conflation** — jumped to "100 nodes" with no derivation,
   mixing a load-evenness mechanism with a capacity/failure-headroom decision. Needs a clean
   mental separation: physical count = capacity math; virtual node count = evenness tuning on top.
3. **Cost/mitigation pairing sometimes internally inconsistent** even when the underlying choice
   is right (6 nodes' "metadata overhead / mitigate with virtual nodes" didn't hold together).
   Saying *a* cost and *a* mitigation isn't sufficient if they don't logically connect — same
   checklist item as Mock #2 #6, but the new failure mode is "incoherent pairing" rather than
   "benefit-only, no cost at all."
4. Minor: units arithmetic (10ns vs 10μs) — low priority, didn't derail the session because the
   conceptual point survived the error.
5. Minor: didn't account for per-key metadata overhead in memory sizing — not pushed on this
   session, worth tightening before it's asked directly.

**Recommendation**: proceed to Week 2 Day 6 (debrief + 弱點補強). Priority for that session is
converting "reach for a memorized term" into the same kind of reflex-with-trigger-condition that
the compound-question paraphrase just became — this is the fourth session (Mock #1 implicitly,
Mock #2 `CAS`, Mock #3 `WAIT`) this exact shape of error has appeared.

## Comments
