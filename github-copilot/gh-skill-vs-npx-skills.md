# gh skill と npx skills の比較調査

> 最終更新: 2026-04-24  
> 対象: GitHub CLI `gh skill` / Vercel Labs `npx skills`  
> 方針: 公式ドキュメント・公式ブログ・技術情報を優先

## Executive Summary

- **`gh skill`** は 2026 年 4 月 16 日に GitHub CLI v2.90.0 で公開された公式コマンドで、エージェントスキルを npm パッケージのように install / pin / update / publish できる。GitHub エコシステムとの深い統合、サプライチェーン整合性（SHA ピン・不変リリース）、エンタープライズ向けガバナンスが強みである。[^1][^2]
- **`npx skills`** は 2026 年 2 月に Vercel Labs がリリースしたオープンソース CLI で、GitHub / GitLab / ローカルなど任意のソースからスキルを管理できる。Claude Code・Codex・Cursor・Copilot など 40 以上のエージェントをサポートする最広義のクロスエージェント対応が特徴である。[^3][^4]
- 両ツールは共通の **Open Agent Skills 仕様**（`SKILL.md` を中心とする構造）を使用しており、スキルの相互運用性が保たれる。どちらを使っても同じスキルリポジトリを参照できる。[^1][^3]
- **GitHub / Copilot を主軸に使うチーム** には `gh skill` が、**複数エージェントを横断する柔軟な運用** を優先するチームには `npx skills` が向く。

---

## 1. 背景と位置づけ

### 1.1 gh skill（GitHub 公式）

`gh skill` は 2026 年 4 月 16 日に GitHub Changelog で発表され、GitHub CLI v2.90.0 以降で利用可能になった。[^1]

背景にあるのは、AI コーディングエージェントの普及に伴って「スキル（エージェントに特定の作業方法を教える指示・スクリプトの集合体）」を共有・管理するニーズが高まったことである。`gh skill` はこの問題に対して、**npm が JavaScript パッケージを管理するのと同じ発想** でスキルを扱うアプローチを採った。

- 公式サポート対象エージェント: Copilot、Claude Code、Cursor、Codex、Gemini CLI、Antigravity ほか
- スキルの実体: GitHub リポジトリ上にホストされた `SKILL.md` ベースの構造体

### 1.2 npx skills（Vercel Labs）

`npx skills` は 2026 年 2 月に Vercel Labs がリリースしたオープンソース CLI ツール。[^3][^4]

npm パッケージとして公開されており、`npx` で即時実行できる（グローバルインストール不要）。スキルの取得元は GitHub / GitLab / 任意の git URL / ローカルパスと幅広く、特定プラットフォームへの依存がない設計になっている。

