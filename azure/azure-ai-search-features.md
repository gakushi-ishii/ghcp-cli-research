# Azure AI Search 機能詳細ガイド

> 最終更新: 2026-03-25
> 対象 API バージョン: 2025-09-01 (GA) / 2025-11-01-preview (Preview)

## Executive Summary

Azure AI Search は、Microsoft Azure が提供するフルマネージドのクラウド検索サービスであり、フルテキスト検索・ベクトル検索・セマンティック検索・ハイブリッド検索を統合した高度な情報検索プラットフォームである。インデクサーによる自動データ取り込み、AI エンリッチメント（スキルセット）、ナレッジストア、そして最新のエージェンティックリトリーバル（Agentic Retrieval）まで、RAG（Retrieval-Augmented Generation）パターンを含む幅広いユースケースに対応する[^1][^2]。

---

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [フルテキスト検索（Classic Search）](#2-フルテキスト検索classic-search)
3. [ベクトル検索（Vector Search）](#3-ベクトル検索vector-search)
4. [セマンティック検索（Semantic Search）](#4-セマンティック検索semantic-search)
5. [ハイブリッド検索（Hybrid Search）](#5-ハイブリッド検索hybrid-search)
6. [エージェンティックリトリーバル（Agentic Retrieval）](#6-エージェンティックリトリーバルagentic-retrieval)
7. [インデクサー（Indexers）](#7-インデクサーindexers)
8. [AI エンリッチメント（Skillsets）](#8-ai-エンリッチメントskillsets)
9. [ナレッジストア（Knowledge Store）](#9-ナレッジストアknowledge-store)
10. [関連度チューニング（Relevance Tuning）](#10-関連度チューニングrelevance-tuning)
11. [ユーザーエクスペリエンス機能](#11-ユーザーエクスペリエンス機能)
12. [セキュリティ](#12-セキュリティ)
13. [SDK・API リファレンス](#13-sdkapi-リファレンス)
14. [主要リポジトリ一覧](#14-主要リポジトリ一覧)

---

## 1. アーキテクチャ概要

```
┌──────────────────────────────────────────────────────────────────┐
│                        クライアントアプリケーション                    │
│            (Web App / Agent / Copilot / LLM Orchestrator)        │
└──────────┬───────────────────────────────────────┬───────────────┘
           │ REST API / Azure SDK                  │
           ▼                                       ▼
┌─────────────────────┐              ┌──────────────────────────┐
│  Azure AI Search    │              │  Azure OpenAI / LLM      │
│  サービス            │◄────────────│  (埋め込み / チャット補完)    │
│                     │              └──────────────────────────┘
│  ┌───────────────┐  │
│  │ インデックス    │  │    ┌───────────────────────┐
│  │ (テキスト+     │  │◄───│ インデクサー            │
│  │  ベクトル)     │  │    │ (データソース接続)       │
│  └───────────────┘  │    └───────┬───────────────┘
│                     │            │
│  ┌───────────────┐  │    ┌───────▼───────────────┐
│  │ スキルセット    │  │    │ Azure Blob / SQL /     │
│  │ (AI エンリッチ) │  │    │ Cosmos DB / OneLake    │
│  └───────────────┘  │    └───────────────────────┘
│                     │
│  ┌───────────────┐  │    ┌───────────────────────┐
│  │ ナレッジストア  │──┼───▶│ Azure Storage          │
│  └───────────────┘  │    │ (Table/Blob/File)      │
└─────────────────────┘    └───────────────────────┘
```

Azure AI Search サービスの主要コンポーネント[^1]:

| コンポーネント | 説明 |
|------------|------|
| **インデックス** | 検索可能なドキュメントの永続的なコレクション。テキストフィールドとベクトルフィールドの両方を格納可能 |
| **インデクサー** | 外部データソースからの自動データインポート |
| **スキルセット** | インデクシング時の AI エンリッチメントパイプライン |
| **ナレッジストア** | エンリッチメント出力の Azure Storage への永続化 |
| **ナレッジベース** | エージェンティックリトリーバル用のマルチソース検索設定（Preview） |

---

## 2. フルテキスト検索（Classic Search）

### 概要

Azure AI Search のフルテキスト検索は、Apache Lucene をベースとした転置インデックスを使用し、BM25 ランキングアルゴリズムにより関連度スコアを計算する[^3]。

### 主要機能

| 機能 | 説明 |
|------|------|
| **シンプルクエリ構文** | `search=hotel near beach` のようなキーワード検索 |
| **フル Lucene 構文** | 正規表現、ワイルドカード、近接検索、ブースト演算子 `^` をサポート |
| **フィルター** | OData フィルター式（`$filter=Rating gt 4`） |
| **ファセット** | カテゴリ別の集計結果（`facet=Category`） |
| **ソート** | 任意のフィールドでの並べ替え |
| **ページネーション** | `$top` / `$skip` パラメータ |
| **地理空間検索** | `Edm.GeographyPoint` 型による距離・範囲検索 |

### アナライザー

テキスト処理用のアナライザーが豊富に用意されている[^4]:

- **Standard Lucene アナライザー**: デフォルトの汎用アナライザー
- **言語アナライザー**: 50 以上の言語に対応（`ja.lucene` で日本語対応）
- **カスタムアナライザー**: トークナイザー + トークンフィルターを組み合わせて独自定義可能

### 使用例（REST API）

```http
POST https://{service-name}.search.windows.net/indexes/{index-name}/docs/search?api-version=2025-09-01
Content-Type: application/json
Authorization: Bearer {token}

{
  "search": "luxury hotel near beach",
  "queryType": "simple",
  "select": "HotelName, Description, Rating",
  "filter": "Rating ge 4",
  "facets": ["Category"],
  "orderby": "Rating desc",
  "top": 10
}
```

### 使用例（Python SDK）

```python
from azure.search.documents import SearchClient
from azure.identity import DefaultAzureCredential

client = SearchClient(
    endpoint="https://<service-name>.search.windows.net",
    index_name="hotels-sample",
    credential=DefaultAzureCredential()
)

results = client.search(
    search_text="luxury hotel near beach",
    select=["HotelName", "Description", "Rating"],
    filter="Rating ge 4",
    facets=["Category"],
    order_by=["Rating desc"],
    top=10
)

for result in results:
    print(f"{result['HotelName']}: {result['@search.score']}")
```
[^5]

---

## 3. ベクトル検索（Vector Search）

### 概要

ベクトル検索は、埋め込みモデル（Azure OpenAI `text-embedding-3-small/large` など）で生成されたベクトル表現に基づき、意味的類似性で検索結果を返す機能である[^6]。

### インデックス設定

ベクトルインデックスの作成には以下の 3 つの構成要素が必要[^7]:

1. **アルゴリズム構成** (`vectorSearch.algorithms`)
2. **圧縮構成** (`vectorSearch.compressions`) — オプション
3. **プロファイル** (`vectorSearch.profiles`)

#### アルゴリズム

| アルゴリズム | 説明 |
|-----------|------|
| **HNSW** (Hierarchical Navigable Small World) | 近似最近傍探索。高速だが近似的。パラメータ: `m`(4-10), `efConstruction`(100-1000), `efSearch`(100-1000) |
| **Exhaustive KNN** | 完全探索。精度は最高だが低速。小規模データまたは検証用 |

#### 距離メトリック

| メトリック | 用途 |
|----------|------|
| `cosine` | Azure OpenAI モデル推奨 |
| `dotProduct` | 正規化済みベクトル向け |
| `euclidean` | ユークリッド距離 |
| `hamming` | バイナリデータ用 |

#### 圧縮（Quantization）

| 圧縮方式 | 説明 |
|---------|------|
| **スカラー量子化** | float → int8 に圧縮。メモリ使用量を削減 |
| **バイナリ量子化** | float → 1bit に圧縮。最もメモリ効率が良い |

圧縮使用時は `rescoringOptions` で元のベクトルによるリスコアリングが可能（デフォルト有効）[^7]。

### インデックス定義例

```json
{
  "name": "vector-index",
  "fields": [
    { "name": "id", "type": "Edm.String", "key": true, "filterable": true },
    { "name": "title", "type": "Edm.String", "searchable": true },
    { "name": "content", "type": "Edm.String", "searchable": true },
    {
      "name": "contentVector",
      "type": "Collection(Edm.Single)",
      "searchable": true,
      "retrievable": false,
      "stored": false,
      "dimensions": 1536,
      "vectorSearchProfile": "my-vector-profile"
    }
  ],
  "vectorSearch": {
    "algorithms": [
      {
        "name": "my-hnsw",
        "kind": "hnsw",
        "hnswParameters": {
          "m": 4,
          "efConstruction": 400,
          "efSearch": 500,
          "metric": "cosine"
        }
      }
    ],
    "compressions": [
      {
        "name": "my-scalar",
        "kind": "scalarQuantization",
        "scalarQuantizationParameters": {
          "quantizedDataType": "int8"
        },
        "rescoringOptions": {
          "enableRescoring": true,
          "defaultOversampling": 10
        }
      }
    ],
    "profiles": [
      {
        "name": "my-vector-profile",
        "algorithm": "my-hnsw",
        "compression": "my-scalar"
      }
    ]
  }
}
```
[^7]

### ベクトルクエリ（Python SDK）

```python
from azure.search.documents.models import VectorizedQuery

vector_query = VectorizedQuery(
    vector=embedding_vector,        # 事前計算済みの埋め込みベクトル
    k_nearest_neighbors=10,
    fields="contentVector",
    exhaustive=True                 # True=Exhaustive KNN, False=HNSW
)

results = client.search(
    search_text=None,               # ベクトルのみの検索
    vector_queries=[vector_query],
    select=["title", "content"],
    top=10
)
```
[^8]

### 統合ベクトル化（Integrated Vectorization）

インデクサーのスキルセットと連携し、テキストデータを自動でベクトル化する機能。手動での埋め込み計算が不要になる[^9]。

```
データソース → インデクサー → テキスト分割スキル → 埋め込みスキル → インデックス
                              (チャンク化)        (ベクトル化)     (テキスト+ベクトル格納)
```

---

## 4. セマンティック検索（Semantic Search）

### 概要

セマンティックランキング（Semantic Ranker）は、Microsoft の機械学習モデルを使用して、BM25 による初期結果をクエリの意図と文脈に基づいて再ランキングする機能である[^10]。

### セマンティック構成（Semantic Configuration）

インデックスに対してセマンティック構成を定義する必要がある[^11]:

| プロパティ | 説明 |
|----------|------|
| `titleField` | ドキュメントのタイトル（25 語以下推奨） |
| `contentFields` | 自然言語のテキストフィールド（複数指定可能） |
| `keywordFields` | キーワード・タグフィールド（複数指定可能） |

```json
{
  "semanticSearch": {
    "defaultConfiguration": "my-semantic-config",
    "configurations": [
      {
        "name": "my-semantic-config",
        "prioritizedFields": {
          "titleField": { "fieldName": "HotelName" },
          "contentFields": [
            { "fieldName": "Description" }
          ],
          "keywordFields": [
            { "fieldName": "Tags" }
          ]
        }
      }
    ]
  }
}
```

### クエリ実行

```json
{
  "search": "walking distance to live music",
  "queryType": "semantic",
  "semanticConfiguration": "my-semantic-config",
  "captions": "extractive|highlight-true",
  "answers": "extractive|count-3",
  "select": "HotelId, HotelName, Description"
}
```

### セマンティック検索の 3 つの機能

| 機能 | 説明 | レスポンスフィールド |
|------|------|-----------------|
| **セマンティックランキング** | ML モデルによる結果の再ランキング | `@search.rerankerScore` |
| **キャプション** | 各結果から最も関連性の高いパッセージを抽出 | `@search.captions` |
| **アンサー** | クエリが質問形式の場合、直接的な回答を抽出 | `@search.answers` |

### 前提条件

- Basic 以上の価格レベル（Free では利用不可）
- セマンティックランキングの[有効化](https://learn.microsoft.com/azure/search/semantic-how-to-enable-disable)が必要
- [対応リージョン](https://learn.microsoft.com/azure/search/search-region-support)で利用可能[^10]

---

## 5. ハイブリッド検索（Hybrid Search）

### 概要

ハイブリッド検索は、フルテキスト検索（BM25）とベクトル検索を単一のリクエスト内で同時に実行し、**Reciprocal Rank Fusion (RRF)** アルゴリズムで結果をマージする機能である[^12]。

### なぜハイブリッド検索が有効か

```
┌─────────────────────────────────────────────────────┐
│                 ハイブリッドクエリ                      │
│  search_text = "historic hotel"                     │
│  vector_query = [0.12, -0.03, ...]                  │
└───────────┬────────────────────┬────────────────────┘
            │                    │
     ┌──────▼──────┐     ┌──────▼──────┐
     │ BM25 検索    │     │ ベクトル検索  │
     │ (キーワード   │     │ (意味的類似性) │
     │  マッチング)  │     │              │
     └──────┬──────┘     └──────┬──────┘
            │                    │
     ┌──────▼────────────────────▼──────┐
     │       RRF (Reciprocal Rank       │
     │         Fusion) マージ           │
     └──────────────┬───────────────────┘
                    │
     ┌──────────────▼───────────────────┐
     │   セマンティックランキング（任意）   │
     │     (ML による再ランキング)        │
     └──────────────┬───────────────────┘
                    │
                    ▼
              最終結果セット
```

### 使用例（Python SDK）

```python
from azure.search.documents.models import VectorizedQuery

vector_query = VectorizedQuery(
    vector=query_vector,
    k_nearest_neighbors=10,
    fields="DescriptionVector",
    exhaustive=True
)

results = client.search(
    search_text="historic hotel walk to restaurants and shopping",
    vector_queries=[vector_query],
    select=["HotelName", "Description", "Address/City"],
    top=10
)
```
[^8]

### 使用例（REST API）

```http
POST https://{service}.search.windows.net/indexes/{index}/docs/search?api-version=2025-09-01
Content-Type: application/json

{
  "search": "historic hotel walk to restaurants and shopping",
  "vectorQueries": [
    {
      "vector": [-0.009154, 0.018708, ...],
      "fields": "DescriptionVector",
      "kind": "vector",
      "exhaustive": true,
      "k": 10
    }
  ],
  "select": "HotelName, Description, Address/City",
  "top": 10
}
```
[^13]

### ハイブリッド + セマンティック

ハイブリッド検索にセマンティックランキングを追加することで、最も高い関連度が得られる:

```json
{
  "search": "historic hotel walk to restaurants",
  "vectorQueries": [{ "vector": [...], "fields": "DescriptionVector", "kind": "vector", "k": 10 }],
  "queryType": "semantic",
  "semanticConfiguration": "my-semantic-config",
  "captions": "extractive|highlight-true",
  "top": 10
}
```

---

## 6. エージェンティックリトリーバル（Agentic Retrieval）

### 概要

エージェンティックリトリーバルは、LLM を活用した複雑な質問に対するマルチクエリパイプラインであり、RAG パターンのために設計されたプレビュー機能である[^14]。

### 動作の流れ

```
┌────────────────┐
│  ユーザーの質問   │  "2023年以降に入社したリモートワーカーの有給休暇は？"
└───────┬────────┘
        │
┌───────▼────────┐
│  LLM による      │  複雑な質問を分解:
│  クエリ計画      │  → "有給休暇ポリシー"
│                 │  → "リモートワーカー規定"
│                 │  → "2023年入社条件"
└───────┬────────┘
        │ 並列実行
   ┌────┼────┐
   ▼    ▼    ▼
┌────┐┌────┐┌────┐
│サブ ││サブ ││サブ │  各サブクエリはセマンティックランキング付き
│Q1  ││Q2  ││Q3  │
└──┬─┘└──┬─┘└──┬─┘
   │     │     │
   └──┬──┘──┬──┘
      │     │
┌─────▼─────▼────┐
│  結果マージ +     │
│  セマンティック    │
│  リランキング     │
└───────┬────────┘
        ▼
┌────────────────┐
│  構造化レスポンス  │  グラウンディングデータ + 引用 + 実行計画
└────────────────┘
```

### 主要コンポーネント

| コンポーネント | 説明 |
|------------|------|
| **ナレッジソース** | インデックス付きコンテンツ、SharePoint、OneLake、Web などのデータソース接続 |
| **ナレッジベース** | 複数のナレッジソースを統合し、検索パイプラインを管理 |
| **リトリーブアクション** | アプリケーションコードから呼び出す検索実行アクション |
| **推論エフォート** | LLM の関与度を調整（minimal / low / medium） |

### Classic RAG との比較

| 項目 | エージェンティックリトリーバル | Classic RAG |
|------|--------------------------|------------|
| クエリ計画 | LLM がサブクエリに分解 | 単一クエリ |
| 実行方式 | 並列サブクエリ | 単一リクエスト |
| レスポンス | 構造化（引用・実行計画付き） | フラットな結果セット |
| セマンティックランキング | 組み込み | 別途設定が必要 |
| ステータス | Preview | GA |
| 推奨用途 | 新規 RAG 実装 / エージェント | 既存コードの維持 / シンプルさ重視 |

[^14][^15]

### 使用開始

```python
# ナレッジベースの作成例（REST API を使用）
# 詳細: https://learn.microsoft.com/azure/search/search-get-started-agentic-retrieval
```

---

## 7. インデクサー（Indexers）

### 概要

インデクサーは、外部データソースから自動的にデータをインポートし、検索インデックスに取り込む「プルモデル」のデータ取得機構である[^16]。

### サポートされるデータソース

| データソース | ステータス |
|-----------|----------|
| Azure Blob Storage | GA |
| Azure Cosmos DB (NoSQL) | GA |
| Azure Data Lake Storage Gen2 | GA |
| Azure SQL Database | GA |
| Azure Table Storage | GA |
| Azure SQL Managed Instance | GA |
| Microsoft OneLake | GA |
| SQL Server on Azure VMs | GA |
| Azure Files | Preview |
| Azure MySQL | Preview |
| SharePoint in Microsoft 365 | Preview |
| Azure Cosmos DB for MongoDB | Preview |
| Azure Cosmos DB for Apache Gremlin | Preview |
| Logic Apps コネクタ | Preview |

[^16]

### インデクサーの処理段階

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Stage 1       │    │ Stage 2       │    │ Stage 3       │    │ Stage 4       │
│ ドキュメント   │───▶│ フィールド     │───▶│ スキルセット   │───▶│ インデックス   │
│ クラッキング   │    │ マッピング     │    │ 実行（任意）   │    │ への出力       │
│ (ファイル解析)  │    │ (ソース→宛先)  │    │ (AI エンリッチ) │    │               │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### インデクサー定義例（REST API）

```http
POST https://{service}.search.windows.net/indexers?api-version=2025-09-01
Content-Type: application/json

{
  "name": "my-blob-indexer",
  "dataSourceName": "my-blob-datasource",
  "targetIndexName": "my-index",
  "skillsetName": "my-skillset",
  "schedule": {
    "interval": "PT2H"
  },
  "parameters": {
    "configuration": {
      "parsingMode": "json",
      "imageAction": "generateNormalizedImages"
    }
  },
  "fieldMappings": [
    { "sourceFieldName": "metadata_storage_path", "targetFieldName": "id", "mappingFunction": { "name": "base64Encode" } },
    { "sourceFieldName": "metadata_storage_name", "targetFieldName": "title" }
  ],
  "outputFieldMappings": [
    { "sourceFieldName": "/document/content/organizations", "targetFieldName": "organizations" }
  ]
}
```

### 変更検出と削除検出

- **変更検出**: Blob Storage は自動検出。SQL/Cosmos DB は高水位ポリシーで設定
- **削除検出**: ソフトデリートポリシーを設定して検出可能
- **スケジュール**: 最短 5 分間隔で定期実行可能[^16]

---

## 8. AI エンリッチメント（Skillsets）

### 概要

AI エンリッチメントは、インデクサーパイプラインの一部として、ドキュメントからの情報抽出・変換・エンリッチを行うスキルセットの仕組みである[^17]。

### 組み込みスキル一覧

| カテゴリ | スキル名 | 説明 |
|---------|---------|------|
| **Vision** | OCR | 画像からのテキスト抽出 |
| **Vision** | Image Analysis | 画像の説明・タグ生成 |
| **Language** | Entity Recognition | 人名・組織・場所等のエンティティ抽出 |
| **Language** | Language Detection | 言語検出 |
| **Language** | Key Phrase Extraction | キーフレーズ抽出 |
| **Language** | Sentiment Analysis | 感情分析 |
| **Language** | Text Translation | テキスト翻訳 |
| **Language** | PII Detection | 個人情報検出 |
| **Utility** | Text Split | テキストの分割（チャンク化） |
| **Utility** | Text Merge | テキストの結合 |
| **Utility** | Shaper | 出力の整形（ナレッジストア用） |
| **Embedding** | Azure OpenAI Embedding | テキストのベクトル化（統合ベクトル化） |
| **Custom** | Web API Skill | 外部 API 呼び出し（Azure Functions 等） |

[^17]

### スキルセット定義例

```json
{
  "name": "my-skillset",
  "description": "AI enrichment pipeline",
  "skills": [
    {
      "@odata.type": "#Microsoft.Skills.Text.V3.EntityRecognitionSkill",
      "name": "entity-recognition",
      "categories": ["Organization", "Person", "Location"],
      "defaultLanguageCode": "ja",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "organizations", "targetName": "organizations" },
        { "name": "persons", "targetName": "persons" },
        { "name": "locations", "targetName": "locations" }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.KeyPhraseExtractionSkill",
      "name": "keyphrases",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "keyPhrases", "targetName": "keyPhrases" }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.SplitSkill",
      "name": "text-split",
      "textSplitMode": "pages",
      "maximumPageLength": 2000,
      "pageOverlapLength": 500,
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "textItems", "targetName": "chunks" }
      ]
    }
  ],
  "cognitiveServices": {
    "@odata.type": "#Microsoft.Azure.Search.CognitiveServicesByKey",
    "key": "<cognitive-services-key>"
  }
}
```

### カスタムスキル（Azure Functions 連携）

```json
{
  "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
  "name": "custom-translation",
  "uri": "https://my-function.azurewebsites.net/api/translate",
  "httpHeaders": {
    "x-functions-key": "<function-key>"
  },
  "batchSize": 10,
  "inputs": [
    { "name": "text", "source": "/document/content" }
  ],
  "outputs": [
    { "name": "translatedText", "targetName": "translatedContent" }
  ]
}
```
[^17]

---

## 9. ナレッジストア（Knowledge Store）

### 概要

ナレッジストアは、AI エンリッチメントの出力を Azure Storage に永続化する機能であり、検索インデックスとは別に、テーブル・オブジェクト・ファイルとしてエンリッチ済みコンテンツを格納する[^18]。

### プロジェクションの種類

| プロジェクション | ストレージ | 用途 |
|---------------|----------|------|
| **テーブル** | Azure Table Storage | 行と列で表現可能なデータ。分析ツール・BI 向け |
| **オブジェクト** | Azure Blob Storage | JSON 形式の完全なドキュメント |
| **ファイル** | Azure Blob Storage | 正規化済み画像ファイル |

### 定義例

```json
{
  "knowledgeStore": {
    "storageConnectionString": "DefaultEndpointsProtocol=https;AccountName=<acct>;AccountKey=<key>;",
    "projections": [
      {
        "tables": [
          {
            "tableName": "documentsTable",
            "generatedKeyName": "documentKey",
            "source": "/document/shapedData"
          },
          {
            "tableName": "keyPhrasesTable",
            "generatedKeyName": "keyPhraseKey",
            "source": "/document/shapedData/keyPhrases/*"
          }
        ],
        "objects": [
          {
            "storageContainer": "enriched-documents",
            "source": "/document/shapedData"
          }
        ],
        "files": [
          {
            "storageContainer": "document-images",
            "source": "/document/normalized_images/*"
          }
        ]
      }
    ]
  }
}
```

### プロジェクショングループの特性

- **相互排他性**: 各グループは他のグループから完全に独立
- **関連性**: 同一グループ内のテーブル間はリレーションシップが保持される
- 複数のシナリオに対応するため、グループを分けて定義可能[^18]

---

## 10. 関連度チューニング（Relevance Tuning）

### 概要

検索結果の関連度を向上させるための複数のチューニング手法が提供されている[^19]。

### チューニング手法一覧

| 手法 | 対象 | 説明 |
|------|------|------|
| **スコアリングプロファイル** | テキスト/数値フィールド | フィールドの重み付け、関数（freshness/magnitude/distance/tag）によるブースト |
| **セマンティックランキング** | テキストフィールド | ML モデルによる意味的再ランキング |
| **BM25 パラメータ調整** | テキストフィールド | ドキュメント長と用語頻度の影響度を調整 |
| **ベクトルフィールド重み** | ハイブリッド検索 | ベクトル成分の重要度を調整 |
| **ブースト演算子** | Full Lucene 構文 | `term^2` で特定語句をブースト |

### スコアリングプロファイル定義例

```json
{
  "scoringProfiles": [
    {
      "name": "boost-by-rating",
      "text": {
        "weights": {
          "HotelName": 2.0,
          "Description": 1.5
        }
      },
      "functions": [
        {
          "type": "magnitude",
          "fieldName": "Rating",
          "boost": 10,
          "interpolation": "linear",
          "magnitude": {
            "boostingRangeStart": 1,
            "boostingRangeEnd": 5,
            "constantBoostBeyondRange": false
          }
        },
        {
          "type": "freshness",
          "fieldName": "LastRenovationDate",
          "boost": 5,
          "freshness": {
            "boostingDuration": "P365D"
          }
        },
        {
          "type": "distance",
          "fieldName": "Location",
          "boost": 5,
          "distance": {
            "referencePointParameter": "currentLocation",
            "boostingDistance": 10
          }
        }
      ],
      "functionAggregation": "sum"
    }
  ],
  "defaultScoringProfile": "boost-by-rating"
}
```

### シノニムマップ

同義語をマッピングして検索範囲を暗黙的に拡張する機能[^20]:

```json
POST /synonymmaps?api-version=2025-09-01
{
  "name": "geo-synonyms",
  "format": "solr",
  "synonyms": "USA, United States, United States of America\nWashington, Wash., WA => WA\n"
}
```

- **等価ルール**: `USA, United States` — 双方向展開
- **明示的マッピング**: `WA, Washington => WA` — 一方向のみ
- フィールドの `synonymMapNames` プロパティで割り当て

---

## 11. ユーザーエクスペリエンス機能

### オートコンプリートとサジェスト

| 機能 | 説明 |
|------|------|
| **Autocomplete** | 部分入力から検索用語を補完 |
| **Suggestions** | 部分入力からインデックス内の実際のドキュメントを提案 |

Suggester の定義が必要[^21]:

```json
{
  "suggesters": [
    {
      "name": "sg",
      "searchMode": "analyzingInfixMatching",
      "sourceFields": ["HotelName", "Category"]
    }
  ]
}
```

### ヒットハイライティング

検索結果中のマッチしたキーワードにハイライト（`<em>` タグ）を適用:

```json
{
  "search": "luxury hotel",
  "highlight": "Description",
  "highlightPreTag": "<mark>",
  "highlightPostTag": "</mark>"
}
```

---

## 12. セキュリティ

### 認証方式

| 方式 | 説明 | 推奨度 |
|------|------|--------|
| **RBAC (Microsoft Entra ID)** | ロールベースアクセス制御。条件付きアクセス、監査証跡あり | **推奨** |
| **API キー** | 管理キー / クエリキー | レガシー（移行推奨） |

### 主要ロール

| ロール | 権限 |
|-------|------|
| `Search Service Contributor` | サービス管理（インデックス、インデクサー等の作成・削除） |
| `Search Index Data Contributor` | インデックスデータの読み書き |
| `Search Index Data Reader` | インデックスデータの読み取りのみ |

### ネットワークセキュリティ

| 機能 | 説明 |
|------|------|
| **IP ファイアウォール** | 受信元 IP アドレス範囲の制限 |
| **プライベートエンドポイント** | Azure Private Link による VNet 経由のアクセス |
| **ネットワークセキュリティ境界** | Azure NSP によるリソース間のネットワーク一括管理 |

### データ暗号化

| 方式 | 説明 |
|------|------|
| **Microsoft 管理キー** | 組み込み。透過的な保存時暗号化 |
| **カスタマーマネージドキー (CMK)** | Azure Key Vault の鍵によるインデックス・シノニムマップの追加暗号化 |

### ドキュメントレベルセキュリティ

- フィルターベースのセキュリティトリミング（ACL フィールドをインデックスに追加し、`$filter` でアクセス制御）
- Azure Storage からのアクセス許可メタデータの継承
- SharePoint のアクセス許可の継承（エージェンティックリトリーバル使用時）[^22]

---

## 13. SDK・API リファレンス

### REST API

| エンドポイント | 用途 |
|-------------|------|
| `POST /indexes` | インデックス作成 |
| `POST /indexes/{name}/docs/search` | 検索クエリ実行 |
| `POST /indexers` | インデクサー作成 |
| `POST /skillsets` | スキルセット作成 |
| `POST /datasources` | データソース接続定義 |
| `POST /synonymmaps` | シノニムマップ作成 |
| `POST /indexes/{name}/docs/index` | ドキュメントのアップロード/更新/削除 |

API バージョン: `api-version=2025-09-01`（GA）

### Azure SDK

| 言語 | パッケージ | 主要クライアント |
|------|----------|---------------|
| **.NET** | `Azure.Search.Documents` | `SearchClient`, `SearchIndexClient`, `SearchIndexerClient` |
| **Python** | `azure-search-documents` | `SearchClient`, `SearchIndexClient`, `SearchIndexerClient` |
| **Java** | `com.azure:azure-search-documents` | `SearchClient`, `SearchIndexClient`, `SearchIndexerClient` |
| **JavaScript** | `@azure/search-documents` | `SearchClient`, `SearchIndexClient`, `SearchIndexerClient` |

### Python SDK 使用例 — 完全なワークフロー

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    SimpleField,
    SearchableField,
    VectorSearch,
    HnswAlgorithmConfiguration,
    VectorSearchProfile,
    SemanticConfiguration,
    SemanticSearch,
    SemanticPrioritizedFields,
    SemanticField
)

endpoint = "https://<service-name>.search.windows.net"
credential = DefaultAzureCredential()

# 1. インデックス作成
index_client = SearchIndexClient(endpoint=endpoint, credential=credential)

fields = [
    SimpleField(name="id", type=SearchFieldDataType.String, key=True),
    SearchableField(name="title", type=SearchFieldDataType.String, sortable=True),
    SearchableField(name="content", type=SearchFieldDataType.String, analyzer_name="ja.lucene"),
    SearchableField(name="category", type=SearchFieldDataType.String, filterable=True, facetable=True),
    SearchField(
        name="contentVector",
        type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
        searchable=True,
        vector_search_dimensions=1536,
        vector_search_profile_name="my-profile"
    )
]

vector_search = VectorSearch(
    algorithms=[HnswAlgorithmConfiguration(name="my-hnsw")],
    profiles=[VectorSearchProfile(name="my-profile", algorithm_configuration_name="my-hnsw")]
)

semantic_config = SemanticConfiguration(
    name="my-semantic-config",
    prioritized_fields=SemanticPrioritizedFields(
        title_field=SemanticField(field_name="title"),
        content_fields=[SemanticField(field_name="content")]
    )
)
semantic_search = SemanticSearch(configurations=[semantic_config])

index = SearchIndex(
    name="my-index",
    fields=fields,
    vector_search=vector_search,
    semantic_search=semantic_search
)

index_client.create_or_update_index(index)

# 2. ドキュメントアップロード
search_client = SearchClient(endpoint=endpoint, index_name="my-index", credential=credential)

documents = [
    {"id": "1", "title": "東京タワー", "content": "東京タワーは...", "category": "観光地",
     "contentVector": [0.1, 0.2, ...]},
]

search_client.upload_documents(documents=documents)

# 3. ハイブリッド検索 + セマンティックランキング
from azure.search.documents.models import VectorizedQuery

results = search_client.search(
    search_text="東京の観光スポット",
    vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="contentVector")],
    query_type="semantic",
    semantic_configuration_name="my-semantic-config",
    select=["title", "content", "category"],
    top=5
)
```
[^5][^8]

---

## 14. 主要リポジトリ一覧

| リポジトリ | 説明 |
|----------|------|
| [Azure/azure-search-vector-samples](https://github.com/Azure/azure-search-vector-samples) | ベクトル検索のサンプルコード集（Python, .NET, Java, JS） |
| [Azure-Samples/azure-search-rest-samples](https://github.com/Azure-Samples/azure-search-rest-samples) | REST API サンプル（ACL, カスタムアナライザー, ナレッジストア等） |
| [Azure-Samples/azure-search-dotnet-samples](https://github.com/Azure-Samples/azure-search-dotnet-samples) | .NET SDK サンプル |
| [Azure-Samples/azure-search-classic-rag](https://github.com/Azure-Samples/azure-search-classic-rag) | Classic RAG パターンの実装例 |
| [Azure-Samples/azure-search-sample-data](https://github.com/Azure-Samples/azure-search-sample-data) | サンプルデータセット（ホテル、NASA 地球画像等） |

---

## Confidence Assessment

| 項目 | 信頼度 | 備考 |
|------|--------|------|
| フルテキスト検索の機能・API | ✅ 高 | GA 機能、公式ドキュメントで確認済み |
| ベクトル検索の設定・クエリ | ✅ 高 | GA 機能、REST API 仕様で確認済み |
| セマンティック検索の設定 | ✅ 高 | GA 機能、公式ドキュメントで確認済み |
| ハイブリッド検索の動作 | ✅ 高 | GA 機能、RRF アルゴリズムは公式ドキュメントで確認済み |
| エージェンティックリトリーバル | ⚠️ 中〜高 | Preview 機能。API が変更される可能性あり |
| インデクサー・データソース | ✅ 高 | GA 機能、サポートリスト確認済み |
| AI エンリッチメント | ✅ 高 | GA 機能、スキル一覧は公式ドキュメントで確認済み |
| ナレッジストア | ✅ 高 | GA 機能、プロジェクション定義は公式ドキュメントで確認済み |
| セキュリティ | ✅ 高 | 公式ドキュメントで確認済み |
| コードサンプル | ✅ 高 | 公式サンプルリポジトリおよび Microsoft Learn から取得 |

---

## Footnotes

[^1]: [Features of Azure AI Search](https://learn.microsoft.com/azure/search/search-features-list) — Azure AI Search の全機能一覧
[^2]: [Introduction to Azure AI Search](https://learn.microsoft.com/azure/search/search-what-is-azure-search) — サービス概要
[^3]: [Full-text search in Azure AI Search](https://learn.microsoft.com/azure/search/search-lucene-query-architecture) — BM25 スコアリングの詳細
[^4]: [Analyzers in Azure AI Search](https://learn.microsoft.com/azure/search/search-analyzers) — アナライザーの概要と設定
[^5]: [Quickstart: Full-text search using Python](https://learn.microsoft.com/azure/search/search-get-started-text?pivots=python) — Python SDK でのインデックス作成・検索
[^6]: [Vector search overview](https://learn.microsoft.com/azure/search/vector-search-overview) — ベクトル検索の概要
[^7]: [Create a vector index](https://learn.microsoft.com/azure/search/vector-search-how-to-create-index) — ベクトルインデックスの作成手順
[^8]: [Hybrid search how-to query](https://learn.microsoft.com/azure/search/hybrid-search-how-to-query) — ハイブリッド検索クエリの Python/REST サンプル
[^9]: [Integrated vectorization](https://learn.microsoft.com/azure/search/vector-search-integrated-vectorization) — 統合ベクトル化の設定
[^10]: [Semantic ranking overview](https://learn.microsoft.com/azure/search/semantic-search-overview) — セマンティックランキングの概要
[^11]: [Configure semantic ranker](https://learn.microsoft.com/azure/search/semantic-how-to-configure) — セマンティック構成の設定方法
[^12]: [Hybrid search overview](https://learn.microsoft.com/azure/search/hybrid-search-overview) — ハイブリッド検索と RRF の解説
[^13]: [Hybrid search how-to query (REST)](https://learn.microsoft.com/azure/search/hybrid-search-how-to-query#set-up-a-hybrid-query) — REST API でのハイブリッドクエリ例
[^14]: [Agentic retrieval overview](https://learn.microsoft.com/azure/search/agentic-retrieval-overview) — エージェンティックリトリーバルの概要
[^15]: [RAG in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview) — RAG パターンの全体像
[^16]: [Indexers in Azure AI Search](https://learn.microsoft.com/azure/search/search-indexer-overview) — インデクサーの概要とサポートデータソース
[^17]: [AI enrichment in Azure AI Search](https://learn.microsoft.com/azure/search/cognitive-search-concept-intro) — AI エンリッチメントの概要
[^18]: [Knowledge store concept](https://learn.microsoft.com/azure/search/knowledge-store-concept-intro) — ナレッジストアの概要とプロジェクション
[^19]: [Relevance in Azure AI Search](https://learn.microsoft.com/azure/search/search-relevance-overview) — 関連度チューニングの概要
[^20]: [Add synonyms](https://learn.microsoft.com/azure/search/search-synonyms) — シノニムマップの作成と割り当て
[^21]: [Autocomplete and suggestions](https://learn.microsoft.com/azure/search/search-add-autocomplete-suggestions) — オートコンプリートとサジェスト
[^22]: [Secure an Azure AI Search service](https://learn.microsoft.com/azure/search/search-security-best-practices) — セキュリティのベストプラクティス
