# System Design Interview Prep — 8 週衝刺計畫

Status: active

## 目前進度

> 每次上課/mock interview 後更新此區塊，作為進度的唯一權威來源（換 agent session 時，先讀這裡再繼續）。

- **目前所在**：Week 2 Day 3 完成（**Mock #2 debrief 行動化 + Consistent Hashing 概念**；記錄見 `daily_road_map/2026-09-10-day9.md`）。準備進入 Week 2 Day 4：**Cache eviction policy、Replication for cache → Distributed Cache 設計**
- **已完成的 Mock Interview**：#1 URL Shortener（`issues/01-url-shortener.md`）、#2 Rate Limiter（`issues/02-rate-limiter.md`）
- **累積弱點清單**：
  - 容易停在「概念知道」層次，較少主動講到具體技術機制（如 L4/L7 路由如何影響設計決策）
  - Trade-off 分析時，容易漏抓系統的不對稱性（例如流量比例懸殊時，哪一半才是真正需要優化的對象）
  - 待加強：主動展現「不過度設計」的判斷力（例如先用 deployment pool 而非直接上完整 microservices；Day 4 Q3 全表掃描排程 vs TTL 自然過期，同一類問題）
  - **Back-of-envelope 估算容易失準**（Day 3 Q2：keyspace 大小估算錯了兩個數量級以上），需要刻意練習容量估算
  - 容易混淆「機率極低」與「保證」的差異（誤以為 random+hash 方法可以完全不查 DB 就保證唯一）
  - 部分名詞定義有誤記（如 Federation 的作用方向），需要每次先核對 primer 原文再下結論
  - **多處寫入的一致性/順序問題容易被忽略**（Day 4 Q2：dual write 到 DB 與 cache 時，沒先考慮寫入順序與失敗處理）
  - 選定策略時，容易只講優點、不主動點出該策略本身的代價與緩解方式（Day 4 Q1）
  - **【今日新增，優先處理】Compound/backward-referencing 問題的追蹤能力**：Mock #1 中兩次（collision-check 該查 Redis 還是 DB；DB 是否真的是第一瓶頸）被追問時,答非所問,答成相鄰但不同的主題,需追問到第三次才對到題——這比單純英文詞彙錯誤更值得優先處理,因為面試官可能解讀成「沒跟上對話」而非「語言不夠流利」。建議下次 mock interview 刻意練習「回答前先用一句話覆述問題」
  - **精確技術原語（exact primitive）掌握不牢**：如 HTTP redirect header 正確名稱是 `Location` 不是 `Set-Location`，即使被直接問兩次仍未答對 —— 概念性理解到位,但沒有把具體字面答案背熟
  - 新答案沒有主動跟前幾天的決策做交叉檢查（如 Day 5 一開始用 `hash(long_url)` 當 key,跟 Day 3 schema 的 `owner_id` 設計互相矛盾,靠面試官指出才發現）
  - **【Mock #2 新增，最高優先】選定策略時只講優點、不主動講代價**：整場被追問「代價是什麼」至少 3 次（capacity 選 20、global Redis、payments fail-open）。要求每次選型都在同一句話內補「代價是 ___，緩解是 ___」
  - **【Mock #2】compound 問題的 paraphrase drill 仍未內化**：Mock #1 的頭號 carry-over,Mock #2 依然沒有未經提示就覆述,且把 fail-open/fail-closed 答反（payments 答成 fail-open + general 答成 fail-closed）還堅持不改
  - **【Mock #2】容量估算沒有帶進 mock**：面試官給了 1M keys / 200k active / 500k req/s,一個數字都沒用（沒算 Redis 記憶體、沒驗證單區 Redis 吞吐、capacity 20 無推導）。Day 7 Part 1 練的東西沒有 transfer
  - **【Mock #2】不確定怎麼答時,會硬套一個記得的術語（CAS、local cache）到它其實無法解決的問題上**,而不是說「我不確定,讓我想一下」
- **每日問答記錄**：`daily_road_map/2026-08-19-day2.md`、`daily_road_map/2026-08-20-day3.md`、`daily_road_map/2026-08-21-day4.md`、`daily_road_map/2026-09-01-day6.md`、`daily_road_map/2026-09-02-day7.md`、`daily_road_map/2026-09-08-day8.md`、`daily_road_map/2026-09-10-day9.md`
- **Mock Interview 記錄**：`issues/01-url-shortener.md`、`issues/02-rate-limiter.md`
- **最後更新**：2026-09-10

