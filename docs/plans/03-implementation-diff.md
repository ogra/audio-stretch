# 具体的な実装指示 (diff)

以下、ファイルごとに具体的な変更内容を示す。

---

## ファイル 1: stretch.h

### 変更 1.1: コメント修正

```diff
-// to stretch the timing of a 16-bit PCM signal (either mono or stereo) from
+// to stretch the timing of a PCM or IEEE Float signal (either mono or stereo) from
```

### 変更 1.2: float API の追加

```diff
 StretchHandle stretch_init (int shortest_period, int longest_period, int num_chans, int flags);
 int stretch_output_capacity (StretchHandle handle, int max_num_samples, float max_ratio);
+
+// Original int16 API (maintained for backward compatibility)
 int stretch_samples (StretchHandle handle, const int16_t *samples, int num_samples, int16_t *output, float ratio);
 int stretch_flush (StretchHandle handle, int16_t *output);
+
+// New float32 API
+int stretch_samples_float (StretchHandle handle, const float *samples, int num_samples, float *output, float ratio);
+int stretch_flush_float (StretchHandle handle, float *output);
+
 void stretch_reset (StretchHandle handle);
```

---

## ファイル 2: stretch.c

### 変更 2.1: 定数定義の置換 (行 35-43)

```diff
-#if INT_MAX == 32767
-#define MERGE_OFFSET    32768L      /* promote to long before offset */
-#define abs32           labs        /* use long abs to avoid UB */
-#else
-#define MERGE_OFFSET    32768
-#define abs32           abs
-#endif
-
-#define MAX_CORR    UINT32_MAX  /* maximum value for correlation ratios */
+// Remove all int16-specific constants (MERGE_OFFSET, abs32, MAX_CORR)
+// float32 processing uses standard fabsf() and direct comparison
```

### 変更 2.2: struct stretch_cnxt (行 45-53)

```diff
 struct stretch_cnxt {
     int num_chans, inbuff_samples, shortest, longest, tail, head, fast_mode;
-    int16_t *inbuff, *calcbuff;
+    float *inbuff, *calcbuff;
     float outsamples_error;
-    uint32_t *results;
+    float *results;

     struct stretch_cnxt *next;
-    int16_t *intermediate;
+    float *intermediate;
 };
```

### 変更 2.3: 前方宣言の更新 (行 55-57)

```diff
-static void merge_blocks (int16_t *output, int16_t *input1, int16_t *input2, int samples);
-static int find_period_fast (struct stretch_cnxt *cnxt, int16_t *samples);
-static int find_period (struct stretch_cnxt *cnxt, int16_t *samples);
+static void merge_blocks_float (float *output, const float *input1, const float *input2, int samples);
+static int find_period_fast (struct stretch_cnxt *cnxt, const float *samples);
+static int find_period (struct stretch_cnxt *cnxt, const float *samples);
```

### 変更 2.4: stretch_init() 内部 (行 91-98)

`sizeof` は型変更に自動追従するため、明示的な変更不要。
ただし `results` の型が変わる点に注意:

```diff
-    cnxt->results = calloc (longest_period, sizeof (*cnxt->results));
+    cnxt->results = calloc (longest_period, sizeof (*cnxt->results));  // float now
```

### 変更 2.5: stretch_samples_float() — 新規追加 (行 186 付近)

`stretch_samples()` の内容をリネームし、`int16_t` → `float` に置換したバージョン:

