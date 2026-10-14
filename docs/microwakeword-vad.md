# ESPHome microWakeWord VAD model — origin, training, and how it's used

Researched 2026-10-14. Sources: `esphome/micro-wake-word-models` (main), `kahrendt/microWakeWord` (main @ 4665173),
`esphome/esphome` (dev @ 25d4b88, `esphome/components/micro_wake_word/`), and a direct parse of `models/v2/vad.tflite`.

## TL;DR

- **Who trained it:** Kevin Ahrendt (`kahrendt`), author of microWakeWord and of the ESPHome `micro_wake_word` component.
  It shipped with microWakeWord v2 in July 2024.
- **Training data:** there is **no published recipe** for the VAD specifically. No notebook, config, or release note
  describes it. The most likely sources are the speech and non-speech feature sets in Ahrendt's Hugging Face dataset,
  but that is **inferred, not documented** (see below).
- **Architecture:** a tiny streaming MixedNet (MixConv) CNN, the same family as the v2 wake-word models.
  It has about 14k int8 parameters, takes 3 × 40 micro-frontend features (30 ms) per inference, and outputs one
  sigmoid probability. The temporal receptive field is about 0.75 s.
- **How ESPHome uses it:** the VAD is a **gate**, applied with AND. A wake word is accepted only if the VAD's
  sliding-window **average** (5 × 30 ms, cutoff 0.5) says "speech" at that moment. Blocked detections are logged and
  do **not** reset the wake-word model, so the wake word can still fire a moment later if the VAD catches up.

---

## 1. Provenance

