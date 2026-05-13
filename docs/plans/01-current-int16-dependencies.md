# 現状の int16 依存箇所 詳細マッピング

## 1. stretch.h — パブリック API (3箇所)

### 1.1 関数シグネチャ

```c
// stretch.h:39 — stretch_samples()
int stretch_samples (StretchHandle handle, const int16_t *samples, int num_samples,
                     int16_t *output, float ratio);

// stretch.h:40 — stretch_flush()
int stretch_flush (StretchHandle handle, int16_t *output);
```

両関数の入出力バッファが `int16_t *` に固定されている。

### 1.2 ドキュメント文言

```c
// stretch.h:14
// to stretch the timing of a 16-bit PCM signal (either mono or stereo)
```

---

## 2. stretch.c — コア実装

### 2.1 データ構造 `struct stretch_cnxt` (stretch.c:45-53)

```c
struct stretch_cnxt {
    int num_chans, inbuff_samples, shortest, longest, tail, head, fast_mode;
    int16_t *inbuff, *calcbuff;     // ← 2つの int16_t バッファ
    float outsamples_error;
    uint32_t *results;

    struct stretch_cnxt *next;
    int16_t *intermediate;          // ← カスケード用中間バッファも int16_t
};
```

影響範囲:
- `inbuff`: 入力サンプルのリングバッファ。stretch_init, stretch_reset, stretch_samples, stretch_flush で使用
- `calcbuff`: ピッチ検出用の計算バッファ（ステレオ→モノ変換、fast mode の 2:1 圧縮）
- `intermediate`: カスケードインスタンス間の中間バッファ

### 2.2 定数定義 (stretch.c:35-43)

```c
#if INT_MAX == 32767
#define MERGE_OFFSET    32768L      // int16 → unsigned 変換用オフセット
#define abs32           labs
#else
#define MERGE_OFFSET    32768
#define abs32           abs
#endif

#define MAX_CORR    UINT32_MAX      // 相関計算の最大値
```

- **MERGE_OFFSET (32768)**: 符号付き int16 を unsigned に変換するためのオフセット。float32 化で不要になる
- **abs32**: int32 の絶対値。float32 化で `fabsf()` に置換
- **MAX_CORR (UINT32_MAX)**: 整数相関の最大値。float32 では不要（相関値が 0.0～1.0 の範囲に正規化される）

### 2.3 merge_blocks() (stretch.c:603-610)

```c
static void merge_blocks (int16_t *output, int16_t *input1, int16_t *input2, int samples)
{
    for (i = 0; i < samples; ++i)
        output [i] = (int32_t)(((uint32_t)(input1 [i] + MERGE_OFFSET) * (samples - i) +
            (uint32_t)(input2 [i] + MERGE_OFFSET) * i) / samples) - MERGE_OFFSET;
}
```

**処理内容**: 2つのピッチ周期ブロックを線形クロスフェードで合成。
符号付き→符号なし変換（+32768）、整数乗算・除算、符号なし→符号付き変換（-32768）の
パターンで、C のゼロ方向丸めによる圧縮歪みを回避している。

**float32 化時の注意**: このゼロ付近の歪み問題は float では発生しないため、
MERGE_OFFSET を使わない単純な線形補間に置き換えられる。

### 2.4 find_period() (stretch.c:418-490)

```c
static int find_period (struct stretch_cnxt *cnxt, int16_t *samples)
```

**int16 依存箇所**:

| 行 | 内容 | 説明 |
|----|------|------|
| 429-433 | ステレオ→モノ変換 | `((int32_t)samples[i] + samples[i+1]) >> 1` — ビットシフト平均 |
| 436-437 | モノ累積和 | `abs32(calcbuff[i]) + abs32(calcbuff[i+cnxt->longest])` |
| 442 | スケーラ計算 | `scaler = (MAX_CORR - 1) / sum` — 整数除算による正規化係数 |
| 449 | 短期間累積和 | `abs32(calcbuff[i]) + abs32(calcbuff[i+period])` |
| 462 | 差分計算 | `abs32((int32_t)*--ref - *--comp)` |
| 471 | 相関値計算 | `factor = diff ? (sum * scaler) / diff : MAX_CORR` |
| 485 | 累積和更新 | `abs32(calcbuff[period*2]) + abs32(calcbuff[period*2+1])` |

**アルゴリズム概要**:
```
  factor = (Σ|A[i]| + Σ|B[i]|) × scaler / Σ|A[i] - B[i]|
```
ここで:
- A, B は連続する2つのピッチ周期ブロック
- scaler = (UINT32_MAX - 1) / (全サンプルの絶対値和)
- factor が最大となる period が最適ピッチ周期

### 2.5 find_period_fast() (stretch.c:502-584)

```c
static int find_period_fast (struct stretch_cnxt *cnxt, int16_t *samples)
```

**int16 依存箇所**:

| 行 | 内容 | 説明 |
|----|------|------|
| 512-513 | ステレオ 2:1 圧縮 | 4サンプル合計を `>> 2` |
| 516-517 | モノ 2:1 圧縮 | 2サンプル合計を `>> 1` |
| 529 | 短期間累積和 | 同上 |
| 542 | 差分計算 | 同上 |
| 551 | 相関値計算・保存 | `cnxt->results[period] = ...` |
| 565 | 累積和更新 | 同上 |
| 569-578 | 補間による微調整 | 隣接ピークとの比較で最適周期を小数点精度に |

**find_period との違い**: データを 2:1 に圧縮し、偶数ピッチ周期のみを試行する。
最後に隣接ピリオドの相関値を比較して補間する。計算量は約 1/4。

### 2.6 stretch_init() (stretch.c:72-117)

