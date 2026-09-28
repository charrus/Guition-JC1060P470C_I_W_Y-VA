# ESPHome project troubleshooting notes

## Push-to-talk release aborted the command before intent handling

Recorded: 27 September 2026. Device: Guition JC1060P470C_I_W_Y.

### Observed symptom

The user's log showed a running Assist pipeline and microphone stream, then:

```text
Starting STT by VAD
Signaling stop
Assist Pipeline ended
State changed from STOPPING_MICROPHONE to IDLE
```

There was no `Speech recognised as` or `Intent started`. The release handler in the supplied configuration called `voice_assistant.stop` after setting the UI to Thinking.

### Cause confirmed in source

The firmware's normal release action cancelled the pipeline before speech recognition and intent processing completed:

1. In ESPHome 2026.9.0, `voice_assistant.stop` calls `VoiceAssistant::request_stop()`. While streaming microphone audio, that calls `signal_stop_()`, sending `VoiceAssistantRequest` with `start = false`.
2. The inspected `aioesphomeapi` client routes that request to `handle_stop(True)`, meaning abort.
3. Home Assistant 2026.9.0 routes an abort to `_abort_pipeline()`, which cancels the pipeline task.

The protocol distinguishes this from `VoiceAssistantAudio.end`, which the client routes to `handle_stop(False)` to end the audio stream without aborting processing. The stock ESPHome stop action does not send that audio-end message.

The ESPHome documentation inspected on this date still described calling stop to finish a command with `silence_detection: false`. That advice does not match the implementation combination inspected here. Recheck both ESPHome and Home Assistant source when upgrading; do not assume an older example's release behaviour remains valid.

This log establishes premature cancellation. It does not yet establish transcription accuracy, correct entity exposure, intent success or working response audio. Changing intent permissions or microphone pins would not address this cancellation path.

### Fix used in this project

- Start with `silence_detection: true`.
- On every accepted press, explicitly unmute `panel_microphone` before starting the assistant. Keep the retained master-mute switch as a separate user preference.
- On normal release, apply `microphone.mute: panel_microphone`. The microphone component supplies zero samples while muted, allowing Home Assistant's silence detection to finish the command. Do not stop microphone capture directly; the pipeline still needs the silence tail.
- Let `on_stt_vad_end` mark speech capture complete. The 28 September revision explicitly stops the named microphone source at this stage, with STT-end and TTS-start fallbacks, to release the shared I²S bus. See the handoff note below.
- If recognition or playback already started before release, preserve that phase and leave the request running.
- Use `voice_assistant.stop` only for explicit cancellation, including Stop/Reset, master mute or a release before listening begins.
- Retain the busy guard until the pipeline ends, the assistant becomes idle and speaker playback finishes. Do not unlock merely because the user released the button.
- Keep the timeout recovery, which can cancel an unresponsive request.

This preserves a hold-to-talk interface with silence-based completion. It is not a forced end-of-speech API: Home Assistant may also finish when it detects a pause while the button is held. Release-to-processing latency depends on the pipeline's finished-speaking detection setting.

### Verification and regression checks

The revised complete YAML compiled successfully with ESPHome 2026.9.0 / ESP-IDF 5.5.5 using temporary test credentials. Device confirmation of this PTT fix is still pending.

After flashing, hold until Listening, say a complete command, then release. Expect:

```text
Released; microphone muted, waiting for silence detection
STT by VAD end
Speech recognised as: "..."
Intent started
```

Check these behaviours after future edits:

- A normal release does not log `Signaling stop` and does not abort the request.
- A second request captures speech, because press unmutes the microphone again.
- Releasing while Thinking or Speaking does not interrupt the response.
- A quick tap before Listening cancels cleanly.
- Master mute and Stop/Reset still cancel or silence the current request.
- Backend or no-speech errors appear on the panel instead of being mistaken for intent success.

If recognition completes but no intended action occurs, inspect the Home Assistant Assist trace and entity exposure next. That is a separate failure stage.

### Primary references

