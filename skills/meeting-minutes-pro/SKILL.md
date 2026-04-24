---
name: meeting-minutes-pro
description: VTT 形式の会議文字起こしから、組織文脈に沿った議事録・トピック別詳細・standup 抽出・knowledge 差分候補・status 差分候補を一括生成する汎用スキル。VTT ファイルおよび MP4 パスが明示的に渡され、かつ「議事録を作成して」「会議のまとめを作って」「ミーティングの内容を整理して」などが要求されたときに使用する。利用対象プロジェクト（CLAUDE.md に `<target-project>` マーカーが記載されている、または `meetings/` ディレクトリが存在する）でのみ発火する。汎用の議事録作成は `anthropic-skills:meeting-minutes-from-vtt` に委譲し、本スキルは言葉遣いルール・蒸留ルーブリック・既存 knowledge／status との差分検出レイヤを上乗せする。
version: 0.1.0
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Skill
---

# meeting-minutes-pro — 議事録＋差分候補スキル

## このスキルの位置付け

汎用の議事録作成（`anthropic-skills:meeting-minutes-from-vtt`）に、運用現場で効く 4 レイヤを上乗せする：

1. **言葉遣いレイヤ**（`RULES.md`）：個人名除去・さん付け・「案」使い分け・自明省略
2. **蒸留レイヤ**（`RUBRIC.md`）：新規性 AND 再利用性 AND 矛盾なし の 3 条件で knowledge 反映候補を判定
3. **差分検出レイヤ**（`SEARCH_ALIASES.md`）：日本語の表記ゆれに強い既存 knowledge／status との類似検出
4. **精度チェックレイヤ**（`CHECKLIST.md`）：固有名詞確認・VTT 発言者ラベル崩壊検出・MP4 ピンポイント参照・PII 赤旗

出力は `meetings/YYYY-MM-DD_会議名/` 配下に 7 ファイル構成で生成し、`status/` および `knowledge/` への書き込みは候補提示＋ユーザー承認後の Apply に限定する（自動書込しない）。

---

## 発火条件

### トリガーフレーズ（いずれか）

- 「議事録を作成して」／「議事録を作って」
- 「会議のまとめを作って」／「ミーティングの内容を整理して」
- 「/meeting-minutes-pro」

### かつ、以下を同時に満たすこと

- **VTT ファイルが明示されている**（ユーザーからパスが提示、または `inbox/meetings` / `Downloads` 等に特定できる）
- **対象プロジェクトを cwd としている**（次節の cwd 判定で確認）

### 非トリガー条件（スキルを起動しない）

- VTT ファイルが提示されていない場合（「この内容を議事録にして」のような生テキストは対象外）
- 対象プロジェクト外のディレクトリで作業している場合
- 汎用的な議事録作成だけが求められている場合（`anthropic-skills:meeting-minutes-from-vtt` で十分）

---

## 冒頭：cwd 前提チェック（必須）

スキル実行の最初に必ず以下のいずれかを確認する：

1. `Glob` で `meetings/` ディレクトリの存在を確認、または
2. `Read` で `CLAUDE.md` を読み、あらかじめ合意したプロジェクトマーカー（例：`<target-project>`）が記載されているか確認

どちらも満たさない場合、ユーザーに `mkdir meetings` を実行してよいか確認し、承認されたら現 cwd に `meetings/` を作成して判定を通す。拒否された場合は **即座に中断してユーザーに以下を報告**：

```
cwd 判定で対象プロジェクトではないと判断されました。
誤発火防止のため本スキルは起動しません。
対象プロジェクトのルートで実行するか、明示的に cwd を指定してください。
```

cwd 判定が通った場合、その cwd を `<project-root>` として以降の外部データ参照に使う。

この判定は誤発火による他プロジェクト書き込み事故の前段ガードとして機能する。

---

## Progressive Disclosure：関連ドキュメント

### スキル本体（portable、`~/.claude/skills/meeting-minutes-pro/`）

実行前に以下を読み込む：

- **`RULES.md`**：言葉遣いルール（R1〜R11）。成果物出力直前に全ルール適用
- **`RUBRIC.md`**：蒸留 3 条件ルーブリック。候補判定時に参照
- **`SEARCH_ALIASES.md`**：同義語・表記ゆれテーブル（汎用）。差分検出時に参照
- **`CHECKLIST.md`**：精度チェック項目。実行中と完了直前に参照
- **`BENCHMARK.md`**：回帰テスト基準の構造定義。回帰テスト時のみ読む

### 外部データ（プロジェクト固有、`<project-root>/skills-data/meeting-minutes-pro/`）

skill 本体から以下のデータファイルを参照する。存在しない場合はフォールバック（R1〜R11 の抽象ルールのみで動作、精度は低下）：

