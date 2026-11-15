---
name: satellite1-speakers
description: Playbook for operating, flashing, and diagnosing the Farr household's Satellite1 speakers from the TheMoatAI fork — OTA flashing per-room configs, LED status meanings, HA entity controls, device log capture, and firmware-change conventions. Load whenever flashing, configuring, diagnosing, or modifying Satellite1 firmware or speaker behavior.
---

# Satellite1 Speakers — Fork Operations Playbook

The household's two FutureProofHomes Satellite1 speakers run firmware we build
from **our fork**: `/Users/chrisfarr/code/Satellite1-ESPHome` (remote
`TheMoatAI/Satellite1-ESPHome`). The fork is the source of truth for how the
speaker interface works — when speaker behavior surprises you, read
`config/common/*.yaml` there before consulting vendor docs. Full fork context
and divergence notes live in `docs/FORK-NOTES.md` in that repo.

## Ground rules

- **Never push to `FutureProofHomes/*`** — there is no `upstream` remote and it
  must stay that way (an accidental PR against the parent repo happened once).
  Push branches to `TheMoatAI/Satellite1-ESPHome`; when using `gh pr create`
  from this repo pass `--repo TheMoatAI/Satellite1-ESPHome` explicitly.
- **Firmware changes ship with docs**: any change to `config/` must update
  `docs/FORK-NOTES.md` in the same commit — the notes are the reference we
  consult instead of re-deriving behavior each session.
- **Vendor docs describe stock firmware.** Our fork changes lighting, capture
  behavior, and the error policy. For LED meanings and state behavior, the
  fork's `config/common/led_ring.yaml` and `config/common/voice_assistant.yaml`
  are ground truth.

## The units and their identities (verify, don't assume)

Config names are keyed to the **WiFi network the unit was flashed with**, not
the room it currently sits in — the physical units have been swapped between
rooms. Verify current location with the user before assuming.

| HA device | Host IP | Flashed config | WiFi | Entity-ID prefix | As of 2026-11-10 |
|---|---|---|---|---|---|
| `Satellite1 c2f84c` | `192.168.86.206` | `config/satellite1-kitchen.yaml` | PrettyFlyForAWifi | `living_room_satellite1_c2f84c_*` (original enrollment) | physically in living room |
| `Satellite1 c5a9ac` | `192.168.86.207` | `config/satellite1-livingroom.yaml` | Hummingbird | `kitchen_satellite1_c5a9ac_*` (original enrollment) | physically in kitchen |

Entity-ID prefixes are permanent (assigned at first enrollment) and do **not**
track physical location. The HA→device host mapping is authoritative for "which
unit is which": read it from HA config entries when in doubt.

## OTA flashing

The venv lives in the fork repo: `Satellite1-ESPHome/.venv` (esphome 2026.7.3
from PyPI — stock CLI; the fork's custom pieces arrive via
`external_components` pinned in `config/common/components.external.yaml`).
Stock FPH firmware builds report 2025.4.0.

```sh
cd /Users/chrisfarr/code/Satellite1-ESPHome
.venv/bin/esphome run config/satellite1-kitchen.yaml --device 192.168.86.206
```

- OTA over WiFi; **no USB needed**. Compile is ~10–15 min cold (ESP-IDF) and
  near-instant warm; OTA upload ~10s. Build dir: `config/.esphome/build/`.
- Redirect output to a file and tail it — never pipe `esphome` through `head`
  (SIGPIPE kills the build), and macOS has no `timeout` command.
- The `esphome run` invocation can outlive a tool-call timeout while still
  having completed fine: check the log for `OTA successful` before retrying.
  Killing mid-upload is safe — the device keeps the old firmware.
- After a flash, verify the `Free speech capture` switch state in HA — the
  switch restores from the device's flash prefs (`RESTORE_DEFAULT_OFF`), not
  from HA, so a fresh flash can drop it.
