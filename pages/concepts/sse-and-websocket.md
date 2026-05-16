---
title: SSE 與 WebSocket
tags: [concept, networking, real-time, application-layer]
sources: 1
updated: 2026-05-16
---

當需要即時推送資料給客戶端時，HTTP 請求-回應模型不夠用，SSE 和 WebSocket 是兩種主要替代方案。更極端的點對點場景則有 WebRTC。

## SSE（Server-Sent Events）：伺服器單向推送

SSE 是建立在 HTTP 之上的規範，讓伺服器能透過單一 HTTP 連線向客戶端推送多則訊息。

**運作方式**：它是 HTTP 之上的一個巧妙改裝——伺服器在單一回應中隨時間持續串流多則訊息，客戶端逐行處理 body，對每則資料立即反應。

```
data: {"id": 1, "event": "price_update", "price": 42.5}

data: {"id": 2, "event": "price_update", "price": 43.0}
```

**限制：**
- SSE 連線不能保持太長時間（伺服器、負載平衡器、中間代理可能關閉連線）
- `EventSource` 物件會自動用最後收到的訊息 ID 重新連線（斷線重連是內建的）
- 某些不規範的網路可能把 SSE 回應批次聚合成單一回應，讓流式行為退化

**什麼時候用**：客戶端需要接收通知或事件更新，但不需要向伺服器回傳資料。例如：拍賣當前價格即時更新、新聞推播。

## WebSocket：即時雙向通訊

WebSocket 提供客戶端和伺服器之間持久的、類似 TCP 的連線，實現即時雙向通訊。

**建立流程：**
1. 客戶端透過 HTTP 發起 WebSocket 握手（底層是 TCP 連線）
2. 連線升級到 WebSocket 協定
3. 客戶端和伺服器都可以透過連線互相發送二進位訊息
4. 連線保持開啟，直到明確關閉為止

**優勢**：廣泛的瀏覽器支援。可利用現有的 HTTP 工作階段資訊（cookies、headers）完成升級。

> **注意**：客戶端能從 HTTP 升級到 WebSocket，不代表基礎設施一定支援它。每個基礎設施元件（防火牆、代理、負載平衡器）都需要支援 WebSocket 連線。

**什麼時候用**：客戶端和伺服器之間需要高頻、持久、雙向通訊——即時應用、遊戲、需要訊息立刻發送和接收的場景。

> **面試警告**：在沒有說明原因的情況下就跳進 WebSocket 實作，是讓面試官給你「thumbs down」的好方法。WebSocket 雖然強大，但支援它所需的基礎設施可能很昂貴，有狀態連線的開銷在規模下也需要大量設計配合。**除非真的需要，否則不要用。**

## WebRTC：點對點通訊（P2P）

WebRTC 讓瀏覽器之間能夠直接進行點對點通訊，不需要中間伺服器做資料交換。**這是我們介紹的唯一一個使用 UDP 的應用層協定。**

**建立 WebRTC 連線的四個步驟：**
1. 客戶端連線到中央**信令伺服器（signaling server）**，了解自己的 peer
2. 客戶端聯繫 **STUN 伺服器**取得自己的公網 IP 和 port（NAT 穿透）
3. 客戶端透過信令伺服器與對方分享連線資訊
4. 客戶端建立直接的點對點連線，開始傳送資料

**STUN vs TURN：**
- **STUN**：讓 peer 建立可公開路由的位址和 port（打洞），是 NAT 穿透的標準方式
- **TURN**：當 STUN 失敗時的中繼備案，把請求透過中央伺服器轉送

**什麼時候用**：音視訊通話和會議應用。

> **誠實的警告**：WebRTC 極難做對，即使是最好的實作也會有連線中斷的問題。面試中，試圖用 WebRTC 設計點對點系統而搞錯方向的面試者，比成功實作的要多得多。大多數問題不需要點對點連線。

## SSE vs WebSocket 選擇

- **需要單向推送（伺服器 → 客戶端）** → SSE（更簡單，HTTP 原生支援）
- **需要雙向即時通訊** → WebSocket
- **WebSocket 的負載平衡**：使用 L4 Load Balancer（見 [[負載平衡]]），因為 WebSocket 是長連線，L4 可以維持持久的 TCP 連線

## See also

- [[客戶端-伺服器架構（Client-Server Architecture）]] — WebRTC/P2P 是 Client-Server 的例外，99% 的情況用 Client-Server
- [[HTTP/HTTPS]] — SSE 和 WebSocket 都建立在 HTTP 之上
- [[TCP vs UDP]] — WebSocket 用 TCP，WebRTC 用 UDP
- [[負載平衡]] — WebSocket 需要 L4 LB；SSE 用最少連線演算法
