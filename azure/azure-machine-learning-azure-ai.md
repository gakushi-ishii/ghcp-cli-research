# Azure Machine Learning で出来ること & Azure AI サービス全体像（2026年4月時点）

## Executive Summary

Azure Machine Learning（Azure ML）は、データ準備からモデル学習・デプロイ・運用監視まで、機械学習の全ライフサイクルをカバーするマネージドサービスである。AutoML、Managed Online Endpoints、Prompt Flow、Responsible AI ダッシュボードなどの主要機能を提供し、カスタム ML モデル開発の中核プラットフォームとして位置付けられている。一方、2025〜2026年にかけて SDK v1 の廃止や一部プレビュー機能の終了、Azure ML Studio (classic) の完全終了など、大幅なサービス整理が進行中である。また、Microsoft のAI サービス全体は「Microsoft Foundry（旧 Azure AI Foundry / Azure AI Studio）」を中心に再編されており、生成 AI・エージェント開発は Foundry、カスタム ML は Azure ML、プリビルト AI API は Azure AI Services（旧 Cognitive Services）という棲み分けが明確化されている。

---

## 1. Azure Machine Learning の主要機能

### 1.1 AutoML（自動機械学習）

自動でアルゴリズム選択、ハイパーパラメータチューニング、特徴量エンジニアリングを行い、最適なモデルを構築する機能。分類・回帰・時系列予測・コンピュータビジョン・NLP に対応[^1][^2]。

- データサイエンスの専門知識が限定的でも高品質なモデルを構築可能
- SDK v2 および Azure ML Studio の GUI から利用可能
- Microsoft Fabric の AutoML としても利用可能（Power BI 統合の後継）

### 1.2 Managed Online Endpoints / Batch Endpoints

リアルタイム推論用の Managed Online Endpoints とバッチ推論用の Batch Endpoints を提供[^3][^4]。

- HTTPS エンドポイントの即時作成
- Blue/Green デプロイメントによる安全なロールアウト・A/B テスト
- 自動スケーリング（CPU / GPU 対応）
- MLflow モデルおよび非 MLflow モデルの両方をサポート
- ネットワーク分離・マネージド VNet との統合

### 1.3 サーバーレスコンピュート

VM のプロビジョニング不要でトレーニングジョブや推論を実行可能[^5]。

- オンデマンドでコンピュートリソースを管理
- Standard VM および Spot（低優先度）VM をサポート
- ファインチューニング、環境構築、Responsible AI タスクにも対応

### 1.4 モデルカタログ

Microsoft、OpenAI、Hugging Face、Meta、Cohere 等のファウンデーションモデルを集約したリポジトリ[^6]。

- Managed Online Endpoints またはサーバーレスデプロイメント（Standard Deployments）として直接デプロイ可能
- 従量課金制のサーバーレスデプロイメント対応
- Responsible AI ダッシュボードとの統合

### 1.5 Prompt Flow

LLM を活用したアプリケーションの開発・評価・デプロイを効率化するツール[^7]。

- ビジュアルなプロンプトワークフロー作成
- プロンプトバリアント管理・評価
- エンドポイントデプロイとモニタリング統合
- **ステータス**: GA（一般提供）

### 1.6 ML パイプライン

データ準備→学習→評価→デプロイの一連のワークフローをオーケストレーション[^8]。

- Azure ML SDK v2 / CLI v2 で定義
- スケジュール実行・イベントトリガー対応
- 再利用可能なコンポーネントベースの設計

### 1.7 Feature Store（特徴量ストア）

ML 特徴量の一元管理・共有・再利用を可能にする機能[^6]。

- バージョニングとガバナンス
- トレーニングと推論間の一貫性確保
- クロスチーム・クロスワークスペースでのコラボレーション

### 1.8 Data Labeling（データラベリング）

画像やテキストのラベリングプロジェクトを管理する機能[^6]。

- ML 支援ラベリング（自動ラベル提案）
- 複数ラベラーの管理・品質チェック
- Azure ML パイプラインとの統合

### 1.9 Responsible AI ダッシュボード

モデルの公平性・解釈可能性・エラー分析・反事実分析・因果分析を統合的に提供[^9][^10]。

| コンポーネント | 説明 | ステータス |
|---|---|---|
| Error Analysis | モデルエラーの分布・パターン分析 | GA |
| Fairness Assessment | 公平性メトリクスの評価 | GA |
| Model Interpretability | 特徴量重要度・説明可能性 | GA |
| Counterfactual What-if | 反事実シナリオ分析 | GA |
| Causal Analysis | 因果推論 | GA |
| Responsible AI Scorecard | PDF 出力可能な監査用スコアカード | **Preview** |

