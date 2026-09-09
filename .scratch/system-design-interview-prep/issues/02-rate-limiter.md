# Mock Interview #2 — Design an API Rate Limiter

Status: resolved
Type: mock-interview
Date: 2026-09-09 (Week 2, Day 2)
Format: AI mock interview, full English, 45-min style

---

## Prompt

Design a rate limiter for a large, global web API platform (Stripe/Twitter-style; external
customers with API keys). Core scope: limit requests per client per time window, reject the
excess. Given constraints:

- Global: 3 regions (NA / EU / Asia), single global hostname, DNS geo-routing. **Same API key
  is legitimately active in multiple regions at once** (not anycast-pinned).
- Rate limiter sits at the API gateway / edge, in front of dozens of backend microservices —
  one chokepoint, per-API-key, platform-wide.
- ~1M registered keys, ~200k active per minute. Platform peak ~500k req/s.
- Default limit 1,000 req/min per key; single global bucket per key (all endpoints count).

## Clarifying questions asked (candidate)

1. Global vs region-local traffic origin
2. Whether there are many backend services
3. Number of API keys; whether a key can hit any endpoint

Reasonable coverage of the distributed dimension (good — that's the crux of this problem).
Did **not** ask about: read/write nature of the limit check, burst-tolerance requirement
(assumed it), latency budget for the check, what happens on limiter failure (fail-open vs
fail-closed) — these had to be drawn out by the interviewer later.

---

## Timeline & Findings

### 1. Algorithm selection — ✅ solid, good spontaneous trade-off framing
- Led with **token bucket** for burst tolerance; correctly named the alternatives and why he
  rejected them: fixed window → boundary bursts; sliding window log → memory grows with request
  volume. This is the Day 6/Day 8 material and it came out cleanly and unprompted.
- Mentioned **Redis + Lua script** for atomicity and **named the race condition** (`read → check
  → increment` non-atomic) without being asked. This was a specific Mock #2 acceptance item from
  Day 6 — **passed**.

### 2. Token bucket primitive — ⚠️ config vs state conflated, corrected under pushing
- First answer: "two variables, `last_update` and `refill_rate`" — mixed configuration and
  mutable state.
- After two rounds of pushing, landed on the correct split:
  - **config**: `capacity` (burst ceiling), `refill_rate`
  - **per-key state**: `{ tokens, last_update }`
- **Capacity sizing was not justified**: chose 20 "to accommodate the requirement." Could not
  articulate that capacity *is* the burst-policy knob (idle key can fire `capacity` requests
  instantly before throttling). Same shape as Mock #1: conceptual understanding present, but the
  precise primitive and its rationale needed to be extracted rather than volunteered.

### 3. Reject response — ⚠️ `Retry-After` format wrong first, corrected under pushing
- `429` — correct immediately.
- `Retry-After` — first said the value is `<timestamp>`. Corrected to **delta-seconds** only
  when explicitly challenged ("Is it a timestamp?").
- Did not mention `X-RateLimit-*` headers at all.
- Pattern identical to Mock #1's `Location`/`Set-Location`: the exact literal primitive is not
  cold yet.

### 4. Distributed design / global Redis — ⚠️ started at "cluster + replication, eventual
consistency" (mechanism-free), improved when pushed
- Initial answer was the Mock #1 failure mode verbatim: naming technologies ("redis cluster and
  replication, allow eventual consistency") without a mechanism.
- **Good**: when asked to pick one global Redis vs one-per-region and defend it, he committed to
  one global Redis, then immediately recognised on challenge that a synchronous cross-ocean
  round trip (+100–150ms on *every* API call) is unacceptable, and switched to **one Redis per
  region**.
- Correctly identified the consequence: per-region counters → customer can exceed the global
  limit (up to ~3× with 3 regions).

### 5. Bounding the cross-region overshoot — ❌ keyword-matching, no mechanism / no math
- Asked how to bound the 3× overshoot: answered "async counter and CAS optimistic transaction."
  CAS is single-store atomicity — it does not address how three regional counters share one
  1,000/min budget. This is reaching for a familiar term rather than answering the question.
- Second attempt: "monitor API-key usage across the three regions, then split the budget based
  on the result" — this *is* a legitimate direction (dynamic weighted budget allocation,
  periodically rebalanced), but he never gave the data flow (what syncs where, how often) or the
  worst-case number, despite being asked twice for "the math."

### 6. Cost of a chosen strategy — ❌ **recurring across the whole session**, benefits stated,
costs only on demand
- Had to ask "what's the cost / what do you give up" **at least three times** — for the capacity
  choice, for global Redis, and for fail-open on payments.
- Fail-open example: stated the *benefit* ("payments is critical, must not have downtime") twice
  before naming the *cost* (unbounded flood to payment backends during a Redis outage → whole
  system degrades). Even then, the mitigation offered was "local cache to prevent each request
  going to the backend" — the right instinct (degrade to local enforcement) but the wrong
  primitive: what's needed is a **coarse local fallback limiter** per gateway, not a cache.