```c
int stretch_samples_float (StretchHandle handle, const float *samples, int num_samples,
                           float *output, float ratio)
{
    struct stretch_cnxt *cnxt = (struct stretch_cnxt *) handle;
    int out_samples = 0, next_samples = 0;
    float *outbuf = output;
    float next_ratio;

    if (cnxt->next) {
        outbuf = cnxt->intermediate;

        if (ratio < 0.5) {
            next_ratio = ratio / 0.5;
            ratio = 0.5;
        }
        else if (ratio > 2.0) {
            next_ratio = ratio / 2.0;
            ratio = 2.0;
        }
        else
            next_ratio = 1.0;
    }

    num_samples *= cnxt->num_chans;

    if (ratio < 0.5)
        ratio = 0.5;
    else if (ratio > 2.0)
        ratio = 2.0;

    while (num_samples) {
        int samples_to_copy = num_samples;

        if (samples_to_copy > cnxt->inbuff_samples - cnxt->head)
            samples_to_copy = cnxt->inbuff_samples - cnxt->head;

        memcpy (cnxt->inbuff + cnxt->head, samples, samples_to_copy * sizeof (cnxt->inbuff [0]));
        num_samples -= samples_to_copy;
        samples += samples_to_copy;
        cnxt->head += samples_to_copy;

        while (cnxt->tail >= cnxt->longest &&
               cnxt->head - cnxt->tail >= cnxt->longest * (cnxt->fast_mode ? 3 : 2)) {
            float process_ratio;
            int period;

            if (ratio != 1.0 || cnxt->outsamples_error)
                period = cnxt->fast_mode ?
                    find_period_fast (cnxt, cnxt->inbuff + cnxt->tail) :
                    find_period (cnxt, cnxt->inbuff + cnxt->tail);
            else
                period = cnxt->longest;

            if (cnxt->outsamples_error == 0.0)
                process_ratio = floor (ratio * 2.0 + 0.5) / 2.0;
            else if (cnxt->outsamples_error > 0.0)
                process_ratio = floor (ratio * 2.0) / 2.0;
            else
                process_ratio = ceil (ratio * 2.0) / 2.0;

            if (process_ratio == 0.5) {
                merge_blocks_float (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                    cnxt->inbuff + cnxt->tail + period, period);
                cnxt->outsamples_error += period - (period * 2.0 * ratio);
                out_samples += period;
                cnxt->tail += period * 2;
            }
            else if (process_ratio == 1.0) {
                memcpy (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                    period * 2 * sizeof (cnxt->inbuff [0]));

                if (ratio != 1.0)
                    cnxt->outsamples_error += (period * 2.0) - (period * 2.0 * ratio);
                else
                    cnxt->outsamples_error = 0;

                out_samples += period * 2;
                cnxt->tail += period * 2;
            }
            else if (process_ratio == 1.5) {
                memcpy (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                    period * sizeof (cnxt->inbuff [0]));
                merge_blocks_float (outbuf + out_samples + period, cnxt->inbuff + cnxt->tail + period,
                    cnxt->inbuff + cnxt->tail, period);
                memcpy (outbuf + out_samples + period * 2, cnxt->inbuff + cnxt->tail + period,
                    period * sizeof (cnxt->inbuff [0]));
                cnxt->outsamples_error += (period * 3.0) - (period * 2.0 * ratio);
                out_samples += period * 3;
                cnxt->tail += period * 2;
            }
            else if (process_ratio == 2.0) {
                merge_blocks_float (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                    cnxt->inbuff + cnxt->tail - period, period * 2);

                cnxt->outsamples_error += (period * 2.0) - (period * ratio);
                out_samples += period * 2;
                cnxt->tail += period;

                if (cnxt->fast_mode) {
                    merge_blocks_float (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                        cnxt->inbuff + cnxt->tail - period, period * 2);

                    cnxt->outsamples_error += (period * 2.0) - (period * ratio);
                    out_samples += period * 2;
                    cnxt->tail += period;
                }
            }
            else
                fprintf (stderr, "stretch_samples: fatal programming error: process_ratio == %g\n", process_ratio);

            if (cnxt->next) {
                next_samples += stretch_samples_float (cnxt->next, outbuf, out_samples / cnxt->num_chans,
                    output + next_samples * cnxt->num_chans, next_ratio);
                out_samples = 0;
            }

            int samples_to_move = cnxt->inbuff_samples - cnxt->tail + cnxt->longest;

            memmove (cnxt->inbuff, cnxt->inbuff + cnxt->tail - cnxt->longest,
                samples_to_move * sizeof (cnxt->inbuff [0]));

            cnxt->head -= cnxt->tail - cnxt->longest;
            cnxt->tail = cnxt->longest;
        }
    }

    if (ratio == 1.0 && !cnxt->outsamples_error && cnxt->head != cnxt->tail) {
        int samples_leftover = cnxt->head - cnxt->tail;

        if (cnxt->next)
            next_samples += stretch_samples_float (cnxt->next, cnxt->inbuff + cnxt->tail,
                samples_leftover / cnxt->num_chans,
                output + next_samples * cnxt->num_chans, next_ratio);
        else {
            memcpy (outbuf + out_samples, cnxt->inbuff + cnxt->tail,
                samples_leftover * sizeof (*output));
            out_samples += samples_leftover;
        }

        memmove (cnxt->inbuff, cnxt->inbuff + cnxt->head - cnxt->longest,
            cnxt->longest * sizeof (cnxt->inbuff [0]));
        cnxt->head = cnxt->tail = cnxt->longest;
    }

    return cnxt->next ? next_samples : out_samples / cnxt->num_chans;
}
```

