---
title: REST vs GraphQL vs gRPC
tags: [comparison, api, networking, application-layer]
sources: 4
updated: 2026-05-16
---

三種主要的 API 設計範式，面試預設選 REST，除非有明確理由選其他。

## 快速對照

| | REST | GraphQL | gRPC |
|--|------|---------|------|
| 基礎協定 | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| 資料格式 | JSON | JSON | Protocol Buffers（二進位） |
| 吞吐量 | 基準 | 基準 | **約 10x** |
| 型別系統 | 無 | Schema | Proto 定義（強型別） |
| 瀏覽器支援 | ✅ 原生 | ✅ 原生 | ❌ 需要 grpc-web |
| 適用場景 | 預設 / 外部 API | 前端靈活查詢 | 內部微服務 |

## REST：簡單且靈活

核心原則：客戶端對「資源（resource）」執行簡單的操作。資源對應到資料庫表格或伺服器上的物件。

```
GET    /users/{id}        → User
PUT    /users/{id}        → User
POST   /users             → User（新建）
GET    /users/{id}/posts  → [Post]
```

**什麼時候用**：幾乎所有情況的預設選擇。除非有 REST 無法滿足的特定需求，才考慮 GraphQL、gRPC、SSE 或 WebSocket。

> **設計提醒**：要以「資源」而非「操作」思考。`updateUser` 應變成 `PUT /users/{id}`；`startGame` 應變成 `PATCH /games/{id}` 加上 `{ "status": "started" }`。

### 如何傳遞資料給 API

| 方式 | 用途 | 例子 |
|------|------|------|
| **Path Parameter** | 標示唯一資源 | `GET /users/123` |
| **Query Parameter** | 篩選、排序、分頁等查詢條件 | `GET /users?age=30&sort=desc&page=2` |
| **Request Body** | 傳送結構化資料，用於 POST、PUT、PATCH | `POST /users { "name": "Bohr" }` |

### 分頁回應設計

```json
{
  "data": [...],
  "page": 2,
  "page_size": 20,
  "total_pages": 10
}
```

## GraphQL：彈性的資料獲取

Facebook 2015 年開源，讓客戶端能夠請求它精確需要的資料，解決前後端分屬不同團隊時的協作問題。

**解決的問題：**
- **欠獲取（Under-fetching）**：需要多個請求才能取得所有資料
- **過獲取（Over-fetching）**：回應包含太多不需要的資料

GraphQL 支援三種操作：**Query**（讀取）、**Mutation**（新增/更新/刪除）、**Subscription**（即時推送，類似 WebSocket/SSE）。

**N+1 問題**：因為 resolver 是 field-based 設計，巢狀查詢很容易造成 N+1 問題（查一組 users，再對每個 user 各查一次 orders）。解法是使用 DataLoader 做 batch loading。

**快取困難**：GraphQL 所有查詢都走同一個 endpoint `/graphql`，不同 query body 產生不同結果，無法直接用 URL 當快取 key（不像 REST 可以直接用 CDN/瀏覽器 HTTP caching）。

**什麼時候用**：前端團隊需要快速迭代、靈活調整的場景，以及多個團隊對重疊資料進行廣泛查詢的情況。

> **面試注意**：在系統設計面試中，GraphQL 的好處其實比較模糊——面試需求是固定的（不像真實的迭代開發），而 GraphQL 的後端執行有時會引入複雜的 N+1 問題。建議只在問題明顯聚焦於靈活性時才提出 GraphQL。

## gRPC：高效能內部通訊

Google 推出的高效能 RPC 框架，使用 HTTP/2 和 Protocol Buffers。

**為什麼快**：Protocol Buffers 是二進位格式，比帶有嵌入式綱要的 JSON 小很多（同樣資料約 15 bytes vs 40 bytes），解析 CPU 消耗也更少，吞吐量約為 JSON over HTTP 的 10 倍。

**Proto 定義範例：**
```protobuf
message User {
  string id = 1;
  string name = 2;
}

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}
```

**什麼時候用**：微服務架構中服務間的高效通訊。強型別有助於在編譯時發現錯誤。

**不適合外部 API**：二進位協定，瀏覽器不支援，工具生態不如 JSON over HTTP 成熟。

**最佳策略：內部 API 用 gRPC，外部 API 用 REST。**

> **面試提示**：不要過早優化 RPC 協定選擇。在解決其他實質性瓶頸之前就跳到 gRPC 並不是好做法。過早的優化是萬惡之源。

## See also

- [[HTTP/HTTPS]] — 三者都建立在 HTTP 之上
- [[SSE 與 WebSocket]] — 需要即時推送時的替代方案
- [[負載平衡]] — gRPC 內建客戶端負載平衡