### 7. Compound / backward-referencing question — ❌ **not paraphrased, and answered backwards**
- Deliberate compound probe (per Day 6/7 plan): "(a) if only Frankfurt's Redis is down and the
  customer's traffic is split across all 3 regions, what happens, and does it interact with your
  budget split; (b) does your fail-open choice change if this is the payments API?"
- Did **not** restate the question before answering (the paraphrase drill was the #1 carry-over
  item from Mock #1 — **still not applied unprompted**).
- Answered part (b) **inverted**: "fail-open on payments, fail-closed on general-purpose." That
  is the opposite of the conventional answer — a general API prioritises availability
  (fail-open, which is Stripe's actual published choice); payments is where you'd more plausibly
  fail-closed or add strong local fallback because an unbounded flood there has financial/fraud
  consequences. When the definitions were spelled out and he was asked to defend or correct, he
  **dug in** on the inverted answer rather than re-examining it.
- Part (a) — the interaction between fail-open and the per-region budget split — was not
  addressed at all.

### 8. Back-of-envelope estimation — ❌ essentially absent this session
- The interviewer supplied concrete numbers (1M keys, 200k active/min, 500k req/s). The
  candidate **never used any of them**: did not size Redis memory (200k active keys × ~2 ints +
  key overhead — this is literally Day 7 練習題 B), did not check whether one regional Redis can
  absorb the per-region request rate, did not derive `refill_rate` or `capacity` from the stated
  1,000/min limit (16.66/s is right but 1000/60 was not shown; capacity 20 was arbitrary).
- Day 7 Part 1 was built specifically to fix this and it did not transfer into the mock.

### 9. Over-reach at the top of the design — ⚠️ minor
- Volunteered the full Stripe 4-layer architecture (request limiter / concurrent limiter / fleet
  load shedder / worker load shedder) early, before designing the single per-key limiter that
  was actually asked. Good that the material is known, but leading with it before the core design
  reads as reciting. Lead with the simple design, then offer the extension.

---

## Communication / English notes

- "where do requests come from, globally or locality" → "…globally, or is traffic
  region-specific?"
- "it could happen boundary burst" → "boundary bursts can happen" / "it's vulnerable to
  boundary bursts"
- "the cost of memory will be large of amount" → "the memory cost will be large" / "it uses a
  large amount of memory"
- "I will avoid the rate limiter which can not be a bottleneck in the traffic flow" → garbled;
  intended "the rate limiter must not become a bottleneck in the request path"
- "it can have to work without downtime" → "it has to keep working even during an outage"
- "the requests flood into backend servers make the whole system go worst" → "…make the whole
  system worse" / "…degrade the whole system"
- **Pattern (same as Mock #1, and more consequential than the grammar)**: under a compound or
  backward-referencing question, the answer drifts or inverts, and the paraphrase-first drill is
  still not being applied. Additionally this session: reaching for a familiar technical term
  (`CAS`, "local cache") and attaching it to a problem it doesn't actually solve, instead of
  saying "I'm not sure how to bound that — let me think."

---

## Overall Assessment

**Strengths**
- Algorithm solution space is genuinely internalised — token bucket vs fixed/sliding window,
  the memory trade-off, Lua-script atomicity, the race condition, fail-open philosophy, caching
  the throttle decision locally, the Stripe layered model. Breadth from Day 6/Day 8 is there.
- Commits to a decision when pushed rather than staying vague.
- Recognised the global-synchronous-Redis latency problem instantly once it was framed
  concretely.
- Named the distributed race condition unprompted (Mock #2 acceptance item — passed).

**Weaknesses to carry into next mock (priority order)**
1. **Stating the cost of a chosen strategy** — this session's clearest and most repeated gap.
   Every strategy choice needs "…the cost is ___, and I'd mitigate it with ___" *in the same
   breath*, not after being asked. (Day 7 checklist #6.)
2. **Compound / backward-referencing questions** — still answering without paraphrasing, and
   this time inverting the answer (fail-open/closed) and defending the inversion. The
   paraphrase-first drill from Mock #1 has not transferred. (Day 7 checklist #1.)
3. **Back-of-envelope estimation not transferring into mocks** — was handed the numbers and used
   none of them. Redis memory sizing and per-region throughput checks should be reflexive here.
   (Day 7 checklist #3, Part 1.)
4. **Exact primitives still need extraction** — `Retry-After` value format, token-bucket
   config-vs-state split, capacity as the burst knob. Same class of gap as Mock #1's `Location`.
   (Day 7 checklist #4.)
5. When unsure how to answer (bounding the overshoot), say so and reason aloud — don't attach a
   memorised term (`CAS`) to it.

**Recommendation**: proceed to Week 2 Day 3 (debrief + Consistent Hashing) as planned. For
Mock #3 (Distributed Cache), the interviewer should again plant one compound/backward-referencing
question and one "what's the cost of that" pause, and should refuse to supply any capacity number
the candidate could derive — force the estimation out.

## Comments
