---
title: PostgreSQL JSONB，當關聯式資料庫遇上文件儲存
author: Cooper
date: 2026-01-28 00:30:00 +0800
categories: [Database]
tags: [postgresql, jsonb, nosql]
image:
  path: /postgresql-jsonb/banner.jpg
---

## 前言

在過去的專案中，我一直在尋找一個既能處理嚴謹的關聯資料，又能靈活儲存非結構化資料的資料庫解決方案。相信很多開發者都跟我有過同樣的掙扎：當我們在使用傳統的關聯式資料庫（如 MySQL 或 MariaDB）時，往往會遇到一些難以預先定義 Schema 的場景，例如使用者的自訂偏好設定、第三方 API 的原始回應內容，或是電商平台中千變萬化的商品屬性。

過去，為了解決這種靈活性問題，我們可能會選擇引入 MongoDB 這樣的 NoSQL 資料庫。然而，這也帶來了新的挑戰：資料一致性的維護變得複雜、需要維運兩套不同的資料庫系統、開發者需要學習兩套查詢語法，而且跨資料庫的 JOIN 操作簡直是噩夢。

直到我深入研究了 PostgreSQL 的 JSONB 功能，我才發現原來「魚與熊掌可以兼得」。PostgreSQL 不僅僅是一個強大的關聯式資料庫，它透過 JSONB 提供的文件儲存能力，讓它在處理非結構化資料時，表現甚至不輸給專門的 NoSQL 資料庫。這篇文章將帶你深入了解 PostgreSQL JSONB 的強大之處，以及為什麼越來越多的開發者（包括我自己）正從 MariaDB 或 MongoDB 遷移到 PostgreSQL。

## JSONB vs JSON

在 PostgreSQL 中，處理 JSON 資料有兩種主要的資料類型：`JSON` 和 `JSONB`。雖然它們看起來很像，但底層的處理方式卻截然不同。

![JSONB 對比](/postgresql-jsonb/jsonb-comparison.png)

### JSON：原始文字儲存
`JSON` 類型基本上是將輸入的資料當作「文字」來儲存。它會完整保留你輸入時的樣子，包括空格、換行，甚至是重複的鍵值（Key）。
- **優點**：寫入速度極快，因為資料庫不需要對內容進行解析或重新編碼。
- **缺點**：每次查詢時，資料庫都必須重新解析文字內容，這會導致查詢效能低落。此外，它不支援高效的索引。

### JSONB：二進制格式儲存
`JSONB` 中的 "B" 代表 "Binary"（二進制）。當你存入資料時，PostgreSQL 會先將 JSON 解析並轉換成一種最佳化的二進制格式。
- **優點**：查詢速度極快，因為資料已經是解析過的格式。它會自動移除重複的鍵值，並刪除不必要的空格。最重要的是，它支援強大的 GIN（Generalized Inverted Index）索引。
- **缺點**：寫入時因為需要進行解析和轉換，速度會比純 `JSON` 稍微慢一點點，且儲存空間可能會稍微大一些（因為需要儲存額外的元數據）。

### 特性對比表

| 特性 | JSON | JSONB |
|-----|------|-------|
| 儲存格式 | 文字 (Plain Text) | 二進制 (Binary) |
| 寫入速度 | 較快 | 較慢 (需解析) |
| 查詢速度 | 較慢 (需解析) | 較快 |
| 索引支援 | 無 (僅能使用函數索引) | 支援 GIN/GiST 索引 |
| 保留空格 | 是 | 否 |
| 保留重複鍵 | 是 | 否 (保留最後一個) |

{: .prompt-info }
> **我的建議**：除非你特別需要保留原始 JSON 的格式（例如作為稽核日誌，必須與原始輸入完全一致），否則在 99% 的實際應用場景中，請直接使用 **JSONB**。

## JSONB 基本操作

讓我們來看看如何在 PostgreSQL 中實際操作 JSONB。首先，我們建立一個簡單的 `users` 資料表，其中包含一個 `profile` 欄位來儲存使用者的非結構化資訊。

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT NOT NULL,
    profile JSONB NOT NULL DEFAULT '{}'
);
```

### 插入資料
插入 JSONB 資料非常直覺，直接傳入符合 JSON 格式的字串即可：

```sql
INSERT INTO users (username, profile) VALUES
('cooper', '{"email": "cooper@example.com", "age": 30, "tags": ["developer", "blogger"], "settings": {"theme": "dark", "notifications": true}}'),
('alice', '{"email": "alice@example.com", "age": 25, "tags": ["designer"], "settings": {"theme": "light", "notifications": false}}');
```

### 查詢與操作符
PostgreSQL 提供了一系列強大的操作符來存取 JSONB 內容：

1.  **`->`**：獲取 JSON 物件欄位（回傳 JSONB 類型）。
2.  **`->>`**：獲取 JSON 欄位內容並轉換為文字（回傳 TEXT 類型）。
3.  **`@>`**：包含操作符，檢查左邊的 JSON 是否包含右邊的內容。
4.  **`?`**：檢查鍵值（Key）是否存在。

```sql
-- 獲取所有使用者的主題設定 (回傳 TEXT)
SELECT username, profile->'settings'->>'theme' AS theme
FROM users;

