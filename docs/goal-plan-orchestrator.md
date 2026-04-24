# Goal Plan Orchestrator 運用仕様

## 設計原則（参考観点）

本オーケストレータは「探索 → 接続 → 実装」の3層を直線でなく循環として扱う。Phase 構造は維持しつつ、以下を設計判断の羅針盤とする:

- **探索**（Phase A / A-2）: 原理・問い・抽象を守る。短期成果を求めすぎない
  - goal-interrogator: 問い・KR・Unknown の精度を担保
  - explorer: 事実ベースの research.md を並列 subagent で生成
- **接続**（Phase B / C）: 抽象 ↔ 具体の往復可能な仕様に変換する。置き場所ではなく運動
  - planning-builder: Interview を介して技術選定の意味変換
  - planning-auditor: 逸脱の痕跡を audit.json に見える化
- **実装**（/start 以降）: 具体・納期・収益の責任を引き受ける

接続層の成果物（`plan.md` / `audit.json` / `readiness.md`）は「責任の所在と逸脱の痕跡」を見える化することが目的。

## 目的
1回のユーザー依頼から、`goal/scope` の確定、research 収集、planning docs 生成、監査、改訂までを統合し、実装可能な計画状態まで到達させる。

## 入力
- 必須: `goal`
- 任意: `scope`, `constraints`, `dod`, `session_slug`
- `dod` は Definition of Done（検証可能な完了条件）として扱う

## エントリポイント
- コマンド: `/goal-plan <ゴール>`
- 例: `/goal-plan フォーム完了率を改善したい`

## 生成物
- `goal.md`
- `scope.md`
- `research.md`
- `plan.md`
- `risk.md`
- `contract.md`
- `audit.json`
- `readiness.md`
- `agent-ledger.md`

## 保存先ルール
1. `<cwd>/.claude/plans/sessions/<YYYY-MM-DD--slug>/` が存在する場合は優先
2. なければ `~/.claude/plans/sessions/<YYYY-MM-DD--slug>/`

`always-persist` を既定とし、書き込み不可時のみ chat 出力へフォールバックする。

## 状態遷移
1. `COLLECT_INPUT` — 入力正規化・セッションディレクトリ作成
2. `GOAL_CELL` — goal-interrogator でゴール定義
3. `EXPLORATION_CELL` — explorer で research.md 生成（並列 subagent 投入）
4. `PLANNING_CELLS` — planning-builder で3文書生成（Interview Phase 統合）
5. `AUDIT_LOOP` — planning-auditor + planning-reviser の反復（最大3回）
6. `READY` または `WAIT_USER_INPUT`

## Multi-agent 編成

### Phase A: Goal Cell
- **goal-interrogator** エージェントを呼び出す
- 成果物: `goal.md`, `scope.md`（D-xx 含む）

### Phase A-2: Exploration Cell（新設）
- **explorer** エージェントを呼び出す
- 並列投入: **Explore subagent × 3-5**（domain 別、公式推奨数に準拠）
- 事前読込: `./CLAUDE.md` / `./CLAUDE.local.md` → Inherited constraints 抽出
- 成果物: `research.md`（Inherited constraints / Current behavior / Relevant patterns / External references / Unknowns / Parallel exploration ledger）

### Phase B: Planning Cells
- **planning-builder** エージェントを呼び出す
- 入力: `goal.md`, `scope.md`, `research.md`
- **Interview Phase**: 技術選定で曖昧点があれば AskUserQuestion で interview → plan.md の Interview Record に記録
- **Alternative Plans Mode**（scope_risk_score >= 6 or Goal Action in {ESCALATE, SPLIT}）: 2-3 候補を並列生成 → ユーザー選択
- 成果物: `plan.md`（Inherited constraints / Interview Record / Execution Hints 含む）, `risk.md`, `contract.md`（構造化 Verification）

