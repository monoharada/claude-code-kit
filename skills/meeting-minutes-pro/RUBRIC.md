# RUBRIC.md — 蒸留判断ルーブリック

このドキュメントは `meeting-minutes-pro` スキルの蒸留層。会議から抽出した `key_points` / `decisions` / `next_actions` を `knowledge/` や `status/` に反映する候補として採用するかの判定を行う。

---

## 蒸留 3 条件（AND 判定）

候補が `knowledge/` 反映候補 `T-new / T-update / P-new / P-update / decision / constraint` になるには、**3 条件すべて** を満たす必要がある。

### C1: 新規性（Novelty）

- 既存 `knowledge/` ドキュメント（`design_principles.md` `tradeoffs.md` `a11y_constraints.md` `implementation_notes.md` `thinking.md` `design_decisions.jsonl`）に同じ趣旨の記述が **まだない** こと
- 判定は `SEARCH_ALIASES.md` の同義語テーブルと `アンカー語 4〜7 語のうち ≥ 2 ヒット` で類似判定
- 類似ヒットがあれば `T-update`（既存更新）または `observation`（観察行き）にデマテ

### C2: 再利用性（Reusability）

- 単発判断でなく、将来の類似場面で参照価値がある抽象度を持つこと
- 判定基準：以下 3 つのいずれか 2 つ以上を満たすと再利用性あり
  - 一般化できる原則・判断基準・制約になっている
  - 複数コンポーネント／複数メンバーに適用しうる
  - 外部事業者・新規メンバーへの説明材料になる
- 再利用性が低ければ `observation` 行き（観察・タイムラインに残すが `knowledge/` には上げない）

### C3: 既存と矛盾しない（Consistency）

- 既存 `knowledge/` の記述と矛盾するなら、単純追加は不可
- 矛盾検出時は `conflict` ラベルを付与し、候補出力時に **両論併記＋ユーザー判断委譲**（`requires_review: true`）
- `tradeoffs.md` の同じトレードオフ軸と相反する場合は `P-update` 候補として提示

---

## observation（観察） vs T-xxx（知識化）の切り分け

- 3 条件全達成 → `T-new / T-update / P-new / P-update / decision / constraint` 候補化
- 2 条件以下の達成 → `observation` として `meetings/*/05_観察メモ.md` 等に残す
- **observation は分母から除外**（Recall 計測から外す）

---

## 候補出力フォーマット

蒸留候補は Top N（デフォルト N = 10）を以下のフィールドで出力：

```jsonl
{
  "candidate_id": "cand-001",
  "type": "T-new | T-update | P-new | P-update | decision | constraint | observation",
  "destination": "knowledge/design_principles.md | knowledge/tradeoffs.md | ... | observations",
  "core_claim": "1 文で書いた主張（50〜100 字）",
  "evidence_vtt_line": 1234,
  "evidence_timecode": "00:23:15",
  "priority": "high | medium | low",
  "confidence": 0.0-1.0,
  "requires_review": true | false,
  "rubric_flags": {
    "novelty_pass": true,
    "reusability_pass": true,
    "consistency_pass": true,
    "similar_existing": ["knowledge/tradeoffs.md#<section>"]
  },
  "related_T_ids": ["T-005"]
}
```

### priority の決め方

- `high`：組織の仕組み・採用・予算・リリース判断に関わる / 複数メンバーの動きを変える
- `medium`：コンポーネント単位の設計判断 / 複数コンポーネントに波及する原則
- `low`：単発トピック / 実装メモ

### confidence の決め方

- `≥ 0.8`：VTT 原文の複数箇所で同主旨が繰り返し示され、合意が明示
- `0.5 – 0.8`：発言されているが合意の明示なし、または VTT 原文の表現ゆれあり
- `< 0.5`：要確認（推測依存）。`requires_review: true` を強制付与

### requires_review の強制条件

以下のいずれかに該当する場合、候補に `requires_review: true` を強制付与：

- C3 で矛盾検出（`rubric_flags.consistency_pass: false`）
- `confidence < 0.5`
- 禁止語彙が含まれる（RULES.R11 違反）
- 固有名詞が VTT で確認取れない（RULES.R5）
- `priority: high`（人間最終判断）

---

## 手動オーバーライド

- ユーザーは候補提示時に ID を指定して `priority` / `type` / `destination` を上書き可能
- オーバーライド履歴は `.claude/plans/sessions/<session-id>/override-log.md` に記録（次回以降のルーブリック改善材料）

---

## アンカー語＋補助語の二段判定

1. **アンカー語**：候補の主張から中心概念を 1〜2 語抽出
2. **補助語**：同義語・類義語・関連用語を 4〜7 語展開（`SEARCH_ALIASES.md` 参照）
3. 既存 `knowledge/` に対して **アンカー語または補助語のうち ≥ 2 個** がヒットすれば `類似` 判定 → `T-update` または `observation` にルーティング
4. ヒット 0〜1 個なら `新規性あり` として C2 評価に進む

---

## 判定の実行順

スキル実行中、各候補に対して以下の順で適用：

1. SEARCH_ALIASES を用いてアンカー語＋補助語でインデックス検索
2. 類似ヒット件数を確認し C1（新規性）判定
3. 再利用性 C2 を満たすか確認
4. 既存との矛盾 C3 を確認
5. 3 条件全達成＝候補化、いずれか未達＝observation
6. priority／confidence／requires_review フラグを付与
7. Top N 候補として整形出力
