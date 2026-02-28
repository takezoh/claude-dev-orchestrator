# setup Skill - プロジェクトへの dev-orchestrator セットアップ

## 概要
現在のプロジェクトに dev-orchestrator を対話的にセットアップする。
プロジェクト情報をヒアリングし、設定ファイル・ディレクトリ・パーミッションを一括で構成する。

## 入力
- `$ARGUMENTS`: なし（対話的にヒアリング）

## 手順

### Step 1: プロジェクト状態の確認
1. 以下のファイル/ディレクトリの存在を確認する:
   - `.claude/dev-orchestrator.yml`
   - `tasks/`
   - `.claude/settings.json`（permissions.allow）
   - `.gitignore`
2. 既にセットアップ済みの項目をリストアップする
3. 未セットアップの項目があれば続行、すべて済みなら「セットアップ済みです」と伝えて終了

### Step 2: プロジェクト情報のヒアリング
`.claude/dev-orchestrator.yml` が存在しない場合のみ実施する。
存在する場合は「既存の設定を使用します」と伝えて Step 3 へ進む。

ユーザーに以下を質問する（AskUserQuestion を使用）:

1. **プロジェクト名**: リポジトリ名やディレクトリ名からデフォルトを提案
2. **使用言語/フレームワーク**: プロジェクトのファイルを調査して自動検出し、確認を求める
   - `package.json` → Node.js/TypeScript
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `requirements.txt` / `pyproject.toml` → Python
   - `Makefile` → Make
3. **テストコマンド**: 検出した言語に基づきデフォルトを提案
   - Node.js: `npm test`
   - Go: `go test ./...`
   - Rust: `cargo test`
   - Python: `pytest`
4. **リントコマンド**（任意）: 同様にデフォルト提案
5. **PRのベースブランチ**: `main` or `master` or `develop` を自動検出して提案

### Step 3: セットアップ実行
確認済みの情報をもとに、以下を順番に実行する。
既に存在するものはスキップし、何をスキップしたか報告する。

#### 3-1. `.claude/dev-orchestrator.yml` の作成
Step 2 でヒアリングした情報をもとに設定ファイルを作成する。
テンプレートの形式に従い、ヒアリング結果を反映する。

テンプレート形式（`.claude/dev-orchestrator.yml`）:
```yaml
# claude-dev-orchestrator 設定ファイル

# プロジェクト情報
project:
  name: "<プロジェクト名>"
  languages:
    - <言語1>
    - <言語2>
  description: "<プロジェクトの概要>"

# テストコマンド
test:
  commands:
    - name: "<言語名>"
      run: "<テストコマンド>"

# リント/フォーマット
lint:
  commands:
    - "<リントコマンド>"

# PRの設定
pr:
  draft: true
  # base: "<ベースブランチ>"

# レビューループ上限回数
review:
  max_retries:
    research: 2
    design: 2
    test: 2
    implement: 3

# Claude Codeモデル設定
model: "claude-sonnet-4-6"

# Artifact（Agent間成果物）の保存先（gitignore対象）
artifacts_dir: ".agent-artifacts"
```

#### 3-2. `tasks/` ディレクトリの作成
- `tasks/` がなければ作成する

#### 3-3. `.gitignore` の更新
以下のエントリが `.gitignore` になければ追加する:
- `.agent-artifacts/`
- `agent-logs/`

#### 3-4. パーミッション設定
`.claude/settings.json` に Agent/Skill が使うコマンドの allow ルールを追加する。

1. プラグインの `templates/permissions.json` を読み込む
   - `permissions.core.allow` — 全Agent共通（ファイル操作・探索）
   - `permissions.vcs.allow` — バージョン管理・GitHub連携
   - `permissions.languages.<lang>` — 言語別のビルド・テストコマンド
2. 現在の `.claude/settings.json` の `permissions.allow` を読み取り、追加が必要なものを特定する

**ユーザーに確認を求める（AskUserQuestion を使用）:**
- core + vcs は必須として提示する
- languages は Step 2 で検出した言語を元に、該当するものをデフォルト選択として提案する
- 新規追加されるパーミッションの一覧を提示する
- 既に設定済みのものは「設定済み」と表示する
- 選択肢:
  - 「すべて追加」（推奨）— 全言語のパーミッションを含む
  - 「検出言語のみ追加」— core + vcs + 検出された言語のみ
  - 「スキップ」— パーミッション設定を行わない

ユーザーが承認した場合のみ `.claude/settings.json` を更新する。
既存設定を保持したままマージし、既にリストにあるパーミッションは重複追加しない。

### Step 4: 結果の報告
セットアップした内容をサマリーで報告する:

```
セットアップ完了:
  ✅ .claude/dev-orchestrator.yml を作成
  ✅ tasks/ を作成
  ✅ .gitignore を更新
  ✅ パーミッション設定を追加（N項目）
  ⚠️ スキップ: <既に存在した項目>

次のステップ:
  1. .claude/dev-orchestrator.yml を確認・調整
  2. 「〜のタスクを作って」でタスクを作成
  3. 「task-1 を実行して」で対話実行、または /run でバッチ実行
  4. /check で結果確認
```

## 注意
- 既存ファイルは上書きしない（スキップして報告）
- `.claude/settings.json` のパーミッションはマージ方式（既存設定を壊さない）
- プロジェクト固有のハードコードを避ける（テンプレートベース）
- **CLAUDE.md を変更しない** — ワークフロー定義はプラグイン側に存在するため、プロジェクトの CLAUDE.md への追記は不要
