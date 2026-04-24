---
description: "Run multi-agent goal→plan orchestration: define goal/scope, generate planning docs, audit, and revise until implementation-ready"
category: workflow-orchestration
---

# Goal Plan Orchestrator

粗い依頼から goal/scope 確定 → planning docs 生成 → 監査 → 改訂までを multi-agent で統合実行する。
各工程で **dig（深掘り質問）** によりユーザーと曖昧点を解消してから最終プランをユーザーに提示する。

## 入力
ユーザーからの自然言語入力を受け取る。$ARGUMENTS をゴール候補として扱う。

## オーケストレーション手順

以下の手順を厳密に順番に実行すること。

### Step 0: セッション準備

1. $ARGUMENTS からゴール候補を抽出する
2. ゴールテキストから slug を自動生成する（英数字・ハイフン、30文字以内）
3. セッションディレクトリを決定する:
   - 優先: `<cwd>/.claude/plans/sessions/<YYYY-MM-DD--slug>/`
   - フォールバック: `~/.claude/plans/sessions/<YYYY-MM-DD--slug>/`
4. セッションディレクトリを作成する（mkdir -p）

### Step 1 (Phase A): Goal Cell — goal-interrogator

**goal-interrogator エージェント**を呼び出す:

- 入力: $ARGUMENTS（粗いゴール）
- 期待出力: Goal Scorecard, goal.md, scope.md

エージェント完了後:
- `goal.md` と `scope.md` をセッションディレクトリに保存する
- Goal Action を確認する:
  - **KEEP**: dig → Phase B へ進む
  - **REWRITE / ESCALATE / SPLIT**: 下記「質問ポリシー」に従い AskUserQuestion でユーザーに確認してから、goal-interrogator を再実行する

#### Step 1-dig: Goal の深掘り

**`/dig:dig` を実行する**（Skill ツールで `dig:dig` を呼び出す）:

- 対象: セッションディレクトリの `goal.md` と `scope.md`
- 目的: ゴール定義・スコープの曖昧点をユーザーと徹底的に解消する
- dig が検出した曖昧点についてユーザーに質問し、回答を `goal.md` / `scope.md` に反映する
- dig の結果 scope が大きく変わった場合は goal-interrogator を再実行する
- **dig が「曖昧点なし」と判断するまで繰り返す**

### Step 1.5 (Phase A-2): Exploration Cell — explorer

**explorer エージェント**を呼び出す:

- 入力: セッションディレクトリの `goal.md`, `scope.md`
- 事前読込: プロジェクトの `./CLAUDE.md`, `./CLAUDE.local.md`, `@import` ファイル → Inherited constraints を抽出する
- 並列投入: goal の `observed domain` に応じて **Explore subagent を 3-5 並列**で投入する（UI/Backend/Infra/Data/Policy 等）
  - 各 subagent は独立 context で事実調査（実装提案は含めない）
  - 公式推奨: 1 subagent あたり 5-6 個の調査項目
- 期待出力: `research.md`（Inherited constraints / Scope of reading / Current behavior / Relevant patterns / External references / Unknowns / Parallel exploration ledger）

エージェント完了後:
- `research.md` をセッションディレクトリに保存する
- Inherited constraints の件数と Unknowns 件数をサマリとして表示する

#### Step 1.5-dig: Research の深掘り

**`/dig:dig` を実行する**（Skill ツールで `dig:dig` を呼び出す）:

- 対象: セッションディレクトリの `research.md`
- 目的: 調査の抜け漏れ、競合する発見、不明瞭な Unknown の解消
- 特に以下の観点で深掘りする:
  - Inherited constraints に CLAUDE.md の重要項目が含まれているか
  - Current behavior に引用・file:line が付いているか
  - Unknowns 各件に検証方法があるか
- dig の回答を `research.md` に反映する
- **dig が「曖昧点なし」と判断するまで繰り返す**

### Step 2 (Phase B): Planning Cells — planning-builder

**planning-builder エージェント**を呼び出す:

- 入力: セッションディレクトリの `goal.md`, `scope.md`, `research.md`
- 期待出力: `plan.md`, `risk.md`, `contract.md`（research.md は Phase A-2 で生成済み）
- Interview Phase（§planning-builder の 2.5）:
  - research.md を読んだ直後、技術選定で曖昧点があれば **AskUserQuestion** で interview
  - 1回あたり 2-4問、各問 2-4選択肢、pros/cons 必須、推奨に「(Recommended)」
  - 回答は `plan.md` の `## Interview Record` に記録する
