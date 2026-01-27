---
title: Elasticsearch，分散式的全文搜索引擎
author: Cooper
date: 2024-04-14 00:30:00 +0800
categories: [Database]
tags: [nosql, elasticsearch]
image:
  path: /elasticsearch/banner.jpg
---

## 前言

在現代的 Web 應用程式中，搜尋功能幾乎是標配。但你有沒有想過，當資料量達到百萬、千萬甚至上億等級時，傳統的關聯式資料庫（如 MySQL 或 PostgreSQL）還能扛得住嗎？

最近在工作上遇到一個棘手的問題：我們的產品目錄搜尋變得越來越慢。原本只是簡單的 `SELECT * FROM products WHERE name LIKE '%關鍵字%'`，但在資料量成長後，這種全表掃描（Full Table Scan）簡直是災難。沒錯，就是這麼簡單的問題，卻讓資料庫伺服器的 CPU 經常飆到 90% 以上。

這時候，Elasticsearch（簡稱 ES）就派上用場了。

Elasticsearch 是一個基於 Lucene 的分散式搜尋引擎。它之所以快，是因為它採用了「倒排索引」（Inverted Index）的技術。想像一下，如果你要在一本書中找某個單字，你是會從第一頁翻到最後一頁，還是直接翻到書末的索引表？倒排索引就是那張索引表。

當你需要處理大量文字搜尋、日誌分析或是需要毫秒級的搜尋反應時，Elasticsearch 絕對是你的首選。

## 部署 Elasticsearch

要開始使用 Elasticsearch，最快的方式就是使用 Docker。現在 ES 已經出到 8.x 版本了，安全性設定比以前嚴格許多，但透過 `docker-compose` 還是可以輕鬆搞定。

### docker-compose.yml 設定

以下是一個簡單的 `docker-compose.yml`，包含了 Elasticsearch 8.12.0 和可視化工具 Kibana：

```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - 9200:9200
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    ports:
      - 5601:5601
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elastic

networks:
  elastic:
    driver: bridge
```

> 注意：為了開發方便，我這裡將 `xpack.security.enabled` 設為 `false`。在正式環境中，請務必開啟安全性設定並配置密碼。
{: .prompt-warning }

### 安裝 IK 分詞器

對於中文使用者來說，內建的分析器（Analyzer）效果並不理想。我們通常會安裝 `elasticsearch-analysis-ik` 插件。

你可以進入容器執行以下指令：

```bash
docker exec -it elasticsearch bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.12.0/elasticsearch-analysis-ik-8.12.0.zip
```

安裝完後記得重啟容器。你可以透過以下指令驗證 ES 是否正常運作：

```bash
curl http://localhost:9200
```

> Elasticsearch 非常吃記憶體！建議開發環境至少給它 2GB 的 RAM，否則啟動時可能會因為 OOM (Out Of Memory) 而崩潰。
{: .prompt-info }

## Analyzer 深入解析

要理解 Elasticsearch 為什麼能精準搜尋，就必須先搞懂 **Analyzer（分析器）**。當你存入一段文字（Document）或進行搜尋（Query）時，ES 都會透過 Analyzer 對文字進行處理。

### 分析器流程

一個 Analyzer 由三個核心組件組成，它們按順序執行：

![分析器流程](/elasticsearch/analyzer-pipeline.png)

1.  **Character Filters (字符過濾器)**：
    這是第一道工序。它會在分詞之前處理原始文字。例如 `html_strip` 可以去除 HTML 標籤，或者你可以自定義字符映射（Mapping），把 `&` 轉換成 `and`。

2.  **Tokenizer (分詞器)**：
    這是最核心的部分。它負責將一長串文字切分成一個個獨立的詞元（Tokens）。例如將 "I love coding" 切成 ["I", "love", "coding"]。

3.  **Token Filters (詞元過濾器)**：
    最後一道工序。它會對切好的詞元進行加工。常見的操作包括：
    *   `lowercase`：將所有字母轉為小寫。
    *   `stop`：去除停用詞（如 "a", "the", "is"）。
    *   `synonym`：處理同義詞（如將 "quick" 關聯到 "fast"）。

### 內建分析器

ES 提供了多種內建分析器：
*   **Standard Analyzer**：預設值。按詞彙邊界切分，轉小寫，移除標點符號。
*   **Simple Analyzer**：只要不是字母的地方就切開，並轉小寫。
*   **Whitespace Analyzer**：只按空格切分，不轉小寫。
*   **Keyword Analyzer**：不進行任何切分，將整段文字當作一個詞元。

### 使用 _analyze API 測試

你可以直接在 Kibana 的 Dev Tools 中測試分析器的效果：

```json
POST _analyze
{
  "analyzer": "standard",
  "text": "Elasticsearch is AWESOME!"
}
```

你會發現 "AWESOME!" 被切成了 "awesome"，標點符號不見了，且全部變成了小寫。

## 中文分詞

如果你直接用 `standard` 分析器處理中文，你會發現它會把每個中文字都切開。例如「今天天氣很好」會變成 ["今", "天", "天", "氣", "很", "好"]。這在搜尋時會產生大量噪音，效果極差。

