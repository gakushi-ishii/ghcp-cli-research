# Microsoft Agent Framework 調査レポート

> **調査日**: 2026-04-24  
> **調査対象**: Microsoft Agent Framework（AutoGen + Semantic Kernel 統合フレームワーク）  
> **対象読者**: AI エージェント開発に関心を持つエンジニア・アーキテクト  

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [背景・歴史](#2-背景歴史)
3. [Microsoft Agent Framework の概要](#3-microsoft-agent-framework-の概要)
4. [アーキテクチャ](#4-アーキテクチャ)
5. [主要機能](#5-主要機能)
6. [類似ライブラリとの比較](#6-類似ライブラリとの比較)
7. [クイックスタート](#7-クイックスタート)
8. [まとめと選択指針](#8-まとめと選択指針)
9. [参考リンク](#9-参考リンク)

---

## 1. エグゼクティブサマリー

**Microsoft Agent Framework（MAF）** は、Microsoft の 2 つの主要 AI エージェントフレームワーク — 研究発の **AutoGen** と エンタープライズ向け **Semantic Kernel** — を統合した、オープンソースの統一エージェント SDK です。

2025 年 10 月に発表・開発開始、2026 年 3 月に GA（一般提供開始）を迎えました。Python / .NET 双方でネイティブサポートされ、グラフベースのオーケストレーション・MCP/A2A プロトコル対応・エンタープライズグレードの観測性と安全機能を兼ね備えます。

AutoGen および Semantic Kernel は **メンテナンスモード**（バグ修正・セキュリティパッチのみ）に移行しており、Microsoft は新規プロジェクトへの MAF 採用を強く推奨しています。

---

## 2. 背景・歴史

### 2.1 AutoGen（旧 Microsoft Research 発）

| 項目 | 内容 |
|------|------|
| 初リリース | 2023 年 |
| 開発元 | Microsoft Research (MSR) |
| 特徴 | マルチエージェント会話・協調の研究特化型フレームワーク |
| 強み | 柔軟なエージェント間の会話パターン、迅速なプロトタイピング |
| 弱み | 本番環境向けの状態管理・コンプライアンス機能が不十分 |
| v0.4 リリース | 2025 年 1 月（非同期イベント駆動アーキテクチャへ全面刷新） |
| GitHub Stars | 約 57,000（2026 年 4 月時点） |
| 最終リリース | python-v0.7.5（2025 年 9 月） |

**AutoGen v0.4 の主な変更点：**
- 同期処理から**アクターモデル（非同期メッセージパッシング）**へ移行
- コア層 / AgentChat 層 / Extensions 層の 3 層モジュール構造を採用
- OpenTelemetry によるビルトイン観測性
- AutoGen Studio（Web ベースのローコード開発 UI）導入

### 2.2 Semantic Kernel（Azure AI 発）

| 項目 | 内容 |
|------|------|
| 初リリース | 2023 年 |
| 開発元 | Microsoft Azure AI チーム |
| 特徴 | エンタープライズ向け LLM オーケストレーション SDK |
| 強み | .NET との深い統合、プラグイン・テレメトリ・セキュリティ・コンプライアンス |
| 弱み | マルチエージェント会話フローには多くのカスタム実装が必要 |

### 2.3 統合の経緯

```
┌────────────────────┐     ┌────────────────────────┐
│    AutoGen          │     │    Semantic Kernel      │
│ (MSR 研究向け)      │  +  │ (Azure エンタープライズ) │
│ ・マルチエージェント│     │ ・プラグインシステム    │
│ ・会話オーケストレ  │     │ ・テレメトリ・観測性   │
│ ・迅速プロトタイプ  │     │ ・セキュリティ・準拠   │
└────────────────────┘     └────────────────────────┘
              │                         │
              └───────────┬─────────────┘
                          ▼
           ┌──────────────────────────────┐
           │  Microsoft Agent Framework   │
           │    (MAF) — GA March 2026     │
           │  "Best of Both Worlds"       │
           └──────────────────────────────┘
```

2025 年 10 月に発表された理由：双方のフレームワークの利用者が増える中、開発者が「AutoGen か Semantic Kernel か」の選択を迫られる状況が生じ、Microsoft として統一した指針を提供する必要があったため。

---

## 3. Microsoft Agent Framework の概要

### 3.1 基本情報

| 項目 | 内容 |
|------|------|
| GitHub | `github.com/microsoft/agent-framework` |
| ライセンス | MIT |
| 言語サポート | Python、.NET（Java / JavaScript は開発中） |
| GA 日付 | 2026 年 3 月 |
| Azure 統合 | Azure AI Foundry ネイティブサポート |
| 標準プロトコル | MCP（Model Context Protocol）、A2A（Agent-to-Agent）|

### 3.2 パッケージ構成（Python）

```bash
# フルインストール
pip install agent-framework

# コア機能のみ
pip install agent-framework-core

# Azure AI Foundry 統合
pip install agent-framework-foundry

# .NET
dotnet add package Microsoft.Agents.AI
```

---

## 4. アーキテクチャ

### 4.1 概念アーキテクチャ

```
┌────────────────────────────────────────────────────────┐
│                  ユーザー / ビジネスロジック            │
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────┐
│          エージェント定義層                            │
│  (ロール / ツール / ポリシー / メモリ / セキュリティ)  │
└───────────────────────────┬────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────┐
│              ワークフローエンジン                       │
│     グラフベース状態機械 (Sequential / Parallel /      │
│                           Conditional / Checkpoint)    │
└───────┬──────────────────────────┬─────────────────────┘
        │                          │
┌───────▼──────────┐   ┌──────────▼──────────────────────┐
│  MCP プロトコル  │   │  A2A プロトコル                  │
│ (ツール発見・実行) │   │ (エージェント間通信)            │
└──────────────────┘   └─────────────────────────────────┘
        │
┌───────▼─────────────────────────────────────────────────┐
│  外部統合レイヤー                                        │
│  Azure AI Foundry / Microsoft 365 / OpenAPI /           │
│  Human-in-the-Loop / カスタムプラグイン                 │
└─────────────────────────────────────────────────────────┘
```

### 4.2 3 層モジュール構造

| 層 | 説明 |
|----|------|
| **Core 層** | 非同期メッセージパッシング・アクターモデル・イベント駆動基盤 |
| **AgentChat 層** | グループチャット・タスク駆動エージェント・高レベル API |
| **Extensions 層** | LLM クライアント（OpenAI, Azure）・ツール・メモリプラグイン |

### 4.3 ワークフロータイプ

| タイプ | 説明 | ユースケース |
|--------|------|------------|
| **Sequential** | エージェントが順番に実行 | 手順が明確なパイプライン処理 |
| **Parallel** | 複数エージェントが並列実行 | 独立したサブタスクの同時処理 |
| **Conditional** | 条件分岐でフローを切り替え | ビジネスロジックが複雑な承認フロー |
| **Checkpoint** | 中断・再開・人間介入をサポート | 長時間実行・ヒューマンインザループ |

---

## 5. 主要機能

### 5.1 マルチエージェントオーケストレーション

- **グラフベース状態機械**: ノード = エージェント、エッジ = 状態遷移またはツール呼び出しとして定義
- **会話型協調**: AutoGen 由来のエージェント間会話パターン
- **並列・条件分岐フロー**: 複雑なビジネスロジックを宣言的に定義可能

### 5.2 メモリ管理

- **短期メモリ**: セッション内のコンテキスト保持
- **長期メモリ**: ベクターストア（Pinecone、Azure AI Search など）や SQL/MongoDB と統合
- **エージェント別メモリ**: 各エージェントが独自のコンテキストを維持

### 5.3 ツール統合

- **MCP（Model Context Protocol）**: ツールの発見・呼び出しを標準化
- **A2A（Agent-to-Agent）プロトコル**: 組織をまたいだエージェント間通信
- **OpenAPI 統合**: REST API をツールとして簡単にラップ
- **Semantic Kernel プラグイン**: 既存の SK プラグイン資産をそのまま利用可能

### 5.4 観測性・デバッグ

- **OpenTelemetry 統合**: メトリクス・トレース・ログの統一収集
- **ブラウザベース UI デバッガ**: エージェント実行フローをリアルタイム可視化
- **ホットリロード**: 開発中の変更を即時反映

### 5.5 エンタープライズ機能

| 機能 | 詳細 |
|------|------|
| **認証・認可** | Microsoft Entra（旧 Azure AD）との深い統合 |
| **コンプライアンス** | SOC 2、HIPAA 対応認定 |
| **プロンプトインジェクション対策** | ビルトインの安全フィルタリング |
| **監査証跡** | 全エージェント操作の完全なログ記録 |
| **Responsible AI** | タスク遵守監視・有害コンテンツフィルタ |

---

## 6. 類似ライブラリとの比較

### 6.1 主要フレームワーク比較表

| 比較軸 | **Microsoft Agent Framework** | **LangGraph** | **CrewAI** | **LlamaIndex** |
|--------|-------------------------------|---------------|------------|----------------|
| **開発元** | Microsoft | LangChain Inc. | CrewAI Inc. | LlamaIndex Inc. |
| **主な強み** | エンタープライズ・Azure統合・.NET対応 | 複雑なステートフルワークフロー | 迅速プロトタイプ・ロールベース | RAG・企業データ連携 |
| **アーキテクチャ** | グラフ + アクターモデル | グラフ型ステートマシン | ロール/クルー型 | RAG ファースト + エージェント |
| **言語サポート** | Python / .NET（Java/JS 予定） | Python | Python | Python |
| **学習コスト** | 中〜高 | 高 | 低〜中 | 中 |
| **本番適性** | ◎ 高（GA・SLA あり） | ○ 高 | △ 中 | ○ 中〜高 |
| **マルチエージェント** | ◎ ネイティブサポート | ○ サポート | ◎ ロールベース協調 | △ 基本サポート |
| **Human-in-the-Loop** | ◎ ネイティブサポート | ○ サポート | △ 限定的 | △ 限定的 |
| **Azure 統合** | ◎ ネイティブ | △ アドオン | △ アドオン | △ アドオン |
| **MCP 対応** | ◎ ネイティブ | ○ 対応 | △ 限定的 | △ 限定的 |
| **OSS フレンドリー** | ○ MIT ライセンス | ○ MIT | ○ MIT | ○ MIT |
| **GitHubスター数（概算）** | 開発中（agent-framework） | 約 13,000 | 約 30,000 | 約 38,000 |

### 6.2 詳細比較

#### LangGraph（by LangChain）

**特徴**:
- LangChain エコシステムの公式エージェントワークフロー後継
- グラフ型実行：ステートフル・トレース可能・デバッグ容易なフロー定義
- 複雑なマルチツール・マルチステップ・分岐ワークフローに強い

**強み**:
- LangChain の豊富なエコシステムとの統合
- 宣言的なグラフ記述で複雑ロジックを整理しやすい
- LangSmith による観測性

**弱み**:
- LangChain 由来のドキュメント散逸と学習コスト
- .NET サポートなし

**選ぶべきケース**: 複雑なステートフル・マルチアクターの本番アプリケーション（カスタマーサポート、研究パイプライン、ポリシーワークフロー）

---

#### CrewAI

**特徴**:
- 「チーム・ロール」メタファー：エージェントに役割・バックストーリー・スキルを定義し「クルー」として協調
- 迅速なプロトタイピングに特化した直感的な API

**強み**:
- 最少のボイラープレートで動作するシンプルな API
- 標準的な QA タスクでグラフベースより 5 倍以上高速な場合がある
- 自律協調（クルー）と開発者制御フロー（フロー機能）のデュアルモデル

**弱み**:
- 深く複雑なワークフローのステート・メモリ管理が粗い
- .NET サポートなし

**選ぶべきケース**: ビジネス自動化・シンプル〜中程度のマルチエージェント設定・高速な開発テスト

---

#### LlamaIndex

**特徴**:
- RAG ファーストから出発し、広範なエージェントオーケストレーションに拡張
- 企業データへの深いアクセス・インデックス・検索に特化

**強み**:
- 多様なデータソースへのシームレスな接続
- エンタープライズ規模の知識アクセスに最適化
- ベクターストア・ドキュメント処理のエコシステムが豊富

**弱み**:
- 汎用マルチエージェント・会話フローは上記 3 フレームワークより成熟度が低い

**選ぶべきケース**: 社内文書・ナレッジベース・エンタープライズデータを活用するインテリジェントアシスタント

---

### 6.3 選択フローチャート

```
Azure / Microsoft エコシステムを使用？
├── YES → .NET アプリまたは高いコンプライアンス要件？
│         ├── YES → Microsoft Agent Framework ✅
│         └── NO  → Microsoft Agent Framework または LangGraph
└── NO  → RAG / 社内データ中心のユースケース？
           ├── YES → LlamaIndex ✅
           └── NO  → 迅速なプロトタイプ or ロールベース協調？
                      ├── YES → CrewAI ✅
                      └── NO  → 複雑なステートフルワークフロー？
                                 ├── YES → LangGraph ✅
                                 └── NO  → CrewAI or LangGraph
```

---

## 7. クイックスタート

### 7.1 Python クイックスタート

#### 環境準備

```bash
# Python 3.10 以上が必要
python --version

# 仮想環境の作成（推奨）
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# インストール
pip install agent-framework

# OpenAI API キーの設定（.env ファイルまたは環境変数）
export OPENAI_API_KEY="sk-..."
# Azure OpenAI を使う場合
# export AZURE_OPENAI_API_KEY="..."
# export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
```

#### シングルエージェント（最小構成）

```python
# single_agent.py
import asyncio
from agent_framework import Agent
from agent_framework.openai import OpenAIChatClient

async def main():
    agent = Agent(
        client=OpenAIChatClient(),
        instructions="あなたは親切なアシスタントです。日本語で答えてください。"
    )
    response = await agent.run("Pythonでフィボナッチ数列を計算する関数を書いてください。")
    print(response)

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
python single_agent.py
```

#### マルチエージェント（グループチャット）

```python
# multi_agent.py
import asyncio
from agent_framework import Agent, GroupChat, GroupChatManager
from agent_framework.openai import OpenAIChatClient

async def main():
    client = OpenAIChatClient()

    # エージェントの定義
    researcher = Agent(
        name="Researcher",
        client=client,
        instructions="あなたはコードを書く専門家です。要求に従って実装を行います。"
    )

    reviewer = Agent(
        name="Reviewer",
        client=client,
        instructions="あなたはコードレビュアーです。コードの品質・ベストプラクティスを確認し改善点を提案します。"
    )

    # グループチャットのセットアップ
    groupchat = GroupChat(
        agents=[researcher, reviewer],
        max_rounds=4
    )
    manager = GroupChatManager(groupchat=groupchat, client=client)

    # タスク実行
    result = await manager.run(
        "Pythonで二分探索アルゴリズムを実装し、レビューしてください。"
    )
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

#### ツール付きエージェント

```python
# agent_with_tools.py
import asyncio
import httpx
from agent_framework import Agent, tool
from agent_framework.openai import OpenAIChatClient

# ツールの定義（デコレータで登録）
@tool
async def get_weather(city: str) -> str:
    """指定した都市の現在の天気を取得します。"""
    # 実際には天気 API を呼ぶ
    return f"{city}の天気: 晴れ、気温 22°C"

@tool
async def calculate(expression: str) -> str:
    """単純な四則演算を計算します（数値リテラルのみ対応）。"""
    import ast
    try:
        # ast.literal_eval は数値リテラルのみ安全に評価する
        # 複雑な演算が必要な場合は simpleeval 等の専用ライブラリを使用すること
        result = ast.literal_eval(expression)
        return f"計算結果: {result}"
    except (ValueError, SyntaxError) as e:
        return f"エラー: 数値リテラルのみ対応しています ({e})"

async def main():
    agent = Agent(
        client=OpenAIChatClient(),
        instructions="あなたは天気情報と計算ができるアシスタントです。",
        tools=[get_weather, calculate]
    )
    response = await agent.run("東京の天気を教えて、それと 15 * 24 を計算して。")
    print(response)

if __name__ == "__main__":
    asyncio.run(main())
```

#### Human-in-the-Loop（人間介入ワークフロー）

```python
# human_in_loop.py
import asyncio
from agent_framework import Agent, HumanApprovalStep
from agent_framework.openai import OpenAIChatClient

async def main():
    agent = Agent(
        client=OpenAIChatClient(),
        instructions="あなたは重要な業務決定を行うエージェントです。"
    )

    # 重要な決定の前に人間の承認を要求
    workflow = agent.with_approval(
        approval_step=HumanApprovalStep(
            message="このアクションを実行しますか？",
            on_approve=lambda: print("承認されました"),
            on_reject=lambda: print("却下されました")
        )
    )

    result = await workflow.run("予算 100 万円の新規プロジェクトを承認してください。")
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

---

### 7.2 .NET クイックスタート

#### プロジェクトのセットアップ

```bash
dotnet new console -n MyAgentApp
cd MyAgentApp
dotnet add package Microsoft.Agents.AI
```

#### シングルエージェント（C#）

```csharp
// Program.cs
using Microsoft.Agents.AI;

var apiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")!;

var agent = new OpenAIClient(apiKey).AsAIAgent(
    name: "HelpfulBot",
    instructions: "あなたは親切なアシスタントです。日本語で答えてください。"
);

var response = await agent.RunAsync("Pythonでクイックソートを実装してください。");
Console.WriteLine(response);
```

```bash
dotnet run
```

#### マルチエージェント（C#）

```csharp
// Program.cs
using Microsoft.Agents.AI;
using Microsoft.Agents.AI.Orchestration;

var apiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")!;
var client = new OpenAIClient(apiKey);

var coder = client.AsAIAgent(
    name: "Coder",
    instructions: "コードを書く専門家です。"
);

var reviewer = client.AsAIAgent(
    name: "Reviewer",
    instructions: "コードを厳しくレビューします。"
);

var groupChat = new GroupChat(new[] { coder, reviewer }, maxRounds: 3);
var manager = new GroupChatManager(groupChat, client);

var result = await manager.RunAsync(
    "C# で非同期ファイル読み込みのサンプルを実装しレビューしてください。"
);
Console.WriteLine(result);
```

---

### 7.3 AutoGen（現行版）を使ったクイックスタート

AutoGen はメンテナンスモードですが、現在も広く使われているため参考として記載します。

```bash
pip install pyautogen openai
```

```python
# autogen_quickstart.py
import autogen

llm_config = {
    "model": "gpt-4o",
    "api_key": "YOUR_OPENAI_API_KEY",
}

assistant = autogen.AssistantAgent(
    name="Assistant",
    system_message="あなたは優秀なソフトウェアエンジニアです。",
    llm_config=llm_config,
)

user_proxy = autogen.UserProxyAgent(
    name="User",
    code_execution_config={"use_docker": False},
    human_input_mode="NEVER",  # 自動実行モード
)

# 会話を開始
user_proxy.initiate_chat(
    assistant,
    message="Pythonで素数を列挙するエラトステネスの篩を実装してください。"
)
```

> ⚠️ **注意**: AutoGen は v0.7.5 でメンテナンスモードに移行しました。新規プロジェクトは **Microsoft Agent Framework** を推奨します。

---

### 7.4 開発ツール

| ツール | 説明 |
|--------|------|
| **VS Code + Microsoft 365 Agents Toolkit** | エージェントの雛形生成・ホットリロード・デバッグ |
| **AutoGen Studio** | Web ベースのビジュアル開発環境（AutoGen 時代の資産も利用可） |
| **Azure AI Foundry** | クラウド上でのエージェントのホスティング・モニタリング・管理 |

---

## 8. まとめと選択指針

### 8.1 Microsoft Agent Framework の採用を推奨するケース

| ケース | 理由 |
|--------|------|
| **Microsoft / Azure エコシステム** | ネイティブの Azure AI Foundry・Entra 統合 |
| **.NET アプリケーション** | Python と .NET で同一フレームワークを使用可能 |
| **高コンプライアンス要件** | SOC 2・HIPAA 対応、完全な監査証跡 |
| **複雑なマルチエージェントワークフロー** | グラフベースのオーケストレーションとチェックポイント機能 |
| **長期安定性** | Microsoft の GA サポートと SLA 保証 |

### 8.2 他フレームワークを検討すべきケース

| 状況 | 推奨 |
|------|------|
| Python のみで迅速なプロトタイプ | **CrewAI** |
| 複雑なステートフルワークフローかつ LangChain 資産あり | **LangGraph** |
| 社内ドキュメント・ナレッジ中心 | **LlamaIndex** |
| ベンダー非依存を優先 | **LangGraph** or **CrewAI** |

### 8.3 移行ロードマップ

```
既存 AutoGen ユーザー
    ↓
  段階的移行（既存コードは動作継続）
    ↓
  新機能 → Microsoft Agent Framework で実装
    ↓
  完全移行（長期ロードマップ）

既存 Semantic Kernel ユーザー
    ↓
  プラグイン・DI 設定はほぼそのまま移行可能
    ↓
  マルチエージェント機能を MAF の新 API で追加
    ↓
  完全移行
```

---

## 9. 参考リンク

| リソース | URL |
|----------|-----|
| Microsoft Agent Framework GitHub | https://github.com/microsoft/agent-framework |
| Microsoft Learn ドキュメント | https://learn.microsoft.com/en-us/agent-framework/ |
| Azure AI Foundry | https://ai.azure.com |
| AutoGen GitHub（メンテナンスモード） | https://github.com/microsoft/autogen |
| AutoGen ドキュメント | https://microsoft.github.io/autogen/stable/ |
| Microsoft Agent Framework Samples | https://github.com/microsoft/Agent-Framework-Samples |
| 公式発表ブログ (Foundry) | https://devblogs.microsoft.com/foundry/introducing-microsoft-agent-framework-the-open-source-engine-for-agentic-ai-apps/ |
| AutoGen → MAF 統合発表 (VentureBeat) | https://venturebeat.com/ai/microsoft-retires-autogen-and-debuts-agent-framework-to-unify-and-govern |
| Semantic Kernel + AutoGen 統合 (Visual Studio Magazine) | https://visualstudiomagazine.com/articles/2025/10/01/semantic-kernel-autogen--open-source-microsoft-agent-framework.aspx |
| LangGraph 公式 | https://langchain-ai.github.io/langgraph/ |
| CrewAI 公式 | https://docs.crewai.com/ |
| LlamaIndex 公式 | https://docs.llamaindex.ai/ |
