# ReSpeaker XVF3800 → ESPHome voice satellite

ESPHome firmware config for the **Seeed ReSpeaker XVF3800 4-mic array with onboard XIAO ESP32-S3**. It turns the board into a
Home Assistant Assist voice satellite, replacing the stock firmware.

It builds on [formatBCE/Respeaker-XVF3800-ESPHome-integration](https://github.com/formatBCE/Respeaker-XVF3800-ESPHome-integration)
(pinned to commit `e9b1137`) and adds:

- **Time-of-day spoken wake greetings** ("Good morning / afternoon / evening") instead of the chime, ported from a
  FutureProofHomes Satellite1 setup. There is a male and a female voice set, chosen per wake word.
- **Custom wake words**: "Frederick" (v2) and "Ok Emily", loaded from
  [wsieber/microwakewords](https://github.com/wsieber/microwakewords), alongside the upstream ones.
- **VAD gate raised to 0.5.** Upstream sets the microWakeWord VAD cutoff to 0.05, which lets almost any sound through.
- **`micro_wake_word` logging at DEBUG**, so detections and VAD blocks are visible in the logs.
- **One shared config for many units.** Each device is a 6-line file.

Built and running on two units with **ESPHome 2026.9.1** (Python 3.13). The XMOS DSP firmware is **1.0.7**, which the
config flashes automatically.

## Layout

```
respeaker-xvf3800.yaml     # unit 1: name + friendly_name, includes xvf3800/common.yaml
xvf3800-2.yaml             # unit 2: same, different name
xvf3800/common.yaml        # everything shared: upstream packages, Wi-Fi/OTA, greetings, wake words, VAD, logging
xvf3800/voice-assistant.yaml  # local copy of upstream packages/voice-assistant.yaml with the greeting handler
sounds/                    # greeting audio, embedded into the firmware at build time
secrets.yaml.example       # copy to secrets.yaml
docs/microwakeword-vad.md  # how the microWakeWord VAD model works and how ESPHome uses it
```

## Build and flash a new unit

1. **Install ESPHome**:
   ```sh
   python3 -m venv .venv
   .venv/bin/pip install "esphome==2026.9.1"
   ```
2. **Add secrets**: `cp secrets.yaml.example secrets.yaml` and fill in Wi-Fi and an OTA password.
3. **Create the device file**: copy `respeaker-xvf3800.yaml` to a new file and give it a unique `name` (hostname) and
   `friendly_name`. Don't reuse an existing name; duplicate names collide in Home Assistant.
   ```yaml
   substitutions:
     name: xvf3800-kitchen
     friendly_name: ReSpeaker XVF3800 Kitchen

   packages:
     common: !include xvf3800/common.yaml
   ```
4. **First flash over USB.** Plug the board's USB-C into the computer and run:
   ```sh
   .venv/bin/esphome run xvf3800-kitchen.yaml      # pick the serial port when asked
   ```
   This replaces the stock firmware, and there's no going back without a backup of it. Take one first with `esptool`
   if you might want it.
5. **XMOS update on first boot.** The ESP32 updates the XMOS DSP firmware over I2C to the bundled 1.0.7. This takes
   about 45 s; leave the board powered until it finishes. Beamforming, echo cancellation and noise
   suppression run on the XMOS chip.
6. **Add to Home Assistant.** The device shows up under *Settings → Devices & services* as a discovered ESPHome
   device. Add it, then pick the Assist pipeline and wake word on the device page.
7. **Later updates** go over the network:
   ```sh
   .venv/bin/esphome run xvf3800-kitchen.yaml --device xvf3800-kitchen.local
   ```

Every unit builds from `xvf3800/common.yaml`. After changing that file or anything in `sounds/`, reflash **every** unit.

## Customising

### Wake words
`micro_wake_word.models` in `xvf3800/common.yaml` adds models on top of the upstream defaults (Okay Nabu, Hey Jarvis,
and others). The active wake word is chosen in Home Assistant.

One quirk: the `.json` manifests for wake words trained with the Tater trainer carry extra keys, which ESPHome rejects
with a misleading `[micro_wake_word] is an invalid option` error. Use stripped `*esp32.json` copies, like
`frederickv2esp32.json`.

To tune a model per deployment, add `probability_cutoff` on the model entry. A higher value means fewer false wakes and
more missed wakes.

### Greetings
- `sounds/morning|afternoon|evening.mp3` is the male set and `*_f.mp3` is the female set. Midnight to noon is morning,
  noon to 6 pm is afternoon, and 6 pm to midnight is evening. Until the clock syncs after boot, the device plays
  `sounds/yo.flac` instead.
- `female_wake_words` in `common.yaml` lists the wake words, by the manifest's `wake_word` phrase, that get the female
  set. Every other wake word gets the male set.
- The current files are 24 kHz stereo 160 kbps CBR MP3. The male set is levelled to −14 dB speech RMS and the female
  set to −12 dB, with peaks kept under −1 dBFS so the greetings sit with Home Assistant's TTS replies. Re-level new
  recordings to match.
- The greeting handler waits for the greeting to finish before opening the mic. Otherwise STT transcribes "Good
  morning." as the request.

### VAD gate
```yaml
micro_wake_word:
  vad:
    probability_cutoff: 0.5
```
A wake word is accepted only if the VAD model also thinks someone is talking. 0.5 is the model's own default.

The VAD only filters out **non-speech** false wakes such as music, clatter or a dog. Speech-like false wakes, from TV
dialogue for example, pass the VAD whatever the cutoff; raise the wake word's own `probability_cutoff` for those. If
real wakes get blocked, the log shows `Wake word model predicts '…', but VAD model doesn't.`, and lowering the cutoff
(try 0.3) fixes it. See [docs/microwakeword-vad.md](docs/microwakeword-vad.md).

### Logging
`common.yaml` compiles logging in at DEBUG but runs at INFO, with only `micro_wake_word` raised to DEBUG. To watch
detections:
```sh
.venv/bin/esphome logs xvf3800-kitchen.yaml --device xvf3800-kitchen.local
```

## Updating the upstream package

`respeaker_ref` in `common.yaml` pins the upstream commit. `xvf3800/voice-assistant.yaml` is a copy of that commit's
`packages/voice-assistant.yaml` with the greeting change applied. When you bump the ref:

1. Copy the new upstream `packages/voice-assistant.yaml` over the local one.
2. Re-apply the greeting block in `on_wake_word_detected`. It's the `sound_file: !lambda` that picks
   `wake_good_<part>[_f]_sound`, followed by the wait for the announcement to finish.

The file is copied rather than patched because ESPHome merges packages by **concatenating** action lists. A
package's `on_wake_word_detected` can't be replaced: `!remove` plus a redefinition leaves both handlers running.

## Credits and license

This repository is released under the [MIT License](LICENSE).

- Base firmware: [formatBCE/Respeaker-XVF3800-ESPHome-integration](https://github.com/formatBCE/Respeaker-XVF3800-ESPHome-integration).
  `xvf3800/voice-assistant.yaml` is a modified copy of its `packages/voice-assistant.yaml`, used under the MIT license.
  See [xvf3800/LICENSE.upstream.md](xvf3800/LICENSE.upstream.md).
- Wake word engine: [microWakeWord](https://github.com/kahrendt/microWakeWord) and the ESPHome
  [`micro_wake_word`](https://esphome.io/components/micro_wake_word/) component.