### 1.10 Model Monitor（モデルモニタリング）

デプロイ済みモデルのパフォーマンス・データドリフト・安全性を監視[^11]。

- 従来 ML モデル向け: GA
- **生成 AI アプリケーション向け**: **Preview** — Groundedness、Coherence、Fluency、Relevance 等の評価メトリクス対応
- Model Data Collector による本番データ収集
- アラート設定・定期レポート

### 1.11 Designer（デザイナー）

ドラッグ＆ドロップの GUI でパイプラインを構築する機能[^12]。

- コーディング不要で ML パイプラインを設計
- **注意**: 一部プレビュー機能が 2026年3月31日に廃止予定（後述）

### 1.12 コンピュートインスタンス / クラスター

開発用のマネージド VM（コンピュートインスタンス）と学習用のスケーラブルなコンピュートクラスターを提供。

- GPU / CPU 対応
- 分散トレーニングサポート
- アイドル時自動シャットダウン

---

## 2. 廃止済み・廃止予定の機能

### 2.1 既に廃止済み

| 機能 | 廃止日 | 後継 |
|---|---|---|
| **Azure ML Studio (classic)** | 2024年8月31日 | Azure Machine Learning（現行版）[^13] |
| **LUIS（Language Understanding）** | 廃止済み | Azure AI Language / Conversational Language Understanding[^14] |

### 2.2 廃止予定（2025〜2026年）

| 機能 | 廃止日 | 後継・対応 |
|---|---|---|
| **Azure ML SDK v1** | 2025年3月31日 非推奨化 / 2026年6月30日 サポート終了 | Azure ML SDK v2 へ移行[^15] |
| **Power BI 内の Cognitive Services / Azure ML 統合** | 2025年9月15日 完全廃止（新規作成は2025年8月11日より不可） | Microsoft Fabric の AutoML / Azure AI REST API[^16] |
| **Designer プレビュー機能**（ステップグルーピング、パイプラインジョブ比較、データラベリングプロジェクトへのインポート、v2 データの使用） | 2026年3月31日 | GA 機能への移行[^17] |
| **Data Drift (preview)** | 廃止済み | Model Monitor に置換[^18] |
| **Synapse Analytics と Azure ML のリンク / Apache Spark プール接続** | SDK v1 廃止に伴い非推奨 | Azure ML SDK v2 のネイティブ Spark 統合[^19] |
| **Image Analysis 4.0（Azure Vision）** | 2025年9月非推奨化 / 2028年9月完全廃止 | 新しい Vision API[^20] |

---

## 3. Azure AI サービス全体像

Microsoft の AI サービスは、2025〜2026年にかけて大きくブランドリニューアルされ、以下の3層構造に整理されている。

