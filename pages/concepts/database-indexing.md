---
title: 資料庫索引（Database Indexing）
tags: [concept, database, performance]
sources: 1
updated: 2026-05-16
---

索引（Index）就像書本的目錄。它能讓你快速找到需要的資料，而不用從頭到尾把整本「書」（資料表）翻一遍。

## 沒有索引 vs 有索引

**沒有索引**：資料庫必須做 **Full Table Scan（全表掃描）**，逐筆比對，時間複雜度 O(n)。資料量越多，查詢越慢。

**有索引**：資料庫透過索引快速定位資料（像查目錄找頁碼），時間複雜度接近 **O(log n)**。

```sql
-- 沒有索引：Full Table Scan
SELECT * FROM users WHERE email = 'bohr@example.com';

-- 建立索引後：快速定位
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'bohr@example.com'; -- O(log n)
```

## 索引能加速哪些操作

- `WHERE` 條件查詢
- `JOIN` 操作
- `ORDER BY` 排序
- 前綴搜尋 `LIKE 'abc%'`

## 索引的缺點

- **佔用額外儲存空間**
- **寫入（INSERT/UPDATE/DELETE）會變慢**：因為每次寫入都要更新索引
- **建太多索引反而拖累效能**：索引本身需要維護

> 索引是以空間換時間，以寫入代價換讀取速度的取捨。

## 常見索引種類

| 類型 | 適用場景 |
|------|---------|
| **B-Tree Index** | 最常見，支援等值查詢、範圍查詢、排序 |
| **LSM Tree** | 寫入密集型，Cassandra、RocksDB 使用 |
| **Hash Index** | 只支援等值查詢（不支援範圍），極快 |
| **Geospatial Index** | 地理位置查詢（附近的餐廳） |
| **Inverted Index** | 全文搜尋（Elasticsearch） |

## See also

- [[資料庫交易（Database Transactions）]] — 索引是查詢優化；Transaction 是資料完整性保障
- [[分片（Sharding）]] — Sharding 前應先確認索引是否已優化（慢查詢 → 加 index / 優化 SQL 是比 sharding 更簡單的解法）
- [[快取（Caching）]] — 快取是索引之外另一個加速讀取的手段
