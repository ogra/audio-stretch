# 制限事項と検証計画

## 1. 依存ライブラリへの影響

### 1.1 結論: 影響なし

本プロジェクトは外部オーディオライブラリに依存していない。WAV ファイルのパース・生成は
すべて `main.c` 内で独自実装されている。依存するのは以下だけであり、いずれも変更不要:

| 依存物 | 用途 | 影響 |
|--------|------|------|
| `<stdlib.h>` | malloc, free, calloc, strtol, strtod | なし |
| `<stdint.h>` | int16_t, uint32_t, uint16_t 等 | なし（int16_t は互換ラッパーで引き続き使用） |
| `<string.h>` | memcpy, memmove, memset, strncmp, strcmp | なし |
| `<stdio.h>` | FILE I/O, fprintf | なし |
| `<math.h>` | log10, sin, ceil, floor, fabsf | fabsf の使用が増えるが標準関数 |

### 1.2 libsndfile 非依存

`.gitignore`, `build.sh`, ソースコードのいずれにも `libsndfile` への言及はない。
`libsndfile` の追加は不要。

---

## 2. メモリ使用量の変化

### 2.1 コアライブラリ (stretch.c)

最悪ケース（longest_period=2400, 48kHz stereo, fast mode）:

| バッファ | int16 | float32 | 差分 |
|----------|-------|---------|------|
| inbuff | 2400×2×4×2 = 38,400 bytes | ×2 = 76,800 bytes | +38,400 |
| calcbuff | 2400×2×2 = 9,600 bytes | ×2 = 19,200 bytes | +9,600 |
| intermediate | 2400×2×4×2 = 38,400 bytes | ×2 = 76,800 bytes | +38,400 |
| results | 2400×4 = 9,600 bytes | 2400×4 = 9,600 bytes | ±0 |
| **合計** | **96,000 bytes** | **182,400 bytes** | **+86,400 bytes** |

増加は最大でも約 86 KB。組み込み用途でも許容範囲。

### 2.2 CLI デモ (main.c)

audio_window_ms=100, 48kHz stereo の場合:

| バッファ | int16 | float32 |
|----------|-------|---------|
| inbuffer | 4800×4 = 19,200 bytes | 4800×8 = 38,400 bytes |
| outbuffer | ~38,400 bytes | ~76,800 bytes |
| prebuffer | 19,200 bytes | 38,400 bytes |

+57 KB 程度の増加。

### 2.3 変換用一時バッファ（互換ラッパー）

`stretch_samples()` / `stretch_flush()` の int16 互換ラッパー内で、
呼び出しごとに `malloc`/`free` する一時 float バッファが発生する。
これはヒープ確保のオーバーヘッドになるため、長時間の連続処理では注意が必要。
必要に応じてバッファを再利用する最適化も検討できる。

---

## 3. 演算パフォーマンス

### 3.1 期待される変化

| 関数 | 変更内容 | パフォーマンス影響 |
|------|----------|-------------------|
| `merge_blocks` | 整数演算 → 浮動小数点 | 同等（除算がなくなり乗算+加算のみになった） |
| `find_period` | 整数相関 → 浮動小数点相関 | 微減（整数除算→浮動小数点除算）だが、scaler計算が不要になる分相殺 |
| `find_period_fast` | 同上 + ビットシフト → 乗算 | 同等 |
| `stretch_samples` (float) | memcpyサイズが2倍 | ほぼゼロ影響 |
| int16 互換ラッパー | int16↔float 変換追加 | オーバーヘッドあり（後述） |

### 3.2 変換オーバーヘッドの定量化

int16 API 互換ラッパーでの追加コスト:

```
int16→float: v * (1.0f / 32768.0f)   // 乗算1回
float→int16: v * 32767.0f + clamp     // 乗算1回 + 分岐2回
```

48kHz stereo, 1分間の処理 = 5,760,000 フレーム × 2 チャンネル = 11,520,000 サンプル。
変換のみで約 0.5～2 ms（モダンCPU、gcc -Ofast）。

これは全体の処理時間に比べて無視できるレベルだが、
どうしても気になる場合は `-DSTRETCH_NO_INT16_COMPAT` で
互換APIをコンパイル対象外にするビルドオプションを用意できる。

### 3.3 SIMD 最適化の可能性

float32 化により、以下の最適化が容易になる:

1. `merge_blocks_float()` のループは SSE/AVX で 4～8 サンプル同時処理が可能
2. `find_period()` の差分絶対値和は SIMD で高速化可能
3. gcc `-Ofast -ftree-vectorize` で自動ベクトル化が期待できる

int16 時代の MERGE_OFFSET による unsigned 変換は SIMD 化の障壁だったが、
float32 ではその制約がなくなる。

### 3.4 非正規化数 (Denormals) の注意

無音区間で float 値がきわめて小さくなり、非正規化数が発生すると
CPU の演算速度が急激に低下する（一部の x86 CPU で顕著）。