```
┌─────────────────────────────────────────────────────────────┐
│              Microsoft Foundry（旧 Azure AI Foundry）         │
│    生成 AI・マルチモデル・エージェントオーケストレーション       │
│  ┌──────────────────────┐  ┌────────────────────────────┐   │
│  │  Azure OpenAI Service│  │  Foundry Models Catalog    │   │
│  │  (GPT-4o, o3 等)     │  │  (Meta, Cohere, xAI 等)   │   │
│  └──────────────────────┘  └────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│         Azure AI Services（旧 Cognitive Services）           │
│              プリビルト AI API（コード不要〜最小限）           │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────────────┐ │
│  │ Vision │ │ Speech │ │Language│ │ Document Intelligence│ │
│  └────────┘ └────────┘ └────────┘ └──────────────────────┘ │
│  ┌────────────┐ ┌──────────────┐ ┌────────────────────┐    │
│  │ Translator │ │Content Safety│ │ Content Understand.│    │
│  └────────────┘ └──────────────┘ └────────────────────┘    │
│  ┌──────────┐ ┌────────────┐ ┌───────────────┐            │
│  │AI Search │ │Custom Vision│ │    Face API   │            │
│  └──────────┘ └────────────┘ └───────────────┘            │
├─────────────────────────────────────────────────────────────┤
│              Azure Machine Learning                         │
│        カスタム ML モデル開発・MLOps・AutoML                  │
│  ┌────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐   │
│  │ AutoML │ │Pipelines │ │ Designer  │ │ Feature Store│   │
│  └────────┘ └──────────┘ └───────────┘ └──────────────┘   │
│  ┌──────────────┐ ┌───────────────┐ ┌────────────────┐    │
│  │Managed Endpts│ │Responsible AI │ │ Model Monitor  │    │
│  └──────────────┘ └───────────────┘ └────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Microsoft Foundry（旧 Azure AI Foundry / Azure AI Studio）

生成 AI およびエージェント開発の統合プラットフォーム[^21][^22]。

| 特徴 | 説明 |
|---|---|
| マルチモデル対応 | OpenAI、Meta Llama、Cohere、Mistral、xAI 等 |
| エージェントオーケストレーション | 複雑なマルチエージェントワークフロー |
| Prompt Flow 統合 | プロンプト設計・評価・デプロイ |
| ファインチューニング | カタログモデルのカスタマイズ |
| ガバナンス | RBAC、監査ログ、ポリシー管理 |
| Foundry Local | オンプレミス/エッジデプロイ |
| ポータル | ai.azure.com |

### 3.2 Azure OpenAI Service

OpenAI モデル（GPT-4o、GPT-4.1、o3、o4-mini 等）への Azure マネージドアクセス[^23]。

- Azure の SLA・セキュリティ・コンプライアンス下で OpenAI モデルを利用
- DALL-E による画像生成、Embeddings、Whisper 音声認識
- シンプルな API ベースの利用に最適

### 3.3 Azure AI Services（旧 Cognitive Services）

プリビルト AI API 群。ML の専門知識不要で AI 機能をアプリに追加可能[^24]。

| サービス | 機能 | ステータス |
|---|---|---|
| **Azure Vision** | 画像分析、OCR、空間分析、物体検出 | GA |
| **Azure Face** | 顔検出・認証・認識 | GA |
| **Azure Speech** | 音声→テキスト、テキスト→音声、音声翻訳、話者認識 | GA |
| **Azure Translator** | 100以上の言語でのテキスト・ドキュメント翻訳 | GA |
| **Azure Language** | 感情分析、キーフレーズ抽出、PII 検出、要約、QnA | GA |
| **Conversational Language Understanding** | 意図・エンティティ認識（LUIS の後継） | GA |
| **Azure AI Document Intelligence** | フォーム・領収書・ドキュメントからのデータ抽出 | GA |
| **Azure AI Content Safety** | コンテンツモデレーション、有害コンテンツ検出 | GA（カスタムカテゴリ API は **Preview**）|
| **Azure AI Search** | セマンティック検索、ベクトル検索、ハイブリッド検索、RAG | GA |
| **Azure Custom Vision** | カスタム画像分類・物体検出モデルの構築 | GA |
| **Azure Content Understanding** | マルチモーダル（文書・画像・動画・音声）データ分析 | GA（2025年11月 API v1） |
| **Anomaly Detector** | 時系列データの異常検知 | GA |
| **Azure Bot Service** | マルチチャネルチャットボット構築 | GA |

---

## 4. シナリオ別 Azure AI サービス選択ガイド

### 4.1 判定フロー

```
生成 AI / LLM が必要？
  ├─ Yes → OpenAI モデルだけで十分？
  │         ├─ Yes → Azure OpenAI Service
  │         └─ No（マルチモデル・エージェント）→ Microsoft Foundry
  └─ No → プリビルト AI API で対応可能？
           ├─ Yes → Azure AI Services
           └─ No → カスタムモデル学習が必要？
                    ├─ Yes → Azure Machine Learning
                    └─ No → Azure AI Search / Bot Service 等
