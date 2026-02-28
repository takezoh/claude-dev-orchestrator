# dev Skill - Orchestrator型開発自動化

## 変数定義
- `$D` = `.claude/claude-dev-orchestrator` — 生成物のベースディレクトリ

## 概要
タスクIDを受け取り、情報収集→設計→テスト作成→実装→PR作成を全自動で実行するOrchestrator。
各ステップはSubAgentで独立実行し、Artifactはファイル経由で受け渡す。

## 設定の読み込み
最初に `.claude/dev-orchestrator.yml` を読み込み、以下を把握する:
- テストコマンド、リントコマンド
- レビューループの上限回数
- PR設定（draft、ラベル、ベースブランチ）
設定ファイルが存在しない場合は以下のデフォルトを使用する:
- テスト: 検出不可（スキップ）
- レビュー上限: research=2, design=2, test=2, implement=3
- PR: draft=true

生成物の保存先は `$D/` 配下に固定:
- Artifact: `$D/artifacts/<task-id>/`
- ログ: `$D/logs/`

## タスク定義の読み込み
タスクIDをもとに `$D/tasks/<task-id>.md` を読み込む。

## 実行フロー

### Step 0: 準備
1. `.claude/dev-orchestrator.yml` を読み込む（なければデフォルト使用）
2. `$D/tasks/<task-id>.md` を読み込む
3. `$D/artifacts/<task-id>/` ディレクトリを作成する
4. タスク内容を `$D/artifacts/<task-id>/task.md` に保存する
5. 設定内容を `$D/artifacts/<task-id>/config-snapshot.md` に保存する（デバッグ用）

### Step 1: 情報収集 (Research)
1. `researcher` AgentをSubAgentとして起動する
   - 入力: `$D/artifacts/<task-id>/task.md`
   - 出力: `$D/artifacts/<task-id>/research.md`
2. `reviewer` AgentにSubAgentとしてレビューを依頼する
   - 入力: `$D/artifacts/<task-id>/research.md`
   - レビュー観点: 「調査は十分か、見落としている既存コードや依存関係はないか」
   - 出力: `$D/artifacts/<task-id>/research-review.md`
3. FAILなら修正を依頼する（設定の max_retries.research 回まで）

### Step 2: 設計 (Design)
1. `designer` AgentをSubAgentとして起動する
   - 入力: task.md + research.md
   - 出力: `$D/artifacts/<task-id>/design.md`
2. `reviewer` AgentにSubAgentとしてレビューを依頼する
   - レビュー観点: 「設計はタスク要件を満たしているか、既存アーキテクチャと整合性があるか」
   - 出力: `$D/artifacts/<task-id>/design-review.md`
3. FAILなら修正を依頼する（設定の max_retries.design 回まで）

### Step 3: テスト作成 (Test)
1. `tester` AgentをSubAgentとして起動する
   - 入力: task.md + design.md + config-snapshot.md
   - 実行: 設計書のテスト戦略に基づきテストコードを作成
   - 出力: `$D/artifacts/<task-id>/test-report.md`
2. `reviewer` AgentにSubAgentとしてテストレビューを依頼する
   - レビュー観点: 「テストケースは要件を正しく検証しているか、網羅性は十分か」
   - 出力: `$D/artifacts/<task-id>/test-review.md`
3. FAILなら修正を依頼する（設定の max_retries.test 回まで）

### Step 4: 実装 (Implement)
1. `implementer` AgentをSubAgentとして起動する
   - 入力: task.md + design.md + config-snapshot.md + test-report.md（テストコードは既にワークツリーに存在）
   - 実行: コード実装 + テスト実行（設定のtest.commandsを使用）
   - 出力: `$D/artifacts/<task-id>/implement-report.md`
2. `reviewer` AgentにSubAgentとしてレビューを依頼する
   - レビュー観点: 「設計通りか、テスト全パスか、規約違反ないか」
   - 出力: `$D/artifacts/<task-id>/implement-review.md`
3. FAILの場合:
   - reviewerが `test_issue: true` と判定 → tester (Step 3) に差し戻し
   - それ以外 → implementer をリトライ（設定の max_retries.implement 回まで）

### Step 5: PR作成 (Create PR)
1. `pr-creator` AgentをSubAgentとして起動する
   - 入力: `$D/artifacts/<task-id>/` 配下の全Artifact + 設定のpr.*
   - 実行: git commit + git push + gh pr create
   - 出力: `$D/artifacts/<task-id>/pr-url.txt`

## エラーハンドリング
- レビュー上限到達でもFAILの場合: 現状態でcommitし、PR本文に「⚠️ レビュー未通過: <ステップ名>」を明記してDraft PR作成
- SubAgent異常終了: `$D/artifacts/<task-id>/error.md` にエラー記録、可能な範囲でPR作成
- 設定ファイル不在: デフォルト値で続行（エラーにしない）

## Artifact
- `$D/` はgitignore対象
- PR作成後も残す（デバッグ用）
- worktree削除時に消える