-- 尋找標籤中包含 "developer" 的使用者
SELECT username
FROM users
WHERE profile->'tags' ? 'developer';

-- 使用包含操作符尋找特定設定的使用者
SELECT username
FROM users
WHERE profile @> '{"settings": {"theme": "dark"}}';
```

### PostgreSQL 14+ 的新語法：下標存取 (Subscripting)
如果你使用的是 PostgreSQL 14 或更高版本，你可以使用更像程式語言的語法來存取 JSONB 欄位：

```sql
-- 使用下標語法獲取 email
SELECT profile['email'] FROM users;

-- 更新特定欄位的值
UPDATE users 
SET profile['settings']['theme'] = '"solarized"'
WHERE username = 'cooper';
```

這種語法讓 SQL 看起來更簡潔，也更符合現代開發者的直覺。

### 進階 JSONB 函數
除了基本的操作符，PostgreSQL 還提供了一系列強大的函數來修改和轉換 JSONB 資料：

1.  **`jsonb_set`**：修改 JSONB 中的特定值。
2.  **`jsonb_insert`**：在陣列的特定位置插入新元素。
3.  **`jsonb_each`**：將 JSON 物件展開為鍵值對（Key-Value pairs）。
4.  **`jsonb_array_elements`**：將 JSON 陣列展開為多列資料。

```sql
-- 使用 jsonb_set 更新年齡
UPDATE users 
SET profile = jsonb_set(profile, '{age}', '31')
WHERE username = 'cooper';

-- 使用 jsonb_insert 在標籤陣列開頭插入新標籤
UPDATE users
SET profile = jsonb_insert(profile, '{tags, 0}', '"senior"')
WHERE username = 'cooper';

-- 將使用者的標籤展開為多列，方便進行統計
SELECT username, jsonb_array_elements_text(profile->'tags') AS tag
FROM users;
```

### SQL/JSON Path (PostgreSQL 12+)
PostgreSQL 12 引入了 SQL/JSON Path 語言，這讓複雜的 JSON 查詢變得更加強大且易讀。它類似於 JSONPath 或 XPath。

```sql
-- 尋找所有年齡大於 28 且設定中 theme 為 dark 的使用者
SELECT username
FROM users
WHERE jsonb_path_exists(profile, '$.age ? (@ > 28) && $.settings.theme == "dark"');
```

## 索引優化

如果你的 JSONB 欄位資料量很大，且需要頻繁查詢，那麼索引就是效能的關鍵。這也是 PostgreSQL 完勝 MySQL/MariaDB 的地方之一。

### GIN 索引：JSONB 的黃金標準
GIN（Generalized Inverted Index）是處理 JSONB 最常用的索引類型。它可以對 JSONB 中的所有鍵值對進行索引。

```sql
-- 建立標準的 GIN 索引
CREATE INDEX idx_profile ON users USING GIN (profile);
```

這個索引支援 `@>`, `?`, `?&`, `?|` 等操作符。如果你只需要支援 `@>` 操作符，可以使用 `jsonb_path_ops` 參數，這會產生更小、更快的索引：

```sql
-- 建立更高效的路徑 GIN 索引
CREATE INDEX idx_profile_path ON users USING GIN (profile jsonb_path_ops);
```

### 表達式索引 (Expression Indexes)
如果你經常查詢 JSONB 中的某個特定欄位，建立表達式索引會比全域 GIN 索引更有效率：

```sql
-- 針對 profile 中的 email 欄位建立索引
CREATE INDEX idx_user_email ON users ((profile->>'email'));
```

建立這個索引後，像 `SELECT * FROM users WHERE profile->>'email' = 'cooper@example.com'` 這樣的查詢將會變得極快。

{: .prompt-info }
> 在實際專案中，我通常會先觀察慢查詢日誌。如果查詢模式很固定，我會優先選擇表達式索引；如果查詢非常動態且不可預測，則會使用 GIN 索引。

## 實際應用場景

為什麼我們需要 JSONB？以下是幾個我在實際開發中經常遇到的場景：

### 1. 使用者偏好設定
每個使用者的設定可能都不一樣，有些使用者開啟了電子郵件通知，有些使用者則設定了自訂的介面顏色。使用 JSONB 可以輕鬆應對這種多變的需求，而不需要為每個新設定都增加一個資料庫欄位。

```sql
-- 查詢所有關閉通知的使用者
SELECT * FROM users WHERE profile @> '{"settings": {"notifications": false}}';
```

### 2. 事件日誌 (Event Logging)
在微服務架構中，我們經常需要記錄各種系統事件。不同類型的事件會有不同的屬性。

```sql
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    event_type TEXT,
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 插入不同結構的日誌
INSERT INTO audit_logs (event_type, payload) VALUES
('login', '{"ip": "192.168.1.1", "browser": "Chrome"}'),
('purchase', '{"item_id": 101, "amount": 50.5, "currency": "USD"}');
```

### 3. API 回應快取
當我們呼叫外部 API（如 GitHub API 或 Stripe API）時，可以將原始的 JSON 回應直接存入 JSONB 欄位。這樣既保留了原始資料供日後稽核，又能利用 PostgreSQL 的查詢能力來分析這些資料。

### 4. 多租戶配置 (Multi-tenant Configurations)
在 SaaS 應用中，不同的租戶（客戶）可能有不同的配置需求。JSONB 讓我們可以在同一個資料表中儲存這些差異化的配置。

### 5. 電商商品屬性
這是最經典的例子。衣服有「尺寸」和「顏色」，而硬碟有「容量」和「介面類型」。使用傳統的正規化設計（EAV 模型）會導致查詢極其複雜且效能低下，而 JSONB 則能完美解決這個問題。

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes JSONB
);

-- 尋找所有容量為 1TB 的硬碟
SELECT * FROM products 
WHERE attributes @> '{"category": "HDD", "capacity": "1TB"}';
```

