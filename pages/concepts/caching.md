---
title: 快取（Caching）
tags: [concept, performance, scalability, redis]
sources: 1
updated: 2026-05-16
---

快取是當資料庫讀取太慢或太昂貴時的解法。從 Postgres 讀一個用戶個人資料可能需要 50 毫秒，但從 Redis 這樣的記憶體快取讀取只需要 1 毫秒，足足快了五十倍。

**面試預設答案**：高流量系統的快取 = Redis 做外部快取（External Caching）。

## 快取的位置

**外部快取（External Caching）**：一個獨立的快取服務（Redis/Memcached），所有 application server 共用同一個快取。這是面試預設答案。支援 LRU 淘汰和 TTL。

**CDN**：把靜態內容快取在靠近用戶的地方。沒有 CDN，每個圖片請求都要跑到你的後端伺服器；有了 CDN，從附近的邊緣伺服器傳遞只需要 20-40 毫秒（vs 直連 origin server 250-300ms）。

**客戶端快取（Client-Side Caching）**：瀏覽器（HTTP cache、localStorage）或行動 App 的本地快取。後端控制程度有限。

**行程內快取（In-Process Caching）**：直接在 application 行程內快取資料，零網路開銷。適合設定值、feature flag、小型參考資料集、熱門 key。**限制**：每個 application instance 有自己的快取，不在各台伺服器之間共享。在面試中只在介紹外部快取之後，才提這個作為優化層。

**延遲順序**：In-Process Cache（零網路） << External Cache (Redis <1ms) << CDN (20-40ms) << origin server (250-300ms)

## 四種快取架構模式

### Cache-Aside（旁載快取）— 預設首選

這是最常見的快取模式，面試預設使用的那個。

1. Application 先查快取
2. 如果資料在，直接回傳（cache hit）
3. 如果不在，從資料庫取，存入快取，再回傳（cache miss）

**如果你只記得一種快取模式，記 cache-aside。**

### Write-Through（同步寫穿）

Application 寫入快取，快取再同步地把資料寫到資料庫，整個寫入操作要等到快取和資料庫都更新完才算完成。適用場景：讀取必須永遠回傳新資料，而且系統可以接受稍微慢一點的寫入。

### Write-Behind（非同步回寫）

Application 只寫入快取，快取在背景非同步地把資料批次寫到資料庫。寫入非常快，但引入了風險：如果快取在把資料 flush 到資料庫之前崩潰，資料就遺失了。適用場景：需要高寫入吞吐量，且可以接受最終一致性（分析和 metrics pipeline）。

### Read-Through（讀穿快取）

快取充當智慧代理層，application 不直接和資料庫溝通。當 cache miss 發生，是由快取本身去資料庫取資料、存起來，再傳給 application。CDN 本質上是一種 read-through 快取。在系統設計面試中很少有理由主動提出這個模式（除非在討論 CDN）。

## 快取淘汰策略（Eviction Policy）

**LRU（Least Recently Used）**：移除最久沒有被存取的項目。大多數系統的預設選擇，適合「最近用過的資料很可能再次被用」的工作負載。

**LFU（Least Frequently Used）**：移除被存取次數最少的項目。適合某些 key 長期持續熱門的場景（如熱門影片或排行榜播放清單）。

**FIFO（First In First Out）**：只根據插入時間移除，忽略使用模式，可能移除還在被熱用的項目。除了簡單的快取層，在真實系統中很少使用。

**TTL（Time To Live）**：不是淘汰策略，而是為每個 key 設定過期時間。通常和 LRU 或 LFU 搭配使用，在資料新鮮度和記憶體用量之間取得平衡。只要資料必須最終刷新（如 API 回應或 session token），TTL 就是必備的。

## 常見快取問題

### 快取雪崩（Cache Stampede / Thundering Herd）

當一個熱門的快取項目過期，大量請求同時嘗試重建它，就發生了快取雪崩。原本一個查詢，瞬間變成幾百甚至幾千個，可能把資料庫打垮。

**解法：**
- **請求合併（Request coalescing / Single flight）**：只讓一個請求去重建快取，其他請求等待那個結果。這是最有效的解法。
- **快取預熱（Cache warming）**：在熱門 key 過期之前主動刷新。

### 快取一致性（Cache Consistency）

快取和資料庫對同一份資料回傳不同值的時候就發生了不一致。因為大多數系統從快取讀取，但先寫到資料庫，這製造了一個快取還拿著舊資料的窗口。

**處理策略：**
- **寫入時失效快取**：更新資料庫後刪除快取項目，讓下次讀取時以新資料填充快取
- **短 TTL 容許過時資料**：如果可以接受最終一致性，就讓稍微過時的資料暫時存在
- **接受最終一致性**：對 feed、動態牆、通知列表、metrics 而言，短暫的延遲通常沒問題

### 熱 Key（Hot Keys）

熱 key 是一個比其他所有項目收到多得多流量的快取項目。就算整體 cache hit rate 很高，單一熱 key 也可能讓某個快取節點或某個 Redis shard 過載，變成瓶頸（例如 Taylor Swift 的個人資料 key）。

**解法：**
- **複製熱 key**：把同樣的值存在多個快取節點上，把讀取負載分散出去（注意：副本的過期時間要隨機錯開，避免同時過期造成快取雪崩）
- **加行程內備援快取**：把極端熱門的值存在 application 行程記憶體中，避免一直打 Redis

## 面試中怎麼談快取

不要一上來就直接說快取——你需要先確立為什麼有必要用它：

1. **確認瓶頸**：指出快取要解決的具體問題（資料庫負載、查詢延遲、昂貴的計算）
2. **決定快取什麼**：讀取頻繁、不常變動、而且取得或計算成本昂貴的資料
3. **選擇快取架構**：預設 cache-aside；靜態媒體用 CDN；極端熱 key 可考慮行程內快取
4. **設定淘汰策略**：LRU 是穩妥的預設，TTL 是防止資料過時的必備手段
5. **說明缺點**：快取失效、快取故障時怎麼辦、快取雪崩

> 最重要的是，不要快取所有東西。展示你知道什麼時候快取值得那份複雜度，什麼時候一個 index 設計良好的資料庫就已經夠了。

## See also

- [[資料庫索引（Database Indexing）]] — 快取之前先考慮的優化（更簡單）
- [[一致性雜湊（Consistent Hashing）]] — 分散式快取 sharding 的路由機制
- [[CDN 與地理延遲]] — CDN 是 read-through 快取的一種形式
- [[熔斷器（Circuit Breaker）]] — 如果 Redis 掛了，circuit breaker 可以降級到直查資料庫
- [[系統設計關鍵數字]] — Redis 的具體延遲和吞吐量數字
