# check Skill - エージェント実行結果の確認

## 概要
バッチ実行中・完了後のエージェント状態を一覧表示する。
tmux セッション、Artifact、レビュー結果、Draft PR、worktree を確認する。

## 入力
- `$ARGUMENTS`: なし

## 手順

### Step 1: 設定読み込み
`.claude/dev-orchestrator.yml` を読み込む（なければデフォルト使用）:
- `artifacts_dir`: Artifact保存先（デフォルト: `.agent-artifacts`）

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
`<artifacts_dir>/*/` の各ディレクトリについて:
1. タスクIDを取得（ディレクトリ名）
2. `tasks/<task-id>.md` から `status` を取得
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

🌳 Worktree:
   (git worktree list の出力)

🧹 クリーンアップ:
   git worktree remove .claude/worktrees/<name>
   git worktree prune
   tmux kill-server
```

## 注意
- 読み取り専用の操作のみ行う（ステータス更新を除く）
- `gh` や `tmux` が未インストールの場合は該当セクションをスキップする
