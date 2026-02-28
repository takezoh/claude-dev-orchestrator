# check Skill - エージェント実行結果の確認

## 概要
バッチ実行中・完了後のエージェント状態を一覧表示する。
tmux セッション、Artifact、レビュー結果、Draft PR、worktree を確認する。

## 入力
- `$ARGUMENTS`: なし

## 手順

### Step 1: 設定読み込み
`.claude/dev-orchestrator.yml` を読み込む（なければデフォルト使用）:
- 生成物保存先: `.claude/claude-dev-orchestrator/`

### Step 2: 情報収集
以下の情報を並列で収集する:

#### 2-1. tmux セッション状態
```bash
tmux ls 2>/dev/null | grep "agent-"
```
- 実行中のセッションがあれば一覧表示
- なければ「全て完了」と表示

#### 2-2. Draft PR 一覧
```bash
gh pr list --draft --json number,title,url
```

#### 2-3. タスク別 Artifact 確認
`.claude/claude-dev-orchestrator/artifacts/*/` の各ディレクトリについて:
1. タスクIDを取得（ディレクトリ名）
2. `.claude/claude-dev-orchestrator/tasks/<task-id>.md` から `status` を取得
3. `*-review.md` ファイルから各ステップのレビュー判定（PASS/FAIL）を取得
4. `pr-url.txt` が存在すれば PR URL を表示
5. `error.md` が存在すれば警告を表示

#### 2-4. worktree 一覧
```bash
git worktree list
```

### Step 3: ステータス自動更新
PR が作成済み（`pr-url.txt` が存在）かつタスクが `in_progress` のままの場合:
- タスクファイルの `status` を `done` に更新する

### Step 4: 結果を整形して表示

```
エージェント実行結果
================================

📋 tmuxセッション:
   agent-task-1: (実行中)
   (または: 全て完了)

📝 Draft PR:
   #42 feat: add authentication
      https://github.com/user/repo/pull/42

📁 タスク結果:
   [task-1] (status: done)
      research: PASS
      design: PASS
      test: PASS
      implement: PASS
      PR: https://github.com/user/repo/pull/42
      ✅ ステータス → done

   [task-3] (status: in_progress)
      research: PASS
      design: FAIL
      PR: 未作成
```

### Step 5: 自動クリーンアップ
全ての `agent-*` tmux セッションが終了済み（Step 2-1 で検出なし）の場合、
残存する worktree と tmux セッションを自動でクリーンアップする。

**実行条件**: `agent-*` tmux セッションが**1つも実行中でない**こと。
1つでも実行中なら、このステップはスキップしてクリーンアップコマンドを案内するだけにする。

#### 5-1. worktree の削除
`git worktree list` で `.claude/worktrees/` 配下のエントリを検出した場合:
```bash
git worktree remove .claude/worktrees/<name>
```
各 worktree に対して実行する。削除に失敗した場合（未コミットの変更がある等）は `--force` を使わず警告を表示する。

#### 5-2. worktree prune
```bash
git worktree prune
```

#### 5-3. tmux セッションの終了
`agent-*` の tmux セッションが残っている場合（「Press Enter to close...」で待機中）:
```bash
tmux kill-session -t agent-<task-id>
```
各セッションに対して実行する。`agent-` プレフィックスのないセッションには触れない。

#### 5-4. クリーンアップ結果の表示
```
🧹 クリーンアップ:
   ✅ worktree task-1: 削除済み
   ✅ worktree task-3: 削除済み
   ✅ tmux agent-task-1: 終了
   ✅ tmux agent-task-3: 終了
   ✅ worktree prune: 完了
```

削除できなかった worktree がある場合:
```
🧹 クリーンアップ:
   ⚠️ worktree task-2: 未コミットの変更があるため削除できません
      手動で確認: cd .claude/worktrees/task-2
```

## 注意
- クリーンアップは全セッション終了後にのみ自動実行する
- 実行中のセッションがある場合はクリーンアップコマンドを案内にとどめる
- `--force` による強制削除は行わない（未コミットの変更を保護する）
- `agent-` プレフィックスのない tmux セッションには触れない
- `gh` や `tmux` が未インストールの場合は該当セクションをスキップする
