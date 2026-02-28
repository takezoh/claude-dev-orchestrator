# CLAUDE.md

## このリポジトリは何か

Claude Code プラグイン「dev-orchestrator」のソースコード。
ユーザーのプロジェクトにインストールして使う。このリポジトリ自体を開発対象として変更する場合の指示が以下。

## 構成

```
agents/          SubAgent定義（frontmatter + markdown）
commands/        スラッシュコマンド（/setup, /run, /check）
skills/          Skill定義（dev, task, setup, run, check）
templates/       テンプレート（config.yml, permissions.json）
.claude-plugin/  プラグインメタデータ
```

## Skill一覧

| Skill | 概要 | 呼び出し方 |
|---|---|---|
| dev | Orchestrator型の全自動実行 | 「task-1 を実行して」 |
| task | タスク定義の作成 | 「認証機能のタスクを作って」 |
| setup | プロジェクト初期セットアップ | `/setup` |
| run | tmuxバッチ実行 | `/run [task-id...]` |
| check | 実行結果の確認 | `/check` |

## Agent定義の書き方

`agents/*.md` は以下の形式:

```markdown
---
name: agent-name
description: 1行の説明
tools:
  - Read
  - Bash(git:*)
---

# Agent Name

## 役割
## 入力
## 手順
## 出力
## 注意
```

- `tools:` は必要最小限にする
- 読み取り専用Agentには Write を与えない
- 出力は必ずファイル経由（`.claude/claude-dev-orchestrator/artifacts/<task-id>/` 配下）
- パーミッションを追加・変更した場合は `templates/permissions.json` も更新する

## タスクファイルの書き方

`.claude/claude-dev-orchestrator/tasks/task-N.md` は以下の形式:

```markdown
---
status: pending
---
# task-N: 短い名前（kebab-case）
タスクの概要を1-2文で説明。

## 要件
- 具体的な実装要件をリストで記載

## 受け入れ条件
- テストが通ること
```

- `status`: `pending` | `in_progress` | `done`
- ファイル名は `task-N.md`（番号のみ、タスク名は含めない）
- 見出しレベル: タイトルは `#`、セクションは `##`

## 守るべきルール

- ドキュメント・コメントは日本語
- プラグインとして配布するため、特定プロジェクトへのハードコード禁止
- `.claude/claude-dev-orchestrator/` は gitignore 対象（artifacts と logs を格納）
- `templates/permissions.json` はパーミッション定義の単一ソース — agents の tools 変更時は必ず同期する
