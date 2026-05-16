---
title: OSI 模型（系統設計必知三層）
tags: [concept, networking, fundamentals]
sources: 1
updated: 2026-05-16
---

系統設計面試只需要掌握三個關鍵層次，其餘可以當作已抽象化的黑盒子。

## 三個關鍵層次

| 層次 | OSI 編號 | 代表協定 |
|------|----------|---------|
| 網路層（Network） | Layer 3 | IP、routing |
| 傳輸層（Transport） | Layer 4 | TCP、UDP、QUIC |
| 應用層（Application） | Layer 7 | HTTP、DNS、WebSocket、WebRTC |

- **Layer 3（IP）**：負責路由和定址，把資料拆成封包在網路間轉送，提供 best-effort 交付。
- **Layer 4（Transport）**：在 IP 之上加入可靠性、排序、流量控制（TCP），或幾乎不加任何東西只求速度（UDP）。
- **Layer 7（Application）**：開發者花最多時間的地方，定義應用程式如何通訊。

## 一個 Web 請求的完整流程

當瀏覽器輸入 URL：

1. **DNS 解析**：域名 → IP 位址
2. **TCP 三次握手**：SYN → SYN-ACK → ACK
3. **HTTP 請求/回應**
4. **TCP 四次揮手**：FIN → ACK → FIN → ACK

## 為什麼分層重要

作為應用程式開發者，你可以假設：
- TCP 層保證資料正確有序送達（或通知失敗）
- IP/DNS/路由處理如何找到目標伺服器

越往上層走，每個請求的延遲和處理就越多——這在討論 [[負載平衡]] 時尤其重要（L4 vs L7 的效能差異就來自此）。

## See also

- [[TCP vs UDP]] — Layer 4 兩大協定的取捨
- [[HTTP/HTTPS]] — Layer 7 最核心的協定
- [[負載平衡]] — L4/L7 Load Balancer 的差異源於此
