# run Skill - pendingタスクをバッチ実行

## 概要
`.claude/claude-dev-orchestrator/tasks/` 内の pending タスクを tmux セッションで並列バッチ実行する。
各タスクは独立した worktree と Claude Code プロセスで処理される。

## 入力
- `$ARGUMENTS`: タスクID（省略時は全 pending タスクを対象）
  - 例: `task-1`
  - 例: `task-1 task-3`（スペース区切りで複数指定）

## 手順

### Step 1: 設定読み込み
1. `.claude/dev-orchestrator.yml` を読み込む（なければデフォルト使用）
   - `model`: 使用モデル（デフォルト: `claude-sonnet-4-6`）
   - 生成物保存先: `.claude/claude-dev-orchestrator/`

### Step 2: 対象タスクの特定
1. `$ARGUMENTS` が指定されていればそれを対象とする
2. 未指定なら `.claude/claude-dev-orchestrator/tasks/task-*.md` から `status: pending` のものを番号順に収集する
3. 対象タスクが0件なら「pending タスクがありません」と報告して終了

### Step 3: 事前チェック
以下のコマンドが利用可能か確認する:
- `claude` — Claude Code CLI
- `gh` — GitHub CLI（`gh auth status` で認証済みか確認）
- `git`
- `tmux`

不足があればエラーを報告して終了する。

### Step 4: ユーザーに確認
対象タスクの一覧を表示し、実行確認を求める（AskUserQuestion を使用）:
- タスクID・タスク名の一覧
- 使用モデル
- 選択肢:
  - 「実行」（推奨）
  - 「キャンセル」

### Step 5: エージェント起動
ユーザーが承認した場合、各タスクについて以下を実行する:

1. 既に同名の tmux セッション（`agent-<task-id>`）が存在する場合はスキップ
2. `.claude/claude-dev-orchestrator/logs/` ディレクトリを作成
3. `.claude/claude-dev-orchestrator/artifacts/<task-id>/` ディレクトリを作成
4. タスクファイルの `status` を `in_progress` に更新（`sed` で `pending` → `in_progress`）
5. tmux セッションを作成し、以下のコマンドを実行:

```bash
tmux new-session -d -s "agent-<task-id>" \
  "cd <repo-dir> && \
   unset CLAUDECODE CLAUDE_CODE_ENTRYPOINT && \
   claude -p \"<プロンプト>\" \
     --worktree <task-id> \
     --dangerously-skip-permissions \
     --model <model> \
     --verbose \
     --output-format stream-json \
     2>&1 | tee \".claude/claude-dev-orchestrator/logs/<task-id>_<timestamp>.log\"; \
   echo 'Press Enter to close...'; read"
```

プロンプトは `/dev` コマンドと同等の内容を使用する。

各タスク起動後、5秒待ってから次のタスクを起動する。

### Step 6: 結果報告
```
エージェント起動完了:
  ✅ task-1 → tmux session: agent-task-1
  ✅ task-3 → tmux session: agent-task-3
  ⚠️ task-5 → スキップ（既存セッション）

確認コマンド:
  tmux ls                          # セッション一覧
  tmux attach -t agent-<task-id>   # 接続 (Ctrl+B, D で離脱)
  /check                           # 結果確認
```

## 注意
- `--dangerously-skip-permissions` でバッチ実行するため、事前にユーザー確認を必ず行う
- `.gitignore` に `.claude/claude-dev-orchestrator/` が含まれていなければ追加する
