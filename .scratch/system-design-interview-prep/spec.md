# System Design Interview Prep — 8 週衝刺計畫

Status: active

## 目前進度

> 每次上課/mock interview 後更新此區塊，作為進度的唯一權威來源（換 agent session 時，先讀這裡再繼續）。

- **目前所在**：尚未開始，準備進入 Week 1 Day 1
- **已完成的 Mock Interview**：無
- **累積弱點清單**：無
- **最後更新**：2026-08-18

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
