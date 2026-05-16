---
title: 系統設計關鍵數字
tags: [concept, fundamentals, numbers, performance]
sources: 1
updated: 2026-05-16
---

你不需要精準記得每個數字，但要有基本的量級直覺。這些數字最大的價值是**幫你避免 over-engineering**——最常犯的錯誤是在只有幾 TB 資料、幾千 QPS 時就急著說要 sharding、要微服務。

**Latency 順序**：memory << disk << network

## 快取（Caching）

Redis 不再只是 32GB～64GB 規模的小型快取，而是可以輕鬆支援數百 GB 甚至 TB 級的資料。

| 指標 | 數字 |
|------|------|
| 記憶體容量 | 最高可達 1TB（甚至更多的特殊配置） |
| 讀取延遲 | 同區域 < 1ms；跨區寫入 1–2ms |
| 吞吐量 | 單節點可支援 100k 以上讀取/秒，寫入達數十萬次/秒 |

**什麼情況下需要 Cache Sharding**：
- 資料集逼近 1TB
- 長期吞吐量超過 100k ops/sec
- 讀取 latency 需要穩定在 0.5ms 以下

## 資料庫（Databases）

現代資料庫的處理能力常常讓人意外。單一 PostgreSQL 或 MySQL 實例就能處理數十 TB 的資料，並保持毫秒級 latency。

| 指標 | 數字 |
|------|------|
| 儲存 | 64TiB（Aurora 可達 128TiB） |
| 讀取 latency | 快取讀取 1–5ms；磁碟讀取 5–30ms；commit latency 5–15ms |
| 吞吐量 | Aurora/RDS 單節點讀取可達 50k TPS；寫入 10–20k TPS |
| 連線數 | 5,000–20,000 |

**什麼情況下需要 DB Sharding**：
- 資料規模逼近 50TiB
- 寫入吞吐長期超過 10k TPS
- 未快取的讀取要求 < 5ms
- 需要跨區域部署
- 備份時間過長或不切實際

> **「良好調校」具體指什麼**？Sharding 之前有很多更簡單的優化手段：慢查詢 → 加 index / 優化 SQL；讀取瓶頸 → 加 cache → 加 read replica；寫入瓶頸 → 調 DB 參數 → 升級硬體。一個良好調校的單一資料庫，已經能支撐數百萬甚至上千萬用戶。

## 應用伺服器（Application Servers）

現代伺服器資源比過去豐富許多，讓許多傳統設計模式已經不再受限。

| 指標 | 數字 |
|------|------|
| 連線數 | 單機 100k+ |
| CPU | 8–64 核心 |
| 記憶體 | 64–512GB（最高可達 2TB） |
| 網路 | 25Gbps |
| 啟動時間 | Container apps 30–60 秒 |

**什麼情況下需要 Scale Out**：
- CPU 長期使用率超過 70–80%
- Latency 無法符合 SLA
- 記憶體使用率長期超過 70–80%
- 頻寬逼近 20Gbps

## 訊息佇列（Message Queues）

以 Kafka 為例，訊息佇列已經演進成高效能的資料高速公路。

| 指標 | 數字 |
|------|------|
| 吞吐量 | 每 broker 可達 1M 訊息/秒 |
| 延遲 | 1–5ms（同區域） |
| Message Size | 1KB–10MB |
| 儲存 | 單 broker 可達 50TB |
| Retention | 數週到數月 |

**什麼情況下需要 Kafka Sharding**：
- Throughput 逼近 800k messages/sec
- Partition Count 逼近 200k
- Consumer Lag 持續增加
- 需要 Cross-Region Replication

## 速查表（Cheat Sheet）

| 元件 | 關鍵指標 | 需要擴展的觸發條件 |
|------|---------|-----------------|
| 快取 (Redis) | Latency ~1ms；每秒 100k 以上操作；記憶體上限 1TB | hit rate < 80%；latency > 1ms；記憶體 > 80% |
| 資料庫 | 最高 50k TPS；cached read < 5ms；儲存 64TiB+ | 寫入 > 10k TPS；uncached read > 5ms；需跨區分布 |
| 應用伺服器 | 100k+ concurrent 連線；8–64 核心；64–512GB RAM | CPU > 70%；Response latency 超過 SLA |
| 訊息佇列 (Kafka) | 每 broker 每秒 100 萬訊息；latency < 5ms；50TB | Throughput > 800k/sec；partition > 200k |

## 面試中如何使用這些數字

**用「大概量級 + 推理」的方式**，不需要背誦精確數字：

- 「假設一台機器有幾十 GB memory，大概可以放幾億個這種 object」
- 「大概超過多少 TPS 才需要考慮 database sharding」

現代系統中真正的瓶頸多半是**每秒操作數（ops/sec）**或**網路頻寬**，而不是記憶體大小。這跟幾年前的狀況完全相反。

## See also

- [[可擴展性（Scalability）]] — 何時從垂直擴展轉向水平擴展
- [[快取（Caching）]] — Redis 的延遲和吞吐量數字的應用
- [[分片（Sharding）]] — DB sharding 的觸發條件
- [[複製（Replication）]] — Read replica 可以處理的讀取負載規模
