# float32 移行 設計詳細

## 1. データ構造の変更

### 1.1 `struct stretch_cnxt` の改修 (stretch.c:45-53)

```c
// Before
struct stretch_cnxt {
    int num_chans, inbuff_samples, shortest, longest, tail, head, fast_mode;
    int16_t *inbuff, *calcbuff;
    float outsamples_error;
    uint32_t *results;
    struct stretch_cnxt *next;
    int16_t *intermediate;
};

// After
struct stretch_cnxt {
    int num_chans, inbuff_samples, shortest, longest, tail, head, fast_mode;
    float *inbuff, *calcbuff;
    float outsamples_error;
    float *results;           // uint32_t → float (相関値を浮動小数点で保持)
    struct stretch_cnxt *next;
    float *intermediate;
};
```

**変更点**:
- `int16_t *inbuff` → `float *inbuff`
- `int16_t *calcbuff` → `float *calcbuff`
- `int16_t *intermediate` → `float *intermediate`
- `uint32_t *results` → `float *results`（相関値も float 化）

**メモリ影響**:
- `inbuff`: `longest_period * num_channels * max_periods * sizeof(float)` = 従来の 2 倍
- `calcbuff`: `longest_period * num_channels * sizeof(float)` = 従来の 2 倍
- `intermediate`: 同上、2 倍
- `results`: `longest_period * sizeof(float)` = 従来と同じ（uint32_t と float はどちらも 4 bytes）

### 1.2 API の拡張 (stretch.h)

後方互換を保つため、既存 API を維持しつつ `_float` バリアントを追加する：

```c
// 既存 API (後方互換、内部で int16→float→int16 変換)
int stretch_samples (StretchHandle handle, const int16_t *samples, int num_samples,
                     int16_t *output, float ratio);
int stretch_flush (StretchHandle handle, int16_t *output);

// 新規 float API
int stretch_samples_float (StretchHandle handle, const float *samples, int num_samples,
                           float *output, float ratio);
int stretch_flush_float (StretchHandle handle, float *output);
```

**設計判断**: 既存 API を残す根拠
- 既存ユーザーとの互換性
- int16 API の内部実装は「int16→float 変換 → float 処理 → float→int16 変換」となり、
  変換オーバーヘッドはあるがコード重複は最小限

---

## 2. WAV ヘッダー処理の拡張

### 2.1 追加する定数 (main.c)

```c
#define WAVE_FORMAT_PCM          0x1
#define WAVE_FORMAT_IEEE_FLOAT   0x3    // 新規追加
#define WAVE_FORMAT_EXTENSIBLE   0xfffe
```

### 2.2 フォーマット検出ロジックの変更

```c
// Before (main.c:281-305)
if (bits_per_sample != 16) {
    fprintf(stderr, "\"%s\" is not a 16-bit .WAV file!\n", infilename);
    return 1;
}
if (WaveHeader.BlockAlign != WaveHeader.NumChannels * 2) { ... }
if (format == WAVE_FORMAT_PCM) { ... }
else { ... } // reject

// After
int bytes_per_sample;

if (format == WAVE_FORMAT_IEEE_FLOAT ||
    (format == WAVE_FORMAT_EXTENSIBLE && WaveHeader.SubFormat == WAVE_FORMAT_IEEE_FLOAT)) {
    bytes_per_sample = 4;  // float32
} else if (format == WAVE_FORMAT_PCM) {
    bytes_per_sample = 2;  // int16
    if (bits_per_sample != 16) { error; }
} else {
    error("unsupported format");
}

if (WaveHeader.BlockAlign != WaveHeader.NumChannels * bytes_per_sample) { error; }
```

### 2.3 出力フォーマットの選択

出力はデフォルトで入力フォーマットを継承する。CLI オプションで明示指定も可能にする：

```
-o<int|float>  = output format (int16 or float32, default: same as input)
```

もしくはシンプルに「入力が float なら float 出力、int16 なら int16 出力」とする。

### 2.4 write_pcm_wav_header() の汎用化

