---
title: HTTP/HTTPS
tags: [component, networking, application-layer, protocol]
sources: 1
updated: 2026-05-16
---

HTTP 是網路上資料通訊的事實標準，是一個請求-回應協定，建立在 [[TCP vs UDP|TCP]] 之上。HTTPS 在 HTTP 之上加了 TLS/SSL 加密層。

## HTTP 核心概念

HTTP 是**無狀態**協定——每個請求都是獨立的，伺服器不需要維護先前請求的資訊。這是件好事：讓 HTTP 伺服器可以被描述為請求參數的函式，天然無狀態，易於水平擴展。

### 請求方法

| 方法 | 用途 | 冪等？ |
|------|------|--------|
| GET | 取得資源 | ✅ 是 |
| PUT | 更新/替換資源 | ✅ 是 |
| DELETE | 刪除資源 | ✅ 是 |
| PATCH | 部分更新資源 | 視實作而定 |
| POST | 建立新資源 | ❌ 否 |

> **面試考點**：GET、PUT、DELETE 是冪等的；**POST 不是**。詳見 [[冪等性]]。

### 常見狀態碼

- **2xx 成功**：200 OK、201 Created
- **3xx 重定向**：301 Moved Permanently、302 Found
- **4xx 客戶端錯誤**：400 Bad Request、401 Unauthorized、403 Forbidden、404 Not Found、429 Too Many Requests
- **5xx 伺服器錯誤**：500 Server Error、502 Bad Gateway

### Headers

Headers 是鍵值對（key-value），非常靈活。值得學習的設計哲學：`Accept-Encoding` 讓客戶端宣告它能處理的編碼方式，伺服器用最有效率的方式（gzip、brotli）回應，同時保持後向相容性。

## HTTPS

在 HTTP 之上加了 TLS/SSL 安全層，加密通訊，防止竊聽和中間人攻擊。公開網站毫無例外都要用 HTTPS。

> **重要安全提醒**：HTTPS 加密的是傳輸中的內容，但不代表請求來自合法客戶端。API 在沒有驗證的情況下，永遠不應信任 request body 的內容（例如 body 裡的 user ID）。攻擊者可以修改 body 來竊取其他用戶的資料。

## HTTP/2 的多工（Multiplexing）

HTTP/2 支援多路複用，讓多個請求共享同一個 TCP 連線，避免每次請求都重新建立連線的開銷。gRPC 就建立在 HTTP/2 之上。

## See also

- [[REST vs GraphQL vs gRPC]] — 建立在 HTTP 之上的三種 API 風格
- [[SSE 與 WebSocket]] — HTTP 的即時推送擴展
- [[TCP vs UDP]] — HTTP 建立在 TCP 之上
- [[負載平衡]] — L7 LB 理解 HTTP，L4 LB 不理解
- [[冪等性]] — HTTP 方法冪等性與安全重試的關係