| Item | Value |
|---|---|
| Manifest | `models/v2/vad.json`: `"author": "Kevin Ahrendt"`, `"website": "https://www.kevinahrendt.com/"`, `"trained_languages": ["en"]` |
| Added in | micro-wake-word-models commit `9a72109c`, PR #7 "Add V2 Models", 2024-07-11. That is the only commit that touched `vad.tflite`/`vad.json`, so the model has never been retrained. |
| PR #7 description | "…includes two new models, one for VAD to reduce false acceptances and a 'Hey Mycroft' wake word model." |
| Originally distributed as | `https://github.com/kahrendt/microWakeWord/releases/download/v2.1_models/vad.json` (cited in esphome PR #7032). **That release no longer exists.** The repo now has only the `okay_nabu_20241226.3`, `stop` and `stop-20241016.2` releases/tags. |
| TFLite metadata | Converted with TF/MLIR; `min_runtime_version` 2.15.1; file size 34,328 bytes |
| License | Apache-2.0, which covers the weights per the models repo README |

## 2. Training data: what's known vs. inferred

**Documented:** nothing VAD-specific. I checked these places and found no VAD training details:

- `kahrendt/microWakeWord`: no VAD notebook, config, or doc. The only "VAD" in the code is `audio_utils.py`, which uses
  **webrtcvad** to trim silence from generated clips (`remove_silence`). That is unrelated to this model.
- No commits mention VAD, and no issues or PRs in either repo discuss VAD training.
- ESPHome docs and PRs #7032 / #7164 say only that the model "reduce[s] false activations from non-speech noises."

**Most likely data sources (inference):** Ahrendt's Hugging Face dataset
[`kahrendt/microwakeword`](https://huggingface.co/datasets/kahrendt/microwakeword) (CC-BY-NC-4.0) dates from the same
period (card references the June 8 2024 microWakeWord version). It contains pre-computed micro-frontend features split
into exactly the two classes a VAD needs:

| Subset | Sources | Natural VAD label |
|---|---|---|
| `speech_background` | LibriSpeech train-other, VOiCES | speech (positive) |
| `no_speech_background` | FMA-medium (music), FSD50K (sound events), WHAM! (noise) | non-speech (negative) |
| `dinner_party_background` | CHiME-6 (train/dev/eval), DiPCo | mixed speech and ambient sound, used for validation/test of false accepts |

The general microWakeWord data docs (`documentation/data_sources.md`) list the same corpora, plus Common Voice and BIRD
room impulse responses for augmentation. `trained_languages: ["en"]` fits LibriSpeech and VOiCES, which are both English.
**This mapping is an educated guess. Nothing confirms that the VAD was trained this way.** To confirm it, ask Kevin Ahrendt
or the OHF-Voice/microWakeWord maintainers directly.

## 3. Architecture (read from `vad.tflite`)

I parsed the flatbuffer directly (`tflite` Python package). The tensor names (`model/stream*/…`, `depthwise_conv2d_*`)
match microWakeWord's `mixednet.py`, which uses the streaming layers from Google's `kws_streaming`.

```
input  int8 [1, 3, 40]      3 new 10 ms feature slices × 40 mel-ish channels (scale 0.102, zp -128)
 └ Stream(Conv2D 5×1, 32 filters, stride 3) + ReLU          ring buffer 2 frames
 └ MixConv block 1: Stream(DepthwiseConv 5×1)  → Conv1×1 32→48 + BN + ReLU   ring buffer 4
 └ MixConv block 2: Stream(DepthwiseConv 8×1)  → Conv1×1 48→48 + BN + ReLU   ring buffer 7
 └ MixConv block 3: Stream(DepthwiseConv 13×1) → Conv1×1 48→48 + BN + ReLU   ring buffer 12
 └ Dense 48→1 → Logistic (sigmoid)
output uint8 [1, 1]         probability, scale 1/256
```

- **Equivalent training flags** (inferred from the graph): `--first_conv_filters 32 --first_conv_kernel_size 5 --stride 3
  --pointwise_filters "48,48,48" --mixconv_kernel_sizes "[5],[8],[13]" --residual_connection "0,0,0,0" --pooled 0`.
  `mixednet.py` sets ring buffer = kernel − 1, which is how the kernel sizes 5 / 8 / 13 were recovered.
  The middle kernel of 8 is unusual; the repo default is `[5],[9],[13],[21]`.
- **No residual ADDs and no pooling.** The final dense layer reads only the newest time step.
- **Parameters:** about 14.1k (first conv 6.4k; the three blocks 1.6k + 2.4k + 2.4k plus depthwise; dense 49). All
  int8, with int32 biases.
- **State:** 4 resource variables (`stream*/states`) hold the ring buffers between inferences. These need the
  `CALL_ONCE` / `VAR_HANDLE` / `READ_VARIABLE` / `ASSIGN_VARIABLE` ops that ESPHome registers.
- **Rate and receptive field:** one inference every 30 ms. The receptive field is
  1 + 4 + 7 + 12 = 24 strided steps → (24 − 1)·3 + 5 = 74 feature frames ≈ **0.74 s** of audio (plus the 30 ms window).
- **Manifest runtime config:** `probability_cutoff 0.5`, `sliding_window_size 5`, `feature_step_size 10`,
  `tensor_arena_size 16772` bytes, `minimum_esphome_version 2024.7.0`.

The VAD gets the same input features as the wake-word models: the 16 kHz micro_speech frontend with noise suppression
and AGC, 30 ms window, 10 ms step, quantized to int8 in `MicroWakeWord::generate_features_`.

## 4. How ESPHome combines VAD with wake-word detections

Code: `esphome/components/micro_wake_word/{__init__.py, micro_wake_word.cpp, streaming_model.cpp}`.

**Config / codegen (`__init__.py`)**
- `vad:` is optional. An empty `vad:` loads the default model `vad` (resolved to
  `github://esphome/micro-wake-word-models/models/v2/vad.json`). You can override `model`, `probability_cutoff` and
  `sliding_window_size`.
- If `vad:` is present, codegen defines `USE_MICRO_WAKE_WORD_VAD`. The model goes through the normal model-loading path.
  Because the manifest says `wake_word == "vad"`, codegen calls `add_vad_model(...)` for it instead of creating a
  `WakeWordModel`. The cutoff is quantized as `int(cutoff * 255)`, so 0.5 becomes 127.

**Inference (`update_model_probabilities_`)**
- For every feature slice, each wake-word model and the VAD run `perform_streaming_inference()`. Each model fills its
  `[1,3,40]` input in steps and calls `Invoke()` every 3 slices (30 ms). It then pushes the uint8 output into a
  ring buffer of `sliding_window_size` recent probabilities.

**Decision (`process_probabilities_`)**
```cpp
DetectionEvent vad_state = vad_model_->determine_detected();   // computed first, every pass
vad_state_ = vad_state.detected;                               // exposed via get_vad_state()
for each wake-word model with a new probability:
    ww = model->determine_detected();
    if (ww.detected) {
        if (vad_state.detected) { queue detection; wake loop; model->reset_probabilities(); }
        else                    { ww.blocked_by_vad = true; queue it (logged only); }
    }
```
- **VAD rule (`VADModel::determine_detected`):** `sum(last 5 probs) > cutoff_q * 5`. That is a sliding-window
  **average** over about 150 ms, compared with 0.5. If the VAD model is disabled or failed to load, it returns
  `detected = true` (fail-open), so it never blocks wake words.
- **Wake-word rule:** the same averaging test, plus a "cool-off". After a detection, `reset_probabilities()` zeroes the
  window and sets `ignore_windows_ = -100`. The model then needs 100 below-cutoff inferences before it can fire again.
  The VAD has no cool-off.
- **Blocked detections:** the main loop logs `"Wake word model predicts '<ww>', but VAD model doesn't."` and does not
  fire `on_wake_word_detected`. **The wake-word model is not reset**, so on the next 30 ms window it can be accepted if
  its average is still above cutoff and the VAD average has risen. In practice the VAD vetoes only while the window
  around the wake word looks like non-speech.
- `get_vad_state()` is commented as "intended for the voice assistant component to fetch VAD status". Only
  `micro_wake_word` itself uses it for gating.

**History**
- esphome PR #7032 (2024-07-11, ESPHome 2024.7.0) added microWakeWord v2: MixedNet ops, multiple models, and the VAD.
- esphome PR #7164 (2024-07-30) **fixed VAD being "nearly always detected"**. The code had compared the quantized uint8
  output with a float in 0–1. The PR also switched the VAD from the window **max** to the window **average** for
  stability.

**Docs discrepancy:** the esphome.io "Model JSON" section still says "VAD models use the maximum of all probabilities in
the sliding window." Since PR #7164 the code and the `vad:` option docs both use the **average**. The docs line is stale.

## 5. Open questions / next steps

- The exact training data, augmentation, and training steps for `vad.tflite` are unpublished. The `v2.1_models` release
  that originally hosted it has been removed. Ask upstream if this matters, e.g. on OHF-Voice/micro-wake-word or the
  ESPHome Discord.
- `trained_languages: ["en"]` means non-English speech may score lower and could veto valid wake words. If you see
  `"…but VAD model doesn't"` in logs, lower `vad: probability_cutoff` or remove `vad:`.

## Sources

- https://github.com/esphome/micro-wake-word-models (models/v2/vad.json, vad.tflite; PR #7)
- https://github.com/kahrendt/microWakeWord (README, `documentation/data_sources.md`, `microwakeword/mixednet.py`, `audio/audio_utils.py`)
- https://github.com/esphome/esphome/tree/dev/esphome/components/micro_wake_word
- https://github.com/esphome/esphome/pull/7032, https://github.com/esphome/esphome/pull/7164
- https://esphome.io/components/micro_wake_word/
- https://huggingface.co/datasets/kahrendt/microwakeword