這就是為什麼我們需要 **IK Analyzer**。

### ik_smart vs ik_max_word

IK 提供了兩種主要模式：

1.  **ik_smart**：最少切分。它會嘗試用最粗略的方式切分文字。
    *   「今天天氣很好」 → ["今天", "天氣", "很好"]
2.  **ik_max_word**：最細粒度切分。它會窮盡所有可能的組合。
    *   「今天天氣很好」 → ["今天", "天天", "天氣", "氣很", "很好"]

通常在建立索引（Indexing）時使用 `ik_max_word` 以增加搜尋覆蓋率，而在搜尋（Searching）時使用 `ik_smart` 以提高精準度。

### 自定義詞庫

在某些專業領域，IK 可能認不出特定的詞彙。例如「歐買尬」可能會被切開。你可以透過修改 IK 的設定檔 `IKAnalyzer.cfg.xml` 並加入自己的 `.dic` 檔案來擴充詞庫。

```xml
<!-- IKAnalyzer.cfg.xml 範例 -->
<properties>
    <comment>IK Analyzer 擴展配置</comment>
    <entry key="ext_dict">my_custom.dic</entry>
</properties>
```

在 `my_custom.dic` 中，每一行寫一個詞即可。

## 搜尋優化

有了索引後，如何讓搜尋結果更符合使用者預期？這涉及到相關性評分（Relevance Scoring）。ES 預設使用 **BM25** 演算法。

### Boosting (權重提升)

有時候我們希望標題（Title）匹配到的權重比內容（Content）高。我們可以使用 `^` 符號來提升權重。

```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "蘋果手機",
      "fields": ["title^5", "description"]
    }
  }
}
```

在這個例子中，如果關鍵字出現在 `title`，其評分貢獻會是 `description` 的 5 倍。

### multi_match 的不同類型

`multi_match` 查詢非常強大，它有幾種常見的 `type`：

*   **best_fields** (預設)：在多個欄位中尋找匹配，取評分最高的那個欄位作為最終分數。適合處理「同一個概念分佈在不同欄位」的情況。
*   **most_fields**：將所有欄位的評分加總。適合處理「同一個欄位有多種分析方式」（例如一個欄位存原文字，另一個欄位存拼音）的情況。
*   **cross_fields**：將多個欄位視為一個大欄位。適合處理「姓」在一個欄位、「名」在另一個欄位的情況。

### 實戰範例

假設我們要搜尋產品，且希望精準匹配：

```bash
curl -X GET "localhost:9200/products/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "iPhone 15" } }
      ],
      "filter": [
        { "term": { "status": "active" } },
        { "range": { "price": { "gte": 20000 } } }
      ]
    }
  }
}
'
```

這裡使用了 `bool` 查詢，將 `must`（影響評分）和 `filter`（不影響評分，且有快取優勢）結合起來。

## 與其他資料庫比較

很多人會問：「我的資料庫已經有全文檢索功能了，為什麼還要學 Elasticsearch？」

讓我們來做個簡單的比較：

| 特性 | Elasticsearch | PostgreSQL (FTS) | MySQL (Full-Text) | MongoDB Atlas Search |
| :--- | :--- | :--- | :--- | :--- |
| **索引類型** | 倒排索引 (Lucene) | GIN / GIST | B-Tree / Fulltext | Lucene (Atlas 專屬) |
| **擴展性** | 極強 (原生分散式) | 中等 (需靠分片) | 較弱 | 強 (雲端託管) |
| **分析器自定義** | 極高 | 高 | 低 | 中 |
| **效能 (大數據)** | 極快 | 一般 | 較慢 | 快 |
| **適用場景** | 複雜搜尋、日誌分析 | 中小型應用、關聯資料 | 簡單關鍵字匹配 | MongoDB 用戶快速整合 |

**我個人建議：**
如果你的資料量在十萬筆以下，且搜尋邏輯不複雜，直接用 PostgreSQL 的全文檢索其實就夠了，還能省下一台伺服器的維護成本。但如果你需要處理毫秒級的複雜過濾、聚合分析（Aggregations），或者資料量會持續增長，那麼早點導入 Elasticsearch 會讓你少走很多彎路。

## 結語

Elasticsearch 是一個深不見底的坑，今天我們只是淺嚐了它的分析器與搜尋邏輯。在實際應用中，你還會遇到 Mapping 衝突、分片（Shards）規劃、集群健康度管理等各種問題。

但不可否認，ES 帶來的搜尋體驗是傳統資料庫難以企及的。如果你正在開發一個電商平台、內容管理系統或是監控系統，Elasticsearch 絕對值得你花時間深入研究。

想了解更多，可以參考以下資源：
*   [**Elasticsearch 官方文件**][1]
*   [**Elastic Stack (ELK) 介紹**][2]

希望這篇文章能幫你快速上手 Elasticsearch。如果有任何問題，歡迎在下方留言討論！

---

[1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
[2]: https://www.elastic.co/what-is/elk-stack
