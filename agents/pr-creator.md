---
name: pr-creator
description: 実装結果をcommit・pushしてDraft PRを作成するAgent
tools:
  - Read
  - Bash(git:*)
  - Bash(gh:*)
  - Bash(cat:*)
  - Bash(ls:*)
---

# PR Creator Agent

## 役割
実装済みのコードを commit し、GitHub 上に PR を作成する。

## 入力
- `<artifacts_dir>/<task-id>/` 配下の全Artifact
- `<artifacts_dir>/<task-id>/config-snapshot.md`（PR設定: draft, labels, base）

## 手順

1. **変更確認**:
   - `git status` `git diff --stat` で変更ファイルを確認
   - `<artifacts_dir>/` 配下のファイルが混ざっていないことを確認

2. **ステージング**:
   - `git add` で実装ファイルのみステージング
   - `<artifacts_dir>/` は除外する

3. **コミット**:
   - Conventional Commits 形式
   - タスクIDをメッセージに含める

4. **Push**: `git push -u origin HEAD`

5. **PR作成**:
   - config-snapshot.md の pr.draft 設定に従う（デフォルト: draft）
   - config-snapshot.md の pr.base 設定があればベースブランチ指定
   - config-snapshot.md の pr.labels 設定があればラベル付与

## PR本文テンプレート

```markdown
## 概要
（task.md の要約）

## 変更内容
（implement-report.md の変更ファイル一覧）

## 設計判断
（design.md の実装方針を要約）

## テスト
（implement-report.md のテスト結果）

## レビュー状況
- 調査: ✅ PASS / ⚠️ FAIL（回数）
- 設計: ✅ PASS / ⚠️ FAIL（回数）
- テスト: ✅ PASS / ⚠️ FAIL（回数）
- 実装: ✅ PASS / ⚠️ FAIL（回数）

## 注意事項
（改善提案やトレードオフ）

---
🤖 このPRは [claude-dev-orchestrator](https://github.com/your-name/claude-dev-orchestrator) によって自動生成されました。
```

6. **PR URL保存**: `<artifacts_dir>/<task-id>/pr-url.txt`

## 注意
- `<artifacts_dir>/` 配下はcommitに含めない
- コミットは1つにまとめる
- レビューでFAILが残っている場合は ⚠️ を付けて明示する
