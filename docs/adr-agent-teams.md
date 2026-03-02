# ADR: パイプライン全体評価 + Agent Teams 採用検討

**日付**: 2026-03-02
**ステータス**: Agent Teams 不採用 / パイプライン改善提案あり

---

## 1. パイプライン全体評価

### 1.1 概要

```
[researcher] → reviewer    ← デフォルト無効
[designer]  → reviewer
[tester]    → reviewer
[implementer] → reviewer   ← test_issue 時は tester 差し戻し
[pr-creator]
```

最小構成: SubAgent 7回 / 最大構成: 9回 / リトライ1回: +2回

### 1.2 各コンポーネント評価

#### Researcher（デフォルト無効）— 存在意義が薄い
- デフォルト無効であること自体が「通常は不要」と認めている
- designer が自己調査機能を持つ（`designer.md:28-34`）ため、researcher は重複
- 有効にすると SubAgent +2回（researcher + review）の追加コスト
- 削除候補。残すとしてもデフォルト無効は妥当

#### Designer — パイプラインの最重要コンポーネント、品質のボトルネック
- design.md が全下流工程の唯一の仕様書。品質上限を決定する
- テスト戦略を含む設計は良い判断
- 関数シグネチャレベルの具体性を要求（`designer.md:103`）も良い
- **懸念**: reviewer（同じLLM）が設計ミスを見抜けない場合、全工程が無駄になる

#### Tester — TDD の理念は正しいが、独立性は見せかけ
- tester も implementer も同じ design.md を同じ LLM が解釈する
- 人間の TDD のような「仕様の独立した解釈」は実質的に発生しない
- テストは実行されない（構文確認のみ、`tester.md:55-56`）
- tester が作るテストは「design.md の言い換え」であり、独立した検証にはなっていない

#### Implementer — 制約設計は合理的、「テスト変更不可」は過剰
- 「テストファイルを変更しない」（`implementer.md:83`）はテスト信頼性を守るが、テストにバグがある場合の代償が大きい（test_issue ルーティングで最低2サイクル追加）

#### Reviewer — 一貫性チェックとしては機能、真の品質ゲートではない
- PASS/FAIL 基準が明確、軽微な問題は PASS + 改善提案（`reviewer.md:46`）は良い設計
- **根本的限界**: LLM が LLM をレビュー。同じバイアスを共有
- テスト実行結果（`reviewer.md:70`）は唯一の客観的検証
- **4回のレビューは過剰**: research review は design review に包含される。test review の価値はテスト実行で代替可能。本当に必要なのは design review と implement review の2回

#### PR Creator — 適切、変更不要

### 1.3 アーキテクチャの強み（維持すべき）
- ファイルベース artifact 体系（監査証跡・デバッグ・リカバリ）
- agent ごとの最小権限
- tmux + worktree タスク並列化
- ⚠️ 付き Draft PR のエラーハンドリング
- 設定ファイル駆動

### 1.4 アーキテクチャの弱み

| 弱点 | 重大度 | 説明 |
|------|--------|------|
| design.md ボトルネック | 高 | 全工程が依存。LLM レビューでは盲点を共有 |
| LLM が LLM をレビュー | 中 | 微妙な設計ミスを検出できない可能性 |
| SubAgent 回数過多 | 中 | 最小7回。小タスクに過剰 |
| tester/implementer 分離の費用対効果 | 疑問 | 同じ LLM + 同じ仕様で真の独立性なし |
| research review | 低 | design review に包含される |

---

## 2. Agent Teams 評価

### 2.1 Agent Teams が解決する問題
- フェーズ内のリアルタイム協調（reviewer が途中介入）
- test_issue 時の直接対話による高速解決

### 2.2 Agent Teams が解決しない問題
- design.md ボトルネック（LLM 同士の盲点は Teams でも同じ）
- SubAgent 回数（フルセッション×人数でむしろコスト増）
- tester/implementer 分離の根本問題（同じ仕様 × 同じ LLM の構造は変わらない）

### 2.3 Agent Teams 固有のコスト
- トークン3倍（teammate ごとにフルセッション）
- 実験的フラグ `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 必須
- パーミッション分離不可（全 teammate が lead 権限を継承）
- 非決定論的協調（タスクステータスの遅延あり）
- 監査証跡の分散（`~/.claude/teams/` と `$D/artifacts/` の二重管理）

### 2.4 判定: 不採用

Agent Teams はこのパイプラインの本質的問題を解決しない。表面的な効率化（レビュー往復削減）に対して、コスト増・リスクが見合わない。プラグイン配布の観点からも実験的機能への依存は避けるべき。

---

## 3. 改善提案（優先度順）

### 提案1: design.md 後の人間チェックポイント（効果: 大）
- `human_review.after_design: true` 設定で orchestrator を一時停止
- パイプラインの最大の弱点（design.md ボトルネック + LLM レビューの限界）を直接補う

### 提案2: tester/implementer 並列化（効果: 中）
- 両者の入力は同じ design.md、出力は異なるファイル → 並列可能
- tester review は省略可（テスト実行結果で事実検証される）
- SubAgent 並列で実現可能（Agent Teams 不要）

### 提案3: research review の廃止（効果: 小）
- design review が research の不足もカバー（`SKILL.md:58` で対応済み）

### 提案4: タスク複雑度別パイプライン（効果: 中）
- `complexity: simple` 時は designer → implementer（テストも作成）→ review → PR の短縮版

### 提案5: researcher の廃止（効果: 小）
- designer の自己調査機能と重複。コード簡素化

---

## 4. 不確実性の明記

- tester/implementer 並列化で design.md 曖昧さによるテスト不整合が増加するリスクは、実データなしでは定量化できない
- tester/implementer 統合 vs 分離 vs 並列化のどれが最善かは実運用データが必要
- Agent Teams のリアルタイム協調の実効性も実データがない
- 提案1（人間チェックポイント）の効果は高いと推定するが、ユーザーが自動化を重視する場合は方向性と矛盾する

---

## 5. Agent Teams 再評価のタイミング

1. 正式機能化 + teammate 単位のパーミッション設定が可能になった場合
2. 設定ファイル駆動のチーム定義がサポートされた場合
3. 並列化後の実データで design.md 曖昧性によるリトライ増が問題と判明した場合
