# GitHub Copilot とサンドボックス環境での利用調査

> 最終更新: 2026-04-23  
> 対象: GitHub Copilot / Claude Code / OpenAI Codex  
> 方針: 公式ドキュメント・公式ブログ・技術論文を優先

## Executive Summary

- GitHub Copilot をサンドボックス環境で使う主な方法は、**GitHub Actions 上の Copilot cloud agent** を使う方法と、**Docker Sandbox 上で Copilot CLI を動かす方法**の 2 系統に整理できる。前者は GitHub ネイティブ、後者はローカル隔離向けである。[^1][^2][^3]
- GitHub Copilot cloud agent は **ephemeral な GitHub Actions 環境**で動作し、`.github/workflows/copilot-setup-steps.yml` によって依存関係・OS・ランナー種別・環境変数・サービスを事前定義できる。[^1]
- Copilot cloud agent には標準でファイアウォールがあり、外向き通信を allowlist ベースで制御できるが、**Bash から起動したプロセスにしか効かない**、**MCP サーバーや setup steps には効かない**などの限界も明記されている。[^2]
- Claude Code と Codex はどちらも **ローカル実行寄りのコーディングエージェント**であり、GitHub Copilot よりも「CLI/ローカル実行時の sandbox・approval policy」の設計が前面に出ている。Claude Code は sandbox + auto mode、Codex CLI は bubblewrap / seccomp / approval policy / non-interactive 実行が特徴的である。[^4][^5][^6][^7][^8][^9]
- AI による自立開発では、単に高性能モデルを使うだけでなく、**harness engineering**（実行環境、権限、フィードバック、検証、ログ、レビュー線引きの設計）が重要だという議論が 2025–2026 年に急速に強まっている。[^10][^11][^12][^13]

---

## 1. GitHub Copilot をサンドボックス環境で使う手段

### 1.1 GitHub Copilot cloud agent（GitHub Actions ベース）

GitHub の公式ドキュメントでは、Copilot cloud agent は **GitHub Actions によって提供される ephemeral development environment** 上で動作すると説明されている。エージェントはこの環境でコード探索、変更、テスト、lint 実行などを行う。[^1]

この方法の実務上のポイントは以下。

- リポジトリに `.github/workflows/copilot-setup-steps.yml` を置くことで、Copilot が作業を始める前の環境準備を固定化できる。[^1]
- workflow は **単一の `copilot-setup-steps` job** を持つ必要があり、ファイルは **default branch 上に存在していないと有効化されない**。[^1]
- カスタマイズできるのは `steps`, `permissions`, `runs-on`, `services`, `snapshot`, `timeout-minutes`（最大 59 分）に限定される。[^1]
- 標準 GitHub-hosted runner だけでなく、**larger runner** や **self-hosted runner** も使える。[^1]
- self-hosted runner を使う場合は、GitHub は **ephemeral / single-use runner** を推奨しており、ARC（Actions Runner Controller）や Runner Scale Set Client の利用を案内している。[^1]

つまり GitHub Copilot の「サンドボックス」は、ローカル PC のコンテナというより、**GitHub Actions を制御平面として使うリモート実行サンドボックス**として理解するのが正確である。[^1]

### 1.2 ネットワーク制御とファイアウォール

Copilot cloud agent では、既定でインターネットアクセスがファイアウォールで制限される。GitHub Docs はこの目的を **data exfiltration risk の抑制**と明記している。[^2]

重要なのは「強いが万能ではない」点である。

- 既定では推奨 allowlist が有効で、OS パッケージ、主要言語のパッケージレジストリ、証明書検証、Playwright 用ブラウザ取得などの一般的ホストが許可される。[^2]
- ブロックされた通信があると、PR 本文またはコメントに blocked address と実行コマンドが記録される。[^2]
- ただし firewall は **agent の Bash tool から起動したプロセスにしか効かず**、**MCP サーバー**や**Copilot setup steps で起動されたプロセスには適用されない**。[^2]
- GitHub 自身も、この firewall は **comprehensive security solution ではない**と明言している。[^2]
- self-hosted runner 利用時は、組み込み firewall は互換性がないため無効化し、自前のネットワーク制御を構成する必要がある。[^1][^2]