### 変更 2.6: 既存 stretch_samples() → 互換ラッパー化

```c
// 既存の stretch_samples() を int16→float→int16 のラッパーに置き換え
int stretch_samples (StretchHandle handle, const int16_t *samples, int num_samples,
                     int16_t *output, float ratio)
{
    struct stretch_cnxt *cnxt = (struct stretch_cnxt *) handle;
    int num_chans = cnxt->num_chans;
    int total = num_samples * num_chans;
    int i, result;
    float *float_in, *float_out;
    int max_out;

    float_in = malloc (total * sizeof (float));
    if (!float_in) return 0;

    for (i = 0; i < total; i++)
        float_in[i] = samples[i] * (1.0f / 32768.0f);

    max_out = stretch_output_capacity (handle, num_samples, ratio);
    float_out = malloc (max_out * num_chans * sizeof (float));
    if (!float_out) {
        free (float_in);
        return 0;
    }

    result = stretch_samples_float (handle, float_in, num_samples, float_out, ratio);

    for (i = 0; i < result * num_chans; i++) {
        float v = float_out[i];
        if (v > 1.0f) v = 1.0f;
        if (v < -1.0f) v = -1.0f;
        output[i] = (int16_t)(v * 32767.0f);
    }

    free (float_in);
    free (float_out);
    return result;
}
```

### 変更 2.7: stretch_flush_float() — 新規追加

```c
int stretch_flush_float (StretchHandle handle, float *output)
{
    struct stretch_cnxt *cnxt = (struct stretch_cnxt *) handle;
    int samples_leftover = cnxt->head - cnxt->tail;
    int samples_flushed = 0;

    if (cnxt->next) {
        if (samples_leftover)
            samples_flushed = stretch_samples_float (cnxt->next, cnxt->inbuff + cnxt->tail,
                samples_leftover / cnxt->num_chans, output, 1.0);

        if (!samples_flushed)
            samples_flushed = stretch_flush_float (cnxt->next, output);
    }
    else {
        memcpy (output, cnxt->inbuff + cnxt->tail, samples_leftover * sizeof (*output));
        samples_flushed = samples_leftover / cnxt->num_chans;
    }

    cnxt->tail = cnxt->head;
    memset (cnxt->inbuff, 0, cnxt->tail * sizeof (*cnxt->inbuff));

    return samples_flushed;
}
```

### 変更 2.8: 既存 stretch_flush() → 互換ラッパー化

