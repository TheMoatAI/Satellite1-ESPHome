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
- If an edit to an *included* file (`config/common/*`) isn't picked up, delete
  the `.esphome/` cache dir and rebuild — the validation cache can pin stale
  included config.
- WiFi model-manifest checks may warn `Connection reset by peer` and use cached
  wake-word files — harmless.

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

- Reboot a unit: press `button.satellite1_XXX_restart_sat1`.
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
