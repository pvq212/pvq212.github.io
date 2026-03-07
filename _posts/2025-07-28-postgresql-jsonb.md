---
title: PostgreSQL JSONB，什麼時候真的該用？我現在的判斷方式
author: Cooper
date: 2025-07-28 00:30:00 +0800
categories: [Database]
tags: [postgresql, jsonb, nosql]
image:
  path: /postgresql-jsonb/banner.jpg
---

## 前言

`JSONB` 是那種很容易被吹過頭的功能。

很多人一講到它，就會直接下結論說：「PostgreSQL 根本可以取代 MongoDB。」這句話不是完全錯，但如果你真的拿它做過專案，就會知道答案沒有這麼簡單。

我自己現在對 `JSONB` 的理解比較務實：

> 它很適合處理半結構化資料，但前提是你知道哪些欄位該放進 JSONB，哪些欄位還是應該老實正規化。

## JSON 跟 JSONB，到底差在哪裡

官方文件現在還是把差異講得很清楚。[1]

- `json`：保留原始文字
- `jsonb`：存成二進位格式，查詢更快，也能做索引

最實際的差別就是：

- 你如果只是想原樣存一份 JSON，`json` 可以
- 你如果之後還想查、過濾、索引、更新局部欄位，大多直接選 `jsonb`

![JSONB 對比](/postgresql-jsonb/jsonb-comparison.png)

## 為什麼我大多數情況都會直接選 JSONB

因為現實世界裡，資料放進資料庫後，很少只是拿來存而已。

通常你後面還會想做：

- 查某個 key
- 判斷某個欄位有沒有存在
- 用陣列內容過濾
- 幫常查的欄位建索引

而這些事情，`jsonb` 都比較像是正規做法。[1][2]

## 現在 PostgreSQL 18 的 JSON 功能，其實已經很完整

我去看現在最新文件，`PostgreSQL 18.3` 已經把這幾塊都整理得很完整：[1][2]

- `jsonb` containment
- `jsonb` indexing
- subscripting
- `jsonpath`
- `jsonb_set`
- `jsonb_insert`

也就是說，現在的 JSONB 不是只有「可存 JSON」而已，而是整套查詢與處理能力都長得很完整。

## 最適合 JSONB 的幾種場景

### 1. 使用者偏好設定

像是：

- 主題色
- 通知設定
- 第三方整合選項

這些欄位常常不是每個使用者都有，而且未來還可能一直長。

### 2. 第三方 API 回應快取

這種我很常用。

你把外部 API 回傳原樣存進來，之後需要查局部欄位時，再用 JSONB operator 來撈。

### 3. 彈性屬性資料

例如商品屬性、活動事件 payload、audit log metadata。

這些東西如果硬拆成很多欄位，schema 會長得很累。

## 什麼時候不要用 JSONB

這點比什麼時候要用更重要。

### 1. 你明明知道欄位是固定的

像 `email`、`status`、`created_at` 這種，如果還硬塞 JSONB，我會覺得是在找自己麻煩。

### 2. 你要拿它當萬用垃圾桶

很多人一覺得 schema 變麻煩，就把東西全部塞 JSONB。短期很爽，後期維護會痛苦。

### 3. 你需要強型別與外鍵保證

這種時候，正常欄位設計還是比較穩。

## 新手先學這幾個操作就很夠用

### 1. 讀欄位

```sql
SELECT profile->>'email' FROM users;
```

### 2. 判斷是否包含

```sql
SELECT *
FROM users
WHERE profile @> '{"settings": {"theme": "dark"}}';
```

### 3. 檢查 key 是否存在

```sql
SELECT *
FROM users
WHERE profile ? 'settings';
```

### 4. 更新局部內容

```sql
UPDATE users
SET profile = jsonb_set(profile, '{settings,theme}', '"solarized"');
```

### 5. 用 `jsonpath` 做進階查詢

```sql
SELECT *
FROM users
WHERE jsonb_path_exists(profile, '$.age ? (@ > 28)');
```

這些其實就夠你做很多事了。[2]

## 索引才是你後面會真的在意的地方

官方文件現在還是把這件事講得很明白：`jsonb` 強不強，很大一部分要看索引。[1]

最常見的有兩種思路：

- 整個欄位上 `GIN`
- 常查特定路徑就做 expression index

我自己的習慣很簡單：

- 查詢很彈性：先看 `GIN`
- 查詢很固定：直接對常用路徑做 index

## 我的感想

如果你問我 JSONB 值不值得學，我一定說值得。

但如果你問我「是不是能把 schema 設計都省掉」，我會直接說不要想太美。

它最有價值的地方，是讓你在關聯式資料庫裡，合理地容納一部分半結構化資料，而不是把所有資料建模問題都推給 JSONB。

## 參考資料

[1]: https://www.postgresql.org/docs/current/datatype-json.html
[2]: https://www.postgresql.org/docs/current/functions-json.html
