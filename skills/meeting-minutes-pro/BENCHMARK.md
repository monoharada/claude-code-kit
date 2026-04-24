# BENCHMARK.md — 回帰テスト基準

このドキュメントは `meeting-minutes-pro` スキルの回帰テスト層。スキル変更時はこの 2 サンプルで再現性を確認する。

---

## サンプル固定（プロジェクト固有データ）

処理モデル差を計測するため、サンプル日付・モデル・VTT 原本パス・ゴールド側議事録の組み合わせは **対象プロジェクト側の外部データ** で固定する：

- `<project-root>/skills-data/meeting-minutes-pro/benchmark-samples.md`

このファイルには S1（下限ライン）と S2（目標ライン）の具体的な日付・モデル・ファイルパスが記載されている。サンプル追加（S3 以降）も同ファイルで管理する。

### 判定ラインの設計意図（汎用）

- **下限サンプル**：その時点での下位モデルでも到達すべき最低ライン（Recall 漏れ率 ≤ 15% 相当）
- **目標サンプル**：現行フルスペックモデルでの目標ライン（Recall 漏れ率 ≤ 10%、Apply 率 ≥ 80%）

> 処理モデル差を吸収するため、下限サンプルは目標サンプルより緩い判定。両サンプルとも同じスキルロジックで動くが、モデルの実力差が顕在化する箇所は下限サンプルの FAIL として扱わない。

---

## 観点（共通）

### 構造観点

- 出力ファイル構成が 7 ファイル形式（`00_議事録_全体.md` + `01〜NN_*.md` + `standup_extract.md` + `drafts/*`）に揃うこと
- `00_議事録_全体.md` が `参加者` → `アジェンダ` → `インサイト（または番号セクション）` → `決定事項` の順で並ぶこと
- `standup_extract.md` が `進捗` `計画` `組織` の 3 ブロックを持つこと

### 内容観点

- ゴールド正解リストに対する Recall 計測（漏れ項目一覧）
- Apply 率計測（ユーザー承認→実反映まで到達した候補率）
- `knowledge/` 差分候補が RULES.R1・R2・R7 を守っていること（個人名・さん付け）
- `status/` 差分候補が `status/components.md` `status/members.md` `status/releases.md` のいずれかにルーティングされていること

### 精度観点

- 固有名詞誤記がないか（`CHECKLIST.md §1` 対応）
- PII 赤旗がマスクされているか（`CHECKLIST.md §4` 対応）
- `source_refs` が空になっていないか

---

## 最小セクションの再現（ゴールド側構造）

### `00_議事録_全体.md` で再現が必要なセクション

- 会議タイトル（H1）
- `## 参加者（記録用・本文には記載しない）` または `## 概要`
- `## アジェンダ`
- `## インサイト` 配下に H3 トピック、または番号付 H2 セクション
- `## 決定事項`

### `standup_extract.md` で再現が必要なセクション

- 会議タイトル（H1）
- `## 進捗` または `## 進捗（デザイン）` / `## 進捗（リリース）`
- `## 計画`（メンバー単位 or コンポーネント単位）
- `## 組織` もしくは 知見・蓄積候補

### `drafts/` の存在

- `drafts/knowledge-YYYY-MM-DD.md`（knowledge 差分候補）
- `drafts/status-components-YYYY-MM-DD.md`（status/components.md 差分候補）
- `drafts/status-members-YYYY-MM-DD.md`（status/members.md 差分候補）
- `drafts/status-releases-YYYY-MM-DD.md`（status/releases.md 差分候補）※リリース言及時のみ

---

## 計測指標

### OK-01: Recall 漏れ率

```
Recall 漏れ率 = （ゴールド正解リストの項目数 - スキル候補出力にヒットした件数）/ ゴールド正解リストの項目数
```

- 分母：`gold_distillation_items.jsonl` のうち `observation` 以外の件数
- 分子：スキル候補出力が `evidence_vtt_line` もしくは `core_claim` で一致しなかった件数

### OK-02: Apply 率

```
Apply 率 = ユーザー承認→実反映に到達した候補数 / スキル候補出力の総数
```

- 分母：スキル候補出力の総数（`requires_review` 含む）
- 分子：ユーザーが承認し `drafts/` への反映（または `knowledge/` への反映）を Apply したもの

---

## 差分レポート（`regression/diff-report.md` の 6 欄）

| 欄 | 内容 |
|-----|------|
| 1 | ゴールド分母（observation 除く件数） |
| 2 | スキル候補出力件数 |
| 3 | 漏れ項目（gold IDs） |
| 4 | 漏れ率 Recall（= 漏れ / ゴールド分母） |
| 5 | Apply 件数（ユーザー承認→反映済み） |
| 6 | Apply 率（= Apply 件数 / スキル候補出力件数） |

---

## 回帰テスト手順

1. スキル変更後に本ドキュメントと `<project-root>/skills-data/meeting-minutes-pro/benchmark-samples.md` に従って各サンプルを再生成
2. 生成結果を `regression/YYYY-MM-DD/` に展開（日付はサンプル定義に従う）
3. `<project-root>/skills-data/meeting-minutes-pro/gold_distillation_items.jsonl` と照合し 6 欄を埋める
4. 判定ラインに達していなければ監査ループ 3 回改訂 → それでも失敗ならスコープ縮退 Rollback をユーザー提案

---

## 備考

- 下限サンプルと目標サンプルの役割分担は崩さない（モデル差計測の要）
- 将来のサンプル追加は `benchmark-samples.md` に別 ID（S3、S4...）で追記する
- VTT 原本が揮発先（Downloads 等）から消えたらバックアップ先に移動してパスを `benchmark-samples.md` で更新する