- Alternative Plans Mode（scope_risk_score >= 6 または Goal Action が ESCALATE/SPLIT）:
  - 2-3 つの plan 候補を並列 subagent で独立生成 → `plan-alt.md`
  - AskUserQuestion でユーザーに選択 → 採用案を `plan.md` に正式化
- Execution Hints:
  - Recommended model / effort / task budget を goal-interrogator 判定から自動算出
  - L4/L3（Strong L2 なし）→ opus-4-7 + xhigh + budget unset
  - L2 + Low risk → sonnet-4-6 + high
  - L2 + Medium/High risk → opus-4-7 + xhigh

エージェント完了後:
- 3文書をセッションディレクトリに保存する

#### Step 2-dig: Planning Docs の深掘り

**`/dig:dig` を実行する**（Skill ツールで `dig:dig` を呼び出す）:

- 対象: セッションディレクトリの `plan.md`, `risk.md`, `contract.md`
- 目的: 計画・リスク・契約の曖昧点・抜け漏れをユーザーと解消する
- 特に以下の観点で深掘りする:
  - plan.md: ステップの粒度・依存関係・技術選定の妥当性、Inherited constraints との整合、Execution Hints の妥当性
  - risk.md: リスクの網羅性・緩和策の実現可能性
  - contract.md: Verification の type/command/success_criteria が実行可能か
- dig の回答を関連文書に反映する
- **dig が「曖昧点なし」と判断するまで繰り返す**

### Step 3 (Phase C): Readiness Cell — 監査ループ（最大3イテレーション）

**planning-auditor エージェント**を呼び出す:

- 入力: セッションディレクトリの全文書
- 期待出力: audit.json

audit.json の verdict に応じて分岐する:

#### READY_FOR_IMPLEMENTATION
- DoD coverage が 100%（全 D-xx が PASS）であることを確認する
- readiness.md を確定する → Step 3-dig へ

#### NEEDS_REVISION
- **planning-reviser エージェント**を呼び出す:
  - 入力: audit.json + 既存 planning docs
  - BLOCKER/HIGH を全件対応
- 改訂後、planning-auditor を再実行する（最大3回）

#### NEEDS_USER_INPUT
- audit.json の suggested_questions を基に AskUserQuestion で質問する（下記「質問ポリシー」参照）
- 回答を関連文書に反映する
- planning-auditor を再実行する

#### NEEDS_GOAL
- AskUserQuestion で goal/scope に関する質問を発行する（下記「質問ポリシー」参照）
- 回答を基に goal-interrogator を再実行する
- goal.md/scope.md 更新後、Phase B から再開する

#### Step 3-dig: 監査通過後の最終深掘り

**`/dig:dig` を実行する**（Skill ツールで `dig:dig` を呼び出す）:

- 対象: セッションディレクトリの全文書（goal.md, scope.md, plan.md, risk.md, contract.md, audit.json）
- 目的: 監査を通過した計画全体に対する最終確認
- 特に以下を確認する:
  - 全体の整合性（goal↔scope↔plan↔contract のトレーサビリティ）
  - ユーザーの意図との一致
  - 実装フェーズに入る前の最終懸念点
- dig の回答を関連文書に反映する
- **dig が「曖昧点なし」と判断したら Step 4（確定 & サマリ）へ**

### Step 4: 確定 & サマリ

1. **readiness.md** を生成してセッションディレクトリに保存する:
   ```md
   # Readiness Report
   - Ready for implementation: YES/NO
   - DoD Coverage: done/total (%)
   - Open issues: <残課題>
   - Session: <セッションディレクトリパス>
   ```

2. **agent-ledger.md** を生成してセッションディレクトリに保存する:
   ```md
   # Agent Ledger
   | Phase | Agent | Status | Key Output |
   |-------|-------|--------|------------|
   | A | goal-interrogator | DONE | goal.md, scope.md |
   | A-dig | dig:dig | DONE | goal/scope 深掘り完了 |
   | A-2 | explorer (+ Explore subagents × N) | DONE | research.md |
   | A-2-dig | dig:dig | DONE | research 深掘り完了 |
   | B | planning-builder | DONE | plan.md, risk.md, contract.md |
   | B-dig | dig:dig | DONE | planning docs 深掘り完了 |
   | C | planning-auditor | DONE | audit.json |
   | C | planning-reviser | DONE/SKIPPED | (revision details) |
   | C-dig | dig:dig | DONE | 最終深掘り完了 |
   ```

