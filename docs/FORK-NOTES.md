# TheMoatAI Fork Notes — Satellite1-ESPHome

Status: living document. Update this file in the same commit as any `config/`
change — these notes are the household's reference for how our firmware
diverges from FutureProofHomes stock, so future sessions don't re-derive
behavior from vendor docs (which describe stock only).

Operational playbook (flashing, diagnostics, LED table) lives in
`.agents/skills/satellite1-speakers/SKILL.md`; this document carries the
why and the change log.

## Why this fork exists

The household runs a private voice assistant ("Jinx") on the HomeAutomation
stack: Satellite1 speakers → Home Assistant (voice-router) → Wyoming
whisper/piper + custom voice-tts (kokoro→RVC) → the Jinx attention gate
(`voice-gate`) → OpenClaw brain on the local vLLM fleet. Stock firmware
behaviors that fight that design get changed here. This fork is a household
gadget, not a product — no copyright concerns apply to personal-use
customizations.

## Fork divergences from stock

### 1. Free-speech capture switch (no-wake-word continuous capture)

- `config/common/voice_assistant.yaml`: a `free_speech_capture` switch
  (`RESTORE_DEFAULT_OFF`, HA entity per unit) opts the unit into
  `voice_assistant.start_continuous` at client-connect — no wake word, mic
  always feeding the pipeline.
