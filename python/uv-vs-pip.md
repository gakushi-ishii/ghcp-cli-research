# uv と pip の違い（2026年4月時点）

## 結論

- **pip** は Python 標準寄りの「パッケージインストーラ」で、依存関係のインストールを中心に使うツール。
- **uv** は Rust 製の高速な統合ツールで、`pip` に加えて **仮想環境管理・ロックファイル管理・Python 本体の導入・ツール実行** までまとめて扱える。

つまり、**「最小構成で広く互換性を重視するなら pip」**、**「速度と開発体験を重視するなら uv」** という違いがある。

---

## 比較表

| 観点 | pip | uv |
| --- | --- | --- |
| 主目的 | Python パッケージのインストール | Python プロジェクトと依存関係の統合管理 |
| 実装 | Python | Rust |
| 速度 | 標準的 | `pip` より大幅に高速と案内されている |
| 仮想環境 | `venv` / `virtualenv` と組み合わせるのが基本 | `uv venv` や `uv run` で標準対応 |
| ロックファイル | `pip lock` は **experimental**、または別ツール併用が一般的 | `uv.lock` を標準利用 |
| Python 本体の管理 | 非対応 | `uv python install` で対応 |
| ツール実行 | `pipx` など別ツールを使うことが多い | `uvx` / `uv tool` で対応 |
| 既存資産との互換性 | 非常に高い | `uv pip` で `pip` 系ワークフローを取り込みやすい |

---

## 1. pip の特徴

`pip` は PyPI などから Python パッケージをインストールするための標準的なツールで、Python エコシステムで最も広く使われている。

強み:

- Python に標準的に付属し、導入コストが低い
- 多くの CI/CD、社内ドキュメント、既存運用が `pip install -r requirements.txt` 前提で組まれている
- 互換性と普及度が非常に高い

注意点:

- 仮想環境作成、Python バージョン切替、CLI ツール隔離実行は別ツールと組み合わせることが多い
- 依存解決でバックトラックが多くなると時間がかかることがある
- ロックファイル機能はあるが、現時点では **experimental**

---

## 2. uv の特徴

`uv` は Astral が開発している Rust 製ツールで、単なる `pip` 互換ではなく、Python 開発で使いがちな複数ツールをまとめて置き換える方向の設計になっている。

強み:

- 高速な依存解決・インストール
- `uv.lock` による再現性の高い環境管理
- `uv venv`、`uv run` により仮想環境運用が簡単
- `uv python install` で Python 本体も管理可能
- `uvx` / `uv tool install` で Python 製 CLI ツールも扱いやすい
- `uv pip install` など、既存の `pip` 風コマンドを使った段階移行がしやすい

注意点:

- 新しい統合ツールなので、古い社内手順や一部の特殊な `pip` 運用では検証が必要
- チーム内で `pyproject.toml` / `uv.lock` ベース運用に寄せる合意があると効果を出しやすい

---

## 3. 使い分けの目安

### pip を選びやすいケース

- 既存のプロジェクトや CI が `pip` 前提
- まずは最小限の構成で運用したい
- 新しいツール導入コストを避けたい

### uv を選びやすいケース

- 新規プロジェクトを始める
- 依存解決やインストール時間を短縮したい
- 仮想環境、ロックファイル、Python バージョン管理を一つにまとめたい
- `pip-tools`、`pipx`、`pyenv` などを整理したい

---

## 4. 一言でいうと

- **pip**: 「標準的で互換性重視のパッケージインストーラ」
- **uv**: 「高速で統合的な Python 開発ツールチェーン」

---

## 参考

- pip documentation: <https://pip.pypa.io/en/stable/>
- pip dependency resolution: <https://pip.pypa.io/en/stable/topics/dependency-resolution/>
- pip lock: <https://pip.pypa.io/en/stable/cli/pip_lock/>
- uv on PyPI: <https://pypi.org/project/uv/>