- **`members.md`**：メンバー一覧（さん付け表記）。`RULES.md` R2 から参照
- **`vendor_names.md`**：事業者・外部企業名の誤記リスク表。`CHECKLIST.md` §1 および `RULES.md` R5 から参照
- **`aliases.md`**：組織・案件固有の表記ゆれ。`SEARCH_ALIASES.md` の拡張として参照
- **`benchmark-samples.md`**：回帰テスト S1／S2 の具体パス。`BENCHMARK.md` から参照
- **`gold_distillation_items.jsonl`**：ゴールド正解リスト。回帰テスト時のみ読む

テンプレートは `data-templates/` 配下に同梱している。導入時は `data-templates/*.example.md` を `<project-root>/skills-data/meeting-minutes-pro/` にコピーして中身を埋める。

`<project-root>` は前節の cwd 判定で特定したルート。

---

## Workflow（12 ステップ）

### Step 1: cwd 判定

上記「冒頭：cwd 前提チェック」を実施。失敗時は中断。

### Step 2: 入力受理

以下をユーザーまたは文脈から取得：

- `source_vtt_path`（絶対パス、必須）
- `source_mp4_path`（絶対パス、任意）
- `meeting_title`（未提示ならファイル名から推定）
- `meeting_date`（未提示ならファイル名・VTT のタイムスタンプから抽出）
- `output_dir`（既定：`meetings/YYYY-MM-DD_会議名/`）
- `requester_note`（任意のヒント）

欠落があれば補完を試み、どうしても不明なら 1 度だけユーザーに確認する。

### Step 3: 委譲呼出

`Skill` ツール経由で `anthropic-skills:meeting-minutes-from-vtt` を呼び出す。`args` に VTT パス（必須）と MP4 パス（ある場合）、目的（構造化された中間表現が欲しい旨）を含める。

委譲スキルが返す出力を受け取り、中間表現を抽出・構造化する。

### Step 4: スキーマ自己診断

最低限スキーマ 4 要素を検証：

- ①トピック一覧
- ②決定事項
- ③ネクストアクション
- ④出典

いずれかの必須項目（①②④）が欠落していれば **フェイルファスト**。ユーザーに以下を提示して中断：

```
委譲スキルの出力が最低限スキーマを満たしませんでした。
欠落フィールド: [missing_fields]
Rollback の検討を推奨します：
  - 手動オーケストレーション維持に切り替え
  - 委譲先スキル仕様の再確認
```

③（ネクストアクション）のみ欠落は警告出力のみ（決定のみで NA なしは仕様上あり得るため）。

### Step 5: 言葉遣いレイヤ適用（`RULES.md`）

中間表現の全テキストフィールドに対して、`RULES.md` の R1〜R11 を適用順で適用する：

1. R1: 個人名除去（本文・見出し）
2. R2: 参加者セクションとメンバー参照はさん付け
3. R3: 「案」の使い分け確認
4. R4: 自明情報の省略
5. R5: 固有名詞の要確認フラグ付与
6. R6: 断定強度調整
7. R7: knowledge 候補からの個人名除去
8. R8: 事業者メール特化（該当時のみ）
9. R9／R10: 整形
10. R11: 禁止語彙検出

### Step 6: 既存との差分検出（`SEARCH_ALIASES.md`）

`Grep` で以下を走査（存在しない場合は該当ステップをスキップ）：

- `knowledge/design_principles.md` `knowledge/tradeoffs.md` `knowledge/a11y_constraints.md` `knowledge/implementation_notes.md` `knowledge/thinking.md` `knowledge/design_decisions.jsonl`
- `status/components.md` `status/members.md` `status/releases.md`

各候補の `core_claim` からアンカー語 1〜2 語＋補助語 4〜7 語を `SEARCH_ALIASES.md` で展開し、**アンカー語 OR 補助語のうち ≥ 2 ヒット** で類似判定。ヒット件数を候補の `rubric_flags.similar_existing` に記録。

### Step 7: 蒸留判断（`RUBRIC.md` + `<project-root>/skills-data/meeting-minutes-pro/gold_distillation_items.jsonl` 参照）

`RUBRIC.md` の 3 条件で各候補を評価：

- C1: 新規性（Step 6 のヒット件数から判定）
- C2: 再利用性（3 サブ基準で 2/3 以上満たすか）
- C3: 既存と矛盾しないか

3 条件全達成 → `T-new / T-update / P-new / P-update / decision / constraint`
いずれか未達 → `observation`

### Step 8: 候補提示（Top N ＋ フラグ）

Top N（既定 N=10）を以下の形式で出力：