**対策**:
- gcc の `-ffast-math` は非正規化数をゼロにフラッシュする
- または `find_period()` の無音検出閾値 (`1e-12f`) より下をゼロ扱いする設計にしている
- `merge_blocks_float()` でも必要に応じて `FTZ` (Flush To Zero) を検討

---

## 4. 精度と音質

### 4.1 相関計算の精度

int16 版では:
- `scaler = (UINT32_MAX - 1) / sum` により、sum が小さいと scaler が極端に大きくなる
- `factor = (sum * scaler) / diff` で、diff が小さいと factor が UINT32_MAX に達する
- 整数演算のため、小さな diff の差が factor に反映されにくい

float32 版では:
- `factor = sum / diff` で直接計算
- 浮動小数点の仮数部 23bit の精度で相関値が得られる
- ピッチ検出精度は同等以上と期待される

### 4.2 merge_blocks の精度

int16 版:
- MERGE_OFFSET で unsigned に変換し、整数除算でクロスフェード
- 丸め方向がゼロ方向なため、ゼロ付近で歪みが生じる（MERGE_OFFSET を使う理由）

float32 版:
- 線形補間（`w1 * a + w2 * b`）
- IEEE 754 丸め（最近接偶数丸め）で、ゼロ付近の歪みは解消
- 理論上の SNR は int16 (96dB) → float32 (138dB) に向上

### 4.3 ダイナミックレンジ

int16: 約 96 dB
float32: 約 768 dB（事実上無制限）

これにより、非常に小さな信号や、複数回のクロスフェードによる累積誤差が改善される。

---

## 5. 互換性

### 5.1 API 互換性

| 関数 | 互換性 |
|------|--------|
| `stretch_init()` | **完全互換**（シグネチャ不変） |
| `stretch_output_capacity()` | **完全互換**（シグネチャ不変） |
| `stretch_samples()` | **完全互換**（シグネチャ不変、内部で変換） |
| `stretch_flush()` | **完全互換**（シグネチャ不変、内部で変換） |
| `stretch_reset()` | **完全互換**（シグネチャ不変） |
| `stretch_deinit()` | **完全互換**（シグネチャ不変） |

既存の int16 API ユーザーは再コンパイルのみで移行可能。

### 5.2 WAV 互換性

| 入力 | 出力 |
|------|------|
| int16 PCM WAV | デフォルト: int16 PCM WAV（既存と同じ） |
| float32 IEEE WAV | デフォルト: float32 IEEE WAV |
| その他 (24bit, 8bit等) | 非対応（従来通り） |

### 5.3 エンディアン

WAV ファイルの float32 データはリトルエンディアンが標準。
本コードは従来通りリトルエンディアン環境（x86, ARMv8 等）を前提とする。
ビッグエンディアン環境ではバイトスワップが必要。

---

## 6. 検証計画

### 6.1 ビルド確認

```bash
# リリースビルド
./build.sh rel

# デバッグビルド (Undefined Behavior Sanitizer)
./build.sh ubsan

# Address Sanitizer
./build.sh asan
```

### 6.2 ユニットテスト的確認

1. **int16 入出力（後方互換テスト）**
   - 既存の `samples/mono.wv`, `samples/stereo.wv` + `test.sh` を実行
   - 比率 0.5, 0.75, 1.0, 1.25, 1.5, 2.0 でエラーなく動作すること
   - 出力 WAV が再生可能であること

2. **float32 入出力**
   - 16-bit int WAV を float32 WAV に事前変換し、テスト入力とする
   - `ffprobe` 等で出力ファイルのフォーマットを確認
   - int16 入力→int16 出力と float32 入力→float32 出力で同程度の音質であること

3. **ハイブリッド（int16入力→float32内部→int16出力）**
   - 既存の int16 入力ファイルでテスト
   - 出力が従来版とビットパーフェクトに一致する必要はないが、
     音質劣化がないことを聴感で確認

4. **エッジケース**
   - 無音ファイル（全サンプルが 0）
   - フルスケール正弦波（クリッピングの確認）
   - 極端な比率（0.25, 4.0）
   - ギャップ/サイレンスモード (-g オプション)

### 6.3 メモリチェック

```bash
# Valgrind (Linux) または Address Sanitizer でメモリリークチェック
./audio-stretch -r1.5 samples/mono.wv /tmp/out.wv

# macOS では leaks コマンド
leaks -- audio-stretch -r1.5 samples/mono.wv /tmp/out.wv
```

### 6.4 パフォーマンス比較

```bash
# 大きなWAVファイルで処理時間を計測
time ./audio-stretch-int16 -r1.5 large.wav /tmp/out.wav
time ./audio-stretch-float32 -r1.5 large.wav /tmp/out.wav
```

---

## 7. 将来の拡張可能性

1. **倍精度対応**: 必要に応じて `float` → `double` 化も容易（型エイリアスを使えば1箇所変更で対応可能）
2. **出力フォーマット指定**: CLI オプションで int16/float32 出力を明示指定可能にする
3. **WAV以外のフォーマット**: float32 化により raw PCM との相互運用性も向上
4. **SIMD 最適化**: SSE/AVX/NEON による高速化の下地が整う