3. 最終サマリをユーザーに返す:
   - `Ready for implementation: YES/NO`
   - `DoD coverage: done/total`
   - `Open issues`（あれば）
   - `Session path: <ディレクトリパス>`
   - `Next action`: YES の場合「`/start` で実行フェーズに移行可能」と案内する

## dig 実行ポリシー

各工程での dig（`/dig:dig`）は以下のルールに従う:

1. **実行タイミング**: 各 Phase のエージェント完了直後に必ず実行する
2. **対象文書**: その Phase で生成・更新された文書を対象とする
3. **完了条件**: dig が「曖昧点なし」と判断するまで繰り返す（ただし実用上 3ラウンドを上限とする）
4. **反映**: dig で得られた回答は即座に関連文書に反映する
5. **再実行トリガー**: dig の結果が scope やゴールに大きく影響する場合、該当 Phase のエージェントを再実行する
6. **スキップ不可**: dig は省略できない。各工程の品質ゲートとして機能する

## 質問ポリシー（dig統合）

ユーザーへの質問は必ず以下のルールに従う:

1. **ツール**: 必ず `AskUserQuestion` ツールを使用する（会話的質問は不可）
2. **構造**: 1回あたり 2-4問、各問 2-4選択肢
3. **選択肢**: 各選択肢の description に pros/cons を含める
4. **適用箇所**:
   - Goal Cell: REWRITE/ESCALATE/SPLIT 判定時
   - Exploration Cell: Unknowns の検証方法が不明瞭なとき
   - Planning Cells: Interview Phase（技術選定）、Alternative Plans Mode 時
   - dig: 各工程の深掘り質問時
   - Readiness Cell: NEEDS_USER_INPUT / NEEDS_GOAL 判定時
5. **回答後処理**:
   - Decisions テーブルを出力する
   - 関連文書のみ差分更新する
   - 監査ループに戻る

## 整合ルール

- ID は必ず次で統一: `P-xx`（plan）, `C-xx`（contract）, `R-xx`（risk）, `D-xx`（DoD）, `U-xx`（unknown）
- `risk.md` の HIGH は rollback を必須にする
- `contract.md` の各項目に **構造化 verification**（type / command / success_criteria）を必須にする
  - type: `test` | `lint` | `typecheck` | `screenshot_diff` | `manual_step` | `metric_check`
  - `manual_step` 以外は `command` 必須
- `contract.md` の各 `C-xx` は `D-xx` にトレース可能であること
- `plan.md` の `## Decisions` 冒頭に `Inherited constraints`（research.md から転記）を置く
- `plan.md` に `## Execution Hints`（Recommended model / effort / task budget / rationale）を置く
- `readiness.md` は `DoD Coverage`（PASS/FAIL/PENDING と done/total）を必須にする
- `plan.md` のステップは scope 外を含めない、Inherited constraints に違反しない

## 設計原則（参考観点）

本オーケストレータは「探索 → 接続 → 実装」の3層を直線でなく循環として扱う:

- **探索**（Phase A / A-2）: 原理・問い・抽象を守る。短期成果を求めすぎない
- **接続**（Phase B / C）: 抽象 ↔ 具体の往復可能な仕様に変換する。置き場所ではなく運動
- **実装**（/start 以降）: 具体・納期・収益の責任を引き受ける

接続層の成果物（`plan.md` / `audit.json` / `readiness.md`）は
「責任の所在と逸脱の痕跡」を見える化することが目的。

## ガードレール

- 実装コードを生成・編集しない
- 破壊的コマンドを実行しない
- `APPROVE PLAN` までは実装フェーズへ進まない

## 既存システムとの関係

```
/goal-plan (このコマンド: ゴール→計画の上流)
  ↓ plan.md を生成
  ↓ (dig で各工程の曖昧点を解消)
/start (既存: タスクオーケストレーション)
  ↓ タスク分解
work-orchestrator-ja → task-atomizer-ja → task-implementer
```

- `/goal-plan` = WHY/WHAT を固める（ゴール定義→計画文書→監査）
- `/start` = HOW を実行する（タスク分解→依存分析→実行）

参照: `~/.claude/docs/goal-plan-orchestrator.md`
