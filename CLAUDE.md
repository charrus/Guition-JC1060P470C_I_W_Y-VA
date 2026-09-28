# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single ESPHome firmware config, `guition-voice-panel.yaml`, for the Guition JC1060P470C_I_W_Y. The board is an ESP32-P4 with an ESP32-C6 Wi-Fi co-processor over SDIO, a 1024×600 MIPI-DSI display, GT911 touch and an ES8311 codec. The config provides an LVGL Home Assistant dashboard, push-to-talk (PTT) Assist, a "Panel audio" media player and Sendspin music. It targets **ESPHome 2026.9.0 / ESP-IDF 5.5.5** (`min_version` is enforced) and uses only built-in components, with no external packages or fonts.

- `README.md` is the short project overview and quick start, and it embeds the preview image. Keep its feature list and status in step with `Guition-Setup.md`.
- `Guition-Setup.md` is the user-facing setup guide: substitutions, flashing, HA setup, pin table and troubleshooting table.
- `SKILLS.md` holds engineering notes with the root cause, the source-level evidence and the regression checks for each non-obvious design decision. **Read the relevant section before changing audio, PTT, media-player or network behaviour.** Add a new dated section when you make a comparable fix.
- `secrets.example.yaml` lists the secrets the YAML needs (`wifi_ssid`, `wifi_password`, `guition_api_key`, `guition_fallback_password`). There is no real `secrets.yaml` in this directory.
- `Guition-Panel-Preview.png` is rendered from the LVGL widget positions using sample data. Regenerate it or flag it as stale if the layout changes.

There is no test suite.

## Commands

`esphome` is not on PATH on this machine. Install it with `pip install esphome==2026.9.0` or use the user's ESPHome Device Builder. To validate or build, create a temporary `secrets.yaml` next to the YAML with dummy values. The API key must be valid 32-byte base64 (`openssl rand -base64 32`). Delete the temporary file afterwards and never ship a firmware binary.

```bash
esphome config  guition-voice-panel.yaml   # schema validation (fast)
esphome compile guition-voice-panel.yaml   # full ESP32-P4 build; the real test for lambdas/C++
esphome run     guition-voice-panel.yaml   # build + flash (first flash must be USB to the P4 port)
```

"Verified" in this project means schema plus a full compile. Hardware behaviour (audio, touch, Ethernet failover) can only be confirmed by the user on the device. Say that explicitly instead of claiming it works.

## Architecture of the YAML

The config is organised as a state machine driven by globals and scripts, not as independent components:

- **Globals** (`ptt_held`, `request_pending`, `pipeline_done`, `voice_cancelled`, `voice_phase`, `phase_since`, `press_since`, ...) hold the voice-request state. `refresh_ui` renders the whole LVGL page from these globals together with the media-player and connection state. Change state first, then call `refresh_ui`.
- **Scripts**:
  - `ptt_start` / `ptt_release` handle the press and release of the talk button.
  - `release_microphone` hands the I²S bus from the microphone to the speaker.
  - `audio_output_started` handles external audio that arrives during capture.
  - `cancel_assist` cancels voice activity only, while `stop_voice` (Stop/Reset) also stops both media pipelines.
  - Also present: `reset_music_ducking`, `wake_screen` and `restart_hold`.
- The `interval:` block runs the timeouts and recovery: a 30 s maximum hold and 60 s recovery for an unanswered request.
- **Audio graph:** one `audio_bus` is shared by `panel_microphone` and `panel_speaker`. `panel_speaker` sits behind `panel_audio_mixer`, which has two inputs: `announcement_speaker` and `music_speaker`. `panel_media_player` (platform `speaker_source`) uses the announcement pipeline for the HTTP/TTS source and the media pipeline for the Sendspin and HTTP music sources. `sendspin_group` is only a group controller and plays no audio.

## Invariants (each is backed by a section in SKILLS.md)

- **Never call `voice_assistant.stop` on normal PTT release.** In this ESPHome/HA version it sends an *abort*, which kills STT and intent handling. On release, apply `microphone.mute: panel_microphone` and let HA silence detection finish. Each accepted press must unmute again. Keep `voice_assistant.stop` for explicit cancellation only: Stop/Reset, master mute, or a release before Listening starts.
- **The microphone and speaker cannot run at the same time on the shared I²S bus.** Muting does not release the bus. After capture ends (VAD-end, with STT-end and TTS-start as fallbacks) or on cancellation, stop the *named source* with `id(assist_microphone_source).stop()` via `release_microphone`. Do not call raw `panel_microphone.stop()` or `microphone.stop_capture`, because those break the source's ownership tracking. Do not run the handoff on button release, because the silence tail is still needed then.
- The PTT busy guard unlocks only when the pipeline has ended, the assistant is idle, the media player is idle **and** the physical speaker has stopped.
- The media player owns volume and mute. The `voice_volume` number forwards to it and is mirrored back with `publish_state`. Never restore or set a second volume value.
- All audio runs at 16 kHz / 16-bit. The codec, microphone clock, mixer and Sendspin must stay consistent, so do not change only one sample rate. Opus is excluded because it requires 48 kHz.
- `use_microphone: false` on the ES8311 selects the **analogue** microphone input and is correct. `p4_engineering_sample: 'true'` is correct for the user's v1.3 silicon. Keep `CONFIG_ESP_MAIN_TASK_STACK_SIZE: "16384"`, which is the ESP-IDF #19020 startup workaround.
- Networking uses `network.priority: [ethernet, wifi]` with both interfaces enabled. Do not add automations that disable Wi-Fi, and keep `wifi.reboot_timeout: 0s`. GPIO28/29/30/34/35/49 are reserved for the RMII data lines.
- Pins follow the `_I_W_Y` V1.0 schematic and BSP. Other JC1060P470 variants use different microphone and amplifier pins, so do not copy pins from generic examples.
- Room entity IDs in `substitutions` are placeholders. No Home Assistant instance has been accessed.

When upgrading ESPHome or Home Assistant, recheck the source behaviour cited in SKILLS.md, especially the stop/abort semantics and `MicrophoneSource` ownership. Do not assume the documentation examples still hold.