- [ESPHome 2026.9.0 voice assistant implementation](https://github.com/esphome/esphome/blob/2026.9.0/esphome/components/voice_assistant/voice_assistant.cpp): `request_stop`, `signal_stop_`, `VOICE_ASSISTANT_STT_VAD_END`.
- [ESPHome 2026.9.0 microphone implementation](https://github.com/esphome/esphome/blob/2026.9.0/esphome/components/microphone/microphone.h): muted callbacks receive zero samples.
- [aioesphomeapi client](https://github.com/esphome/aioesphomeapi/blob/main/aioesphomeapi/client.py): `subscribe_voice_assistant`, request-stop versus audio-end callbacks. Also checked in the local ESPHome Python environment.
- [Home Assistant 2026.9.0 ESPHome Assist satellite](https://github.com/home-assistant/core/blob/2026.9.0/homeassistant/components/esphome/assist_satellite.py): `handle_pipeline_stop`, `_stop_pipeline`, `_abort_pipeline`.
- [ESPHome voice assistant documentation](https://esphome.io/components/voice_assistant/).

## Reply playback blocked by the shared I²S bus

Recorded: 28 September 2026. ESPHome 2026.9.0, ESP32-P4 rev1.3.

### Observed symptom and causal chain

The panel booted and connected successfully, then logged:

```text
[E][i2s_audio.speaker.std:401]: Parent bus is busy
[E][i2s_audio.speaker:134]: Driver failed to start; retrying in 1 second
[E][voice_assistant:1001]: Cannot receive audio, buffer is full
```

The errors establish that reply audio arrived while the speaker could not acquire its I²S parent lock. The configuration has one `audio_bus` used by `panel_microphone` and `panel_speaker`. In this configuration the microphone is the competing bus owner. Because playback cannot consume incoming audio, the voice assistant's receive buffer fills and subsequent chunks are dropped.

`microphone.mute` substitutes zeros in delivered audio; it does not stop the I²S microphone driver or release the bus. The earlier PTT release fix intentionally kept capture alive until Home Assistant detected the silence tail. It relied on ESPHome's automatic handoff afterwards, which was insufficient in this run.

The supplied log does not include DEBUG-level STT/TTS events. It therefore does not establish whether VAD-end was absent, events arrived too close together for the scheduled microphone stop, or another capture request retained the microphone. Record the bus conflict as confirmed and the exact event sequence as unresolved until a full trace is available.

### Configuration fix

Give the voice assistant's microphone **source** its own ID:

```yaml
voice_assistant:
  id: va
  microphone:
    id: assist_microphone_source
    microphone: panel_microphone
```

After capture has finished, stop that source:

```yaml
- lambda: |-
    id(assist_microphone_source).stop();
```

The complete configuration wraps this in `release_microphone`, which also mutes input and logs actual driver shutdown. It runs at VAD-end, STT-end, TTS-start, pipeline end, error and explicit cancellation. Calling `MicrophoneSource::stop()` repeatedly is guarded by the source's enabled flag, so normal repeated endpoint callbacks do not return the same microphone listener twice. Stopping the owning source updates that flag, allowing the following PTT request to start it again.

Do not replace this with an unconditional raw `panel_microphone.stop()` or repeated `microphone.stop_capture` calls. A component-owned `MicrophoneSource` tracks its own enabled state; stopping only the physical microphone can leave ownership inconsistent and prevent the next request from restarting capture.

Do not call this handoff script on **normal button release**: the silence tail is still required then. Stop the source only once capture has ended or cancellation is intended. Keep `voice_assistant.stop` reserved for cancellation, preserving the previous release fix.

The script's `wait_until` observes the real microphone's `is_stopped()` state, because the source's `is_stopped()` may become true before physical driver teardown completes. This wait logs success or a one-second warning; it does not block the voice assistant's automatic speaker startup. If a startup race or buffer loss remains, inspect the full DEBUG trace before changing the audio architecture.

### Validation and follow-up

The complete revised YAML compiled successfully with ESPHome 2026.9.0 / ESP-IDF 5.5.5. A host check using the installed `MicrophoneSource::start()` and `stop()` implementations confirmed that repeated stops release the listener only once and the next start rearms capture. The endpoint wiring and preservation of the normal-release silence tail were also checked.

- Capture one full request and look for `Microphone driver stopped; input released the I2S bus`, followed by successful playback.
- Test a second request and Stop/Reset.
- If `Microphone still active after 1s` appears, inspect all microphone capture owners and the full event sequence.
- Do not treat a larger speaker buffer as the primary fix: in the inspected driver it only accepts audio after startup, and it cannot release a busy bus.
- Do not map two independent primary I²S controllers to the same clock pins as a shortcut. This board's clock wiring requires a deliberate shared-clock design if its audio architecture is changed.

Hardware confirmation of this handoff revision is pending. Receiving response audio does not by itself prove that the intended Home Assistant action succeeded.

### Primary references

- [ESPHome 2026.9.0 microphone-source implementation](https://api-docs.esphome.io/microphone__source_8cpp_source): enabled-state ownership in `start()` and `stop()`.
- [ESPHome I²S microphone implementation](https://api-docs.esphome.io/i2s__audio__microphone_8cpp_source): listener accounting, asynchronous stop and parent-bus unlock.
- [ESPHome standard I²S speaker implementation](https://api-docs.esphome.io/i2s__audio__speaker__standard_8cpp_source): the parent-lock failure producing the reported error.
- [ESPHome microphone actions](https://esphome.io/components/microphone/): software mute versus capture control.


## Adding the Home Assistant media player

Recorded: 28 September 2026. Component: built-in `media_player` platform `speaker`. This section records the original addition; the Sendspin revision below supersedes its single-pipeline architecture.

### Integration decisions

- `panel_media_player`, named **Panel audio**, uses one announcement pipeline with `panel_speaker`, FLAC, 16 kHz, mono. Home Assistant can transcode media to that advertised format; direct URLs must already be compatible. This configuration has no separate music pipeline or pause/resume support.
- `voice_assistant.media_player` replaces `voice_assistant.speaker`; those output options are mutually exclusive. Remove `on_tts_stream_start` / `on_tts_stream_end`, which are only valid for direct-speaker output. Use media-player announcement/idle callbacks for display playback state.
- Keep the named microphone-source release and the normal-release silence tail. The media player changes the reply transport to URL playback, but cannot release a microphone-owned I²S bus by itself.
- Both the PTT action and its UI guard check the media player's idle state as well as physical speaker shutdown. Checking only the hardware speaker misses the URL download/decoder-start interval. Request completion and timeout guards also account for media-player state.
- An incoming announcement cancels an ongoing capture and releases the named source. This includes the silence tail after PTT release. Assist's own replies normally arrive after the STT callbacks have already released the source.
- The media player owns volume and mute persistence. The existing `voice_volume` number sends `media_player.volume_set`; `on_volume` mirrors the result with `publish_state`, which does not call the number's control action again. Boot synchronizes the number from the restored media-player volume. Do not also set the raw speaker volume at boot or restore two independent volume values.
- Stop/Reset calls `media_player.stop` with `announcement: true`, allowing the player to stop decoding and clear its queue. It does not persistently mute the player or stop only its underlying speaker while the decoder continues refilling it.

### Validation and device checks

The complete media-player revision passed ESPHome 2026.9.0 schema validation and compiled successfully for ESP32-P4 with ESP-IDF 5.5.5. The build used temporary test secrets; no firmware binary is delivered.

1. Confirm **Panel audio** appears under the ESPHome device in Home Assistant.
2. Run the guide's `tts.speak` example using your actual entity IDs; verify sound and announcement-to-idle status.
3. Change media-player volume in Home Assistant and on the panel; verify the number and slider agree. Check mute display and restoration after reboot.
4. Run two consecutive PTT requests. Verify microphone shutdown precedes reply playback and the talk button unlocks afterwards.
5. Stop an announcement and confirm the next playback remains audible. Send an external announcement during recording and confirm recording is cancelled and the bus is released.

Hardware confirmation remains pending. The earlier direct-audio buffer-full log is retained above as historical evidence; URL playback uses the media player's reader/decoder and different diagnostics.

### References

- [ESPHome speaker media player](https://esphome.io/components/media_player/speaker/).
- [ESPHome 2026.9.0 speaker media-player implementation](https://github.com/esphome/esphome/blob/2026.9.0/esphome/components/speaker/media_player/speaker_media_player.cpp): single-pipeline routing, state, queue stopping and volume persistence.
- [ESPHome 2026.9.0 voice-assistant schema](https://github.com/esphome/esphome/blob/2026.9.0/esphome/components/voice_assistant/__init__.py): exclusive output and stream-hook validation.


## Sendspin integration and shared-speaker ownership

Recorded: 28 September 2026. Target: ESPHome 2026.9.0, ESP32-P4.

### Architecture

- Replace `platform: speaker` with the built-in `platform: speaker_source` while keeping `panel_media_player` / Panel audio as the Assist output and local volume owner.
- Configure `sendspin:` plus a `media_source` of platform `sendspin`. The separate `media_player: platform: sendspin` is a group controller, not an audio decoder or playback path.
- Use two distinct mixer inputs: `announcement_speaker` for a dedicated HTTP announcement source, and `music_speaker` for Sendspin plus a separate HTTP music source. The two pipelines cannot share the same source instance or the same speaker object.
- The mixer downmixes stereo input to the board's existing 16-bit mono speaker. All streams stay at 16 kHz to match the codec and microphone clock. Sendspin advertises `[flac, pcm]`; Opus requires 48 kHz and is deliberately excluded.
- An announcement ducks only the local music input by 24 dB. Playback/idle/pause callbacks restore it. The media player remains the only owner of master volume/mute; source callbacks report this local state back to Sendspin.
- Track and artist metadata are exposed as text sensors and displayed in the existing conversation area while Sendspin plays.

### Recording and stop behaviour

- Retain the silence tail on normal PTT release and stop the named microphone source only after STT or cancellation.
- Disable new PTT requests until Panel audio is idle and the physical speaker has stopped. The user taps Stop/Reset before recording during music; muting does not free I²S. There is no automatic music resume after a voice turn.
- External music or announcements starting during capture invoke `audio_output_started`, cancel that capture and release the named microphone source. Its shutdown wait remains diagnostic, not a hard playback gate.
- Separate `cancel_assist` from `stop_voice`. Microphone master-mute and Assist disconnection cancel voice/announcement activity without stopping independent music. Stop/Reset cancels voice and stops both local pipelines.
- Stopping the Sendspin source updates the client's state to EXTERNAL_SOURCE. This differs from the Sendspin group controller's STOP command, which stops the whole group. Group play/pause/volume likewise affect all members and must be clearly labelled.
- Voice-request cleanup waits for the announcement input to drain, while the separate PTT guard still waits for all output to stop. This allows voice bookkeeping to finish even when background music remains active.

### Validation and device checks

ESPHome 2026.9.0 schema validation and the complete ESP32-P4 firmware build passed with ESP-IDF 5.5.5. Firmware size was 1,934,596 bytes; static RAM use was 112,776 bytes. These build totals do not measure runtime audio buffers or guarantee playback timing. Hardware playback, group synchronization and the handoff still require confirmation on the Guition panel. No test firmware binary is included.

1. Discover the panel in Music Assistant, start a track and verify title/artist and local-volume synchronization.
2. Group it with a second Sendspin player and check audible synchronization.
3. Send Home Assistant TTS to Panel audio while music plays; verify local ducking and restoration.
4. Tap Stop/Reset: check that this panel stops, other group members continue, and PTT enables after physical speaker shutdown.
5. Run two PTT turns, then select the panel for music playback again.
6. Check master microphone mute and Assist disconnection leave Sendspin playback intact.
7. Start music during speech capture and inspect the complete DEBUG trace for source release and recovery.

Discovery requires mDNS and TCP 8928. ESPHome labels Sendspin experimental; recheck server/component compatibility after upgrades. A weak Wi-Fi link can affect synchronized playback independently of the earlier microphone bus-lock error.

### Primary references

- [ESPHome Sendspin hub](https://esphome.io/components/sendspin/).
- [ESPHome Sendspin media source](https://esphome.io/components/media_source/sendspin/).
- [ESPHome speaker-source media player](https://esphome.io/components/media_player/speaker_source/).
- [ESPHome Sendspin group media player](https://esphome.io/components/media_player/sendspin/).
- [Music Assistant Sendspin](https://www.music-assistant.io/player-support/sendspin/).
- Local ESPHome 2026.9.0 source: `sendspin/media_source/sendspin_media_source.cpp`, `speaker_source/speaker_source_media_player.cpp`, `mixer/speaker/mixer_speaker.cpp`.


## Ethernet priority with Wi-Fi fallback

Recorded: 28 September 2026. Target: ESPHome 2026.9.0, Guition JC1060P470C_I_W_Y.

### Implementation

- Configure the IP101 PHY with MDC GPIO31, MDIO GPIO52, external RMII clock GPIO50, PHY address 1 and reset GPIO51 via `power_pin`. These control/data nets match the exact-model schematic and Guition board configuration. The ESP32-P4 default RMII data pins are reserved by ESPHome: GPIO28/29/30/34/35/49. Do not reuse them for another component.
- Use native `network.priority: [ethernet, wifi]`. ESPHome 2026.9.0 validates coexistence only when the priority list includes both configured interfaces. Do not apply older advice that Ethernet and Wi-Fi are always mutually exclusive.
- Keep both interfaces enabled at boot. No Wi-Fi-disable automation is used: a disabled fallback cannot take over until explicitly enabled.
- Prefer Ethernet for the default route and reported address; return to it when reconnected. The installed `network_component.cpp` explicitly sets the ESP-IDF default netif, and `network/util.cpp` implements preferred-interface address selection. It is not just setup order.
- Set Wi-Fi reboot timeout to zero, preserving the API's existing zero timeout. Loss of one interface must not cause an unnecessary reboot of a usable panel.
- Retain the `IP address` entity as a template reading the preferred interface address. Expose separate Ethernet/Wi-Fi IP diagnostics and a Network interface text sensor. Keep Wi-Fi RSSI as an explicitly wireless diagnostic.

### Limits and device checks

Both interfaces use DHCP and may have different addresses. Priority controls the device's default route, not which address an existing Home Assistant connection was opened against. It also does not detect a server or upstream network failure when the Ethernet link/IP remains valid. Existing TCP sessions may reconnect or need playback restarted after failover; seamless audio migration is not claimed.

ESPHome 2026.9.0 schema validation and the complete ESP32-P4 firmware build passed with ESP-IDF 5.5.5. Physical Ethernet and cable-failover tests remain pending:

1. Boot with both networks available; check Ethernet has a DHCP address and the network log selects the Ethernet default interface.
2. Unplug the cable; check Wi-Fi becomes preferred, then reconnect and check Ethernet returns.
3. Boot without a cable and verify Wi-Fi access.
4. Verify Home Assistant reconnection, Assist and Sendspin after each transition. Check fixed-IP entries and per-interface DHCP reservations if discovery reaches an old address.

### References

- [ESPHome multi-interface networking](https://esphome.io/components/network/#multi-interface-support).
- [ESPHome Ethernet](https://esphome.io/components/ethernet/).
- [Exact-model manufacturer schematic mirror](https://github.com/wegi1/ESP32P4-JC1060P470C-I_W_Y/tree/main/5-Schematic).
- Installed ESPHome 2026.9.0: `network/network_component.cpp`, `network/util.cpp`, `ethernet/__init__.py` and `ethernet/ethernet_component_esp32.cpp`.
