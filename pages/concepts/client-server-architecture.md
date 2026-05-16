---
title: 客戶端-伺服器架構（Client-Server Architecture）
tags: [concept, architecture, fundamentals]
sources: 1
updated: 2026-05-16
---

幾乎所有你在系統設計面試中遇到的系統，都建立在 Client-Server 模型上。這是畫架構圖的起點，也是理解系統設計的基礎。

## 核心概念

- **Client（客戶端）**：發起請求的一方。例如瀏覽器、手機 App、或另一個服務。
- **Server（伺服器）**：接收請求、處理邏輯、回傳結果的一方。

兩者透過網路溝通，遵循 **Request-Response（請求-回應）** 模型。

## 為什麼重要

1. **職責分離**：Client 負責呈現和互動，Server 負責商業邏輯和資料處理。讓系統更容易維護和擴展。
2. **集中管理**：資料和邏輯集中在 Server 端，方便更新、維護和安全管控。Client 不需要知道資料怎麼存、邏輯怎麼跑。
3. **多客戶端支援**：同一個 Server 可以同時服務網頁、手機 App、第三方 API 等不同類型的 Client。
4. **擴展基礎**：當流量增加時，只需對 Server 做垂直或水平擴展，不需要改動 Client。

## Client 和 Server 的角色

**Client 的典型職責：**
- 呈現使用者介面（UI）
- 收集使用者輸入
- 向 Server 發送請求，接收並展示回應

**常見 Client：** 瀏覽器（Chrome、Safari）、手機 App（iOS、Android）、桌面應用程式（Slack、VS Code）、**其他服務**（在微服務架構中，一個服務可以是另一個服務的 Client）

**Server 的典型職責：**
- 驗證和授權請求（Authentication & Authorization）
- 執行商業邏輯
- 讀寫資料庫
- 回傳結果給 Client

**Server 通常不是一台機器**，而是由多台伺服器組成的叢集，前面有 [[負載平衡|Load Balancer]] 分配流量。

**Server 通常由三層組成：**
- **Web Server**：接收 HTTP 請求（Nginx、Apache）
- **Application Server**：執行商業邏輯（Node.js、Django、Spring Boot）
- **Database**：儲存和查詢資料（PostgreSQL、MongoDB）

## Thin Client vs Thick Client

Client 可以承擔不同程度的邏輯處理：

| | Thin Client（瘦客戶端） | Thick Client（胖客戶端） |
|--|----------------------|----------------------|
| 邏輯位置 | 大部分在 Server 端 | Client 承擔較多邏輯 |
| 例子 | Server-Side Rendering（SSR） | React/Vue SPA、手機 App |
| Server 角色 | 產生完整 HTML | 主要提供資料 API |
| 優點 | 輕量、易維護、安全性高（邏輯不暴露在前端） | UX 流暢、減少 Server 負擔 |
| 缺點 | 每次互動需向 Server 請求，UX 較慢 | 前端程式碼複雜，部分邏輯暴露在 Client 端 |

> **面試提示**：現代系統設計通常假設 **Thick Client（SPA 或手機 App）+ RESTful API** 的架構。你的設計重點通常在 Server 端的架構，但要知道 Client 端的選擇會影響 API 設計。

## Client-Server vs Peer-to-Peer（P2P）

| 特性 | Client-Server | Peer-to-Peer |
|------|--------------|--------------|
| 通訊方式 | Client 向 Server 請求 | 節點之間直接溝通 |
| 集中控制 | 有（Server 控制一切） | 無（去中心化） |
| 擴展方式 | 擴展 Server | 加入更多 Peer |
| 典型應用 | 幾乎所有 Web 應用 | 視訊通話（WebRTC）、檔案分享（BitTorrent） |

> **面試提示**：系統設計面試中，**99% 的問題都是 Client-Server 架構**。只有在明確涉及視訊/音訊通話時，才需要考慮 P2P（WebRTC）。詳見 [[SSE 與 WebSocket]]。

## 面試中畫架構圖的起點

Client-Server 是畫系統架構圖的第一條線，後續所有設計都在這個框架上展開：

1. **從 Client 開始**：使用者透過什麼介面使用系統？（瀏覽器？App？）
2. **加入 Server**：處理請求的後端服務是什麼？
3. **定義 API**：Client 和 Server 之間怎麼溝通？（REST？GraphQL？WebSocket？）
4. **加入 Database**：Server 的資料存在哪裡？
5. **擴展**：流量大了怎麼辦？加 Load Balancer、Cache、CDN……

## See also

- [[負載平衡]] — Server 通常是伺服器叢集，前面有 LB 分配流量
- [[REST vs GraphQL vs gRPC]] — Client 和 Server 之間的 API 選擇
- [[SSE 與 WebSocket]] — 需要即時推送時的 Client-Server 通訊變體；P2P（WebRTC）的詳細說明
- [[OSI 模型（系統設計必知三層）]] — Client-Server 通訊的網路層基礎