### Phase C: Readiness Cell（反復、最大3回）
- **planning-auditor** → `audit.json`
- Verdict 分岐:
  - `NEEDS_REVISION` → **planning-reviser** → 再監査
  - `NEEDS_GOAL` / `NEEDS_USER_INPUT` → 質問ポリシーで AskUserQuestion
  - `READY_FOR_IMPLEMENTATION` → DoD coverage 100% 確認 → `readiness.md` 確定

## DoD 反映ルール（必須）

- `goal.md`: `Definition of Done` セクションを必須化し、`D-xx` を列挙する
- `contract.md`: 各 `C-xx` を最低1つの `D-xx` にトレースし、verification を付与する
- `readiness.md`: `DoD Coverage` セクション（`D-xx` ごとの PASS/FAIL/PENDING と `done/total`）を必須化する
- `READY_FOR_IMPLEMENTATION` 判定は、DoD coverage 100%（全 `D-xx` が PASS）を満たすこと

## 質問ポリシー（dig統合）

- 必ず `AskUserQuestion` ツールを使用する（会話的質問は不可）
- 1回あたり 2-4問、各問 2-4選択肢
- 各選択肢の description に pros/cons を含める
- 先にエージェント内で補完可能性を検討する
- 補完不能な Unknown のみユーザーへ確認する
- 回答取得後は関連文書のみ差分更新し、監査に戻る

## 整合チェック
- 参照ID整合: `P-xx`, `C-xx`, `R-xx`, `D-xx`, `U-xx`
- `risk.md` HIGH には rollback 必須
- `contract.md` 各項目に **構造化 verification**（type / command / success_criteria）必須
  - type enum: `test` | `lint` | `typecheck` | `screenshot_diff` | `manual_step` | `metric_check`
  - `manual_step` 以外は `command` 必須
- `contract.md` 各 `C-xx` は `D-xx` へトレース必須
- `plan.md` の `## Decisions` 冒頭に `Inherited constraints`（research.md から転記）必須
- `plan.md` に `## Execution Hints`（model / effort / task budget / rationale）必須
- `research.md` に `## Inherited constraints` と `## Parallel exploration ledger` 必須
- `readiness.md` は `DoD Coverage` 必須
- `plan.md` は scope 外タスクを含めない、Inherited constraints に違反しない

## 監査分岐
- `NEEDS_GOAL` → goal/scope 再定義へ戻る（goal-interrogator 再実行）
- `NEEDS_USER_INPUT` → AskUserQuestion で Unknown 解消
- `NEEDS_REVISION` → planning-reviser → 再監査（最大3回反復）
- `READY_FOR_IMPLEMENTATION` → DoD coverage 100% 確認 → `readiness.md` 確定

## 既存システムとの関係

```
/goal-plan (ゴール→計画の上流: WHY/WHAT)
  ↓ plan.md を生成
/start (タスクオーケストレーション: HOW)
  ↓ タスク分解
work-orchestrator-ja → task-atomizer-ja → task-implementer
```

- `/goal-plan` の出力 `plan.md` が `/start` の入力になる
- 自動接続はしない（`/goal-plan` 完了時に「`/start` で実行可能」と案内）

## ACE ログ運用（拡張）

`goal` がログ改善・運用改善を含む場合、次を planning docs に含める:

- 3層ログ定義:
  - `A` (Atomic): イベント単位（開始、終了、BLOCKER、検証）
  - `C` (Context): セッション単位（目的、制約、結果、変更ファイル）
  - `E` (Evolution): 改善ループ単位（仮説、delta、学び、次アクション）
- delta 更新の必須要素: `before` / `after` / `impact`
- Reflector シグナル: `helpful` / `harmful` / `neutral`
- Curator サイクル: 週次 `grow-and-refine`

## 最終出力契約
- `Ready for implementation: YES/NO`
- `DoD coverage`
- `Open issues`
- `Questions asked (if any)`
- `Next action`

## ガードレール
- 実装コードを生成・編集しない
- 破壊的コマンドを実行しない
- `APPROVE PLAN` までは実装フェーズへ進まない