```c
// 行91-92: バッファ確保
cnxt->inbuff = calloc (cnxt->inbuff_samples, sizeof (*cnxt->inbuff));
// → sizeof(int16_t) → sizeof(float) に変わる（メモリ2倍）

// 行95: calcbuff 確保
cnxt->calcbuff = calloc (longest_period * num_channels, sizeof (*cnxt->calcbuff));

// 行98: results 確保 (uint32_t のまま維持可能)
cnxt->results = calloc (longest_period, sizeof (*cnxt->results));

// 行113: intermediate 確保
cnxt->intermediate = calloc (..., sizeof (*cnxt->intermediate));
```

### 2.7 stretch_samples() (stretch.c:186-352)

**int16 依存箇所**:

| 行 | 内容 |
|----|------|
| 186 | 関数シグネチャ `const int16_t *samples, int16_t *output` |
| 190 | `int16_t *outbuf = output` |
| 230 | `memcpy(cnxt->inbuff + cnxt->head, samples, samples_to_copy * sizeof(cnxt->inbuff[0]))` |
| 242-243 | `find_period_fast(cnxt, cnxt->inbuff + cnxt->tail)` |
| 264-268 | merge_blocks + memcpy — 各 process_ratio ケース |
| 319-324 | memmove によるバッファ左詰め |

### 2.8 stretch_flush() (stretch.c:361-383)

```c
// 行375: memcpy output
memcpy (output, cnxt->inbuff + cnxt->tail, samples_leftover * sizeof (*output));
```

### 2.9 stretch_reset() (stretch.c:124-133)

```c
// 行129: memset
memset (cnxt->inbuff, 0, cnxt->tail * sizeof (*cnxt->inbuff));
```

---

## 3. main.c — CLI デモプログラム

### 3.1 WAV 読み取り側の制限

| 行 | 内容 |
|----|------|
| 275-276 | `WAVE_FORMAT_EXTENSIBLE` からの `SubFormat` 読み取りは実装済み（拡張可能） |
| 281-284 | `bits_per_sample != 16` → エラー。float32 を受け付けるように変更必要 |
| 291-294 | `BlockAlign != NumChannels * 2` → エラー。float32 では `NumChannels * 4` |
| 296-305 | `format == WAVE_FORMAT_PCM` のみ許可。float (0x0003) を追加必要 |

### 3.2 バッファ確保・読み書き

| 行 | 内容 |
|----|------|
| 403 | `int16_t *inbuffer = malloc(buffer_samples * WaveHeader.BlockAlign)` |
| 404 | `int16_t *outbuffer = malloc(max_expected_samples * WaveHeader.BlockAlign)` |
| 412 | `prebuffer = malloc(buffer_samples * WaveHeader.BlockAlign)` |
| 423-424 | `fread(..., WaveHeader.BlockAlign, ..., infile)` |
| 474 | `fwrite(outbuffer, WaveHeader.BlockAlign, samples_generated, outfile)` |
| 487 | `memcpy(inbuffer, prebuffer, samples_read * WaveHeader.BlockAlign)` |

### 3.3 write_pcm_wav_header() (main.c:546-577)

```c
// 行395, 524: 呼び出し側が bytes_per_sample = 2 をハードコード
write_pcm_wav_header (outfile, 0, WaveHeader.NumChannels, 2, scaled_rate);

// 行558-562: ヘッダ構築
wavhdr.FormatTag = WAVE_FORMAT_PCM;                      // float時は 0x0003
wavhdr.BytesPerSecond = sample_rate * num_channels * bytes_per_sample;
wavhdr.BlockAlign = bytes_per_sample * num_channels;     // float時は 4*ch
wavhdr.BitsPerSample = bytes_per_sample * 8;             // float時は 32
```

### 3.4 RMS レベル計算 (main.c:579-594)

```c
double rms_level_dB (int16_t *audio, int samples, int channels)
{
    // ...
    return log10 (rms_sum / samples / (32768.0 * 32767.0 * 0.5)) * 10.0;
    //                                ^^^^^^^^^^^^^^^^^^^^^^^^
    //                                int16 のスケーリング定数
}
```

この関数は無音検出（silence mode）に使用される。float32 化の際は
正規化定数を `1.0` にする（float の範囲は -1.0 ～ +1.0）。

### 3.5 WAVE_FORMAT 定義 (main.c:73-74)

```c
#define WAVE_FORMAT_PCM         0x1
#define WAVE_FORMAT_EXTENSIBLE  0xfffe
// WAVE_FORMAT_IEEE_FLOAT (0x0003) が未定義
```

---

## 4. 依存関係のまとめ

```
int16_t が出現するファイル・箇所:
  stretch.h      3箇所 (APIシグネチャ×2 + コメント×1)
  stretch.c      ~40箇所 (バッファ, 演算, memcpy/memset/memmove)
  main.c         ~20箇所 (バッファ, WAVヘッダ, fread/fwrite, RMS計算)

int16 に依存する定数:
  MERGE_OFFSET (32768)    — merge_blocks の符号変換用
  MAX_CORR (UINT32_MAX)   — 相関計算の最大値
  abs32                   — int32絶対値マクロ
  32768.0 * 32767.0       — RMS 正規化定数

int16 に依存するビット演算:
  >> 1                    — ステレオ→モノ平均 (find_period)
  >> 2                    — 4サンプル平均 (find_period_fast ステレオ)
  >> 1                    — 2サンプル平均 (find_period_fast モノ)
```

## 5. 外部依存なしの確認

このプロジェクトは `libsndfile` 等の外部オーディオライブラリに依存していない。
WAV ファイルのパース・生成はすべて `main.c` 内で独自に実装されている。
したがって、float32 対応はこのプロジェクト内のコード変更のみで完結する。
