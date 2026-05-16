---
title: 複製（Replication）
tags: [concept, database, distributed-systems, availability, consistency]
sources: 1
updated: 2026-05-16
---

Replication 是把同一份資料的副本放在多台透過網路連接的機器上。目的是：**降低延遲**（把資料放在靠近用戶的地方）、**提升可用性**（某些節點失效時系統仍能繼續運作）、**擴展讀取吞吐量**（把讀取請求分散到多台機器上）。

真正的挑戰在於：當資料持續更新時，如何讓所有副本保持一致。

## 三種主要架構

### Single-Leader（單主節點）— 最常見

一台 replica 被指定為 **leader（主節點）**，也叫 master 或 primary。所有的寫入都必須先打到 leader。其他 replica 稱為 **follower（從節點）**，也叫 slave、secondary 或 read replica。Leader 把每次寫入包成 replication log，發送給所有 follower，follower 照順序套用這些變更。讀取可以打到 leader 或任何 follower，但寫入只能打到 leader。

被 PostgreSQL（9.0 以後）、MySQL、MongoDB、Kafka、RabbitMQ 等廣泛採用。

**同步 vs 非同步 replication**：

- **同步（Synchronous）**：Leader 在確認 follower 已收到並回報成功後，才通知 client 寫入完成。好處是 follower 保證有最新資料；壞處是一旦那台 follower 沒回應，整個系統的寫入就全部卡住了。
- **非同步（Asynchronous）**：Leader 把訊息發出去，不等 follower 確認就通知 client 成功。寫入更快，但如果 leader 在 follower 同步之前就掉了，那些寫入就永久消失了。
- **半同步（Semi-synchronous）**：保持一台 follower 同步、其他都非同步。這樣至少有兩份節點（leader + 一台 follower）保有最新資料。

> **面試考點**：同步與非同步 replication 的取捨是面試必考點。同步確保一致性但犧牲可用性；非同步確保可用性但有資料遺失風險。沒有哪個更好，要看你的系統對一致性和可用性的要求。

**Replication Lag 造成的三種問題**：

1. **Read-After-Write 不一致**：用戶發布了一則貼文，馬上去看，卻看不到，因為他的讀取打到了一台還沒同步的 follower。解法：讀自己可能改過的東西時，從 leader 讀；追蹤用戶最後一次寫入的位置，只接受同步位置已追上這個標記的 replica。

2. **Monotonic Reads（單調讀取）不一致**：用戶先從一台 lag 小的 follower 讀到了朋友的留言，然後刷新頁面，被路由到一台 lag 大的 follower，留言消失了，看起來像時光倒流。解法：確保同一個用戶的讀取永遠打到同一台 replica（用用戶 ID 的 hash 來決定要讀哪台）。

3. **Consistent Prefix Reads（因果一致性）不一致**：A 問了一個問題，B 回答了。第三個觀察者因為 replication lag 不同，先看到 B 的回答，才看到 A 的問題，因果順序顛倒了。解法：確保有因果關係的寫入都寫到同一個 partition。

**Leader Failover（領導者故障切換）**：

Leader 掌摔比較棘手：必須把某台 follower 提升為新的 leader，讓 client 把寫入打去新 leader。步驟：偵測 leader 失效（timeout）→ 選出新 leader（多數 replica 投票）→ 重新設定系統。

Failover 的地雷：
- **Split Brain**：如果兩個節點都以為自己是 leader，兩邊都接受寫入，資料可能損毀
- **資料遺失**：非同步 replication 下，新 leader 可能還沒收到舊 leader 最後的一些寫入，這些寫入通常會被丟棄
- **Timeout 的長短很難拿捏**：太長則失效後恢復慢；太短則可能在系統本來就很忙的時候不必要地觸發 failover

### Multi-Leader（多主節點）

讓多個節點都能接受寫入。每個 leader 同時也是其他 leader 的 follower，把自己的寫入複製給其他所有 leader 和它自己的 follower。

**適用場景**：多 datacenter 部署（每個 datacenter 有自己的 leader，用戶寫入到最近的 datacenter）、離線操作（手機日曆 App 在沒有網路時也能繼續新增行程）、協作編輯（Google Docs）。

**最大挑戰：寫入衝突**。如果用戶 A 和用戶 B 同時在不同的 leader 修改同一份資料，都成功了，但這兩個寫入在非同步複製時就會衝突。

**衝突解法**：
- **LWW（Last Write Wins）**：每個寫入附上 timestamp，挑最大的當勝者，丟棄其他寫入。簡單但丟失資料。這是 Cassandra 唯一支援的衝突解法。
- **衝突迴避**：確保特定記錄的所有寫入走同一個 leader（例如把同一個用戶的請求永遠路由到同一個 datacenter）。
- **記錄衝突、稍後解決**：保留所有衝突版本，等應用程式或用戶決定怎麼解決。
- **CRDT（Conflict-free Replicated Datatypes）**：可以被多人同時編輯、並自動以合理方式解決衝突的資料結構（集合、計數器、有序列表等）。

### Leaderless（無主節點）

完全拋棄 leader 的概念，讓任何 replica 都能直接接受寫入。Amazon 的內部 Dynamo 系統讓這個架構重新流行起來，Riak、Cassandra、Voldemort 都受到這個設計啟發。

**Quorum Reads 和 Writes**：假設有 n 個 replica，寫入需要 w 個節點確認，讀取需要查詢 r 個節點。只要 **w + r > n**，讀取時至少有一個節點有最新資料。常見配置：n=3, w=2, r=2（容忍一個節點失效）。

**讓 Stale Replica 追上進度**：
- **Read Repair**：Client 並行讀多台 replica，發現某台的 version 比較舊，就把新值寫回那台
- **Anti-entropy process**：背景程序持續掃描 replica 間的差異，把缺少的資料複製過去

**Sloppy Quorum 和 Hinted Handoff**：如果原本應該存這份資料的 n 台「家」節點連不到，但還連得到其他節點，可以接受寫入暫存到現在連得到的節點上（不在原本的 n 台裡）。網路恢復後，這些臨時保管的寫入被送回「家」節點，這個過程叫 **hinted handoff**。Sloppy quorum 提升了寫入可用性，但代價是即使 w+r>n，也無法保證讀到最新值，因為最新值可能在那些「臨時保管」的節點上。

## 面試中什麼時候用哪種架構

| 情況 | 建議架構 |
|------|---------|
| 讀取密集，需要擴展讀取 | Single-Leader + read replica（follower） |
| 需要高可用（服務不能停） | Single-Leader + automatic failover |
| 多 datacenter 部署 | Multi-Leader（每個 DC 有自己的 leader） |
| 高可用寫入 + 可接受最終一致性 | Leaderless（Cassandra quorum 模型） |

> **主動帶進議題**：厲害的面試者不會等到被問才提 replication。在設計讀取密集系統時，主動說：「這個讀取量很高，我會加 read replica 來分散負載，不過需要討論 replication lag 的問題。」

## See also

- [[CAP 定理（CAP Theorem）]] — Replication lag 本質上是 CAP 的 A vs C 取捨
- [[分片（Sharding）]] — Sharding 擴展寫入；Replication 擴展讀取。兩者常一起使用
- [[資料庫交易（Database Transactions）]] — 分散式 transaction 在 replication 環境下的挑戰
- [[系統設計關鍵數字]] — Read replica 的 replication lag 通常在 100ms 以內