## 從其他資料庫遷移

這是我最想分享的部分。為什麼我建議你考慮從 MongoDB 或 MariaDB 遷移到 PostgreSQL？

### 從 MongoDB 遷移
MongoDB 是文件儲存的先驅，但 PostgreSQL 在這方面已經追趕上來，甚至在某些維度上超越了它。

*   **ACID 事務的重要性**：在金融應用或需要嚴謹資料一致性的場景中，PostgreSQL 的強大事務支援是 MongoDB 難以企及的。雖然 MongoDB 後來也加入了事務支援，但在複雜場景下的穩定性和成熟度仍有差距。
*   **單一資料庫 vs 雙資料庫架構**：如果你使用 MongoDB，你通常還需要一個關聯式資料庫來處理核心業務邏輯。這意味著你需要維護兩套系統。使用 PostgreSQL，你可以將關聯資料和文件資料放在同一個資料庫中，甚至在同一個查詢中進行 JOIN。
*   **強大的 JOIN 操作**：在 MongoDB 中，跨集合的查詢（$lookup）效能通常不理想。而在 PostgreSQL 中，你可以輕鬆地將 JSONB 欄位與其他正規化資料表進行 JOIN。

### 從 MariaDB/MySQL 遷移
雖然 MySQL 和 MariaDB 近年來也加強了對 JSON 的支援，但 PostgreSQL 的實作更為先進。

*   **索引能力的差異**：MySQL 的 JSON 索引通常需要透過「虛擬欄位」（Generated Columns）來達成，這在操作上相對繁瑣。PostgreSQL 的 GIN 索引則可以直接作用於 JSONB 欄位，支援更靈活的查詢模式。
*   **操作符的豐富度**：PostgreSQL 提供了極其豐富的 JSONB 操作符和函數（如 `jsonb_set`, `jsonb_insert`, `jsonb_path_query` 等），這讓資料處理變得非常強大。
*   **擴充生態系**：PostgreSQL 有著無與倫比的擴充性，例如 PostGIS（地理資訊）、TimescaleDB（時序資料）以及最近非常火紅的 pgvector（向量資料庫，用於 AI 應用）。

### 遷移注意事項
如果你決定遷移，有幾點需要注意：
1.  **資料型態對應**：確保將 MongoDB 的 BSON 類型正確對應到 PostgreSQL 的 JSONB 或其他原生類型（如 UUID, Timestamp）。
2.  **查詢語法轉換**：你需要將 MongoDB 的查詢物件（Query Object）轉換為 SQL 語法。
3.  **工具推薦**：我個人非常推薦使用 `pgloader`，它可以自動化處理從 MySQL/MariaDB 到 PostgreSQL 的遷移過程，包括資料型態的自動轉換。

## 結語

PostgreSQL 已經不再僅僅是一個「關聯式資料庫」，它已經演變成了一個「多模態資料庫」（Multi-model Database）。透過 JSONB，它成功地吸收了 NoSQL 的靈活性，同時保留了 SQL 的強大與嚴謹。

**什麼時候該用 JSONB？**
- 當資料結構經常變動時。
- 當資料是稀疏的（很多欄位大部分時間都是空的）。
- 當你需要儲存來自外部系統的原始資料時。

**什麼時候該用正規化資料表？**
- 當資料結構非常固定且明確時。
- 當欄位需要頻繁參與複雜的報表計算時。
- 當你需要利用外鍵（Foreign Key）來確保嚴格的參照完整性時。

在實際的專案中，我通常會採用「混合模式」：核心的業務實體（如使用者帳號、訂單主檔）使用正規化的欄位，而擴充屬性、偏好設定或日誌資訊則使用 JSONB。

隨著 AI 時代的到來，PostgreSQL 結合 pgvector 更是展現了強大的潛力。我相信，PostgreSQL 已經成為現代應用程式開發中「the database for everything」的最佳選擇。

---

## 參考資料

[1]: https://www.postgresql.org/docs/current/datatype-json.html
[2]: https://www.postgresql.org/docs/current/functions-json.html
[3]: https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING
[4]: https://github.com/dimitri/pgloader
[5]: https://blog.crunchydata.com/blog/postgresql-vs.-mongodb-2023-edition