このため、Copilot cloud agent を企業内ネットワークで使う場合は、**GitHub 側の firewall 設定 + runner 側のネットワーク境界**の二層で考える必要がある。[^1][^2]

### 1.3 Docker Sandbox 上で Copilot CLI を使う

Docker Docs には、GitHub Copilot を Docker Sandbox 上で使う専用ガイドがある。最小例は以下。[^3]

```bash
sbx run copilot ~/my-project
```

Docker Sandbox 方式の特徴は次の通り。[^3]

- 認証は `sbx secret set -g github` あるいは `GH_TOKEN` / `GITHUB_TOKEN` で与える。[^3]
- sandbox は **ホストのユーザー単位設定を引き継がない**。利用できるのは working directory 内の project-level config のみ。[^3]
- Copilot 用 template は workspace を既定で trust し、承認プロンプトを繰り返さずに動かす前提で構成されている。[^3]
- base image を差し替えたり、カスタム環境でツールを preinstall することもできる。[^3]

この方式は、**ローカル開発者が disposable な隔離実行環境を自分で持ちたい場合**に向いている。一方で、GitHub 上の PR / branch protection / Actions との一体運用は cloud agent 方式の方が自然である。[^1][^3]

### 1.4 GitHub Copilot をサンドボックスで使う際の選択指針

| 目的 | 向く方式 | 理由 |
|---|---|---|
| GitHub 上の Issue/PR 駆動で運用したい | Copilot cloud agent | GitHub Actions, PR, branch protection, session log と統合しやすい[^1][^2] |
| 社内ネットワークや内部 artifact にアクセスさせたい | self-hosted / larger runner 上の cloud agent | runner・ネットワーク境界を自社管理できる[^1] |
| 開発者ローカルで disposable な隔離環境を使いたい | Docker Sandbox + Copilot CLI | ホスト設定を引き継がず、project 単位に閉じやすい[^3] |
| 依存関係導入を trial-and-error ではなく deterministic にしたい | `copilot-setup-steps.yml` | setup を workflow として明示化できる[^1] |

---

## 2. Claude Code / Codex との違い

### 2.1 大きな違いは「どこで実行するか」

GitHub Copilot cloud agent は **GitHub Actions 上のリモート実行**が中心である。これに対し、Claude Code と Codex CLI は **ローカル実行を基本に、sandbox と approvals を前面に出した設計**になっている。[^1][^4][^7]

### 2.2 Claude Code

Anthropic の公開情報では、Claude Code の安全性強化は **permission prompt を増やすだけでなく、filesystem isolation・network isolation を組み合わせる sandboxing** で実現する方向が強調されている。さらに auto mode では、安全にスキップ可能な権限確認を classifier 付きで自動化する方針が示されている。[^4][^5]

要するに Claude Code は、

- ローカル / CLI / IDE 側の作業体験をベースにしつつ、[^6]
- sandbox で危険操作の範囲を狭め、[^4]
- その上で permission prompt の頻度を最適化する、[^5]

という思想が比較的明確である。

GitHub Copilot cloud agent との違いは、GitHub Copilot が **GitHub Actions runner の隔離**と **PR ワークフロー統合**を軸にしているのに対し、Claude Code は **ローカル実行エージェントとしての sandbox / autonomy balance** が前面に出ている点である。[^1][^4][^5][^6]

### 2.3 OpenAI Codex / Codex CLI

OpenAI の Codex CLI は README で **“runs locally on your computer”** と説明されている。[^7] Linux sandbox 実装では bubblewrap を既定にし、`PR_SET_NO_NEW_PRIVS`、seccomp、read-only filesystem、network namespace の分離など、かなり具体的な実装が公開されている。[^8]

また OpenAI の公式ドキュメントでは、

- sandboxing と approvals が既定の安全境界であること、[^9]
- `codex exec` による non-interactive / CI 実行ができること、[^10]
- `--ephemeral` などの automation 向けモードを持つこと[^10]

が案内されている。

したがって Codex は、

- **ローカル CLI を強い OS-level sandbox で囲う設計**
- **approval policy を明示的に切り替える設計**
- **CI に流し込める non-interactive 実行**