## 背景

- **目標**：8 週後（約 2026-10-13）面試一線大廠（FAANG-level）資深工程師職位
- **次要目標**：順便補強 system design 整體實力
- **基礎**：已有基礎概念（CAP theorem、load balancer、cache 等），但未系統化練習過完整設計題
- **參考資料**：[system-design-primer](https://github.com/donnemartin/system-design-primer)
- **每日投入**：2-3 小時
- **語言**：課程說明與討論以 zh-TW 為主，專有名詞維持英文；mock interview 全程英文，並即時糾正溝通表達/用詞

## 教學規範：來源引用

- **每個核心觀念的講解，必須附上出處**，來源優先順序：
  1. [system-design-primer](https://github.com/donnemartin/system-design-primer) 對應章節（含其引用的外部連結）
  2. Primer 未涵蓋或不夠深入的主題（如 Consistent Hashing 細節、Snowflake ID、Geospatial indexing 等），改用其他可信來源：官方文件、知名工程部落格（engineering blog）、論文等，並附上連結
- **不可僅憑訓練資料記憶講解** —— 每次要教新概念前，先用 WebFetch 實際抓取來源內容，再基於抓到的原文整理成中文教材
- 每篇教材/概念筆記結尾附上「參考來源」清單（連結 + 章節/段落標註）
- 若來源之間有衝突或版本差異，需明確標註並說明採用哪個版本的理由

## 課程架構

### Week 1-4：核心 6-8 題打底

概念與實戰並行——每天前半段學一個 building block，後半段直接應用在當週題目上。

| # | 題目 | 情境類型 | 核心 Building Blocks |
|---|------|----------|----------------------|
| 1 | URL Shortener | 讀多寫少 | Hashing、SQL vs NoSQL、DB indexing、Cache-aside |
| 2 | Rate Limiter | 基礎設施型 | Token bucket / Leaky bucket / Sliding window、Distributed rate limiting |
| 3 | Distributed Cache（design a Redis-like system） | 基礎設施型 | Consistent hashing、Eviction policy、Replication |
| 4 | Chat / Messaging System | 寫多讀少 | WebSocket、Message Queue、Delivery guarantee、Consistency model |
| 5 | News Feed / Twitter Timeline | 讀多寫少 + Fan-out | Fan-out on write vs read、Ranking、Caching strategy |
| 6 | Notification System | 非同步/整合型 | Push/Pull、第三方整合、Queue、Retry/Idempotency |

### Week 5-8：擴充 6 題廣度覆蓋 + 綜合實戰

| # | 題目 | 情境類型 | 核心 Building Blocks |
|---|------|----------|----------------------|
| 7 | Search Autocomplete | 搜尋型 | Trie、Ranking、Precomputation |
| 8 | Web Crawler | 批次處理型 | Distributed crawling、Dedup、Politeness/Rate control |
| 9 | Video Streaming（YouTube-like） | 讀多寫少 + 大檔案 | CDN、Transcoding pipeline、Storage tiering |
| 10 | Ride-sharing（Uber-like） | 地理位置型 | Geospatial indexing（Quadtree/Geohash）、Matching algorithm |
| 11 | Distributed Unique ID Generator | 基礎設施型 | Snowflake-like algorithm、Clock skew 處理 |
| 12 | Payment / Ticket Booking System | 強一致性型 | Idempotency、Distributed transaction、Optimistic vs Pessimistic locking |

清單依每次 mock interview 表現動態調整，非固定不變。

## Mock Interview 安排

- **Week 1-6**：每週 2-3 次 AI mock interview（我扮演面試官，45 分鐘 + debrief）
- **Week 7-8**：AI mock interview 持續進行（不減量）+ 額外安排真人 mock interview，兩者疊加、非取代
- 全程英文進行；針對溝通表達、用詞精準度、技術術語即時糾正
- 每次記錄存成 `issues/NN-<題目>.md`

## 8 週 Time Schedule

> 每週 Day 7 為彈性複習/補進度日。

### Week 1（URL Shortener）— 目標日期 2026-08-18 起
- Day 1：Primer 導讀 + Scalability/Availability/Consistency 基本原則、CAP theorem 複習
- Day 2：Load Balancing、DNS、Application layer → 套用於 URL Shortener 需求分析
- Day 3：Database 設計（SQL vs NoSQL、Sharding、Replication）→ URL Shortener DB schema
- Day 4：Caching 策略（Cache-aside/Write-through、CDN）→ 完成 URL Shortener 設計
- Day 5：**AI Mock Interview #1**（URL Shortener）
- Day 6：Debrief + Rate Limiter 概念導入（Token bucket / Leaky bucket / Sliding window）
- Day 7：複習/補進度

### Week 2（Rate Limiter + Distributed Cache）
- Day 1：Rate Limiter 演算法比較與 Distributed rate limiting 挑戰
- Day 2：**AI Mock Interview #2**（Rate Limiter）
- Day 3：Debrief + Consistent Hashing 概念
- Day 4：Cache eviction policy、Replication for cache → Distributed Cache 設計
- Day 5：**AI Mock Interview #3**（Distributed Cache）
- Day 6：Debrief + 弱點補強
- Day 7：複習/補進度

### Week 3（Chat / Messaging System）
- Day 1：WebSocket vs Long polling、Message Queue 基礎
- Day 2：Delivery guarantee（at-least-once/exactly-once）、Consistency model 選擇
- Day 3：**AI Mock Interview #4**（Chat System）
- Day 4：Debrief + 弱點補強
- Day 5：**AI Mock Interview #5**（Chat System 變化題：加群組/已讀回條）
- Day 6：Debrief + 綜合複習 Week 1-3 building blocks
- Day 7：複習/補進度

### Week 4（News Feed + Notification System）
- Day 1：Fan-out on write vs on read、Ranking 演算法概念
- Day 2：**AI Mock Interview #6**（News Feed）
- Day 3：Debrief + Notification System（Push/Pull、第三方整合、Retry/Idempotency）
- Day 4：**AI Mock Interview #7**（Notification System）
- Day 5：Debrief + Week 1-4 弱點總複習
- Day 6：**AI Mock Interview #8**（隨機重測 Week 1-4 任一題型的變化題）
- Day 7：複習/補進度 + Week 5-8 預告

### Week 5（Search Autocomplete + Web Crawler）
- Day 1：Trie 結構、Precomputation for ranking
- Day 2：**AI Mock Interview #9**（Search Autocomplete）
- Day 3：Debrief + Distributed crawling、Dedup 策略
- Day 4：**AI Mock Interview #10**（Web Crawler）
- Day 5：Debrief + 弱點補強
- Day 6：綜合複習
- Day 7：複習/補進度

### Week 6（Video Streaming + Ride-sharing）
- Day 1：CDN、Transcoding pipeline、Storage tiering
- Day 2：**AI Mock Interview #11**（Video Streaming）
- Day 3：Debrief + Geospatial indexing（Quadtree/Geohash）、Matching algorithm
- Day 4：**AI Mock Interview #12**（Ride-sharing）
- Day 5：Debrief + 弱點補強
- Day 6：綜合複習 Week 5-6
- Day 7：複習/補進度 + 開始安排真人 mock interview（Week 7-8）

### Week 7（Distributed ID Generator + Payment System ＋ 真人 mock 開始）
- Day 1：Snowflake-like algorithm、Clock skew 處理
- Day 2：**AI Mock Interview #13**（Distributed ID Generator）
- Day 3：Debrief + Idempotency、Distributed transaction、Locking 策略
- Day 4：**AI Mock Interview #14**（Payment/Booking System）
- Day 5：**真人 Mock Interview #1**
- Day 6：Debrief（AI + 真人回饋整合）
- Day 7：複習/補進度

### Week 8（最終衝刺）
- Day 1：**AI Mock Interview #15**（跨題型綜合，隨機挑選）
- Day 2：**真人 Mock Interview #2**
- Day 3：Debrief + 弱點集中複習
- Day 4：**AI Mock Interview #16**（模擬最可能出現的題型）
- Day 5：**真人 Mock Interview #3**（若可安排）
- Day 6：全面複習：building blocks cheat sheet、trade-off 對照表
- Day 7：面試前緩衝日（輕度複習，避免 burnout）

## 保存架構

- 課程主計畫（本檔案）：`.scratch/system-design-interview-prep/spec.md`
- 每次 mock interview 記錄與回饋：`.scratch/system-design-interview-prep/issues/NN-<題目>.md`（依 `docs/agents/issue-tracker.md` 慣例，Status 記錄 claimed/resolved）
- 核心概念詞彙表：`CONTEXT.md`（透過 `/domain-modeling` 建立，於第一個概念被釐清時建立，之後持續累積）

## Comments
