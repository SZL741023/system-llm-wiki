---
title: 冪等性（Idempotency）
tags: [concept, api, reliability, fault-tolerance]
sources: 1
updated: 2026-05-16
---

冪等 API 可以被呼叫多次，每次都產生相同的結果。這是安全重試的前提——沒有冪等性，重試可能造成重複副作用（例如重複扣款）。

## HTTP 方法與冪等性

| 方法 | 冪等？ | 說明 |
|------|--------|------|
| GET | ✅ 是 | 獲取內容的行為本身不會改變系統狀態 |
| PUT | ✅ 是 | 多次呼叫結果相同 |
| DELETE | ✅ 是 | 刪除已刪除的東西結果相同 |
| PATCH | 視實作 | 取決於實作方式 |
| **POST** | ❌ **否** | 每次呼叫可能建立新資源 |

> **面試考點（Q2）**：以下哪個 HTTP method 不是冪等的？**(A) GET (B) PUT (C) DELETE (D) POST** → 答案是 **(D) POST**。

## 寫入操作的冪等性：冪等鍵（Idempotency Key）

對於寫入資料的場景，常見的做法是在 API 引入一個**冪等鍵（idempotency key）**。

**例子（支付場景）**：如果我們知道用戶每天只會購買一件商品，可以用「用戶 ID + 當天日期」組成冪等鍵。在伺服器端，我們檢查是否已經處理（或正在處理）帶有該冪等鍵的請求，只處理一次。這讓你避免了重複扣款的問題。

## 為什麼重要

[[重試與指數退避（Retry with Exponential Backoff）|重試]]是處理暫時性故障的標準做法，但重試只有在 API 冪等的情況下才安全。如果支付 API 不冪等，網路超時後的重試就可能導致重複收費。

## See also

- [[重試與指數退避（Retry with Exponential Backoff）]] — 重試的前提是 API 冪等
- [[HTTP/HTTPS]] — HTTP 方法的冪等性規則
- [[熔斷器（Circuit Breaker）]] — 和重試、冪等性構成完整的故障處理策略
