# claude-code-public-skills

Claude Code 用の汎用スキル／コマンド 2 点を含む配布パッケージ。私用ドメインに依存しないジェネリック版として整備しており、任意のプロジェクトに導入できる。

---

## 同梱物

### 1. `/goal-plan` — ゴール→計画オーケストレーション（コマンド）

粗い依頼から「ゴール定義 → 計画文書 → 監査 → 改訂」までを multi-agent で統合実行する。各工程で `/dig:dig`（深掘り質問）によりユーザーと曖昧点を解消してから最終プランを提示する。

- `commands/goal-plan.md` — Slash コマンド定義
- `docs/goal-plan-orchestrator.md` — 運用仕様書
- `agents/` — 必須エージェント 5 種
  - `goal-interrogator.md`
  - `explorer.md`
  - `planning-builder.md`
  - `planning-auditor.md`
  - `planning-reviser.md`

### 2. `meeting-minutes-pro` — VTT → 議事録＋差分候補（スキル）

VTT 形式の会議文字起こしから、組織文脈に沿った議事録・standup 抽出・knowledge 差分候補・status 差分候補を一括生成する。言葉遣いルール／蒸留ルーブリック／表記ゆれ検出／精度チェックの 4 レイヤを上乗せ。

- `skills/meeting-minutes-pro/SKILL.md` — スキル本体
- `skills/meeting-minutes-pro/{RULES,RUBRIC,CHECKLIST,SEARCH_ALIASES,BENCHMARK}.md` — 参照ドキュメント
- `skills/meeting-minutes-pro/data-templates/` — プロジェクト固有データのテンプレート群

---

## インストール

### `/goal-plan`

1. `commands/goal-plan.md` を `~/.claude/commands/` にコピー
2. `docs/goal-plan-orchestrator.md` を `~/.claude/docs/` にコピー
3. `agents/*.md` を `~/.claude/agents/` にコピー

### `meeting-minutes-pro`

1. `skills/meeting-minutes-pro/` ディレクトリをまるごと `~/.claude/skills/` にコピー
2. 対象プロジェクトのルートを `<project-root>` とする
3. `data-templates/*.example.*` を `<project-root>/skills-data/meeting-minutes-pro/` にコピーし、`.example` を外す
4. 各データファイルを自プロジェクト固有の値で埋める
5. `<project-root>/meetings/` ディレクトリを作成
6. VTT を渡してスキルを起動して動作確認

---

## 依存関係

### `/goal-plan` の依存

- **`/dig:dig`**: marketplace skill（`kuu-marketplace`）。別途インストールが必要。未導入だと dig ステップでエラーになる。
  - 参考: https://github.com/kuu-marketplace（各自要確認）

### `meeting-minutes-pro` の依存

- **`anthropic-skills:meeting-minutes-from-vtt`**: Anthropic 公式 skills（`anthropic-skills`）配下。汎用の VTT→議事録変換を委譲先として使用する。
- `<project-root>/skills-data/meeting-minutes-pro/` に 5 種のデータファイル（templates から生成）が必要。

---

## 注意事項

### 私用版との違い

本パッケージは **ジェネリック配布版** である。作者の手元にある私用版は以下が追加されている：

- 特定プロジェクト固有のメンバー表・事業者表・エイリアス辞書
- 組織内ワークフロー（knowledge/ ・ status/ の特定ディレクトリ構造）と密結合したロジック
- 作業ログ自動連携フック

配布版ではこれらを全て除去し、テンプレートとドキュメントで各利用者が自プロジェクト向けに組み立てられる形にしている。

### データテンプレートについて

`data-templates/` 配下のファイルはスキーマと使い方を示すダミーデータのみ含む。実運用では利用者側で自プロジェクトの現実データに書き換える必要がある。

---

## ライセンス

MIT License を推奨。利用者側で自由に改変可。

---

## サポート

不具合・改善提案は GitHub Issues にて受け付ける（リポジトリオーナー: `@monoharada`）。
