# Audio-Stretch float32 移行計画 — 概要

## プロジェクト概要

audio-stretch は Time Domain Harmonic Scaling (TDHS) による音声・オーディオ信号の
タイムストレッチライブラリおよびそのデモ CLI プログラムである。

| 項目 | 内容 |
|------|------|
| ソースファイル | `stretch.h` (API), `stretch.c` (コア実装), `main.c` (CLIデモ) |
| 依存ライブラリ | **なし**（標準C + libm のみ） |
| ビルド | gcc (`-Ofast` / `-O0 -g`), `-lm` リンク |
| 現在のデータ型 | `int16_t` (16-bit PCM) のみ |
| 対応WAVフォーマット | `WAVE_FORMAT_PCM` (0x0001) のみ |
| 対応チャンネル | 1 (mono), 2 (stereo) |
| 対応サンプルレート | 8,000 ～ 48,000 Hz |

## float32 移行の目的

1. 16-bit PCM の量子化ノイズ・ダイナミックレンジ制約からの解放
2. 現代的なオーディオ処理パイプラインとの相互運用性向上
3. 内部演算精度の向上による音質改善（特に複数回のクロスフェード処理で顕著）

## 移行の基本方針

**ハイブリッド方式** を採用する：

- **入力**: `int16` PCM および `float32` IEEE Float WAV の両方を受け付ける
- **内部処理**: すべて `float32` (C `float` 型) で統一
- **出力**: 入力フォーマットを継承、または CLI オプションで指定
- **API**: 既存の `int16_t` API を維持しつつ、新たに `float` API を追加

## ドキュメント構成

| ファイル | 内容 |
|----------|------|
| `00-overview.md` (本ファイル) | 全体概要 |
| `01-current-int16-dependencies.md` | 現状の int16 依存箇所の詳細マッピング |
| `02-float32-migration-design.md` | 移行設計の詳細 |
| `03-implementation-diff.md` | 具体的なコード変更指示 |