```c
int stretch_flush (StretchHandle handle, int16_t *output)
{
    struct stretch_cnxt *cnxt = (struct stretch_cnxt *) handle;
    int max_out = stretch_output_capacity (handle, cnxt->inbuff_samples / cnxt->num_chans, 1.0);
    float *float_out;
    int result, i;

    float_out = malloc (max_out * cnxt->num_chans * sizeof (float));
    if (!float_out) return 0;

    result = stretch_flush_float (handle, float_out);

    for (i = 0; i < result * cnxt->num_chans; i++) {
        float v = float_out[i];
        if (v > 1.0f) v = 1.0f;
        if (v < -1.0f) v = -1.0f;
        output[i] = (int16_t)(v * 32767.0f);
    }

    free (float_out);
    return result;
}
```

### 変更 2.9: merge_blocks() → merge_blocks_float() (行 603-610)

```diff
-static void merge_blocks (int16_t *output, int16_t *input1, int16_t *input2, int samples)
+static void merge_blocks_float (float *output, const float *input1, const float *input2, int samples)
 {
-    int i;
+    int i;
+    float inv_samples = 1.0f / (float)samples;

-    for (i = 0; i < samples; ++i)
-        output [i] = (int32_t)(((uint32_t)(input1 [i] + MERGE_OFFSET) * (samples - i) +
-            (uint32_t)(input2 [i] + MERGE_OFFSET) * i) / samples) - MERGE_OFFSET;
+    for (i = 0; i < samples; ++i) {
+        float w2 = (float)i * inv_samples;
+        float w1 = 1.0f - w2;
+        output[i] = input1[i] * w1 + input2[i] * w2;
+    }
 }
```

### 変更 2.10: find_period() の全体書き換え (行 418-490)

```c
static int find_period (struct stretch_cnxt *cnxt, const float *samples)
{
    float sum, diff, best_factor = 0.0f;
    const float *calcbuff = samples;
    int period, best_period;
    int i, j;

    period = best_period = cnxt->shortest / cnxt->num_chans;

    // convert stereo to mono, and accumulate sum for longest period
    if (cnxt->num_chans == 2) {
        calcbuff = cnxt->calcbuff;

        for (sum = 0.0f, i = j = 0; i < cnxt->longest * 2; i += 2)
            sum += fabsf (calcbuff [j++] = (samples [i] + samples [i+1]) * 0.5f);
    }
    else
        for (sum = 0.0f, i = 0; i < cnxt->longest; ++i)
            sum += fabsf (calcbuff [i]) + fabsf (calcbuff [i+cnxt->longest]);

    // if silence, return longest period
    if (sum < 1e-12f)
        return cnxt->longest;

    // accumulate sum for shortest period size
    for (sum = 0.0f, i = 0; i < period; ++i)
        sum += fabsf (calcbuff [i]) + fabsf (calcbuff [i+period]);

    while (1) {
        const float *comp = calcbuff + period * 2;
        const float *ref = calcbuff + period;

        // compute sum of absolute differences
        diff = 0.0f;

        while (ref != calcbuff)
            diff += fabsf (*--ref - *--comp);

        // correlation factor: sum / diff (no scaling needed with float)
        float factor = (diff > 1e-12f) ? (sum / diff) : 1e12f;

        if (factor >= best_factor) {
            best_factor = factor;
            best_period = period;
        }

        // see if we're done
        if (period * cnxt->num_chans == cnxt->longest)
            break;

        // update accumulating sum and current period
        sum += fabsf (calcbuff [period * 2]) + fabsf (calcbuff [period * 2 + 1]);
        period++;
    }

    return best_period * cnxt->num_chans;
}
```

### 変更 2.11: find_period_fast() の全体書き換え (行 502-584)

