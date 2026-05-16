---
title: 資料庫交易（Database Transactions）
tags: [concept, database, consistency, acid]
sources: 1
updated: 2026-05-16
---

Transaction 把一組相關的資料庫操作包成一個邏輯單元，保證它們作為一個整體被執行：要麼全部成功，要麼全部失敗。沒有 transaction，任何一步失敗都會讓資料處於不一致的狀態。

## ACID：四個保證

**A — Atomicity（原子性）**：一個 transaction 內的所有操作，要麼全部成功，要麼全部失敗，不存在中間狀態。透過 WAL（Write-Ahead Log）實現：先記錄「打算做什麼」，如果中途崩潰，重啟後根據 log 判斷繼續完成還是 rollback。

**C — Consistency（一致性）**：Transaction 執行完後，資料庫必須從一個合法狀態轉移到另一個合法狀態。所有你定義的資料完整性規則（constraints）都必須在 transaction 開始前和結束後成立。

> **注意**：ACID 的 C 和 [[CAP 定理（CAP Theorem）]] 的 C 是完全不同的概念。ACID 的 C 指的是資料的「商業邏輯層面的正確性」；CAP 的 C 指的是「所有節點在同一時間看到相同資料」。

**I — Isolation（隔離性）**：同時執行的多個 transactions，彼此之間不應該互相干擾。每個 transaction 看起來就像是在一個沒有其他 transaction 存在的環境下獨立執行。這是四個屬性裡最有取捨空間的。

**D — Durability（持久性）**：一旦 transaction commit 成功，資料就永久保存，即使系統立刻崩潰也不會遺失。

## 隔離等級（Isolation Levels）

| 隔離等級 | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| Read Uncommitted | 可能發生 | 可能發生 | 可能發生 |
| **Read Committed** | 防止 | 可能發生 | 可能發生 |
| Repeatable Read | 防止 | 防止 | 可能發生 |
| Serializable | 防止 | 防止 | 防止 |

- **PostgreSQL 預設**：Read Committed
- **MySQL InnoDB 預設**：Repeatable Read

## 三種並發異常

**Dirty Read（髒讀）**：Transaction A 讀取了 Transaction B 尚未 commit 的資料。如果 B 之後 rollback，A 就讀到了一份「從未真正存在」的資料。

**Non-repeatable Read（不可重複讀）**：同一個 transaction 內，兩次讀取同一筆資料，但兩次得到不同的結果，因為另一個 transaction 在這期間 commit 了對那筆資料的更新。

**Phantom Read（幻讀）**：同一個 transaction 內，兩次執行同樣的查詢（通常是範圍查詢），但第二次多出了新的資料行，因為另一個 transaction 在這期間 insert 了新資料。

## MVCC：現代資料庫如何實現隔離

資料庫不靠「讓每個 transaction 等前一個完成」來實現隔離（那樣效能太差）。現代資料庫（PostgreSQL、MySQL InnoDB）普遍使用 **MVCC（Multi-Version Concurrency Control，多版本並行控制）**。

核心思想：**對同一份資料保存多個版本**。讀操作看的是資料在 transaction 開始時的「快照」，而不是最新版本，所以讀操作不需要等待寫操作完成，寫操作也不會被讀操作擋住。

## 常見陷阱：Lost Update（更新遺失）

當兩個 transaction 同時讀取同一筆資料、各自計算新值、然後都寫回去時，後寫入的 transaction 會覆蓋前一個，讓前一個的更新完全消失。

**解法一：樂觀鎖（Optimistic Locking）**：在更新時帶上版本號，如果版本號不符就拒絕更新。適合衝突不常發生的場景。

```sql
UPDATE accounts SET balance = 800, version = 2
WHERE id = 1 AND version = 1;  -- 如果 version 已被改掉，影響 0 筆
```

**解法二：悲觀鎖（Pessimistic Locking）**：讀取時就把資料鎖住，阻止其他 transaction 讀取或修改，直到當前 transaction 完成。

```sql
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 加排他鎖
UPDATE accounts SET balance = 800 WHERE id = 1;
COMMIT;
```

悲觀鎖較安全，但降低了並行性，也可能引發 **Deadlock（死鎖）**。

## 分散式 Transaction 的挑戰

傳統 ACID transaction 無法跨越服務邊界。當系統拆分成微服務、資料分散在多個資料庫中，有兩種主要解法：

**Two-Phase Commit（2PC，兩階段提交）**：由一個協調者確保所有參與節點都同意 commit。問題：如果協調者在第二階段崩潰，部分節點可能永遠卡在「已 prepare 但未 commit」的狀態，需要人工干預。大多數生產系統避免 2PC。

**Saga Pattern**：把一個分散式 transaction 拆成一系列小的、有補償機制的本地 transaction。如果某步驟失敗，就執行對應的**補償動作**來撤銷已完成的工作。Saga 放棄了強一致性，用**最終一致性**換取更好的可用性和容錯能力。這是現代微服務面對分散式 transaction 的主流選擇。

```
建立訂單流程：
1. 訂單服務：建立訂單記錄（本地 commit）
2. 庫存服務：扣減庫存（本地 commit）
3. 付款服務：扣款（本地 commit）

如果步驟 3 失敗：
- 補償步驟 2：把庫存加回去
- 補償步驟 1：把訂單標記為取消
```

## 面試中如何談 Transaction

- 不要只說「我用 transaction」，要說清楚選哪個隔離等級，以及為什麼
- 庫存扣除、超賣的本質是 **Lost Update**，最簡單的解法是原子 UPDATE：`UPDATE SET qty = qty - 1 WHERE qty > 0`，不需要提升到 Serializable
- 微服務場景：主動說明跨服務 transaction 的挑戰，並說你會用 **Saga 模式**
- 如果用了悲觀鎖，要主動說明 deadlock 風險和解法

## See also

- [[CAP 定理（CAP Theorem）]] — ACID 的 C 和 CAP 的 C 是完全不同的概念
- [[分片（Sharding）]] — Sharding 打破了單一 DB transaction，需要 Saga 或設計來避免跨 shard transaction
- [[冪等性（Idempotency）]] — 重試策略需要配合冪等性，避免補償動作的重複執行
