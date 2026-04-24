# benchmark-samples.md — 回帰テスト固定サンプル（テンプレート）

本ファイルはスキル導入時に `<project-root>/skills-data/meeting-minutes-pro/benchmark-samples.md` にコピーして使用する。
`BENCHMARK.md` から参照され、S1（下限ライン）／S2（目標ライン）の 2 サンプルで再現性を測る。

---

## S1（下限サンプル）

- 日付: `YYYY-MM-DD`
- 使用モデル: `<lightweight-model-id>`（例: haiku 系）
- VTT 原本パス: `<project-root>/inbox/meetings/YYYY-MM-DD_sample.vtt`
- MP4 原本パス: （任意）
- ゴールド側議事録パス: `<project-root>/meetings/YYYY-MM-DD_sample/`
- 判定ライン: Recall 漏れ率 ≤ 15%

## S2（目標サンプル）

- 日付: `YYYY-MM-DD`
- 使用モデル: `<full-spec-model-id>`（例: opus 系）
- VTT 原本パス: `<project-root>/inbox/meetings/YYYY-MM-DD_sample.vtt`
- MP4 原本パス: （任意）
- ゴールド側議事録パス: `<project-root>/meetings/YYYY-MM-DD_sample/`
- 判定ライン: Recall 漏れ率 ≤ 10%、Apply 率 ≥ 80%

---

## サンプル追加

S3 以降を追加する場合は本ファイルに同形式で追記する。