```c
static int find_period_fast (struct stretch_cnxt *cnxt, const float *samples)
{
    float sum, diff, best_factor = 0.0f;
    int period, best_period;
    int i, j;

    best_period = period = cnxt->shortest / (cnxt->num_chans * 2);

    // compress data 2:1 into calcbuff
    if (cnxt->num_chans == 2)
        for (sum = 0.0f, i = j = 0; i < cnxt->longest * 2; i += 4)
            sum += fabsf (cnxt->calcbuff [j++] =
                (samples [i] + samples [i+1] + samples [i+2] + samples [i+3]) * 0.25f);
    else
        for (sum = 0.0f, i = j = 0; i < cnxt->longest * 2; i += 2)
            sum += fabsf (cnxt->calcbuff [j++] =
                (samples [i] + samples [i+1]) * 0.5f);

    // if silence, return longest period
    if (sum < 1e-12f)
        return cnxt->longest;

    // accumulate sum for shortest period
    for (sum = 0.0f, i = 0; i < period; ++i)
        sum += fabsf (cnxt->calcbuff [i]) + fabsf (cnxt->calcbuff [i+period]);

    while (1) {
        const float *comp = cnxt->calcbuff + period * 2;
        const float *ref = cnxt->calcbuff + period;

        diff = 0.0f;

        while (ref != cnxt->calcbuff)
            diff += fabsf (*--ref - *--comp);

        cnxt->results [period] = (diff > 1e-12f) ? (sum / diff) : 1e12f;

        if (cnxt->results [period] >= best_factor) {
            best_factor = cnxt->results [period];
            best_period = period;
        }

        if (period * cnxt->num_chans * 2 == cnxt->longest)
            break;

        sum += fabsf (cnxt->calcbuff [period * 2]) + fabsf (cnxt->calcbuff [period * 2 + 1]);
        period++;
    }

    if (best_period * cnxt->num_chans * 2 != cnxt->shortest &&
        best_period * cnxt->num_chans * 2 != cnxt->longest) {
        float high_side_diff = cnxt->results [best_period] - cnxt->results [best_period+1];
        float low_side_diff = cnxt->results [best_period] - cnxt->results [best_period-1];

        if ((low_side_diff + 1.0f) / 2.0f > high_side_diff)
            best_period = best_period * 2 + 1;
        else if ((high_side_diff + 1.0f) / 2.0f > low_side_diff)
            best_period = best_period * 2 - 1;
        else
            best_period *= 2;
    }
    else
        best_period *= 2;

    return best_period * cnxt->num_chans;
}
```

### 変更 2.12: stretch_reset() (行 129)

`sizeof` が自動追従するため実質的な変更不要だが、型が float になったことを確認。

### 変更 2.13: stretch_deinit() (行 387-401)

変更不要。`free()` に型情報は不要。

---

## ファイル 3: main.c

### 変更 3.1: WAVE_FORMAT 定数の追加 (行 74)

```diff
 #define WAVE_FORMAT_PCM         0x1
+#define WAVE_FORMAT_IEEE_FLOAT  0x3
 #define WAVE_FORMAT_EXTENSIBLE  0xfffe
```

### 変更 3.2: 関数宣言の更新 (行 76-77)

```diff
-static int write_pcm_wav_header (FILE *outfile, uint32_t num_samples, int num_channels, int bytes_per_sample, uint32_t sample_rate);
-double rms_level_dB (int16_t *audio, int samples, int channels);
+static int write_wav_header (FILE *outfile, uint32_t num_samples, int num_channels,
+    int bytes_per_sample, uint32_t sample_rate, int is_float);
+static double rms_level_dB (const float *audio, int samples, int channels);
```

### 変更 3.3: 変数追加 (行 84 付近)

```diff
     int upper_frequency = 333, lower_frequency = 55;
     char *infilename = NULL, *outfilename = NULL;
     int audio_window_ms = AUDIO_WINDOW_MS;
+    int input_bytes_per_sample = 2;  // detected from input file
+    int input_is_float = 0;          // detected from input file
```

### 変更 3.4: WAV フォーマット検出の書き換え (行 275-305)

