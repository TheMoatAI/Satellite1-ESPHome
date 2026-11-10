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

### 3. Error policy — `stt-no-text-recognized` is not an error

- `voice_assistant.yaml on_error` excludes `stt-no-text-recognized` (and
  `duplicate_wake_up_detected`) from taking the red error phase.
- Rationale: in continuous capture the mic hears fan noise / empty room all
  day and STT legitimately returns no text. Treating that as an error put the
  ring in a permanent blue→flashing-red error loop (observed 2026-11-10, the
  unit near the AI cluster's cooling fans). Household decision: noise with
  no words is normal operation; do NOT compensate with mic gain / noise
  suppression changes — keep those stock and fix server-side.
- Real faults (API loss, pipeline failures other than the exclusions) still
  take the red pulse.

## Units, configs, and identity mapping

Two per-room configs exist, keyed to WiFi (not to current physical room):

| Config | WiFi | Current device | Host | Notes |
|---|---|---|---|---|
| `config/satellite1-kitchen.yaml` | PrettyFlyForAWifi | `Satellite1 c2f84c` | `192.168.86.206` | physically in living room (2026-11-10); HA entity prefix `living_room_*` |
| `config/satellite1-livingroom.yaml` | Hummingbird | `Satellite1 c5a9ac` | `192.168.86.207` | physically in kitchen (2026-11-10); HA entity prefix `kitchen_*` |

Diffs between the two configs: WiFi credentials, `node_name`,
`friendly_name`. Everything behavioral is shared via `config/common/`.

Entity-ID prefixes in HA are permanent from first enrollment and do not track
physical location — the units have been swapped physically during bring-up.
Treat the HA config-entry host mapping as authoritative for unit identity.

## Known issues / open threads

- **c2f84c mic capture quality** (2026-11-10): that unit delivers
  near-speech-free audio to whisper even when the user yells (VAD drops
  ~100% of chunks; occasional 1s fragments transcribe as hallucinations).
  Reboots and firmware re-flash did not change it; the other unit works in
  the same conditions. Suspect hardware (half-seated Core/Hat or mic
  connection — units are self-assembled). Next step: power down, reseat,
  retest; then the scripted hearing-accuracy harness (HomeAutomation) decides
  if it needs RMA.
- Transcripts of pure room noise occasionally hallucinate sentences
  containing "Jinx" and open conversations server-side; the gate's echo/dup
  suppression catches exact replays, but noise-hallucination filtering is a
  server-side (gate/orchestrator) concern, not firmware.

## Change log

- 2026-11-10: Added `stt-no-text-recognized` exclusion to `on_error` (stops
  blue→flashing-red error loop in noisy rooms). Added per-room configs
  (kitchen/livingroom), conversation lighting, free-speech capture switch
  (earlier same-day bring-up work).