- **One unit's capture degrades while the other is fine? Check per-unit
  build parity first.** The per-room configs pin DIFFERENT mic channels
  (c2f84c=ch1 via kitchen yaml, c5a9ac=ch0 via livingroom yaml — see
  satellite1.yaml), and a stale per-unit build samples the wrong one.
  Node-name trap: satellite1-livingroom.yaml shares node name
  `satellite1`, so its build dir is `config/.esphome/build/satellite1/`
  — not `satellite1-livingroom/` (`ls config/.esphome/build/` shows
  `satellite1` and `satellite1-kitchen`). Check the binary timestamp,
  not the dir name. 2026-11-14: c5a9ac transcribed nothing at 1ft (yet
  heard its own loud playback — low-gain signature); the ch0-pinned
  livingroom build existed in `config/.esphome/build/satellite1/` but
  the unit was still running the pre-pin ch1 build — the shared node
  name made the stale build look current. `esphome run
  config/satellite1-livingroom.yaml --device 192.168.86.207` fixed it —
  exact transcript at volume 0.60 post-flash. Validate with
  `just voice-hearing-test` from the HomeAutomation repo on the Studio
  (it runs `voice_router/scripts/hearing_calibration.py`) in the
  direction that failed.
- If an edit to an *included* file (`config/common/*`) isn't picked up, delete
  the `.esphome/` cache dir and rebuild — the validation cache can pin stale
  included config. **Cost of that deletion (2026-11-14): a FULL ESP-IDF
  rebuild (~15 min, 2100+ units), and the wake-word models re-download from
  S3 — which can fail with `Connection reset by peer` and kill the
  process.** If S3 resets, pre-seed the cache yourself:
  `config/.esphome/micro_wake_word/<sha256(url)[:8]>/manifest.json` + the
  model file named in the manifest (curl reaches S3 fine even when the
  esphome client doesn't; key = sha256 of the model URL, first 8 hex chars).
- WiFi model-manifest checks may warn `Connection reset by peer` and use cached
  wake-word files — harmless.

## Announce-path traps (2026-11-14, hard-won)

- **NEVER call `voice_assistant.stop` from media_player `on_announcement`.**
  In the assist-satellite announce flow the VA itself streams the TTS
  response into the announcement (VA `STREAMING_RESPONSE` → speaker_source
  `ANNOUNCING`); stopping the VA aborts playback in ~20ms — ANNOUNCING →
  IDLE, decode never runs, **no audio** (verified in device logs). This was
  the intermittent "announce wedges silently" failure. Echo suppression for
  the announcing unit stays on the gate's playing-window rule instead.
- **HA sees NO state change during announcements** — the device publishes
  `MEDIA_PLAYER_STATE_ANNOUNCING` (4) but HA's esphome media_player mapper
  has no ANNOUNCING entry, so the entity stays `idle` and an HA automation
  keyed on media_player state cannot detect replies. Key cross-unit logic
  on the device (`homeassistant.service` calls) or the gate instead.
- Cross-unit echo suppression: the announcing unit flips the sibling's
  `capture_hold` switch via `homeassistant.service` in `on_announcement`;
  its `on_state` keeper (`resume_capture_after_playback` in
  voice_assistant.yaml) reopens capture 400ms after playback ends and
  releases the sibling. Sibling entity ids live in the per-room yamls as
  `sibling_capture_hold`.
- New fork switches register with room-prefixed entity ids — but HA may
  merge device name + area oddly (c2f84c's capture_hold first registered as
  `switch.living_room_kitchen_satellite1_c2f84c_capture_hold`). Verify the
  actual id after first registration and rename in the HA entity registry
  (`config/entity_registry/update` with `new_entity_id`) rather than
  reflashing.

## Device status and diagnostics

### LED meanings (fork ground truth: `config/common/led_ring.yaml`)

| Ring state | Meaning |
|---|---|
| Solid blue | listening / waiting for the next line (conversation active, or idle while `conversation_light` is on) |
| Solid green | thinking |
| Solid orange | talking (replying) |
| **Flashing red** | **error phase** — a pipeline error was accepted as real (e.g. API/pipeline failure). NOT a mute indicator. |
| **Red Twinkle** | no HA connection / assistant not ready — seen during HA restarts; don't misread as a pipeline error. |
| Red LEDs at specific positions | mute indicators (mic/speaker muted) |

- `stt-no-text-recognized` is **benign** in continuous capture (fan noise, empty
  room) and is excluded from the error phase in `voice_assistant.yaml
  on_error`. If red flashing reappears during silence, check which error code
  is being accepted before blaming hardware.
- During "quiet free-speech mode" the ring stays dark while waiting for speech;
  blue appears when the gate has the unit in an active conversation (via the
  `conversation_light` switch) or idle-blue while that switch is on.

### Device logs (the fastest truth)

Device logs require the API encryption key that Home Assistant set at adoption:

1. Pull the key from HA storage:
   `docker exec homeai-voice-router python3 -c "import json; print([e['data']['noise_psk'] for e in json.load(open('/config/.storage/core.config_entries'))['data']['entries'] if e.get('domain')=='esphome' and 'c2f84c' in e['title']][0])"`
2. Temporarily put it under `api: encryption: key:` in
   `config/common/home_assistant.yaml`, run
   `.venv/bin/esphome logs config/satellite1-kitchen.yaml --device <ip>`,
   then **revert the file — never commit the key**.
3. If the CLI still reports `RequiresEncryptionAPIError` after patching, delete
   `.esphome/` and retry (stale validation cache).

### HA-side controls (no reflash needed)

- Reboot a unit: press `button.satellite1_XXX_restart_sat1` (verified working
  2026-11-14 on both units after a wedge: unavailable → idle in ~30-60s;
  Red Twinkle during reconnect is benign, not an error).
- Continuous capture: `switch.*_free_speech_capture` (on = always-listening).
- Conversation lighting: `switch.*_conversation_light` (the voice-gate flips
  the kitchen unit's switch; see `homeai-voice-gate` env in HomeAutomation's
  docker-compose).
- Mute state: `switch.*_mute_microphones` + `binary_sensor.*_button_left_hw_mute`.

## Firmware-change conventions

- Per-room configs (`config/satellite1-livingroom.yaml`,
  `config/satellite1-kitchen.yaml`) differ only in WiFi credentials, node name,
  and friendly name — shared behavior lives in `config/common/*.yaml`.
- Follow the repo's batch-branch workflow: branch `feat/...`, commit, push,
  PR to `TheMoatAI/Satellite1-ESPHome`.
- Whisper STT tuning (VAD threshold, model, initial prompt) does **not** live
  here — it is the `wyoming-whisper` command line in HomeAutomation's
  `docker-compose.yml`. Mic gain / noise suppression knobs live in
  `config/common/voice_assistant.yaml` here, but the household decision (2026-11-10)
  is to leave gain/suppression stock and treat noise-with-no-words as normal.

## Voice-tier testing with the pair (speakers pointed at each other)

The kitchen (`c5a9ac`, 192.168.86.207) and living-room (`c2f84c`, 192.168.86.206)
units get physically aimed at each other for closed-loop audio tests: one unit
plays TTS, the other's mic captures it. Driving entities over the HA REST API
(token `HA_TOKEN`, HA on the Studio at 192.168.86.200:8123):

- Play TTS on a unit: `POST /api/services/assist_satellite/announce`
  `{"entity_id": "assist_satellite.<unit>_assist_satellite", "message": "...",
  "preannounce": false}` — 200 on accept.
- Both units run the fork's free-speech capture, so `assist_satellite.*`
  state `listening` is the steady state, not a wedge. A wedged unit shows
  `unavailable` on its `assist_satellite`/`media_player` entities; reboot via
  the restart button.
- Pipeline debug lines land in the `homeai-voice-router` container logs
  (enable with the `logger/set_level` service, `assist_pipeline: debug`) —
  `default_pipeline speech-to-text result ...` shows what the mic heard.
- Keep-docs rule: when a speaker-test session learns a durable procedure or
  gotcha, fold it back into this skill and `docs/FORK-NOTES.md` the same
  session. See the `homeai-voice-stack` skill for the fork map.