- `candidate_id`
- `type`
- `destination`
- `core_claim`
- `evidence_vtt_line` / `evidence_timecode`
- `priority`（high/medium/low）
- `confidence`（0.0〜1.0）
- `requires_review`（強制条件は `RUBRIC.md` §requires_review）
- `rubric_flags`
- `related_T_ids`

ユーザーに候補を提示し、以下を促す：

- 承認 / 却下 / 編集 / オーバーライド（`priority` / `type` / `destination` を手動変更）

### Step 9: ユーザー承認待ち

ユーザーの応答を待つ。承認された候補のみ Step 10 に進む。却下・保留された候補は `meetings/YYYY-MM-DD_会議名/drafts/observations.md` に記録。

### Step 10: Apply（候補ファイルに出力）

承認された候補を以下のファイルに書き出す（実 `knowledge/` や `status/` には **書き込まない**）：

- `meetings/YYYY-MM-DD_会議名/drafts/knowledge-YYYY-MM-DD.md`
- `meetings/YYYY-MM-DD_会議名/drafts/status-components-YYYY-MM-DD.md`
- `meetings/YYYY-MM-DD_会議名/drafts/status-members-YYYY-MM-DD.md`
- `meetings/YYYY-MM-DD_会議名/drafts/status-releases-YYYY-MM-DD.md`（リリース言及時のみ）
- `meetings/YYYY-MM-DD_会議名/drafts/observations.md`（observation と却下候補）

### Step 11: 議事録 7 ファイルの生成・書き出し

中間表現と RULES 適用結果から以下を生成：

- `meetings/YYYY-MM-DD_会議名/00_議事録_全体.md`
  - 会議タイトル（H1）→ 参加者（記録用・本文には記載しない）→ アジェンダ → インサイト（トピック H3 展開）→ 決定事項
- `meetings/YYYY-MM-DD_会議名/01_NN_トピック名.md` （トピック数分）
- `meetings/YYYY-MM-DD_会議名/standup_extract.md`
  - 会議タイトル（H1）→ 進捗（リリース／デザイン）→ 計画（メンバー単位）→ 組織
- `meetings/YYYY-MM-DD_会議名/drafts/*`（Step 10 の候補出力）

### Step 12: 完了報告

ユーザーに以下を含む完了報告を出力：

- 生成ファイル一覧（7 ファイル構成）
- 候補件数（採用・却下・observation）
- Recall／Apply 率の目視確認（ベンチマーク日付の場合）

本スキル内部で外部ログコマンドは実行しない（`allowed-tools` から Bash を除外）。

---

## 例外時の Rollback（HIGH リスク）

| リスク | トリガー | スキル動作 |
|--------|---------|-----------|
| R-01 委譲仕様変更 | 委譲出力が主要フィールド 50% 以上乖離 or 2 回連続例外 | Step 4 でフェイルファスト、Rollback 提案 |
| R-05 回帰テスト失敗 | BENCHMARK 2 サンプル再現不可 | 監査ループ 3 回後、スコープを Lv3→Lv1 に縮退提案 |
| R-12 誤発火 | cwd 判定失敗 | Step 1 で即中断 |
| R-13 委譲サイレント破壊 | スキーマ検証で欠落 | Step 4 でフェイルファスト |

---

## このスキルを変更したとき

`BENCHMARK.md` の判定基準と `<project-root>/skills-data/meeting-minutes-pro/benchmark-samples.md` で定義された S1／S2 サンプルで回帰テストを実施し、`<project-root>/skills-data/meeting-minutes-pro/gold_distillation_items.jsonl` に対する Recall 漏れ率 ≤10%（S2）／≤15%（S1）と Apply 率 ≥80%（S2）を満たすことを確認する。`regression/diff-report.md` の 6 欄が PASS になること。

未達の場合は監査ループ 3 回改訂、それでも失敗ならスコープ縮退 Rollback をユーザーに提案する。

---

## 非目標（スコープ外）

本スキルは以下を扱わない：

- VTT 自体の改善（話者分離、誤字修正）
- MP4 の OCR・自動画面抽出
- 外部 KMS（Notion 等）への自動書き込み（候補提示のみ、貼り付けは人手）
- チケットシステム（Linear 等）への自動起票

---

## 導入手順（初回セットアップ）

1. このスキルを `~/.claude/skills/meeting-minutes-pro/` に配置する
2. 対象プロジェクトのルートを `<project-root>` とする
3. `data-templates/*.example.md` を `<project-root>/skills-data/meeting-minutes-pro/` にコピーし、`.example` 拡張子を外す
4. 各データファイル（members.md / vendor_names.md / aliases.md / benchmark-samples.md / gold_distillation_items.jsonl）を自プロジェクト固有の値で埋める
5. `<project-root>/meetings/` ディレクトリを作成
6. VTT を渡してスキルを起動して動作確認
