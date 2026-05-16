---
title: 分片（Sharding）
tags: [concept, database, scalability, distributed-systems]
sources: 1
updated: 2026-05-16
---

Sharding 是當單台 database 無法應付你的規模時，把資料拆散到多台機器上的做法，藉此提升儲存容量和吞吐量。**不要過早 shard**——先確認單台 DB 真的無法應付，再來討論 sharding。

「Partitioning」和「Sharding」常被混用。嚴格來說：Partitioning 通常指在單一 database instance 內拆分資料；Sharding 是把資料分散到多台機器上。實際上不用糾結措辭，只要說清楚你的資料是放在一台機器上還是多台就好。

## 什麼時候才需要 Sharding

在討論容量規劃時碰到以下其中一個限制，就可以帶出 sharding：

- **儲存空間**：資料規模逼近 50TiB（單台 Postgres instance 已難應付）
- **寫入吞吐量**：長期寫入超過 10k TPS（replica 幫不上忙，因為寫入只走 primary）
- **讀取吞吐量**：就算有 read replica，服務一億日活用戶每人多個查詢，讀取負載還是需要分散到多台 shard

> **公式**：確認瓶頸 → 解釋為什麼單台 database 無法擴展 → 提出 sharding

面試中到目前為止最常見的 sharding 錯誤，就是在還沒證明必要性之前就引入 sharding。慢下來，做算術，確認 sharding 真的是必要的，再開始解釋你怎麼做。

## 選擇 Shard Key

好的 shard key 需要具備以下條件：

**高基數（High Cardinality）**：key 要有很多不同的值。用布林欄位（true/false）分片最多只能有兩個 shard，根本沒有意義。

**均勻分布（Even Distribution）**：值要能均勻分散到各個 shard 上。用國家分片，而 90% 的用戶在美國，那個 shard 就會比其他 shard 大得多。

**契合查詢模式（Aligns with Queries）**：你最常見的查詢最好只打到一台 shard。如果用 `user_id` 分片，「取得用戶個人資料」或「取得用戶訂單」這類查詢都只打一台 shard。跨所有 shard 的查詢很昂貴。

**好的 shard key 例子**：`user_id`（用戶導向 App）、`order_id`（電商訂單表）
**爛的 shard key 例子**：`is_premium`（布林值）、`created_at`（所有新寫入都到最新 shard，造成熱點）

## 三種 Sharding 策略

### 雜湊分片（Hash-Based）— 預設首選

用雜湊函數把 shard key 均勻分散到各個 shard 上。最大優點是分布均勻。

缺點：增減 shard 時，如果用簡單的 `hash % N`，幾乎所有資料都需要搬移。解法是使用 [[一致性雜湊（Consistent Hashing）]]，把資料搬移量降到最低。

面試預設選擇 Hash-Based（除非你特別說明用其他策略）。

### 範圍分片（Range-Based）

把記錄依照連續的值範圍分組（例如 User ID 1～100 萬 → Shard 1，100～200 萬 → Shard 2）。最大優點是簡單，而且支援高效的範圍掃描。

缺點：容易產生熱點。如果用 `created_at` 分片，幾乎所有流量都會打到最新的 shard。

### 目錄分片（Directory-Based）

用一張查找表來決定每筆記錄放在哪裡（`user_to_shard` mapping table）。強項是靈活性：如果某個用戶產生大量流量，可以把他搬到專屬的 shard。

缺點：每個請求都需要一次額外查找；目錄服務成了關鍵依賴點，如果目錄掛掉，整個系統就停擺。面試中很少是正確答案。

## Sharding 的挑戰

### 熱點與負載不均（Hot Spots）

就算有好的 shard key，某些 shard 還是可能收到遠多於其他 shard 的流量（celebrity problem）。如果用 `user_id` 分片，Taylor Swift 那個 shard 的流量可能是普通用戶的一千倍。

**應對方式**：
- **把熱 key 隔離到專屬 shard**：把名人帳號搬到只處理名人帳號的專屬 shard（這是目錄分片有其用武之地的原因）
- **使用複合 shard key**：不只用 `user_id`，而是結合另一個維度，例如 `hash(user_id + date)`

### 跨 Shard 操作

當資料分散在多台機器上，任何需要從多個 shard 取資料的查詢都會變得昂貴。例如「取得全站最熱門的 10 篇貼文」就必須查詢每一個 shard 並聚合結果。

**解法**：
- **快取結果**：對跨 shard 的聚合查詢結果做快取（可以容忍稍微過時的資料）
- **反正規化（Denormalization）**：讓相關資料放在一起，如果常常需要同時查詢貼文和用戶資料，可以把一些貼文資訊直接存在用戶的 shard 上
- **接受罕見查詢的代價**：如果跨 shard 查詢不常發生，就接受它跑慢一點

> 頻繁的跨 shard 查詢通常是設計訊號，代表 shard key 選錯了或 shard 的邊界劃分有問題。

### 維護一致性

Sharding 打破了單一 DB transaction。如果用戶帳戶在 shard 1，交易記錄在 shard 2，你就不能用單一 database transaction 了。

**最佳解法**：設計成避免跨 shard transaction。如果用 `user_id` 分片，就把一個用戶的所有資料放在他的 shard 上，這樣所有 transaction 都是單 shard 的，快又可靠。

**如果真的無法避免**：用 Saga 模式（見 [[資料庫交易（Database Transactions）]]）。

## 現代資料庫中的 Sharding

你大概不會從頭實作 sharding。現代的分散式資料庫大多自動處理：
- **Cassandra**：consistent hashing + virtual nodes
- **DynamoDB**：對 partition key 做雜湊，自動拆分和合併分區
- **MongoDB**：範圍為基礎的 chunk，balancer 自動維持平衡
- **Vitess/Citus**：架在 MySQL/PostgreSQL 前面的 sharding 層

## See also

- [[可擴展性（Scalability）]] — Sharding 是資料庫水平擴展的手段
- [[一致性雜湊（Consistent Hashing）]] — Hash-Based sharding 的路由機制，避免大規模資料搬移
- [[複製（Replication）]] — Sharding 擴展寫入；Replication 擴展讀取、提高可用性
- [[資料庫交易（Database Transactions）]] — Sharding 打破了 ACID transaction，需要 Saga 模式
- [[系統設計關鍵數字]] — 什麼規模才真正需要 sharding