```c
// Before: bytes_per_sample を引数で受け取っていたが呼び出し側が 2 固定
// After: bytes_per_sample を動的にし、FormatTag も引数化

static int write_wav_header(FILE *outfile, uint32_t num_samples, int num_channels,
                            int bytes_per_sample, uint32_t sample_rate, int is_float)
{
    // ...
    wavhdr.FormatTag = is_float ? WAVE_FORMAT_IEEE_FLOAT : WAVE_FORMAT_PCM;
    wavhdr.BitsPerSample = bytes_per_sample * 8;
    // ...
}
```

---

## 3. 信号処理ロジックのリファクタリング

### 3.1 サンプル値の規格化

| フォーマット | 値域 | 内部表現 |
|--------------|------|----------|
| int16 | [-32768, 32767] | float: [-1.0, +1.0) |
| float32 | [-1.0, +1.0] | float: [-1.0, +1.0] (そのまま) |

変換関数:
```c
// int16 → float
inline float int16_to_float(int16_t v) {
    return v * (1.0f / 32768.0f);
}

// float → int16 (クリッピング付き)
inline int16_t float_to_int16(float v) {
    if (v > 1.0f) v = 1.0f;
    if (v < -1.0f) v = -1.0f;
    return (int16_t)(v * 32767.0f);
}
```

### 3.2 merge_blocks() の float 化 (stretch.c:603-610)

```c
// Before (int16 + MERGE_OFFSET)
static void merge_blocks(int16_t *output, int16_t *input1, int16_t *input2, int samples)
{
    for (i = 0; i < samples; ++i)
        output[i] = (int32_t)(((uint32_t)(input1[i] + MERGE_OFFSET) * (samples - i) +
            (uint32_t)(input2[i] + MERGE_OFFSET) * i) / samples) - MERGE_OFFSET;
}

// After (float)
static void merge_blocks_float(float *output, const float *input1, const float *input2, int samples)
{
    for (i = 0; i < samples; ++i) {
        float w2 = (float)i / (float)samples;
        float w1 = 1.0f - w2;
        output[i] = input1[i] * w1 + input2[i] * w2;
    }
}
```

**注意点**:
- MERGE_OFFSET による符号付き→符号なし変換は不要（float の丸めはゼロ方向ではない）
- 線形補間はシンプルな重み付き平均になる
- 重み計算は毎ループではなく増分加算で最適化可能（ただし可読性重視ならこのまま）

### 3.3 find_period() の float 化 (stretch.c:418-490)

```c
// Before: 整数相関計算
scaler = (MAX_CORR - 1) / sum;
factor = diff ? (sum * scaler) / diff : MAX_CORR;

// After: 浮動小数点相関計算
// 相関値 = (Σ|A[i]| + Σ|B[i]|) / Σ|A[i] - B[i]|
// スケーリング不要、直接除算で factor が求まる
factor = (diff > 1e-12f) ? (sum / diff) : INFINITY;
```

**変更点**:
- `uint32_t sum, diff` → `float sum, diff`
- `abs32(...)` → `fabsf(...)`
- ステレオ→モノ変換: `>> 1` → `* 0.5f`
- `MAX_CORR`, `scaler` は不要に

### 3.4 find_period_fast() の float 化 (stretch.c:502-584)

同上の変更 + 2:1 圧縮処理:
```c
// Before
sum += abs32(cnxt->calcbuff[j++] = ((int32_t)samples[i] + samples[i+1] +
          samples[i+2] + samples[i+3]) >> 2);

// After
sum += fabsf(cnxt->calcbuff[j++] = (samples[i] + samples[i+1] +
          samples[i+2] + samples[i+3]) * 0.25f);
```

### 3.5 相関値の補間処理 (find_period_fast 後半)

```c
// Before (uint32_t での比較)
uint32_t high_side_diff = cnxt->results[best_period] - cnxt->results[best_period+1];
uint32_t low_side_diff = cnxt->results[best_period] - cnxt->results[best_period-1];

// After (float での比較)
float high_side_diff = cnxt->results[best_period] - cnxt->results[best_period+1];
float low_side_diff = cnxt->results[best_period] - cnxt->results[best_period-1];
```

### 3.6 stretch_samples() の memcpy/memmove (stretch.c)

すべての `sizeof(int16_t)` (=2) ベースのメモリ操作を `sizeof(float)` (=4) に変更。
ただし `sizeof(*cnxt->inbuff)` 等を使っていれば型変更だけで自動的に追従する。

