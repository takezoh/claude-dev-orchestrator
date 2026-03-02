# claude-dev-orchestrator

Claude Code 用の Orchestrator 型開発自動化プラグイン。

タスクを定義して寝れば、朝には Draft PR が出来ている。

## 仕組み

```
「認証機能のタスクを作って」 →  .claude/claude-dev-orchestrator/tasks/task-1.md (status: pending)
                                    ↓
「task-1 を実行して」 または  /run (tmux並列)
                              ↓
┌─────────────────────────────────────┐
│  Orchestrator Skill (per task)      │
│                                     │
│  researcher  ←→ reviewer (loop)     │
│       ↓ (file)                      │
│  designer    ←→ reviewer (loop)     │
│       ↓ (file)                      │
│  tester      ←→ reviewer (loop)     │
│       ↓ (file)                      │
│  implementer ←→ reviewer (loop)     │
│       ↓ (file)                      │
│  pr-creator  → Draft PR             │
└─────────────────────────────────────┘
                              ↓
/check  →  結果確認
```

各ステップは SubAgent として独立コンテキストで実行。
Artifact はファイル経由で受け渡し、Orchestrator のコンテキストを軽く保つ。
全ステップに Reviewer Agent がペアで存在し、PASS するまで修正ループを回す。

## 前提条件

- [Claude Code CLI](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- [GitHub CLI (gh)](https://cli.github.com/)
- Git
- tmux（`/run` でのバッチ実行時のみ）

```bash
npm install -g @anthropic-ai/claude-code
export ANTHROPIC_API_KEY="sk-ant-..."
gh auth login
```

## インストール

```bash
claude plugin marketplace add /path/to/claude-dev-orchestrator
claude plugin install dev-orchestrator@claude-dev-orchestrator
```

## セットアップ

```bash
cd /path/to/your-repo
claude
> /setup
```

対話的にプロジェクト情報をヒアリングし、以下を構成する:

- `.claude/dev-orchestrator.yml` — テスト・リントコマンド、PR設定等
- `.claude/claude-dev-orchestrator/tasks/` — タスク定義ディレクトリ
- `.gitignore` — `.claude/claude-dev-orchestrator/` を追加
- `.claude/settings.json` — Agent/Skill が使うパーミッション設定

## 使い方

### 1. タスクを作成

```
> 認証機能にRefresh Token対応を追加するタスクを作って
```

コードを調査し、要件・受け入れ条件を含むタスクファイルを生成する。

### 2. 実行

**対話モード**（1タスクずつ、途中経過を確認しながら）:

```
> task-1 を実行して
```

**バッチ実行**（pending タスクをまとめて並列実行）:

```
> /run
```

### 3. 結果を確認

```
> /check
```

tmux セッション状態、レビュー結果、Draft PR URL を一覧表示する。

### コマンド

| コマンド | 説明 |
|---|---|
| `/setup` | プロジェクトへの初期セットアップ |
| `/run [task-id...]` | tmux セッションでバッチ実行 |
| `/check` | エージェント実行結果の確認 |

それ以外の操作（タスク作成、対話実行）は自然言語で指示する。

## カスタマイズ

### 設定ファイル

`.claude/dev-orchestrator.yml`:

```yaml
test:
  commands:
    - name: "TypeScript"
      run: "npm test"

lint:
  commands:
    - "npm run lint"

review:
  max_retries:
    research: 2
    design: 2
    test: 2
    implement: 3

pr:
  draft: true

model: "claude-sonnet-4-6"
```

### Agent の追加・変更

`agents/` に新しい `.md` ファイルを追加し、
`skills/dev/SKILL.md` のフローに組み込む。

### Skill のフロー変更

`skills/dev/SKILL.md` を直接編集。
例: 設計ステップを省略、テストだけ別ステップにする等。

## アンインストール

```bash
claude plugin remove dev-orchestrator@claude-dev-orchestrator
```

プロジェクト側のファイル（`.claude/dev-orchestrator.yml`、`.claude/claude-dev-orchestrator/` 等）はユーザーデータのため自動削除されない。必要に応じて手動で削除する。

## 設計思想

- **コンテキスト分離**: SubAgent ごとにコンテキストがリセットされ、長いプロンプトによる指示抜けを防ぐ
- **ファイル経由の受け渡し**: Agent 間のデータはファイルで渡し、Orchestrator のコンテキストを軽く保つ
- **設定とロジックの分離**: プラグイン本体（Skills/Agents）はプロジェクト非依存、設定ファイルでカスタマイズ

## License

MIT