が特徴で、GitHub Copilot cloud agent のように GitHub Actions 自体を sandbox として借りる構図とはかなり異なる。[^1][^8][^9][^10]

### 2.4 比較表

| 観点 | GitHub Copilot cloud agent | Claude Code | OpenAI Codex CLI |
|---|---|---|---|
| 主な実行場所 | GitHub Actions 上の ephemeral 環境[^1] | ローカル / IDE / CLI 中心[^4][^6] | ローカル CLI 中心[^7] |
| サンドボックスの主軸 | Actions runner + firewall + runner/network controls[^1][^2] | filesystem / network isolation + permissions[^4][^5] | bubblewrap / seccomp / approvals[^8][^9] |
| CI への組み込み | GitHub に強く統合[^1] | 背景実行や CI 連携を拡張[^6] | `codex exec` による non-interactive 実行[^10] |
| 運用の中心 | PR / branch protection / session logs[^1][^2] | autonomy と user approval の均衡[^4][^5] | local-first agent を安全に自動化[^8][^9][^10] |

---

## 3. 自立開発パイプライン設計・運用論・Human-in-the-Loop

### 3.1 2025–2026 年の論点は「prompt」より「harness」

OpenAI は 2026 年の公式記事で、agent-first world ではモデル単体より **harness engineering** が重要になると論じている。ここでいう harness は、タスクの与え方、利用ツール、チェックポイント、フィードバック、検証、出力フォーマットまで含む実行枠組みである。[^11]

GitHub Copilot の setup steps も同じ方向の設計思想と読める。GitHub Docs は、依存関係導入を LLM に trial-and-error で任せるのは **slow and unreliable** だと説明し、setup を deterministic に与えることを推奨している。[^1]

つまり近年の議論は、

- 良いモデルを使う
- 良い prompt を書く

だけでは不十分で、

- **どんな環境で**
- **どの権限で**
- **どの順序で**
- **どの検証を通して**
- **どこで人間が止めるか**

まで設計しないと自立開発は安定しない、という方向に移っている。[^1][^4][^5][^11]

### 3.2 Human-in-the-Loop は依然として必要

SE 3.0 / AIDev の論文は、Codex, Devin, Copilot, Cursor, Claude Code などによる **45.6 万件超の agentic PR** を集計し、AI エージェントが実運用レベルで広く使われ始めている一方、**受理率では人間との差が残る**ことを示している。[^12]

この示唆は明快である。

- AI は PR を大量に作れる
- しかし merge 判断や信頼形成はまだ人間レビューに強く依存する

よって Human-in-the-Loop / Human-on-the-Loop は、単なる慎重論ではなく、現実の運用データに基づく設計要件とみなすのが妥当である。[^12]

MIT の 2025 年記事も、自律的ソフトウェア工学のボトルネックとして、コーディング単体より **評価・検証・工程横断の統合**を挙げている。[^13]

### 3.3 実務向けの設計パターン

信頼できる議論を総合すると、自動実装パイプラインはおおむね以下のように設計するのが妥当である。[^1][^2][^4][^5][^8][^9][^10][^11][^12][^13]

1. **Task intake / triage**  
   Issue, ticket, spec を構造化し、危険度・影響範囲・必要権限で分類する。
2. **Ephemeral environment provisioning**  
   single-use runner / disposable container / sandbox を都度払い出す。
3. **Deterministic bootstrap**  
   依存関係、SDK、キャッシュ、シークレット注入ルール、ネットワーク allowlist を固定化する。
4. **Least-privilege execution**  
   filesystem, network, secrets, git push, deploy 権限を最小化する。
5. **Automated validation**  
   lint, targeted tests, build, static analysis, security scan を agent 実行の出口に置く。
6. **Human review gates**  
   protected branch, CODEOWNERS, required checks, manual approval を必須化する。
7. **Auditability**  
   session logs, workflow logs, diff, approval 履歴、利用ツールを残す。
8. **Post-merge feedback loop**  
   採択率、revert 率、障害率、レビュー工数、lead time を継続観測する。

### 3.4 ハーネスエンジニアリング観点の要点

実務で特に重要なのは次の 5 点である。[^1][^2][^4][^5][^8][^9][^10][^11]