- Rationale: the household wants natural conversation ("when we say Jinx she
  knows we're talking to her") without a mandatory "Hey Jarvis" prefix. The
  Jinx attention gate (in HomeAutomation) decides which utterances are
  addressed; capture should be dumb and continuous.
- The name-gate mishear tradeoff ("Jinx" → whisper variants) is handled
  server-side in the gate's name patterns, not in firmware.

### 2. Conversation-state lighting

- `config/common/led_ring.yaml`: while the HA switch `conversation_light` is
  on (flipped by the voice-gate for the duration of an active conversation),
  the ring shows solid blue = listening/waiting, solid green = thinking,
  solid orange = talking, and stays dark while waiting for speech otherwise
  (quiet free-speech mode). Idle-blue while `conversation_light` is on reads
  as "still in conversation".
- Pulsing was deliberately replaced with solid colors ("a little
  overwhelming").
- The three color scripts pass `effect: "none"` explicitly: `light.turn_on`
  without an `effect:` leaves any running effect (e.g. the white volume
  level display from a recent volume-button press) running forever — the
  volume indicator only cleared when a restore path ended in
  `light.turn_off`. Fixed 2026-11-10.

### 3. Error policy — `stt-no-text-recognized` is not an error

- `voice_assistant.yaml on_error` excludes `stt-no-text-recognized` (and
  `duplicate_wake_up_detected`) from taking the red error phase. The
  exclusion is global, not continuous-capture-only: in wake-word mode a
  false wake that yields no text also stays silent instead of flashing red.
  We consider that correct behavior for this household.
- Rationale: in continuous capture the mic hears fan noise / empty room all
  day and STT legitimately returns no text. Treating that as an error put the
  ring in a permanent blue→flashing-red error loop (observed 2026-11-10, the
  unit near the AI cluster's cooling fans). Household decision: noise with
  no words is normal operation; do NOT compensate with mic gain / noise
  suppression changes — keep those stock and fix server-side.
- Caveat: a dead or dirty mic manifests as exactly `stt-no-text-recognized`,
  so that hardware-fault class no longer lights red on its own. If a unit
  stops hearing people, check whisper's VAD/transcription output and the
  hearing-accuracy harness rather than trusting the ring.

### 4. Voice assistant samples mic channel per unit (`mic_channels` substitution)

- `voice_assistant.yaml` microphone blocks use `channels: ${mic_channels}`.
  Default is `"1"` (set in `satellite1.yaml`); per-unit yamls override.
  micro_wake_word has always used channel 1 and detects wake words on both
  units. Stock FPH v0.2.1 ships VA on ch0 / MWW on ch1.
- Rationale: side-by-side calibration (scripted phrases cross-played between
  units, transcription scored via the gate's utterance buffer) found the mic
  channels are NOT equivalent and differ per unit:
  - **c2f84c: ch0 near-dead, ch1 good** — ch0 transcribed nothing at any
    volume (even to a human yelling); ch1 passes scripted phrases at 0.40.
  - **c5a9ac: ch1 unusable, ch0 works** — ch0 passed capture at 0.60
    (2026-11-10 runs); flashed ch1 on 2026-11-11 heard VAD-level noise but
    whisper returned "No text recognized" on every pipeline cycle (visible
    in encrypted device logs) — the channel hears sound but no intelligible
    speech. `satellite1-livingroom.yaml` therefore pins `mic_channels: "0"`.
- Calibration data: `HomeAutomation/nocommit/hearing/` (ladder + validation
  JSON per arm; run via `just voice-hearing-test`).
- Reflash caveat: stale `config/.esphome/` build dirs can carry the old
  channel into a rebuilt image — clear `.esphome/` and verify the generated
  `main.cpp` shows `add_channel(<expected>)` before OTA. Also: esphome may
  serve a *validated config cache* (`config/.esphome/storage/*.json`) and
  skip re-reading edited yaml — delete `config/.esphome/storage/` after
  config edits if a run logs "skipping validation".

## Units, configs, and identity mapping

Two per-room configs exist, keyed to WiFi (not to current physical room):

| Config | WiFi | Current device | Host | Notes |
|---|---|---|---|---|
| `config/satellite1-kitchen.yaml` | PrettyFlyForAWifi | `Satellite1 c2f84c` | `192.168.86.206` | physically in living room (2026-11-10); HA entity prefix `living_room_*` |
| `config/satellite1-livingroom.yaml` | Hummingbird | `Satellite1 c5a9ac` | `192.168.86.207` | physically in kitchen (2026-11-10); HA entity prefix `kitchen_*` |

Diffs between the two configs: WiFi credentials, `node_name`,
`friendly_name`, and `mic_channels` (c2f84c=kitchen yaml→ch1,
c5a9ac=livingroom yaml→ch0, see §4). Everything else is shared via
`config/common/`.

Entity-ID prefixes in HA are permanent from first enrollment and do not track
physical location — the units have been swapped physically during bring-up.
Treat the HA config-entry host mapping as authoritative for unit identity.

## Known issues / open threads

- **c2f84c mic capture quality — RESOLVED as a channel issue** (updated
  2026-11-11): the earlier "suspect hardware / RMA" theory is superseded —
  the unit transcribes scripted phrases well on mic channel 1 (minimal
  passing volume 0.40 in calibration) and channel 0 is near-deaf on both
  units. Fixed by sampling ch1 for the voice assistant (divergence §4);
  hardware reseat/RMA no longer indicated unless ch1 degrades too.
- Transcripts of pure room noise occasionally hallucinate sentences
  containing "Jinx" and open conversations server-side; the gate's echo/dup
  suppression catches exact replays, but noise-hallucination filtering is a
  server-side (gate/orchestrator) concern, not firmware.

## Change log

- 2026-11-11: Voice assistant mic channel 0 → 1 (ch0 near-deaf on both
  units; calibration data in HomeAutomation nocommit/hearing). Mute checks
  hoisted above the conversation phase handlers in the LED dispatch (mute
  dots now win over conversation colors; side effect: a muted unit no
  longer shows the ticking-timer countdown — ringing timers still show).
- 2026-11-10 (2): Conversation lighting scripts now pass `effect: "none"` —
  the white volume-level display never cleared after adjusting volume
  because the restore path turned the light on with a color but kept the
  previous effect running.
- 2026-11-10: Added `stt-no-text-recognized` exclusion to `on_error` (stops
  blue→flashing-red error loop in noisy rooms). Added per-room configs
  (kitchen/livingroom), conversation lighting, free-speech capture switch
  (earlier same-day bring-up work).