```diff
             format = (WaveHeader.FormatTag == WAVE_FORMAT_EXTENSIBLE && chunk_header.ckSize == 40) ?
                 WaveHeader.SubFormat : WaveHeader.FormatTag;

             bits_per_sample = (chunk_header.ckSize == 40 && WaveHeader.Samples.ValidBitsPerSample) ?
                 WaveHeader.Samples.ValidBitsPerSample : WaveHeader.BitsPerSample;

-            if (bits_per_sample != 16) {
-                fprintf (stderr, "\"%s\" is not a 16-bit .WAV file!\n", infilename);
+            if (format == WAVE_FORMAT_IEEE_FLOAT ||
+                (format == WAVE_FORMAT_EXTENSIBLE && WaveHeader.SubFormat == WAVE_FORMAT_IEEE_FLOAT)) {
+                input_is_float = 1;
+                input_bytes_per_sample = 4;
+                if (bits_per_sample != 32) {
+                    fprintf (stderr, "\"%s\": float WAV must be 32-bit!\n", infilename);
+                    return 1;
+                }
+            }
+            else if (format == WAVE_FORMAT_PCM) {
+                input_bytes_per_sample = 2;
+                if (bits_per_sample != 16) {
+                    fprintf (stderr, "\"%s\": only 16-bit PCM is supported!\n", infilename);
+                    return 1;
+                }
+            }
+            else {
+                fprintf (stderr, "\"%s\": unsupported format (not PCM or IEEE Float)!\n", infilename);
                 return 1;
             }

-            if (WaveHeader.BlockAlign != WaveHeader.NumChannels * 2) {
-                fprintf (stderr, "\"%s\" is not a valid .WAV file!\n", infilename);
+            if (WaveHeader.BlockAlign != WaveHeader.NumChannels * input_bytes_per_sample) {
+                fprintf (stderr, "\"%s\" has unexpected block alignment!\n", infilename);
                 return 1;
             }
-
-            if (format == WAVE_FORMAT_PCM) {
-                if (WaveHeader.SampleRate < 8000 || WaveHeader.SampleRate > 48000) {
-                    fprintf (stderr, "\"%s\" sample rate is %lu, must be 8000 to 48000!\n",
-                        infilename, (unsigned long) WaveHeader.SampleRate);
-                    return 1;
-                }
-            }
-            else {
-                fprintf (stderr, "\"%s\" is not a PCM .WAV file!\n", infilename);
-                return 1;
-            }
```

### 変更 3.5: write_wav_header() 呼び出し (行 395)

```diff
-    write_pcm_wav_header (outfile, 0, WaveHeader.NumChannels, 2, scaled_rate);
+    write_wav_header (outfile, 0, WaveHeader.NumChannels, input_bytes_per_sample,
+        scaled_rate, input_is_float);
```

### 変更 3.6: バッファ型の変更 (行 403-412)

```diff
-    int16_t *inbuffer = malloc (buffer_samples * WaveHeader.BlockAlign), *prebuffer = NULL;
-    int16_t *outbuffer = malloc (max_expected_samples * WaveHeader.BlockAlign);
+    // Use float buffers internally; allocate based on float size
+    int frame_size = WaveHeader.NumChannels * (int)sizeof(float);
+    float *inbuffer = malloc (buffer_samples * frame_size), *prebuffer = NULL;
+    float *outbuffer = malloc (max_expected_samples * frame_size);

     if (silence_mode)
-        prebuffer = malloc (buffer_samples * WaveHeader.BlockAlign);
+        prebuffer = malloc (buffer_samples * frame_size);
```

### 変更 3.7: 読み取り処理の変更 (行 422-430)