- **環境を宣言的に作る**: setup steps や container image をコード化する。  
- **権限境界を明文化する**: 何が read-only / writable / network denied かを定義する。  
- **失敗時の観測性を確保する**: blocked network, failed setup, failing tests を人間が読める形で残す。  
- **高リスク変更は approval を残す**: infra, secrets, dependency, deploy は fully autonomous にしない。  
- **評価系をベンチマーク外まで広げる**: merge 後品質や人間レビュー負荷まで見る。  

---

## 4. 現時点の結論

### GitHub Copilot をサンドボックスで使うなら

1. **GitHub ネイティブに運用するなら Copilot cloud agent が本命**  
   `copilot-setup-steps.yml`、firewall、branch protection、single-use runner を組み合わせる。[^1][^2]
2. **ローカル隔離が必要なら Docker Sandbox + Copilot CLI が扱いやすい**  
   特にホスト設定を持ち込みたくない場合に向く。[^3]
3. **社内閉域・内部依存が多いなら self-hosted runner だが、ネットワーク制御は自前責任**  
   integrated firewall だけに依存しない。[^1][^2]

### 他エージェントとの差分を一言で言うと

- **GitHub Copilot**: GitHub Actions / PR ワークフロー中心のサンドボックス
- **Claude Code**: ローカル実行エージェントの autonomy と sandbox のバランス重視
- **Codex CLI**: ローカル first かつ OS-level sandbox / approval policy / CI 自動化が明示的

### 自立開発を構想するなら

重要なのは「最強のモデル」よりも、**sandbox・bootstrap・approvals・validation・audit trail をどう設計するか**である。2025–2026 年の一次情報は、その設計全体を **harness engineering** として扱う方向に収束しつつある。[^4][^5][^10][^11][^12][^13]

---

## 参考文献

[^1]: [Configure the development environment - GitHub Docs](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/customize-the-agent-environment) — Copilot cloud agent の ephemeral 環境、setup steps、larger runner / self-hosted runner の公式説明
[^2]: [Customizing or disabling the firewall for GitHub Copilot cloud agent - GitHub Docs](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall) — firewall の目的、allowlist、制限事項
[^3]: [Copilot - Docker Docs](https://docs.docker.com/ai/sandboxes/agents/copilot/) — Docker Sandbox 上での Copilot 利用方法
[^4]: [Making Claude Code more secure and autonomous with sandboxing - Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-sandboxing) — Claude Code の sandboxing 方針
[^5]: [Claude Code auto mode: a safer way to skip permissions - Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-auto-mode) — Claude Code の auto mode と permission 最適化
[^6]: [Claude Code | Anthropic's agentic coding system](https://www.anthropic.com/product/claude-code) — Claude Code の製品説明とワークフロー統合
[^7]: [openai/codex README](https://github.com/openai/codex/blob/main/README.md) — Codex CLI はローカル実行型の coding agent であることの一次情報
[^8]: [openai/codex linux-sandbox README](https://github.com/openai/codex/blob/main/codex-rs/linux-sandbox/README.md) — bubblewrap, seccomp, read-only filesystem, network namespace などの実装情報
[^9]: [Sandbox & approvals - Codex Documentation](https://developers.openai.com/codex/security) — Codex の sandbox / approval の公式説明
[^10]: [Non-interactive mode - Codex Documentation](https://developers.openai.com/codex/noninteractive) — `codex exec` と CI/automation 向け実行の公式説明
[^11]: [Harness engineering: leveraging Codex in an agent-first world - OpenAI](https://openai.com/index/harness-engineering/) — harness engineering を公式に整理した記事
[^12]: [The Rise of AI Teammates in Software Engineering (SE) 3.0: How Autonomous Coding Agents Are Reshaping Software Engineering - arXiv](https://arxiv.org/abs/2507.15003) — 大規模 agentic PR データセットと受理率の分析
[^13]: [Can AI really code? Study maps the roadblocks to autonomous software engineering - MIT CSAIL / MIT Schwarzman College of Computing](https://computing.mit.edu/news/can-ai-really-code-study-maps-the-roadblocks-to-autonomous-software-engineering/) — 自律的ソフトウェア工学のボトルネック整理
