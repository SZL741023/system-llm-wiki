---
title: API 安全：Authentication 與 Authorization
tags: [concept, security, api]
sources: 1
updated: 2026-05-16
---

API 安全的核心是兩個問題：**你是誰（Authentication）** 和 **你能做什麼（Authorization）**。

## Authentication vs Authorization

| | Authentication（身份驗證） | Authorization（授權） |
|--|--------------------------|---------------------|
| 問題 | 你是誰？ | 你能做什麼？ |
| 目的 | 確認身份是否合法 | 決定已驗證身份是否有權限執行某操作 |
| 例子 | 輸入帳號密碼、API Key | Admin 可 DELETE，普通用戶只能 GET 自己的資料 |

簡單記法：**Authentication = Who you are，Authorization = What you can do**

## 常見 Authentication 方式

**API Key**：最簡單，在 header 帶一組 key。缺點：key 洩漏風險高，缺乏細粒度權限控制。

**Basic Auth**：用 `username:password` 做 Base64 編碼放在 header。適合簡單情境，但安全性差。

**Token-based（Session Token / JWT）**：
- 使用者登入後由伺服器發 token，之後呼叫 API 時帶上 token
- **JWT（JSON Web Token）**：內含使用者資訊與簽章，可 stateless 驗證
  - **Access Token**：短有效期（幾分鐘到幾小時），用於每次 API 呼叫驗證
  - **Refresh Token**：長有效期，當 Access Token 過期時用來取得新的 Access Token
- JWT 通常以 `Authorization: Bearer <token>` 方式攜帶

**OAuth 2.0**：常用於第三方 API 授權（Google、Facebook、GitHub 登入）。流程：使用者同意授權 → 發送 access token → API 用 token 驗證。

**mTLS（Mutual TLS）**：雙方都要出示憑證，常用於**內部微服務之間的安全通訊**。

## 常見 Authorization 模型

**RBAC（Role-Based Access Control）**：根據使用者角色控制權限。
- Admin → 可以 `GET /users`、`DELETE /users/123`
- Editor → 可以 `POST /articles`、`PUT /articles/456`
- Viewer → 只能 `GET /articles/456`
- 適用場景：公司內部管理系統

**ABAC（Attribute-Based Access Control）**：根據使用者或資源的屬性決定是否允許存取。
- 屬性：時間、地點、部門等
- 例：只有在辦公室 IP 網段才能呼叫 `/internal-api`
- 適用場景：金融機構、政府系統，需要更細緻的存取控制

**Scope-based（OAuth Scopes）**：權限隨 Token 的 scope 一起發放。
- `repo` scope → 可以存取私人 repo
- `read:user` → 只能讀取使用者基本資訊
- 適用場景：第三方應用授權（Google Drive、GitHub API）

## 面試考點

> **Q**：JWT 通常放在哪裡？**A**：`Authorization: Bearer <token>` header。
>
> **Q**：內部微服務通常用什麼？**A**：mTLS，不是 OAuth 2.0（OAuth 主要用於第三方授權）。

## See also

- [[HTTP/HTTPS]] — HTTPS 加密傳輸，是 Authentication 的基礎安全層
- [[REST vs GraphQL vs gRPC]] — 不同 API 風格下安全機制的應用方式
- [[冪等性（Idempotency）]] — API 設計的另一個重要安全考量