```diff
     while (1) {
-        int samples_read = fread (silence_mode ? prebuffer : inbuffer, WaveHeader.BlockAlign,
-            samples_to_process >= buffer_samples ? buffer_samples : samples_to_process, infile);
+        int samples_to_read = samples_to_process >= buffer_samples ? buffer_samples : samples_to_process;
+        int samples_read;
+
+        if (input_is_float) {
+            // Direct read into float buffer
+            samples_read = fread (silence_mode ? prebuffer : inbuffer,
+                sizeof(float) * WaveHeader.NumChannels, samples_to_read, infile);
+        }
+        else {
+            // Read int16 and convert to float
+            int16_t *raw_buffer = malloc (samples_to_read * WaveHeader.BlockAlign);
+            samples_read = fread (raw_buffer, WaveHeader.BlockAlign, samples_to_read, infile);
+            float *dest = silence_mode ? prebuffer : inbuffer;
+            for (int k = 0; k < samples_read * WaveHeader.NumChannels; k++)
+                dest[k] = raw_buffer[k] * (1.0f / 32768.0f);
+            free (raw_buffer);
+        }
```

### 変更 3.8: 出力処理の変更 (行 458-482)

```diff
         if (samples_to_stretch) {
             int samples_generated;

             if (consecutive_silence_frames >= 3) {
-                samples_generated = stretch_samples (stretcher, inbuffer, samples_to_stretch, outbuffer, silence_ratio);
+                samples_generated = stretch_samples_float (stretcher, inbuffer, samples_to_stretch, outbuffer, silence_ratio);
                 used_silence_frames++;
             }
             else
-                samples_generated = stretch_samples (stretcher, inbuffer, samples_to_stretch, outbuffer, ratio);
+                samples_generated = stretch_samples_float (stretcher, inbuffer, samples_to_stretch, outbuffer, ratio);

             if (samples_generated) {
                 if (samples_generated > max_generated_stretch)
                     max_generated_stretch = samples_generated;

-                fwrite (outbuffer, WaveHeader.BlockAlign, samples_generated, outfile);
+                if (input_is_float) {
+                    fwrite (outbuffer, sizeof(float) * WaveHeader.NumChannels, samples_generated, outfile);
+                }
+                else {
+                    // Convert float → int16 for output
+                    int16_t *raw_out = malloc (samples_generated * WaveHeader.BlockAlign);
+                    for (int k = 0; k < samples_generated * WaveHeader.NumChannels; k++) {
+                        float v = outbuffer[k];
+                        if (v > 1.0f) v = 1.0f;
+                        if (v < -1.0f) v = -1.0f;
+                        raw_out[k] = (int16_t)(v * 32767.0f);
+                    }
+                    fwrite (raw_out, WaveHeader.BlockAlign, samples_generated, outfile);
+                    free (raw_out);
+                }
                 outsamples += samples_generated;
```

### 変更 3.9: フラッシュ処理の変更 (行 497-514)

```diff
     while (1) {
-        int samples_flushed = stretch_flush (stretcher, outbuffer);
+        int samples_flushed = stretch_flush_float (stretcher, outbuffer);

         if (!samples_flushed)
             break;

         if (samples_flushed > max_generated_flush)
             max_generated_flush = samples_flushed;

-        fwrite (outbuffer, WaveHeader.BlockAlign, samples_flushed, outfile);
+        if (input_is_float) {
+            fwrite (outbuffer, sizeof(float) * WaveHeader.NumChannels, samples_flushed, outfile);
+        }
+        else {
+            int16_t *raw_out = malloc (samples_flushed * WaveHeader.BlockAlign);
+            for (int k = 0; k < samples_flushed * WaveHeader.NumChannels; k++) {
+                float v = outbuffer[k];
+                if (v > 1.0f) v = 1.0f;
+                if (v < -1.0f) v = -1.0f;
+                raw_out[k] = (int16_t)(v * 32767.0f);
+            }
+            fwrite (raw_out, WaveHeader.BlockAlign, samples_flushed, outfile);
+            free (raw_out);
+        }
         outsamples += samples_flushed;
```

### 変更 3.10: 最終ヘッダ書き換え (行 524)

