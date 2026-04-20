# GitHub Copilot Coding Agent — 内部構造・挙動 調査レポート

> **調査日**: 2026-04-19  
> **調査対象**: GitHub Copilot Coding Agent（旧称: Copilot SWE Agent）  
> **調査環境**: GitHub Actions runner (Ubuntu x86_64 / Azure)  
> **注意**: 個人設定・機密情報（トークン類）は一切含みません

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [全体アーキテクチャ](#2-全体アーキテクチャ)
3. [ディレクトリ構造](#3-ディレクトリ構造)
4. [エージェント定義（Agent YAML）](#4-エージェント定義agent-yaml)
5. [MCP（Model Context Protocol）サーバー](#5-mcpmodel-context-protocolサーバー)
6. [eBPF ファイアウォール（Padawan-FW）](#6-ebpf-ファイアウォールpadawan-fw)
7. [ツール一覧](#7-ツール一覧)
8. [起動シーケンス](#8-起動シーケンス)
9. [環境変数・設定パラメータ](#9-環境変数設定パラメータ)
10. [~/.copilot ディレクトリの分析](#10-copilot-ディレクトリの分析)
11. [依存ライブラリ・技術スタック](#11-依存ライブラリ技術スタック)
12. [ネットワーク通信とセキュリティ](#12-ネットワーク通信とセキュリティ)
13. [フィーチャーフラグと実験的機能](#13-フィーチャーフラグと実験的機能)
14. [信頼度評価](#14-信頼度評価)

---

## 1. エグゼクティブサマリー

GitHub Copilot Coding Agent は **GitHub Actions ランナー上で動作する TypeScript/Node.js 製の AI コーディングエージェント**です。ユーザーが Issue やコメントでタスクを依頼すると、専用の GitHub Actions ジョブが起動し、エージェントが LLM（Claude / GPT 系モデル）を使用してコードを自律的に修正・コミットします。

主な特徴：
- **6種類のサブエージェント**（research, explore, task, code-review, configure-copilot, rubber-duck）がタスクに応じて使い分けられる
- **MCP（Model Context Protocol）**を通じて GitHub API やブラウザ操作などのツールを呼び出す
- **eBPF ベースのファイアウォール**（padawan-fw）がネットワークアクセスを制限し、セキュリティを担保する
- **Node.js v24.x** で動作し、14MB の単一バンドル JS ファイル（`dist/index.js`）が本体

---

## 2. 全体アーキテクチャ

```
┌──────────────────────────────────────────────────────────────┐
│                   GitHub Actions Runner (Ubuntu x86_64)      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          eBPF Firewall (padawan-fw)                   │   │
│  │  ┌──────────────────────────────────────────────┐    │   │
│  │  │         Node.js v24.x Process                │    │   │
│  │  │                                              │    │   │
│  │  │   dist/index.js (14MB bundled agent)         │    │   │
│  │  │   ├── Main Agent Loop                        │    │   │
│  │  │   ├── Agent Definitions (YAML)               │    │   │
│  │  │   │   ├── research (claude-sonnet-4.6)       │    │   │
│  │  │   │   ├── explore  (claude-haiku-4.5)        │    │   │
│  │  │   │   ├── task     (claude-haiku-4.5)        │    │   │
│  │  │   │   ├── code-review (claude-sonnet-4.5)    │    │   │
│  │  │   │   ├── configure-copilot                  │    │   │
│  │  │   │   └── rubber-duck                        │    │   │
│  │  │   └── Tool Invocation Layer                  │    │   │
│  │  │            │                                 │    │   │
│  │  │            ▼                                 │    │   │
│  │  │   MCP Client (HTTP :2301)                    │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  │            │                                          │   │
│  │   ┌────────┴──────────────────┐                      │   │
│  │   │   MCP Server (port 2301)  │                      │   │
│  │   │   mcp/dist/index.js       │                      │   │
│  │   │   ├── github-mcp-server ──┼──→ api.enterprise.   │   │
│  │   │   │   (remote, read-only) │    githubcopilot.com │   │
│  │   │   ├── playwright MCP      │                      │   │
│  │   │   └── user MCPs (opt.)    │                      │   │
│  │   └───────────────────────────┘                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Additional Binaries:                                        │
│  ├── autofind    (repo auto-discovery, 54MB ELF)            │
│  ├── blackbird-mcp-server (semantic code search; "Bluebird" │
│  │                     in later sections, 11MB ELF)         │
│  ├── github-mcp-server    (GitHub API, 11MB ELF)            │
│  ├── ripgrep     (multi-platform rg binaries)               │
│  └── ebpf/in-path/padawan-fw (eBPF firewall, 18MB ELF)     │
└──────────────────────────────────────────────────────────────┘
          │
          ▼ HTTPS (allowlist only)
┌─────────────────────────────────┐
│  api.enterprise.githubcopilot.com│
│  ├── /agents/swe/agent           │   ← Agent callback (results)
│  └── /mcp/readonly               │   ← GitHub MCP (read-only API)
└─────────────────────────────────┘
```

---

## 3. ディレクトリ構造

エージェントは GitHub Actions ランナーの一時ディレクトリ（`RUNNER_TEMP`）に展開されます。

```
$RUNNER_TEMP/
├── copilot-developer-action-main/   # エージェント本体
│   ├── action.yml                   # GitHub Actions 定義（内部参照用）
│   ├── version                      # コミットSHA + バージョン情報
│   ├── node-req.env                 # 必要なNode.jsバージョン定義
│   ├── package.json                 # 依存パッケージ定義
│   │
│   ├── dist/                        # ビルド済みエージェント
│   │   ├── index.js                 # メインエージェント (~14MB, ESM)
│   │   ├── index.js.map             # ソースマップ (~7MB)
│   │   ├── definitions/             # エージェント定義YAML
│   │   │   ├── research.agent.yaml
│   │   │   ├── explore.agent.yaml
│   │   │   ├── task.agent.yaml
│   │   │   ├── code-review.agent.yaml
│   │   │   ├── configure-copilot.agent.yaml
│   │   │   └── rubber-duck.agent.yaml
│   │   ├── builtin-skills/          # 組み込みスキル
│   │   │   └── customize-cloud-agent/
│   │   │       └── SKILL.md         # スキル定義・使用方法
│   │   ├── gh-gpgsign/              # GPGコミット署名バイナリ
│   │   │   ├── gh-gpgsign-linux-x86_64  (5MB ELF)
│   │   │   └── gh-gpgsign-windows-x86_64.exe (5MB)
│   │   ├── ripgrep/                 # ripgrep バイナリ (全プラットフォーム)
│   │   │   └── bin/
│   │   │       ├── darwin-arm64/rg
│   │   │       ├── darwin-x64/rg
│   │   │       ├── linux-arm64/rg
│   │   │       ├── linux-x64/rg
│   │   │       ├── win32-arm64/rg
│   │   │       └── win32-x64/rg
│   │   ├── node_modules/            # バンドルされない依存モジュール
│   │   │   ├── sharp/               # 画像処理
│   │   │   ├── node-pty/            # 疑似端末
│   │   │   ├── semver/
│   │   │   └── ...
│   │   └── trajectory.md            # セッション中のエージェント行動ログ
│   │
│   ├── mcp/                         # MCP サーバー
│   │   ├── package.json
│   │   └── dist/
│   │       ├── index.js             # MCP サーバー本体 (~8MB)
│   │       ├── runtime-tools.js     # ランタイムツール (~2MB)
│   │       ├── builtin-skills/
│   │       │   └── customize-cloud-agent/
│   │       └── ripgrep/
│   │
│   ├── autofind/                    # リポジトリ自動検出バイナリ
│   │   ├── autofind                 # Linux x86_64 ELF (54MB)
│   │   └── version
│   │
│   ├── blackbird-mcp-server/        # セマンティックコード検索サーバー
│   │   └── blackbird-mcp-server     # Linux x86_64 ELF (11MB)
│   │
│   ├── github-mcp-server/           # GitHub API MCP サーバー
│   │   └── github-mcp-server        # Linux x86_64 ELF (11MB)
│   │
│   ├── ebpf/                        # eBPFファイアウォール
│   │   ├── launch.sh                # ファイアウォール起動スクリプト
│   │   ├── sanitize-domains.sh      # ドメインホワイトリスト検証
│   │   ├── version
│   │   └── in-path/
│   │       ├── padawan-fw           # eBPFファイアウォール本体 (18MB)
│   │       ├── mkcert               # CA証明書生成ツール
│   │       ├── bootstrap
│   │       ├── perf
│   │       ├── profiler
│   │       └── watcher
│   │
│   └── script/                      # ユーティリティスクリプト
│       ├── setup.sh                 # Node.js セットアップ
│       ├── start-mcp-servers.sh     # MCPサーバー起動
│       ├── session-log.sh           # セッションログ送信
│       ├── install-uv.sh / .ps1     # uvx インストール
│       └── install-pipx.sh / .ps1  # pipx インストール
│
├── ghcca-node/                      # Node.js ランタイム
│   └── node/                        # Node.js v24.x (tool cache から)
│
├── mcp-server/                      # MCPサーバーログ
│   └── mcp-server.log
│
└── runtime-logs/                    # ランタイムログ
    ├── params.log                   # 起動パラメータ
    ├── fw.jsonl                     # ファイアウォールイベントログ
    ├── output.log                   # 標準出力ログ
    ├── session-log.log              # セッションログ
    └── mkcert/                      # 一時CA証明書 (実行後削除)
```

---

## 4. エージェント定義（Agent YAML）

エージェントは `dist/definitions/*.agent.yaml` で定義されています。各エージェントは使用するモデル、利用可能なツール、システムプロンプトを持ちます。

### 4.1 エージェント一覧

| エージェント名 | モデル | 用途 |
|--------------|--------|------|
| `research` | `claude-sonnet-4.6` | 深い技術調査・リサーチ |
| `explore` | `claude-haiku-4.5` | 高速コードベース探索 |
| `task` | `claude-haiku-4.5` | コマンド実行（ビルド/テスト/Lint） |
| `code-review` | `claude-sonnet-4.5` | 高精度コードレビュー |
| `configure-copilot` | `claude-haiku-4.5` | MCPサーバー設定管理 |
| `rubber-duck` | 動的選択 | 批判的レビュー・悪魔の代弁者 |

### 4.2 research エージェント

- **目的**: コードベース・API・ライブラリ・アーキテクチャの包括的調査
- **モデル**: `claude-sonnet-4.6`（最も高性能）
- **特徴**:
  - 調査完了後、必ずレポートをMarkdownファイルとして保存する
  - 全ての主張に脚注形式の引用（ファイルパス・行番号・コミットSHA）が必須
  - 並列ツール呼び出しを積極活用するが、GitHub検索は**1回に3〜5並列まで**に制限（レート制限対策）
  - クエリタイプに応じて3種類の出力形式（プロセス型・概念型・技術深掘り型）を使い分け

### 4.3 explore エージェント

- **目的**: 素早い回答が目的（深掘りはしない）
- **モデル**: `claude-haiku-4.5`（軽量・高速）
- **特徴**:
  - Bluebird（セマンティック検索）ツールを多数保有
  - Git履歴検索（コミット、PR、作者、時刻）が可能
  - LSP（Language Server Protocol）ツールも使用可能
  - 「答えが出たら即停止」が原則

### 4.4 task エージェント

- **目的**: `npm test`、`pytest`、`go build` などの開発コマンドを実行
- **モデル**: `claude-haiku-4.5`
- **特徴**:
  - 成功時は **1行サマリーのみ**返却（コンテキスト汚染を最小化）
  - 失敗時は完全なスタックトレース・エラー出力を返却
  - リトライは行わない（1回実行してそのまま報告）

### 4.5 code-review エージェント

- **目的**: 「発見したら$20札を見つけたような感覚」の高品質コードレビュー
- **モデル**: `claude-sonnet-4.5`
- **絶対にコメントしないこと**:
  - スタイル・フォーマット・命名規則
  - コメント・ドキュメントの欠如
  - 「考慮してほしい」系の提案（実際のバグでない限り）
- **コメントする対象**: バグ・セキュリティ脆弱性・競合状態・メモリリーク・クラッシュ

### 4.6 configure-copilot エージェント

- **用途**: Copilot CLI の MCP サーバー設定管理
- **設定ファイルの場所**:
  - ユーザー設定: `{{configDir}}/mcp-config.json`（`{"mcpServers": {...}}`形式）
  - プロジェクト設定: `{{cwd}}/.mcp.json`（`{"servers": {...}}`形式、`${input:id}`変数参照対応）

### 4.7 rubber-duck エージェント

- **目的**: 「悪魔の代弁者」として実装・設計・テストを批判的にレビュー
- **モデル**: 実行時動的選択（ユーザーの現在モデル設定に依存）
- **呼び出しタイミング**: 計画後・実装前が最適（早期の方向修正のため）

---

## 5. MCP（Model Context Protocol）サーバー

### 5.1 MCP アーキテクチャ

MCPサーバーは `mcp/dist/index.js` が起動し、`http://127.0.0.1:2301` でツールを提供します。エージェント本体は HTTP クライアントとして、このエンドポイント経由でツールを呼び出します。

```
エージェント (dist/index.js)
    │
    │ HTTP (localhost:2301)
    ▼
MCP サーバー (mcp/dist/index.js)
    ├── github-mcp-server  (remote/HTTPS → api.enterprise.githubcopilot.com/mcp/readonly)
    ├── playwright MCP     (npx @playwright/mcp@0.0.40)
    └── ユーザー設定MCP   (GITHUB_COPILOT_MCP_JSON環境変数で指定)
```

### 5.2 デフォルト MCP サーバー

| MCPサーバー | 接続方式 | 提供ツール数 |
|------------|---------|------------|
| `github-mcp-server` | リモート HTTPS (read-only) | 28ツール |
| `playwright` | ローカルプロセス (npx) | 21ツール |

### 5.3 github-mcp-server の提供ツール（主要）

GitHub API への read-only アクセスを提供します：

- `get_file_contents` - ファイル内容取得
- `search_code` - コード検索
- `search_repositories` - リポジトリ検索
- `list_commits` / `get_commit` - コミット一覧・詳細
- `list_issues` / `issue_read` - Issue操作
- `list_pull_requests` / `pull_request_read` - PR操作
- `list_branches` / `list_tags` - ブランチ・タグ一覧
- `search_issues` / `search_pull_requests` - 検索

### 5.4 playwright MCP の提供ツール

ブラウザ自動化（Chromium）による Web アクセスを提供：

- スクリーンショット取得、ページナビゲーション
- フォーム入力、クリック操作
- アクセシビリティスナップショット
- 許可オリジン: `localhost`, `127.0.0.1`（セキュリティ制限）

### 5.5 ユーザー設定 MCP

`GITHUB_COPILOT_MCP_JSON` 環境変数（または `GITHUB_COPILOT_MCP_JSON_FROM_INPUT` のBase64）で追加サーバーを指定可能。

`uvx` または `python` 使用時は自動的に `uvx` / `pipx` をインストールします。

---

## 6. eBPF ファイアウォール（Padawan-FW）

### 6.1 概要

eBPF（Extended Berkeley Packet Filter）技術を使用したカーネルレベルのネットワーク制御を実装しています。`padawan-fw` バイナリが eBPF プログラムをカーネルにロードし、全ネットワーク通信を制御します。

### 6.2 SSL/TLS インスペクション

`mkcert` を使用して自己署名 CA 証明書を生成し、エージェントプロセスの HTTPS 通信を透過的に検査します：

1. `mkcert -install` で CA 証明書をシステムストアに登録
2. Node.js / Python / Java / curl などの CA バンドルを上書き
3. エージェント終了後、`mkcert -uninstall` で証明書を削除

### 6.3 ドメイン許可リスト

以下のドメイン・ホストへのアクセスのみ許可（ワイルドカード対応）：

```
localhost
https://github.com/
githubusercontent.com
https://raw.githubusercontent.com/
https://objects.githubusercontent.com/
https://codeload.github.com/
https://uploads.github.com/
https://api.github.com/ (特定パスのみ)
https://*.githubusercontent.com
https://lfs.github.com/
https://github-cloud.s3.amazonaws.com/
https://api.githubcopilot.com/
api.enterprise.githubcopilot.com
https://productionresultssa{0-19}.blob.core.windows.net/
168.63.129.16  (Azure metadata)
172.18.0.1     (Docker ゲートウェイ)
```

### 6.4 ファイアウォールコンポーネント

| バイナリ | サイズ | 用途 |
|---------|-------|------|
| `padawan-fw` | 18MB | メインeBPFファイアウォール |
| `mkcert` | 5MB | CA証明書生成 |
| `bootstrap` | 2MB | 起動補助 |
| `watcher` | 10MB | プロセス監視 |
| `profiler` | 8MB | パフォーマンスプロファイリング |
| `perf` | 4MB | パフォーマンス計測 |

### 6.5 ブロック時の挙動

ブロックされたドメインへのアクセスがあった場合、セッション完了後にレポートが生成されます：

```
⚠️ Warning: I tried to connect to the following addresses, 
but was blocked by firewall rules:
- `blocked-domain.com`
  - Triggering Command: `curl` (dns block)
```

---

## 7. ツール一覧

エージェントが利用可能なツールは YAML 定義の `tools` セクションで制御されます。

### 7.1 ローカルツール

| ツール | 説明 |
|--------|------|
| `bash` | シェルコマンド実行（非同期・同期） |
| `view` | ファイル/ディレクトリ表示 |
| `create` | 新規ファイル作成 |
| `edit` | ファイル編集（文字列置換） |
| `grep` | ripgrep 検索 |
| `glob` | ファイルパターンマッチング |
| `web_fetch` | URL取得（マークダウン変換） |
| `web_search` | ウェブ検索（AI要約付き） |
| `task` | サブエージェント（task型）呼び出し |
| `lsp` | Language Server Protocol ツール |

### 7.2 GitHub MCP ツール（エイリアス）

`github/` プレフィックスで呼び出し可能（`github-mcp-server/` の省略形）。

### 7.3 Bluebird（セマンティック検索）ツール

| カテゴリ | ツール |
|---------|--------|
| コンテンツ検索 | `search_file_content`, `do_fulltext_search`, `do_vector_search`, `do_hybrid_search` |
| ファイル操作 | `get_file_content`, `get_file_chunk`, `search_file_paths` |
| コード構造解析 | クラス/構造体の親子関係、メンバー関数、変数、型階層 |
| 関数分析 | 呼び出し元/先の関数、マクロ展開 |
| Git履歴 | コミット検索（説明文・時刻・作者・PR・SHA） |

---

## 8. 起動シーケンス

```
1. GitHub Actions ジョブ開始
   └── copilot-setup-steps（ユーザー定義セットアップ）実行

2. Node.js セットアップ (setup.sh)
   ├── RUNNER_TOOL_CACHE からキャッシュされた Node.js v24.x を使用
   └── なければ GitHub releases からダウンロード

3. MCPサーバー起動 (start-mcp-servers.sh)
   ├── GITHUB_COPILOT_MCP_JSON を解析
   ├── 必要に応じて uvx/pipx をインストール
   ├── mcp/dist/index.js をバックグラウンド起動
   └── http://127.0.0.1:2301/health をポーリング（最大60秒）

4. eBPF ファイアウォール起動 (ebpf/launch.sh)
   ├── COPILOT_AGENT_FIREWALL_ENABLED を確認
   ├── ドメイン許可リストをサニタイズ
   ├── mkcert で CA 証明書を生成・インストール
   └── padawan-fw でファイアウォールを有効化し、エージェントを起動

5. エージェント本体起動 (dist/index.js)
   ├── Job config を取得（Job ID: COPILOT_AGENT_JOB_ID）
   ├── CAPI セッショントークンを受信
   ├── リポジトリをクローン（GITHUB_WORKSPACE）
   ├── ベースコミットを解決
   ├── MCPサーバーからツールリストを取得
   └── 問題解決ループ開始

6. エージェント実行中
   ├── LLM（Claude/GPT）へリクエスト送信
   ├── ツール呼び出し（MCP経由）
   ├── コード修正・コミット
   └── trajectory.md に行動ログを記録

7. 完了・報告
   ├── PR/コメントを作成/更新
   ├── セッションログを api.enterprise.githubcopilot.com へ送信
   └── ファイアウォール解除・証明書削除
```

---

## 9. 環境変数・設定パラメータ

機密情報を除いた主要な環境変数一覧です。

### 9.1 エージェント制御

| 変数名 | 例・説明 |
|--------|---------|
| `COPILOT_AGENT_SESSION_ID` | UUIDv4（セッション識別子） |
| `COPILOT_AGENT_JOB_ID` | ジョブ識別子 |
| `COPILOT_AGENT_BRANCH_NAME` | エージェントが作業するブランチ名 |
| `COPILOT_AGENT_BASE_COMMIT` | ベースブランチ（例: `refs/heads/main`） |
| `COPILOT_AGENT_TIMEOUT_MIN` | タイムアウト分数（最大59分） |
| `COPILOT_AGENT_ACTION` | アクションタイプ（例: `fix`） |
| `COPILOT_AGENT_CALLBACK_URL` | 結果送信先URL |
| `COPILOT_AGENT_START_TIME_SEC` | 起動時刻（UNIXタイムスタンプ） |

### 9.2 セキュリティ

| 変数名 | 説明 |
|--------|------|
| `COPILOT_AGENT_FIREWALL_ENABLED` | ファイアウォール有効化フラグ |
| `COPILOT_AGENT_FIREWALL_ALLOW_LIST` | ドメイン許可リスト（カンマ区切り） |
| `COPILOT_AGENT_FIREWALL_ENABLE_RULESET_ALLOW_LIST` | ルールセット許可リスト使用フラグ |
| `COPILOT_AGENT_FIREWALL_RULESET_ALLOW_LIST` | gzip+base64エンコードされたルールセット |
| `COPILOT_AGENT_CONTENT_FILTER_MODE` | コンテンツフィルターモード（`hidden_characters`） |

### 9.3 MCP 設定

| 変数名 | 説明 |
|--------|------|
| `COPILOT_MCP_ENABLED` | MCP 有効フラグ |
| `COPILOT_MCP_READ_ONLY_MODE` | 読み取り専用モード |
| `COPILOT_AGENT_MCP_SERVER_TEMP` | MCP サーバー一時ディレクトリ |
| `GITHUB_COPILOT_MCP_JSON` | ユーザー設定MCP JSON |
| `GITHUB_COPILOT_MCP_JSON_FROM_INPUT` | Base64エンコードされたMCP JSON |

### 9.4 セッション管理

| 変数名 | 説明 |
|--------|------|
| `COPILOT_USE_SESSIONS` | セッション機能有効フラグ |
| `COPILOT_USE_ASYNC_SESSIONS` | 非同期セッション機能フラグ |
| `CPD_SAVE_TRAJECTORY_OUTPUT` | trajectory.md の保存パス |
| `COPILOT_FEATURE_FLAGS` | 有効なフィーチャーフラグ一覧 |
| `COPILOT_EXPERIMENTS` | A/B テスト設定 |
| `COPILOT_AGENT_COMMIT_LOGIN` | コミット時のユーザー名 |
| `COPILOT_AGENT_COMMIT_EMAIL` | コミット時のメールアドレス |

### 9.5 ランタイム情報

| 変数名 | 例 |
|--------|---|
| `COPILOT_AGENT_RUNTIME_VERSION` | `runtime-copilot-{commitSHA}` |
| `COPILOT_AGENT_SOURCE_ENVIRONMENT` | `production` |
| `COPILOT_JOB_EVENT_TYPE` | `cli_delegate_command` |
| `COPILOT_AGENT_TIMING_SECTIONS` | タイミング計測結果（例: `launch.firewall_prepare:17ms`） |

---

## 10. ~/.copilot ディレクトリの分析

> **注意**: GitHub Copilot Coding Agent（クラウドエージェント）は GitHub Actions の**エフェメラルランナー**上で動作するため、`~/.copilot` ディレクトリは**存在しません**。このセクションはローカル環境での GitHub Copilot 各種クライアントにおける設定ディレクトリの分析です。

### 10.1 クラウドエージェントの認証

クラウドエージェントでは `~/.copilot` を使用せず、以下の方法で認証します：

- GitHub Actions の `GITHUB_TOKEN` / `COPILOT_AGENT_API_TOKEN` 環境変数
- `COPILOT_AGENT_CALLBACK_URL` を通じたサーバーサイド認証
- GitHub OIDC トークンによるセッション管理

### 10.2 ローカル環境での設定ディレクトリ

| クライアント | 設定ディレクトリ | 主要ファイル |
|------------|----------------|------------|
| GitHub Copilot for VS Code | `~/.vscode/extensions/github.copilot-*/` | 拡張機能設定 |
| gh copilot 拡張 | `~/.config/gh/` | `hosts.yml`（認証トークン） |
| Copilot for Neovim | `~/.copilot/` | `hosts.json`（認証情報） |
| Copilot for JetBrains | `~/.config/github-copilot/` | `hosts.json` |

### 10.3 ~/.copilot ディレクトリ（Neovim/Vim 向け）

GitHub Copilot for Neovim（`github/copilot.vim`）が使用する設定ディレクトリの構造：

```
~/.copilot/
└── hosts.json    # GitHub ホストごとの OAuth トークン（機密情報）
```

**hosts.json の構造（スキーマのみ、値は非公開）**:
```json
{
  "github.com": {
    "user": "<GitHubユーザー名>",
    "oauth_token": "<OAuthトークン（機密）>"
  }
}
```

> ⚠️ `hosts.json` は OAuth トークンを平文で保存するため、このファイルは **絶対に公開しないでください**。

### 10.4 ~/.config/github-copilot/ ディレクトリ

GitHub Copilot を使用する IDE（JetBrains, VS Code など）が共通で使用する場合があります：

```
~/.config/github-copilot/
├── hosts.json     # 認証情報（機密）
└── apps.json      # アプリケーション設定
```

### 10.5 gh copilot 拡張の認証

`gh` CLI 経由の認証は `~/.config/gh/hosts.yml` に保存されます。クラウドエージェントへのアクセス時は GitHub OAuth スコープが必要です。

### 10.6 Copilot CLI（ローカル実行）の設定

`gh copilot` 拡張コマンドを使用する場合の MCP 設定ファイル：

- **ユーザー設定**: `~/.config/github-copilot/mcp-config.json`（または OS依存のconfigDir）
  ```json
  {
    "mcpServers": {
      "my-server": {
        "command": "npx",
        "args": ["my-mcp-server"]
      }
    }
  }
  ```
- **プロジェクト設定**: `.mcp.json`（リポジトリルート）
  ```json
  {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": {
          "API_KEY": "${input:apiKey}"
        }
      }
    }
  }
  ```

---

## 11. 依存ライブラリ・技術スタック

### 11.1 AI/LLM SDK

| ライブラリ | バージョン | 用途 |
|-----------|-----------|------|
| `@anthropic-ai/sdk` | ^0.88.0 | Claude API クライアント |
| `@anthropic-ai/claude-agent-sdk` | ^0.1.62 | Claude エージェントSDK |
| `openai` | ^5.20.1 | OpenAI API クライアント |
| `@openai/codex-sdk` | ^0.66.0 | Codex SDK |
| `@github/copilot-engine-sdk` | ^0.0.1 | Copilot Engine SDK |

### 11.2 MCP / エージェントプロトコル

| ライブラリ | バージョン | 用途 |
|-----------|-----------|------|
| `@modelcontextprotocol/sdk` | ^1.27.1 | MCP クライアント/サーバー |
| `@agentclientprotocol/sdk` | ^0.14.1 | エージェント通信プロトコル |
| `@github/mcp-registry` | ^0.1.5 | MCP レジストリ |

### 11.3 コード解析

| ライブラリ | 用途 |
|-----------|------|
| `web-tree-sitter` + 各言語パーサー | AST 解析（Bash/C/C#/C++/CSS/Go/HTML/Java/JS/JSON/PHP/PowerShell/Python/Ruby/Rust/Scala/TypeScript） |
| `vscode-languageserver-protocol` | LSP 通信 |
| `js-tiktoken` | トークン数カウント |

### 11.4 UI フレームワーク

| ライブラリ | 用途 |
|-----------|------|
| `ink` + `react` | CLI の TUI（Terminal UI）実装 |
| `@inkjs/ui` | Ink コンポーネントライブラリ |

### 11.5 システム統合

| ライブラリ | 用途 |
|-----------|------|
| `node-pty` | 疑似端末（PTY）制御 |
| `@xterm/headless` | ヘッドレス端末エミュレーター |
| `sharp` | 画像処理（スクリーンショット解析） |
| `@github/computer-use-mcp` | コンピューター操作MCP |

### 11.6 ビルドツール

| ツール | 用途 |
|--------|------|
| `esbuild` | 高速バンドル |
| `vitest` | テストランナー |
| `biome` | Linter/フォーマッター |
| `typescript` | 型チェック |

---

## 12. ネットワーク通信とセキュリティ

### 12.1 主要エンドポイント

| エンドポイント | 用途 |
|--------------|------|
| `https://api.enterprise.githubcopilot.com/agents/swe/agent` | エージェント結果コールバック |
| `https://api.enterprise.githubcopilot.com/mcp/readonly` | GitHub MCP サーバー（リモート） |
| `https://api.github.com/` | GitHub REST API |
| `https://github.com/` | リポジトリクローン等 |

### 12.2 セキュリティ設計

1. **最小権限**: GitHub MCP はデフォルトで read-only
2. **ネットワーク分離**: eBPF ファイアウォールによるドメイン制限
3. **SSL インスペクション**: mkcert CA による TLS 通信の監視
4. **コンテンツフィルター**: `hidden_characters` モードで出力をフィルタリング
5. **コミット署名**: `COPILOT_AGENT_SIGN_COMMITS` によるGPG署名（オプション）
6. **エフェメラル環境**: 実行後に証明書・一時ファイルを自動削除

### 12.3 firewall ログ形式

eBPF ファイアウォールのログは JSONL 形式で記録されます：

```json
{
  "time": "2026-04-19T15:50:52.642Z",
  "level": "INFO",
  "msg": "DNS server started",
  "port": 57144,
  "blocked": false,
  "domains": "example.com",
  "cmd": "curl",
  "blockedAt": "dns"
}
```

---

## 13. フィーチャーフラグと実験的機能

現在有効なフィーチャーフラグ（2026-04-19 時点）：

| フラグ名 | 説明 |
|---------|------|
| `copilot_swe_agent_blackbird_tool_use` | Bluebird セマンティック検索の使用 |
| `copilot_swe_agent_firewall_enabled_by_default` | ファイアウォールデフォルト有効 |
| `copilot_swe_agent_vision` | 画像認識（スクリーンショット解析） |
| `copilot_swe_agent_parallel_tool_execution` | 並列ツール実行 |
| `copilot_swe_agent_enable_security_tool` | セキュリティ解析ツール |
| `copilot_swe_agent_code_review` | コードレビューエージェント |
| `copilot_swe_agent_new_out_proc_mcp` | 新しい out-of-process MCP |
| `copilot_swe_agent_enable_dependabot_checker` | Dependabot チェッカー |
| `coding_agent_plan_tags` | プラン タグ機能 |
| `copilot_mission_control_decoupled_mode` | Mission Control 分離モード |
| `copilot_swe_agent_unified_task_tool` | 統合タスクツール |
| `copilot_swe_agent_parallel_validation` | 並列バリデーション |
| `copilot_swe_agent_semantic_issues_search` | セマンティックIssue検索 |
| `coding_agent_commit_signing` | コミット署名 |
| `copilot-feature-agentic-memory` | エージェントメモリ機能 |

---

## 14. 信頼度評価

| 項目 | 信頼度 | 根拠 |
|------|--------|------|
| ディレクトリ構造 | **高** | 実際のファイルシステムを直接観察 |
| エージェント定義 | **高** | YAML ファイルを直接読取り |
| 起動シーケンス | **高** | シェルスクリプトとログを直接確認 |
| MCP アーキテクチャ | **高** | MCP サーバーログと設定を直接確認 |
| eBPF ファイアウォール | **高** | launch.sh と fw.jsonl を直接確認 |
| 環境変数 | **高** | 実行中の `env` コマンド出力から取得（機密値除外） |
| フィーチャーフラグ | **高** | `COPILOT_FEATURE_FLAGS` 環境変数から取得 |
| ~/.copilot（ローカル） | **中** | クラウド環境には存在しないため、公開情報から補完 |
| LLM モデル選択ロジック | **中** | YAML 定義から推測（内部実装は非公開） |
| API 通信の詳細 | **低〜中** | エンドポイントは確認できたが、プロトコル詳細は不明 |

---

## 参考情報

- エージェントバージョン（コミットSHA）: `69cf4482e0cf5287ae6e8fc7f5e8d437a8665d1b`
- eBPF/Autofind バージョン: `69cf4482e0cf5287ae6e8fc7f5e8d437a8665d1b, 0.0.52, 1.4.4`
- Autofind バージョン: `69cf4482e0cf5287ae6e8fc7f5e8d437a8665d1b, 4.4.11`
- Node.js バージョン: v24.x（TARGET: `24.*`, FALLBACK: `24.13.0`）
- 実行OS: Ubuntu x86_64 on Azure (`Linux ... 6.17.0-1010-azure ... GNU/Linux`)
- 調査日時: 2026-04-19T15:50:58 UTC