- サポート対象エージェント: 40 以上（Claude Code、Codex、Cursor、OpenCode、Copilot、Gemini、Continue など）
- スキルの発見: [skills.sh](https://skills.sh) ディレクトリ（コミュニティ駆動のリーダーボード形式）

---

## 2. 共通基盤：Open Agent Skills 仕様

両ツールは **Open Agent Skills 仕様** を共通で使用する。スキルの実体は以下のようなディレクトリ構造のリポジトリである。[^1][^3][^5]

```
my-skill/
├── SKILL.md          # 必須: YAML フロントマター + 指示本文
├── scripts/          # 任意: エージェントが実行するヘルパースクリプト
├── references/       # 任意: ドキュメント・参考資料
└── assets/           # 任意: テンプレート・サンプル
```

`SKILL.md` のフロントマターには `name`・`description` が最低限必要で、インストール元・バージョン・git tree SHA などのプロビナンス情報も埋め込める。この仕様がオープンスタンダードであるため、スキルはどちらのツールを使ってもインストール・利用できる。[^1][^3]

---

## 3. コマンド比較

### 3.1 主要コマンド

| 操作 | `gh skill` | `npx skills` |
|---|---|---|
| スキルの検索 | `gh skill search <keyword>` | `npx skills find <keyword>` |
| インストール（基本） | `gh skill install <repo> <skill>` | `npx skills add <owner/repo>` |
| バージョン指定インストール | `gh skill install <repo> <skill>@v1.2.0` | ─（skills-lock.json で管理） |
| SHA ピン | `gh skill install <repo> <skill>@<SHA>` | ─ |
| インストール前プレビュー | `gh skill preview <repo> <skill>` | ─ |
| 一覧表示 | `gh skill list` | `npx skills list` |
| 更新 | `gh skill update --all` | `npx skills update` |
| 削除 | `gh skill remove <skill>` | `npx skills remove <skill>` |
| 新規スキル初期化 | ─ | `npx skills init <skill-name>` |
| 公開 | `gh skill publish` | リポジトリへの push で代替 |

### 3.2 インストールオプション

**`gh skill`**
```bash
# 基本インストール
gh skill install github/awesome-copilot documentation-writer

# バージョン固定
gh skill install github/awesome-copilot documentation-writer@v1.2.0

# SHA 固定（最強のピン）
gh skill install github/awesome-copilot documentation-writer@<commitSHA>

# 対象エージェントとスコープを指定
gh skill install github/awesome-copilot documentation-writer \
  --agent claude-code --scope user
```

**`npx skills`**
```bash
# GitHub ショートハンドで追加
npx skills add vercel-labs/agent-skills

# 任意の git URL から
npx skills add https://github.com/owner/repo

# ローカルパスから
npx skills add ./local-skill-folder

# グローバルスコープ
npx skills add -g vercel-labs/agent-skills

# 特定スキルのみ
npx skills add vercel-labs/agent-skills --skill my-skill

# 特定エージェント向け
npx skills add vercel-labs/agent-skills --agent claude-code
```

---

## 4. セキュリティとサプライチェーン

### 4.1 gh skill のサプライチェーン機能

`gh skill` は GitHub の既存セキュリティ基盤を活用した強固なサプライチェーン管理を提供する。[^1][^2]

- **プロビナンス追跡**: インストール元リポジトリ・バージョン/タグ・git tree SHA が `SKILL.md` フロントマターに書き込まれる。スキルフォルダを移動しても追跡情報が維持される。
- **バージョンピン**: タグまたは SHA を指定してピン。ピンされたスキルは `update --all` の対象外となり、意図しない更新を防ぐ。
- **不変リリース**: `gh skill publish` で発行したリリースは公開後に変更不可（管理者でも改ざん不可）。
- **内容ハッシュ検出**: バージョンバンプのみ（内容変更なし）の擬似的アップデートを git tree SHA で検出する。
- **インストール前プレビュー**: `gh skill preview` でコード・スクリプト・メタデータを事前確認できる。
- **エンタープライズ連携**: タグ保護、secret/code scanning、組織内プライベート公開と組み合わせ可能。

### 4.2 npx skills のバージョン管理

`npx skills` では **skills-lock.json** が `package-lock.json` に相当する役割を果たし、インストールされたスキルのバージョンを固定して再現性を確保する。[^3]

- コミュニティ向けの透明性重視設計（npm パッケージ化されているため audit が容易）
- セキュリティ報告窓口は security.vercel.com
- エンタープライズ固有のガバナンス機能は `gh skill` に比べて薄い

---

## 5. エージェント対応範囲

| エージェント | `gh skill` | `npx skills` |
|---|---|---|
| GitHub Copilot | ✅ | ✅ |
| Claude Code | ✅ | ✅ |
| Cursor | ✅ | ✅ |
| OpenAI Codex | ✅ | ✅ |
| Gemini CLI | ✅ | ✅ |
| Antigravity | ✅ | ✅ |
| OpenCode | ─ | ✅ |
| Continue | ─ | ✅ |
| Windsurf, Amp, Roo ほか | ─ | ✅ (40 以上) |

`gh skill` は主要エージェントをカバーしているが、`npx skills` の方がより幅広いエージェントへの公式サポートを持つ。[^3][^4]

---

## 6. スキル発見・エコシステム

### gh skill
- `gh skill search` コマンドで GitHub API を通じてスキルを検索
- `SKILL.md` とリポジトリメタデータを検索対象とする
- GitHub 上の公開リポジトリが自然とディスカバラブルになる

### npx skills
- [skills.sh](https://skills.sh) による専用ディレクトリサイト
- インストール数・利用統計によるリーダーボード形式
- コミュニティ主導の透明な評価とトレンド把握が可能[^4]

---

## 7. 比較まとめ

| 観点 | `gh skill` | `npx skills` |
|---|---|---|
| 提供元 | GitHub（公式） | Vercel Labs（オープンソース） |
| リリース | 2026 年 4 月（CLI v2.90.0） | 2026 年 2 月 |
| 必要ツール | GitHub CLI v2.90.0 以上 | Node.js / npx |
| エージェント対応数 | 主要 6 ～ 7 種 | 40 以上 |
| バージョン固定 | タグ / SHA ピン | skills-lock.json |
| インストール前確認 | `gh skill preview` で対応 | なし |
| サプライチェーン | SHA・不変リリース・プロビナンス | skills-lock.json |
| エンタープライズ機能 | 強（タグ保護・code scanning 連携） | 弱 |
| スキル公開 | `gh skill publish` | リポジトリ push + skills.sh 登録 |
| スキル発見 | CLI + GitHub API | CLI + skills.sh ディレクトリ |
| インストール元 | GitHub リポジトリ | GitHub / GitLab / git URL / ローカル |
| オープンスタンダード準拠 | ✅ | ✅ |

---

## 8. 使い分けの目安

### `gh skill` を選びやすいケース

- GitHub / Copilot を中心に開発ワークフローを構築している
- スキルのバージョンを SHA レベルで固定してサプライチェーンを厳密に管理したい
- 組織全体での統一管理（タグ保護・code scanning・不変リリース）が必要
- GitHub Actions や PR ワークフローとシームレスに連携させたい

### `npx skills` を選びやすいケース

- Claude Code・Cursor・OpenCode など複数エージェントを横断して同じスキルを使いたい
- GitHub 以外（GitLab・任意 git URL・ローカル）のソースからスキルを管理したい
- GitHub CLI に依存せず、Node.js / npx だけで動く軽量なワークフローを望む
- コミュニティが評価した skills.sh ディレクトリを活用したい

---

## 9. 一言でいうと

- **`gh skill`**: 「GitHub ネイティブな、エンタープライズ対応のエージェントスキルパッケージマネージャ」
- **`npx skills`**: 「プラットフォーム非依存な、コミュニティ駆動のオープンエージェントスキルマネージャ」

両ツールは競合するというよりも **相補的** な存在であり、Open Agent Skills 仕様を通じて同じスキルエコシステムを共有している。どちらを使っても良質なスキルを導入でき、チームやプロジェクトの構成に合わせて選択すればよい。

---

## 参考文献

[^1]: [Manage agent skills with GitHub CLI - GitHub Changelog](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) — `gh skill` の公式発表。コマンド仕様・プロビナンス・ピン機能・公開機能の詳細
[^2]: [What Is `gh skill`, and How It Changes Agent Skill Management From GitHub - ExpertBeacon](https://expertbeacon.com/what-is-gh-skill-and-how-it-changes-agent-skill-management-from-github/) — `gh skill` の機能詳解
[^3]: [GitHub - vercel-labs/skills: The open agent skills tool](https://github.com/vercel-labs/skills) — `npx skills` の公式リポジトリ。コマンドリファレンス・エージェント対応一覧
[^4]: [Introducing skills, the open agent skills ecosystem - Vercel Changelog](https://vercel.com/changelog/introducing-skills-the-open-agent-skills-ecosystem) — `npx skills` の公式発表。エコシステムと設計思想
[^5]: [Use Agent Skills in VS Code - VS Code Docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills) — Open Agent Skills 仕様（SKILL.md 構造）の公式ガイド