```diff
     rewind (outfile);
-    write_pcm_wav_header (outfile, outsamples, WaveHeader.NumChannels, 2, scaled_rate);
+    write_wav_header (outfile, outsamples, WaveHeader.NumChannels, input_bytes_per_sample,
+        scaled_rate, input_is_float);
     fclose (outfile);
```

### 変更 3.11: memcpy の変更 (行 487)

```diff
-            memcpy (inbuffer, prebuffer, samples_read * WaveHeader.BlockAlign);
+            memcpy (inbuffer, prebuffer, samples_read * WaveHeader.NumChannels * sizeof(float));
```

### 変更 3.12: write_pcm_wav_header() → write_wav_header() (行 546-577)

```diff
-static int write_pcm_wav_header (FILE *outfile, uint32_t num_samples, int num_channels,
-    int bytes_per_sample, uint32_t sample_rate)
+static int write_wav_header (FILE *outfile, uint32_t num_samples, int num_channels,
+    int bytes_per_sample, uint32_t sample_rate, int is_float)
 {
     RiffChunkHeader riffhdr;
     ChunkHeader datahdr, fmthdr;
     WaveHeader wavhdr;

     int wavhdrsize = 16;
     uint32_t total_data_bytes = num_samples * bytes_per_sample * num_channels;

     memset (&wavhdr, 0, sizeof (wavhdr));

-    wavhdr.FormatTag = WAVE_FORMAT_PCM;
+    wavhdr.FormatTag = is_float ? WAVE_FORMAT_IEEE_FLOAT : WAVE_FORMAT_PCM;
     wavhdr.NumChannels = num_channels;
     wavhdr.SampleRate = sample_rate;
     wavhdr.BytesPerSecond = sample_rate * num_channels * bytes_per_sample;
     wavhdr.BlockAlign = bytes_per_sample * num_channels;
     wavhdr.BitsPerSample = bytes_per_sample * 8;

     // ... (rest unchanged)
 }
```

### 変更 3.13: rms_level_dB() の float 化 (行 579-594)

```diff
-double rms_level_dB (int16_t *audio, int samples, int channels)
+static double rms_level_dB (const float *audio, int samples, int channels)
 {
     double rms_sum = 0.0;
     int i;

     if (channels == 1)
         for (i = 0; i < samples; ++i)
-            rms_sum += (double) audio [i] * audio [i];
+            rms_sum += (double) audio [i] * audio [i];
     else
         for (i = 0; i < samples; ++i) {
-            double average = (audio [i * 2] + audio [i * 2 + 1]) / 2.0;
+            double average = ((double) audio [i * 2] + audio [i * 2 + 1]) / 2.0;
             rms_sum += average * average;
         }

-    return log10 (rms_sum / samples / (32768.0 * 32767.0 * 0.5)) * 10.0;
+    // float32 samples are in [-1.0, 1.0], so no int16 scaling needed
+    // 0.5 factor accounts for RMS of a sine wave at full scale
+    return log10 (rms_sum / samples / 0.5) * 10.0;
 }
```

### 変更 3.14: 出力マクロの修正 (main.c:281 のエラーメッセージなど)

エラーメッセージの `"16-bit"` を状況に応じて適切に更新。

### 変更 3.15: stretch_output_capacity 呼び出しの変更

stretch_output_capacity のシグネチャは変わらない（サンプル数と ratio を受け取り int を返す）ため、
呼び出し側の変更は不要。

---

## 注意事項まとめ

1. **コンパイル確認**: 変更後は `build.sh rel` で `-Ofast` ビルドが通ることを確認
2. **テスト実行**: `test.sh` で全テストケースを再実行し、int16 互換ラッパーの正当性を検証
3. **float WAV テスト用ファイル**: 別途 float32 WAV ファイルを用意して動作確認
4. **エンディアン**: float32 WAV ファイルもリトルエンディアンが標準。本コードはリトルエンディアン環境を前提とする
