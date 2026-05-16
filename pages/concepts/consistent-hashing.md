---
title: 一致性雜湊（Consistent Hashing）
tags: [concept, distributed-systems, scalability, hashing]
sources: 1
updated: 2026-05-16
---

一致性雜湊解決的是「當節點數量改變時，如何最小化需要搬移的資料量」的問題。在分散式系統中，資料通常需要分散到多台伺服器上。

## 問題：簡單雜湊的缺陷

最直觀的做法是 `server = hash(key) % N`（N = 節點數）。

**問題**：如果 N 改變（新增或移除伺服器），幾乎所有資料都需要重新分配。

例：N=3 時有 9 個值，N 變成 4 時，其中 7 個值的位置都改變了，需要大量搬遷，造成效能問題與系統不穩定。

## 解法：Hash 環（Hash Ring）

1. 把所有可能的 hash 值映射到一個圓環上（0 → MAX_HASH）
2. 每個節點（server）用 hash 函數映射到圓環上的一個位置
3. 每筆資料也用 `hash(key)` 計算位置
4. **規則**：資料順時針找到第一個節點，交給它存

**加入新節點 E（落在位置 150）**：
- 只有「D 與 E 之間」（0～150）的資料，從 A 搬到 E
- 其他資料**不受影響**

**移除節點 A**：
- 原本屬於 A 的資料移到下一個節點 B
- 其他資料**不受影響**

> **核心價值**：當伺服器數量改變時，只需要搬動一小部分資料（平均 k/n，k 為 key 數，n 為節點數），大部分資料仍然留在原來的節點。

## 虛擬節點（Virtual Nodes）

基本版的問題：
1. **資料分布不均**：節點少時，hash 環上的位置可能集中，導致某些節點負擔過重
2. **節點性能不同**：有的伺服器效能強、有的弱，但一樣只拿到一個區間

**解法**：每個實體節點對應到多個「虛擬節點（vNodes）」，vNodes 也會映射到 hash 環上，就像多個小節點。

**虛擬節點的好處**：
1. **平均分布**：即使節點數少，也能把資料更均勻地切開
2. **依硬體能力分配**：效能較強的機器可以放更多 vNodes，負責更多資料
3. **彈性擴展**：當節點加入/移除時，因為每個節點有多個 vNodes，搬遷的資料更細粒化，負載更平滑

## 面試中什麼時候用 Consistent Hashing？

記憶口訣：**「變動的節點 + 需要穩定歸屬 + 想少搬家」→ 用 consistent hashing**

具體場景：
- **分散式快取 sharding**：Memcached/Redis Sharding（避免 `key % N` 帶來的大規模重分配）
- **Storage sharding/routing**：Cassandra/Dynamo 風格的 partition routing
- **Sticky connections**：聊天室/即時服務將同一使用者或同一聊天室路由到固定節點
- **API Gateway / Sticky Sessions**：減少跨節點 session 同步
- **Rate Limiting**：把 key（user_id、api_key）穩定地分到對應節點做限流統計
- **Metrics / Aggregation**：將相同維度聚到固定節點做累加，降低跨節點匯總成本
- **CDN/Edge Routing**：把 URL 或內容 ID 穩定映射到 edge 節點做快取

## See also

- [[分片（Sharding）]] — Consistent hashing 是 Hash-Based sharding 的解法（避免 resharding 時的大量資料移動）
- [[負載平衡]] — Redis Cluster 使用 client-side LB + consistent hashing
- [[快取（Caching）]] — 分散式快取的 sharding 是 consistent hashing 最常見的應用場景
