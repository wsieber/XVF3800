# Speech-to-text wake check (`wake_verify`)

A second opinion on every wake word detection. When microWakeWord fires, the device sends the last 1.8 s of microphone
audio to Home Assistant's speech-to-text API and checks that the transcript actually contains a wake word. A false
fire from TV dialogue is dropped without a sound, and a wake that the wrong model caught (Okay Nabu firing on "Okay
Batman") goes ahead with the word that was really said.

It runs entirely in the firmware (`components/wake_verify/`). There is no Home Assistant add-on or custom integration:
the device calls Home Assistant's built-in `/api/stt/<engine>` REST endpoint with a long-lived access token, using the
same speech-to-text engine the Assist pipelines use.

## How a wake goes through it

1. **Detection.** micro_wake_word fires and `on_wake_word_detected` runs. The 2 s double-fire guard, the mute check and
   a ringing timer are handled there at once, as before. Everything else is handed to `wake_verify.check`.
2. **Skip list.** A word in `skip_wake_words` (Stop) passes straight through, so Stop always acts instantly.
3. **Recording.** The component keeps a rolling 1.8 s buffer (in PSRAM) of the same microphone channel micro_wake_word
   listens to. After the detection it waits a further 200 ms (`post_roll`) so the end of the word is in the clip.
4. **Level.** A quiet clip is boosted to a −6 dBFS peak, by at most 32×. The Satellite1's clips peak around 200–500
   (out of 32768) and get the full 32×; the XVF3800's are already at full scale and are left alone.
5. **Transcription.** The clip goes to `POST <ha_url>/api/stt/<stt_engine>` as 16 kHz 16-bit mono PCM, from a separate
   task so the main loop keeps running. The local OpenAI-compatible engine answers in 130–340 ms.
6. **Matching.** The transcript is normalised (lower case, punctuation dropped, "ok" read as "okay") and every run of
   neighbouring words is compared with each enabled wake word, spaces removed, by edit distance. The score is
   1 − distance / length, so "Okay, Abu." scores 0.88 against Okay Nabu and "Elfrid." 0.67 against Alfred.
7. **Verdict**, with `match_threshold` 0.65:
   - **ACCEPT**: the detected word scores at least 0.65. The wake goes ahead.
   - **REROUTE**: the detected word doesn't, but another enabled wake word does. The wake goes ahead with that word,
     so Home Assistant runs that word's pipeline and the greeting matches it.
   - **REJECT**: no enabled wake word matches. Nothing happens: no greeting, no beam lock, no stop.
8. **Action.** The `start_wake` script runs with the verdict's word. If the assistant is running it is stopped; if an
   announcement is playing it is stopped; otherwise the greeting plays and the voice assistant starts.

"Enabled" means the words in Home Assistant's active wake word slots at that moment. A model compiled in but not in a
slot is neither a match nor a reroute target.

### Two models on one utterance

Two models often fire on the same speech (Alfred on "Frederick", Frederick on "Alfred"). A detection that arrives while
another is waiting for its verdict joins that check instead of starting its own: the check already weighs every enabled
word, so the wake goes ahead once, with the word that was said. The log shows `'Alfred' joins the check of
'Frederick'`.

### Stops

A wake word heard while the assistant is listening or replying, or while an announcement plays, normally stops it.
That stop also goes through the check, so a false fire in the middle of a conversation no longer cuts it off. Stop
itself skips the check, and a ringing timer is silenced by any wake word without it.

### When the check can't run

The check never makes the device deaf. The detected word goes ahead unchecked if:

- the mode is `off`, or there is no token;
- speech-to-text fails or returns an error (`'Frederick' not checked, waking anyway: …`);
- no verdict arrives within `decision_timeout` (1.5 s) (`No verdict … waking unchecked`);
- a new detection arrives while an earlier check is still running but no longer waiting (log mode, or a check that has
  already timed out).

So while Home Assistant or the speech-to-text engine is down, false fires wake the device as they did before the check.

## Modes

`wake_verify_mode`, a substitution in `xvf3800/common.yaml`, defaults to `off`; a device file overrides it.

| Mode | Behaviour |
|---|---|
| `off` | No check. `on_wake` fires at once with the detected word. |
| `log` | `on_wake` fires at once with the detected word; the check runs in the background and is only logged, as `WOULD ACCEPT`, `WOULD REROUTE` or `WOULD REJECT`. Use it to see what the check would do before trusting it. |
| `reroute` | The wake waits for the verdict, as described above. |

Both XVF3800s run `reroute`.

## Latency

About 0.4 s per wake: the 200 ms post-roll plus the round trip to speech-to-text (median 185 ms over the first 36
checks). The greeting already takes about a second and the assistant only listens once it has finished, so in use the
delay shows only as a slightly later greeting. Stopping a reply with a wake word takes the same extra 0.4 s; Stop does
not.

## Reading the log

Each check logs one line at INFO:

```
ACCEPT 'Frederick' score=1.00 heard='Frederick.' (166 ms, 1.8s clip, peak 271 x32.0) [6 checked: 0 rerouted, 0 rejected]
REROUTE 'Okay Nabu' -> 'Okay Batman' score=0.50 heard='Okay, Batman.' (254 ms, 1.8s clip, peak 32768 x1.0) [...]
REJECT 'Computer' score=0.00 heard='India.' (264 ms, 1.8s clip, peak 327 x32.0) [...]
```

- `score`: how well the **detected** word matched the transcript.
- `heard`: the transcript.
- `(… ms)`: the speech-to-text round trip only.
- `peak N xG`: the clip's loudest sample before boosting, and the boost applied.
- The counters run since boot.

## Setup

1. **Token.** In Home Assistant, open your profile → Security → Long-lived access tokens, create one, and put it in
   `secrets.yaml` as `ha_token` (see `secrets.yaml.example`).
2. **Engine.** `wake_verify_stt_engine` in `xvf3800/common.yaml` is the speech-to-text entity to use. The local
   OpenAI-compatible engine takes about 0.1 s of processing; Home Assistant Cloud took 0.5–0.9 s and needs the
   language `en-US`. `ha_url` is Home Assistant's address.
3. **Mode.** Set `wake_verify_mode: reroute` (or `log` to try it first) in the device file's `substitutions`.
4. Flash, and watch the log for the lines above.

### Options

| Option | Default | Meaning |
|---|---|---|
| `microphone` | required | The mic channel to record; use the one micro_wake_word listens to. Must be 16 kHz. |
| `micro_wake_word` | required | The micro_wake_word instance, for the list of enabled words. |
| `mode` | `log` | `off`, `log` or `reroute`. |
| `ha_url`, `ha_token`, `stt_engine` | required | Where and how to transcribe. |
| `language` | `en` | Sent in the `X-Speech-Content` header. |
| `buffer_duration` | 1800ms | Length of the clip (500 ms–4 s). |
| `post_roll` | 200ms | Wait after the detection before taking the clip. |
| `timeout` | 3s | The HTTP request's own limit. |
| `decision_timeout` | 1500ms | In `reroute`, how long a wake waits for its verdict. |
| `match_threshold` | 0.75 | Minimum score to accept or reroute; the XVF3800s use 0.65. |
| `skip_wake_words` | none | Words that bypass the check and are never reroute targets. |
| `on_wake` | required | Runs with `wake_word`, the word to use. Here it runs the `start_wake` script. |

## Tuning notes (October 2026)

- **Threshold 0.65.** At 0.75, genuine wakes heard as "Okay, now Wu." and "Redruck." (both 0.67) were rejected.
  Speech-to-text often splits "Nabu" into "now" plus another word ("Okay, now who?", 0.62), and those are still
  rejected at 0.65; nothing safe below 0.62 separates them from ordinary "Okay, now …" speech.
- **Lower model cutoffs.** With the check catching false fires, the models can run looser than the 0.98 that kept TV
  quiet without it. Both units run Alfred at 0.96 and Frederick and Computer (Computerv3) at 0.92: the user's own calls
  scored 0.92–0.96, so 0.98 missed many of them, and the extra false fires (Computer four in a night, Alfred a couple a
  day) were all rejected.
- **Speech-to-text inventions.** On noise the local engine sometimes returns text that was never said
  ("Transcribe audio to text (language: en).", "是的。"). These score near zero and are rejected.
- **Rejected clips are labelled false triggers** from the real room, useful as negatives for the next training run.