---

## 4. ハイブリッド対応の設計

### 4.1 int16 API の内部実装（互換レイヤー）

```c
int stretch_samples(StretchHandle handle, const int16_t *samples, int num_samples,
                    int16_t *output, float ratio)
{
    struct stretch_cnxt *cnxt = (struct stretch_cnxt *) handle;
    int num_chans = cnxt->num_chans;
    int total = num_samples * num_chans;

    // int16 → float 変換
    float *float_in = malloc(total * sizeof(float));
    for (int i = 0; i < total; i++)
        float_in[i] = int16_to_float(samples[i]);

    // float バッファ確保（最大出力）
    int max_out = stretch_output_capacity(handle, num_samples, ratio);
    float *float_out = malloc(max_out * num_chans * sizeof(float));

    // コア処理
    int result = stretch_samples_float(handle, float_in, num_samples, float_out, ratio);

    // float → int16 変換
    for (int i = 0; i < result * num_chans; i++)
        output[i] = float_to_int16(float_out[i]);

    free(float_in);
    free(float_out);
    return result;
}
```

### 4.2 データフロー図

```
入力WAV (int16) ──→ int16→float変換 ──┐
                                        ├──→ [float32内部処理] ──→ 出力
入力WAV (float) ──→ そのまま(passthru) ─┘
```

---

## 5. メモリ使用量の変化

### 典型的なパラメータでの試算

| パラメータ | 値 |
|------------|-----|
| サンプルレート | 48000 Hz |
| longest_period | 872 samples (55Hz) |
| max_periods | 3 (normal) / 4 (fast) |
| num_channels | 2 (stereo) |

| バッファ | int16 時 | float32 時 | 増加量 |
|----------|----------|------------|--------|
| inbuff | 872×2×3×2 = 10,464 bytes | ×2 = 20,928 bytes | +10,464 |
| calcbuff | 872×2×2 = 3,488 bytes | ×2 = 6,976 bytes | +3,488 |
| intermediate | 872×2×3×2 = 10,464 bytes | ×2 = 20,928 bytes | +10,464 |
| results | 872×4 = 3,488 bytes | 872×4 = 3,488 bytes | ±0 |
| **合計** | **~28 KB** | **~52 KB** | **+24 KB** |

メモリ増加は約 24 KB と軽微であり、最新の環境では無視できるレベル。

### CLI デモ (main.c) のバッファ

| バッファ | サイズ (25ms @ 48kHz stereo) | int16 | float32 |
|----------|------------------------------|-------|---------|
| inbuffer | 1200 × 4 = 4,800 bytes | 同左 | 9,600 bytes |
| outbuffer | ~2× = ~9,600 bytes | 同左 | ~19,200 bytes |

合計でも +14 KB 程度。

---

## 6. パフォーマンス考察

### 6.1 演算コストの変化

| 操作 | int16 | float32 | 変化 |
|------|-------|---------|------|
| 絶対値 | `abs()` (整数) | `fabsf()` (float) | 同等 |
| 乗算 | 整数乗算 | float乗算 | 同等～微増 |
| 除算 | 整数除算 | float除算 | 微増 |
| 加減算 | 整数 | float | 同等 |
| int16↔float 変換 | なし | 発生（互換レイヤー時） | **追加コスト** |

### 6.2 変換オーバーヘッド

int16 API を使う場合の変換コスト:
- 1サンプルあたり int16→float 変換 ×1, float→int16 変換 ×1
- 48000 Hz, stereo, 1分間 = 5,760,000 サンプル
- 変換のみで約 1-2ms 程度（モダンなCPUでは無視可能）

### 6.3 最適化の余地

- `merge_blocks_float()` は SIMD (SSE/NEON) によるベクトル化が容易
- int16 時代の整数演算よりも、float32 の方が SIMD 最適化との親和性が高い
- gcc `-Ofast` で自動ベクトル化が期待できる

### 6.4 注意点

- **非正規化数**: 無音近くで非正規化数が発生すると急激に遅くなる。
  必要に応じて `-ffast-math` またはゼロへのフラッシュ処理を検討
- **NaN/Inf 伝播**: 無音区間で相関計算の分母が 0 になり Inf が出る可能性がある。
  find_period 系でチェックを入れる