```

### 4.2 シナリオ別採用サービス一覧

| シナリオ | 推奨サービス | 理由 |
|---|---|---|
| **チャットボット / Copilot アシスタント** | Azure OpenAI Service / Microsoft Foundry | GPT モデルによる自然な対話生成 |
| **社内ナレッジ検索（RAG）** | Azure AI Search + Azure OpenAI | ベクトル検索 + LLM による回答生成 |
| **マルチ LLM エージェントプラットフォーム** | Microsoft Foundry | 複数モデル・エージェントオーケストレーション |
| **需要予測・売上予測** | Azure ML (AutoML / 時系列予測) | カスタム予測モデルの構築・MLOps |
| **不正検知・リスクスコアリング** | Azure ML + Anomaly Detector | カスタム分類モデル + 異常検知 |
| **顧客離反予測** | Azure ML (AutoML) | 表形式データの分類問題 |
| **画像分類・物体検出（独自ドメイン）** | Azure Custom Vision / Azure ML | ドメイン特化の画像認識モデル |
| **OCR・ドキュメント処理** | Azure AI Document Intelligence | 請求書・領収書・フォームの自動読取 |
| **音声文字起こし・音声合成** | Azure Speech | STT/TTS/音声翻訳 |
| **多言語翻訳** | Azure Translator | 100+ 言語対応の翻訳 API |
| **コンテンツモデレーション** | Azure AI Content Safety | 有害コンテンツのフィルタリング |
| **予知保全（IoT）** | Azure ML + IoT Hub | センサーデータによる故障予測 |
| **コード生成・分析** | Azure OpenAI (GPT-4.1 / Codex) | コード補完・レビュー・変換 |
| **動画・画像生成** | Azure OpenAI (DALL-E / GPT-image) | 画像生成・編集 |
| **音声対話アプリ** | Azure OpenAI (GPT-4o Audio) + Speech | リアルタイム音声 AI 会話 |

### 4.3 プラットフォーム比較（Microsoft Foundry vs Azure ML vs Azure AI Services）

| 観点 | Microsoft Foundry | Azure Machine Learning | Azure AI Services |
|---|---|---|---|
| **主な用途** | 生成 AI・エージェント開発 | カスタム ML モデル開発・MLOps | プリビルト AI API の利用 |
| **モデルプロバイダ** | OpenAI + Meta, Cohere, xAI 等 | 任意のフレームワーク | Microsoft / パートナー |
| **カスタムモデル学習** | ファインチューニングのみ | フルカスタム（ゼロから学習可） | 一部（Custom Vision 等） |
| **複雑なエージェント** | ✅ | ❌ | ❌ |
| **MLOps** | 限定的 | ✅ フル対応 | ❌ |
| **コーディング不要度** | 中（ポータル操作） | 低〜中（SDK / Designer） | 高（API 呼出のみ） |
| **ハイブリッド / エッジ** | ✅ Foundry Local | ✅（コンテナデプロイ） | 一部（コンテナ版） |
| **ガバナンス** | ✅ フルライフサイクル | ✅ | 標準的 |
| **最適ユーザー** | AI エンジニア・アプリ開発者 | データサイエンティスト | アプリ開発者 |

---

## 5. ブランディングの変遷

Azure の AI サービスは近年、頻繁にブランドが変更されており、混乱が生じやすい状況にある。

| 時期 | 旧名称 | 現名称 |
|---|---|---|
| 〜2023年 | Azure Cognitive Services | Azure AI Services / Foundry Tools |
| 2023〜2024年 | Azure OpenAI Studio | Azure AI Studio → Azure AI Foundry |
| 2025〜2026年 | Azure AI Foundry | **Microsoft Foundry**[^22] |
| 継続 | Azure Machine Learning | Azure Machine Learning（変更なし） |

> **注意**: ポータル上の統合が進み、「Foundry」ブランド下に Azure ML リソースも表示されるが、Azure ML 自体は独立したカスタム ML プラットフォームとして存続している[^21]。

---

## 6. 現在プレビュー中の主要機能

| 機能 | サービス | 説明 |
|---|---|---|
| Responsible AI Scorecard | Azure ML | PDF 出力可能な監査用スコアカード[^10] |
| 生成 AI モデルモニタリング | Azure ML | LLM の品質・安全性メトリクス監視[^11] |
| カスタムカテゴリ検出 API | Content Safety | ユーザー定義のコンテンツカテゴリ検出[^25] |
| Content Understanding | AI Services | マルチモーダルデータ分析（2025年11月 GA[^26]） |

---

## Confidence Assessment

### 高い確信度
- Azure ML の主要機能（AutoML、Managed Endpoints、Pipelines 等）の記述は公式ドキュメントに基づいている
- SDK v1 廃止、Studio (classic) 廃止、Power BI 統合廃止は公式アナウンスに基づいている
- Azure AI Services の各サービス概要は公式ドキュメントに基づいている
- Microsoft Foundry への名称変更は公式ドキュメントで確認済み

### 中程度の確信度
- プレビュー機能の具体的な GA 時期は公式に未発表のため推測を含む
- Designer プレビュー機能の廃止（2026年3月31日）は Azure Feeds / 非公式ソースに基づく
- Image Analysis 4.0 の廃止タイムライン（2028年9月）はウェブ検索結果に基づく

### 注意点
- Azure AI サービスのブランディングは変更が頻繁であり、本レポート執筆時点（2026年4月）以降にさらなる変更がある可能性がある
- 各機能の正確な GA / 廃止日は [Azure Updates](https://azure.microsoft.com/en-us/updates/) および [Azure Deprecation Dashboard](https://azurecharts.com/timeboards/deprecations) で最新情報を確認されたい

---

## Footnotes

[^1]: [What is automated machine learning (AutoML)?](https://learn.microsoft.com/en-us/azure/machine-learning/concept-automated-ml?view=azureml-api-2) — Microsoft Learn
[^2]: [Azure Machine Learning: A Comprehensive Guide](https://www.appliedaicourse.com/blog/azure-machine-learning/) — Applied AI Course
[^3]: [Online endpoints for real-time inference](https://learn.microsoft.com/en-us/azure/machine-learning/concept-endpoints-online?view=azureml-api-2) — Microsoft Learn
[^4]: [Deploy Machine Learning Models to Online Endpoints](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-deploy-online-endpoints?view=azureml-api-2) — Microsoft Learn
[^5]: [Model Training on Serverless Compute](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-serverless-compute?view=azureml-api-2) — Microsoft Learn
[^6]: [Azure Machine Learning - ML as a Service](https://azure.microsoft.com/en-us/products/machine-learning/) — Azure Product Page
[^7]: [What is Azure Machine Learning prompt flow](https://learn.microsoft.com/en-us/azure/machine-learning/prompt-flow/overview-what-is-prompt-flow?view=azureml-api-2) — Microsoft Learn
[^8]: [Enterprise Azure Machine Learning: Deployment and MLOps Guide](https://www.imaginarycloud.com/blog/azure-machine-learning-deployment-and-mlops-guide/) — Imaginary Cloud
[^9]: [Assess AI Systems with Responsible AI dashboard](https://learn.microsoft.com/en-us/azure/machine-learning/concept-responsible-ai-dashboard?view=azureml-api-2) — Microsoft Learn
[^10]: [Use Responsible AI scorecard (preview)](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-responsible-ai-scorecard?view=azureml-api-2) — Microsoft Learn
[^11]: [Model monitoring for generative AI applications (preview)](https://learn.microsoft.com/en-us/azure/machine-learning/prompt-flow/how-to-monitor-generative-ai-applications?view=azureml-api-2) — Microsoft Learn
[^12]: [Mastering Azure Machine Learning: Studio & Pipelines Guide](https://vife.ai/blog/mastering-azure-machine-learning-studio-pipelines) — Vife.ai
[^13]: [Azure Machine Learning Studio (classic) retirement](https://github.com/azure-deprecation/dashboard/issues/193) — Azure Deprecation Dashboard
[^14]: [LUIS retirement & migration to Conversational Language Understanding](https://blog.miraclesoft.com/unlocking-the-power-of-azure-cognitive-services-a-deep-dive-into-luis-and-beyond/) — MiracleSoft Blog
[^15]: [Azure Machine Learning SDK v1 retirement](https://www.azalio.io/retirement-azure-machine-learning-sdk-v1-will-be-retired-on-march-31-2025-transition-to-machine-learning-sdk-v2/) — Azalio
[^16]: [Cognitive Services and Azure ML for Dataflows retirement](https://powerbi.microsoft.com/en-us/blog/cognitive-services-and-azure-ml-for-dataflows-will-be-fully-retired-by-september-15th-2025/) — Power BI Blog
[^17]: [Retirement: Remove dependency on preview features before March 31, 2026](https://azurefeeds.com/2025/10/30/retirement-remove-dependency-on-these-preview-features-before-march-31-2026/) — Azure Feeds
[^18]: [Data drift (preview) replaced by Model Monitor](https://learn.microsoft.com/azure/machine-learning/how-to-monitor-datasets?view=azureml-api-1) — Microsoft Learn
[^19]: [Link Azure Synapse Analytics and Azure ML workspaces (deprecated)](https://learn.microsoft.com/azure/machine-learning/how-to-link-synapse-ml-workspaces?view=azureml-api-1) — Microsoft Learn
[^20]: [Azure Vision - Image Analysis 4.0 deprecation](https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/overview) — Microsoft Learn
[^21]: [Azure AI Foundry vs Azure AI Services vs Azure Machine Learning](https://daniel.mcloughlin.cloud/azure-ai-foundry-vs-azure-ai-services-vs-azure-machine-learning) — Daniel McLoughlin Blog
[^22]: [What is Microsoft Foundry?](https://learn.microsoft.com/en-us/azure/foundry/what-is-foundry) — Microsoft Learn
[^23]: [What is Azure OpenAI in Azure AI Foundry Models?](https://learn.microsoft.com/azure/ai-foundry/openai/overview) — Microsoft Learn
[^24]: [What are Foundry Tools?](https://learn.microsoft.com/azure/ai-services/what-are-ai-services) — Microsoft Learn
[^25]: [What is Azure AI Content Safety?](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview) — Microsoft Learn
[^26]: [Azure Content Understanding documentation](https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/) — Microsoft Learn
