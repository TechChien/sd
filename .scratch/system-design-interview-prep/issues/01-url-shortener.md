# Mock Interview #1 — Design a URL Shortener

Status: resolved
Type: mock-interview
Date: 2026-08-22 (Week 1, Day 5)
Format: AI mock interview, full English, 45-min style

---

## Prompt

Design a URL shortening service (like bit.ly / TinyURL). Core scope agreed during clarification: **create short URL** + **redirect**. Custom aliases/expiration in scope if time allows; analytics and user dashboards explicitly out of scope.

## Clarifying questions asked (candidate)

1. Traffic volume (requests/day)
2. Any functionality besides create + redirect

Both reasonable, standard opening questions. Missing ones a stronger opening would include: read/write ratio (asked as a follow-up separately instead of bundled), custom alias support, expected data retention/expiration policy, whether URLs are anonymous or tied to an account (this later became a real contradiction point — see Finding 3).

---

## Timeline & Findings

### 1. Back-of-envelope estimation — ✅ correct
- 100M creates/day → 1,157 RPS write
- 100:1 read:write → 10B redirects/day → 115,740 RPS read
- Storage: 100M × 50 bytes/day ≈ 5 GB/day
- Math and unit handling were accurate and fast. No corrections needed here.

### 2. Redirect endpoint & header — ❌ two rounds of correction needed
- **Round 1**: proposed `GET /v1/hash(long_url)` returning `Set-Location: short_url`. Both the path and the header/target were wrong.
- **Round 2 (self-correction)**: fixed the endpoint to `GET /v1/{short_code}` and target to `long_url`, but **repeated the same wrong header name** (`Set-Location` instead of `Location`).
- Even on a direct, explicit prompt ("what's the actual HTTP header name"), the candidate did not supply the correct header name and the interview moved on without it being said.
- **Correct answer**: `GET /v1/{short_code}` → `302 Found` with header `Location: <long_url>`.
- **Why it matters**: `Location` is the only header a browser/client honors for redirect targets; `Set-Location` isn't a real HTTP header and would simply be ignored, breaking the entire redirect flow. This is a "know the exact primitive" gap, not a conceptual one — worth explicit memorization before the next mock.

### 3. Key generation method — ⚠️ initial answer created a hidden data-model contradiction, self-corrected once pushed
- Initial answer: `hash(long_url)` truncated to base62 (deterministic).
- **Contradiction surfaced by interviewer**: a deterministic hash means two different users submitting the identical `long_url` get the *same* `short_code`. But the Day 3 schema has one `owner_id` per `short_code` row — so whose row is it?
- Candidate corrected well once challenged: switched to **random + base62**, explicitly confirmed this allows two different users to get two different short codes for the same long URL, each with their own `owner_id`.
- **Takeaway**: the candidate can course-correct fast when a contradiction is pointed out directly, but did not catch it independently while presenting the original design. Cross-checking a new answer against decisions made on previous days (Day 3's schema) is the skill to build.

### 4. Collision-check correctness — ❌ significant gap, required three rounds of prompting
- **Round 1**: "check the redis to see if there are any collision, if exists, then regenerate" — checking a cache (with TTL/eviction) as the sole uniqueness authority at write time is unsound: an evicted-but-still-existing code in the DB would read as "available," producing a genuine duplicate `short_code` row.
- **Round 2**: candidate answered a *different* question (read-path staleness / TTL-vs-expiredAt alignment — which is actually correct and matches Day 4's lesson, but not what was asked).
- **Round 3**: re-asked in maximally concrete terms ("is this code already taken — Redis or DB?") before landing on the correct answer: **query the database**.
- **Takeaway — this is the most important finding from today's session**: the candidate needed the question narrowed three times before answering it, despite having the right underlying knowledge (DB as source of truth) readily available elsewhere in the conversation. This points to a **listening/question-tracking gap under English cognitive load** more than a knowledge gap — when a question is compound, or references something several exchanges back, the answer drifts to an adjacent-but-different topic instead of the one actually asked. Worth deliberately practicing "restate the question in your own words before answering" as a stalling/anchoring technique in real interviews.

### 5. Closing bottleneck question — ❌ did not track the specific probe
- Final probe: "given cache already absorbs most read traffic, is DB read really the first bottleneck, or is it something else (tied to the collision-check discussion)?" — this was designed to lead the candidate to notice that **DB write throughput / collision-check reads on the create path**, not redirect-read throughput, is the more likely pressure point once caching is in place.
- Candidate's answer ("add more backend nodes behind the LB with least-connections") addressed **application-server scaling**, not the database bottleneck question at all — a similar drift pattern to Finding 4.
- This is flagged rather than corrected in depth here, since the pattern is now established (see Finding 4) — worth targeting directly rather than re-explaining the same content again.

---

## Communication / English notes (compounding across the session)

- "any else function which implement" → "any other functionality that I should implement" (article/word-order + infinitive form)
- "I ready to move on" → "I'm ready to move on" (dropped copula "am")
- "I made some mistaken" → "I made some mistakes" / "I was mistaken" (adjective vs. noun confusion)
- **Pattern to watch, not just individual errors**: on at least two occasions (Findings 4 and 5), when a question was compound or referred back to earlier context, the response addressed an adjacent topic rather than the one asked. This showed up more than the vocabulary slips did, and is more consequential in a real interview — an interviewer may read it as not-quite-following the conversation rather than as a language-fluency issue. **Recommended drill**: before answering a multi-part or backward-referencing question, paraphrase it back in one sentence ("So you're asking whether the collision check should hit Redis or the DB — right?") before answering. This buys processing time and self-corrects drift immediately if the paraphrase is wrong.

---

## Overall Assessment

**Strengths**: estimation/math fluent and fast; solid grasp of the caching architecture (local LRU → Redis → DB with cache-aside on write, matching Day 4's lesson); self-corrects well once a contradiction is pointed out explicitly and simply.

**Weaknesses to carry into next mock**:
1. Precise technical primitives (exact header names, exact endpoint semantics) need to be nailed down cold — conceptual understanding alone didn't produce the right literal answer even after being told twice.
2. Cross-referencing today's answer against earlier decisions (schema, prior day's lessons) needs to happen proactively, not just when the interviewer flags a contradiction.
3. **Question-tracking under compound/backward-referencing prompts is the top-priority item** — recommend the "paraphrase before answering" drill in Mock Interview #2.

**Recommendation**: proceed to Week 1 Day 6 (debrief + Rate Limiter concept intro) as planned. Re-test question-tracking specifically in Mock Interview #2 (Rate Limiter, Week 2) by deliberately asking at least one compound/backward-referencing question and watching whether the paraphrase drill gets applied without being prompted.

## Comments
