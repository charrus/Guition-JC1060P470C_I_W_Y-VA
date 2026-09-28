# Guition voice control panel

For the **Guition JC1060P470C_I_W_Y**, 7-inch, 1024 × 600 capacitive touchscreen with ESP32-P4, ESP32-C6 and ES8311 audio. This is a complete ESPHome YAML configuration with an LVGL dashboard and Home Assistant Assist push-to-talk.

Use **ESPHome 2026.9.0 or later**. The configuration uses built-in components and fonts; it does not pull in third-party ESPHome packages.

## What the panel does

- Ethernet-first networking with Wi-Fi fallback and separate interface diagnostics.
- Four large light-control tiles, with room names and live on/off states.
- Clock and date in the Europe/London timezone, including daylight-saving changes.
- Optional indoor temperature and humidity from Home Assistant entities.
- Hold-to-talk voice control, recognised text and a scrollable assistant reply.
- Listening, Thinking, Speaking, muted and disconnected status messages.
- Software microphone mute, retained across restarts.
- A Home Assistant **Panel audio** media player for Assist replies, TTS, HTTP audio and Sendspin music.
- Sendspin discovery, synchronized playback, track/artist display and a separate group-control entity.
- Local voice volume and screen brightness sliders, also exposed to Home Assistant; audio volume stays in sync with the media player.
- Automatic dimming to 15% after two minutes without interaction; touching the screen restores the selected brightness.
- Stop/recovery control and a three-second hold to restart.

`Guition-Panel-Preview.png` is a layout illustration rendered from the YAML widget positions. The time, sensor readings, light states and conversation shown are sample values, not readings from your home or a photograph of a running device. The actual interface uses LVGL's built-in Montserrat fonts.

## 1. Add the configuration

Place `guition-voice-panel.yaml` in the ESPHome configuration directory, normally `/config/esphome/` when using Home Assistant's ESPHome Device Builder. Alternatively, create a new device and replace its generated YAML with this file.

Merge the entries in `secrets.example.yaml` into your existing `secrets.yaml` and fill in the values. Keep existing secrets for your other devices. The API key must be a real 32-byte base64 key, not the placeholder text. To generate one:

```bash
openssl rand -base64 32
```

ESPHome Device Builder can also generate an API encryption key when you create a device. Retain that value under `guition_api_key` if preferred. OTA updates use the same key and require encryption; there is no separate OTA password in this configuration.

## 2. Link the controls to your home

Edit only the substitutions at the top to start with:

```yaml
substitutions:
  panel_title: Home
  room_1_name: Living room
  room_1_entity: light.your_living_room_light
  room_2_name: Kitchen
  room_2_entity: light.your_kitchen_light
  room_3_name: Bedroom
  room_3_entity: light.your_bedroom_light
  room_4_name: Hallway
  room_4_entity: light.your_hallway_light
  temperature_entity: sensor.your_room_temperature
  humidity_entity: sensor.your_room_humidity
```

These entity IDs are placeholders: no Home Assistant account was accessed to discover your devices. Find your actual IDs in Home Assistant's Developer Tools → States. Light groups work too. The tiles currently call `light.toggle`, so their targets must be light entities. To control a switch instead, change that tile's action to `homeassistant.toggle` or `switch.toggle` as well as changing its entity ID.

A tile remains disabled and displays **NOT LINKED** until Home Assistant sends an `on` or `off` state for its target. **OFFLINE** means the state-subscription connection is unavailable. A tap sends a command; the tile changes state only when Home Assistant reports the result. The temperature display expects a Celsius sensor, and humidity expects percent. Missing optional sensors do not prevent light or voice operation.

## 3. Check the ESP32-P4 revision and flash

The default is:

```yaml
p4_engineering_sample: "true"
```

This targets ESP32-P4 silicon revisions below v3.0, including v1.0 and v1.3. If your board's boot log reports **v3.0 or later**, change it to `"false"`. The model name alone does not establish the chip revision. ESPHome's [ESP32 documentation](https://esphome.io/components/esp32/) explains this setting.

Your supplied boot log confirms **v1.3**, so retain `"true"` for your unit.

The configuration also sets `esp32.framework.sdkconfig_options.CONFIG_ESP_MAIN_TASK_STACK_SIZE` to `"16384"`. This is the documented workaround for an [ESP-IDF 5.5.5 ESP32-P4 startup bug](https://github.com/espressif/esp-idf/issues/19020): a small startup-task stack can land in the internal SPM region, which the flash-cache safety check does not recognise. Your assertion and stack pointer `0x30100e70` match that report. Increasing the stack size prevents that allocation from fitting in SPM. The workaround preserves assertions and PSRAM execution. Rebuild and flash over USB if the old firmware crashes before Wi-Fi starts.

Validate and install with ESPHome Device Builder. For the first flash, use the board's P4 programming connection and a USB data cable, following the supplier's boot-mode instructions. This YAML is for the **P4 application**, not a C6 firmware image. Once connected, subsequent updates can be installed over Ethernet or Wi-Fi.

The logger is configured for `USB_SERIAL_JTAG`. Use the USB connection that exposes the P4's native serial/JTAG interface for these logs. If you deliberately use the UART bridge instead, select `hardware_uart: UART0` to match it.

CLI equivalents:

```bash
esphome config guition-voice-panel.yaml
esphome run guition-voice-panel.yaml
```

## Ethernet first, Wi-Fi fallback

Connect the built-in RJ45 port to your router or switch. The configuration enables the board's IP101 Ethernet PHY and keeps C6 Wi-Fi enabled as a fallback. Both interfaces use DHCP.

```yaml
network:
  priority:
    - ethernet
    - wifi

ethernet:
  id: panel_ethernet
  type: IP101
  mdc_pin: GPIO31
  mdio_pin: GPIO52
  clk:
    pin: GPIO50
    mode: CLK_EXT_IN
  phy_addr: 1
  power_pin: GPIO51
  enable_on_boot: true
```

The existing `wifi:` block remains enabled with your saved credentials. ESPHome prefers a connected Ethernet interface for its default route and reported address. If Ethernet loses its connection, Wi-Fi takes over once connected; Ethernet becomes preferred again when it reconnects. Both interfaces remain enabled, so Wi-Fi does not need to be switched on by an automation. `wifi.reboot_timeout: 0s` prevents Wi-Fi loss from initiating a reboot while Ethernet is available. The existing API reboot timeout is also disabled.

Home Assistant gets these diagnostics:

| Entity | Meaning |
|---|---|
| Network interface | Preferred connected interface: Ethernet, Wi-Fi or Disconnected |
| IP address | IPv4 address reported for the preferred available interface |
| Ethernet IP address | Wired interface address |
| Wi-Fi IP address | Wireless interface address |
| Wi-Fi signal | Wireless reception only; it does not indicate which route is preferred |

Ethernet and Wi-Fi normally receive different IP addresses. Keep the device hostname for discovery; do not pin `wifi.use_address` to the old wireless IP if you want address selection to follow the interface. If you create DHCP reservations, reserve each interface's own address. An existing Home Assistant entry configured with a fixed wireless IP may need its connection address updated to reach the panel over Ethernet.

Failover changes routing; it does not move an established TCP connection to another IP. Assist/API and Sendspin sessions may reconnect or playback may need restarting after a cable change. Test this on the device. The priority decision uses link/IP connectivity, not an end-to-end health check of Home Assistant or the internet: a connected Ethernet link with an unreachable server can still remain preferred.

To verify, boot with a cable connected and check for the `Default interface:` log and Network interface = Ethernet. Unplug the cable and check Wi-Fi, then reconnect and check Ethernet again. Finally boot without a cable and confirm Wi-Fi access. Do these checks before relying on remote access.

## 4. Enable Home Assistant control and Assist

1. Add the discovered panel through Settings → Devices & services → ESPHome, or add it by IP address. Supply the configured API key if requested.
2. Open the ESPHome integration's options for this device and enable **Allow the device to perform Home Assistant actions**. This is required for the light buttons.
3. Configure an Assist pipeline under Settings → Voice assistants with a conversation agent, speech-to-text and text-to-speech. Home Assistant Cloud or local services such as Whisper/Speech-to-Phrase and Piper can provide these roles, depending on your setup.
4. Select the desired Assist pipeline for this ESPHome device in its voice-assistant configuration. Expose the entities you want Assist to control to that assistant.
5. Make sure a suitable speaker is connected to the board's speaker socket if your unit was supplied without one. Use the board's recommended power supply.

The panel handles audio capture, playback and the interface. Speech recognition and command handling run through Home Assistant. This configuration does not call OpenAI directly and does not need an OpenAI API key unless you separately choose an OpenAI conversation agent in Home Assistant.

## Using push-to-talk

Press and hold **HOLD TO TALK**. Wait for **Listening - speak now**, speak your command, then release. Release software-mutes the microphone while the audio stream briefly continues with silence. Home Assistant's silence detection then completes speech recognition and runs the intent. The delay depends on the pipeline's finished-speaking detection setting. A quick tap that ends before listening starts is treated as a cancelled attempt.

Silence detection can also finish a command during a pause while the button is still held. Release after this point leaves processing or playback running. For longer pauses, adjust the pipeline's finished-speaking detection in Home Assistant. This is a hold-to-talk interface backed by silence detection; release does not send a native forced end-of-speech command.

**Do not put `voice_assistant.stop` on the normal release path.** In the current API, that sends an abort and can end the pipeline before either `Speech recognised as` or `Intent started` appears. The corrected release path uses `microphone.mute`, and each accepted press explicitly unmutes the microphone. Stop/Reset, master mute and cancellation before listening still use the stop action intentionally. `SKILLS.md` records the cause, source evidence and maintenance checks.

The microphone does not run a wake-word listener. **MUTE MIC** applies software mute and prevents starting a new recording. This is not a physical microphone power-disconnect switch.

After speech capture ends, the `release_microphone` script explicitly stops the voice assistant's named microphone source, allowing its driver to release the shared I²S bus. It runs at `on_stt_vad_end`, with fallbacks at `on_stt_end` and `on_tts_start`. Normal button release still only mutes: capture must continue long enough to supply the silence tail. Stopping the source also resets its internal ownership state so the next PTT press can start it again. The script reports actual driver shutdown; its asynchronous wait is diagnostic and does not delay ESPHome's automatic TTS start.

The talk button stays unavailable while a request or its reply is active. Pipeline completion alone does not unlock it: playback must finish too. Recording has a 30-second maximum hold time. An unanswered request can recover after 60 seconds without active speaker playback; normal replies are not cut off by a fixed playback timer.

Tap **STOP / RESET** to end capture and stop both local audio pipelines, including Sendspin music and Home Assistant announcements. A request already sent to Home Assistant may still finish, so Stop cannot undo an action. The panel waits for its audio/Assist state to become idle before another recording can start. Hold the same button for **three seconds** to restart the device if a session gets stuck. Home Assistant also gets a Restart panel diagnostic button.

## Media player

After installing this revision, Home Assistant exposes a **Panel audio** media player on the Guition device. Assist now sends its response URL to this player through `voice_assistant.media_player: panel_media_player`. The physical `panel_speaker` remains the I²S output beneath the player.

Panel audio now uses ESPHome's built-in `speaker_source` platform, with two pipelines mixed into the existing physical speaker:

| Pipeline | Sources | Audio requested |
|---|---|---|
| Announcements | Dedicated HTTP source for Assist/TTS | FLAC, 16 kHz, mono |
| Media | Sendspin source plus a separate HTTP source | 16 kHz, stereo input, mixed down to mono |

The HTTP media pipeline also advertises FLAC. For arbitrary files, use Home Assistant's media browser/media-source path so Home Assistant can transcode them. Raw URLs sent directly to ESPHome must already supply compatible 16-bit, 16 kHz FLAC and be reachable from the panel. Music is lowered by 24 dB while an announcement runs, then restored over 500 ms. This does not pause the other members of a Sendspin group.

The existing **Voice volume** number and display slider now control Panel audio. Changes made on the Home Assistant media-player card update both. The media player restores its own volume and mute state, with 55% as the first-use default; the previous standalone number no longer restores a competing value. `MUTED` appears beside the slider when playback is muted. Moving the slider to a nonzero volume unmutes playback. Starting PTT preserves the player's mute preference.

While media downloads or plays, the talk button stays unavailable until the media player and physical speaker are idle. External music or an announcement that arrives during capture cancels that capture and releases its microphone source. The microphone-to-speaker handoff remains necessary: the mixer combines playback streams but does not allow the shared I²S bus to run its microphone and speaker drivers at once.

To test independently of speech recognition, call Home Assistant's `tts.speak` action. Replace both example entity IDs with the actual IDs shown in your instance:

```yaml
action: tts.speak
target:
  entity_id: tts.your_tts_provider
data:
  media_player_entity_id: media_player.guition_voice_panel_panel_audio
  message: "The Guition panel speaker is working."
```

Then test a complete PTT request and a second request. The display uses the media player's announcement/idle events for playback status; the direct-speaker-only `on_tts_stream_start` and `on_tts_stream_end` hooks have been removed. Stop/Reset stops both local pipelines without leaving the player persistently muted. Muting the microphone or losing the Home Assistant Assist connection cancels voice activity but allows independent Sendspin music to continue.


## Sendspin setup and controls

This revision includes the Sendspin hub, an actual audio-playing Sendspin media source, and a **Sendspin group** media-player entity. A group controller by itself would not produce audio. Sendspin is currently experimental in ESPHome, so confirm compatibility when upgrading ESPHome or your server.

1. Run a compatible Sendspin server, such as Music Assistant, on a network reachable from the panel. Music Assistant's current Sendspin provider is built in and enabled by default.
2. Flash this YAML and let the panel connect over Ethernet or Wi-Fi. Discovery uses mDNS; allow it and TCP port **8928** between the panel and server. No server address or account credentials are required in this YAML for automatic discovery.
3. In Music Assistant, select the discovered Guition panel and play a track. If discovery fails, its Sendspin provider supports manual discovery addresses; use the panel's IP or hostname there. The panel is a Sendspin playback client, so the separate Music Assistant Sendspin *Source* plugin is unnecessary.
4. Group it with another Sendspin player if desired. Check synchronization by listening on both devices; this configuration has not been calibrated on physical hardware.

**Panel audio** controls this device's volume and mute. Its local volume is reported to Sendspin and changes received from the server update the display slider. The slider keeps its existing Voice volume label and entity for continuity, but it controls music as well as speech.

**Sendspin group** controls the entire active group: play, pause, stop, volume and mute apply to all members. Use Music Assistant to select music and manage the queue. Play/pause forwarded through Panel audio while Sendspin is its source also controls the group. In contrast, the panel's **STOP / RESET** stops local playback and relinquishes its Sendspin stream, without issuing a group-wide stop; select the panel again in Music Assistant when you want it to rejoin playback.

During Sendspin music, the conversation area shows the track title and artist. The status reads **Sendspin - stop to talk**. Tap **STOP / RESET**, wait for output to stop and the talk button to enable, then hold to speak. This board's shared I²S configuration does not support microphone capture while music continues; muting the music alone would not free the bus. It does not automatically resume Sendspin after a voice request.

Audio stays at **16 kHz / 16-bit / mono output**, matching the existing codec and microphone clock. Sendspin advertises FLAC first and PCM second at 16 kHz; incoming stereo is downmixed by the mixer. This is a speech-oriented panel rather than a high-fidelity music configuration. Opus is not advertised because ESPHome's Sendspin implementation requires 48 kHz for that codec. Do not change only the Sendspin sample rate: the mixer inputs and hardware audio clock must remain compatible.

## Hardware choices

The supplied pin assignments follow the manufacturer BSP and `_I_W_Y` V1.0 schematic linked below. Earlier open-board JC1060P470 variants use different microphone and amplifier pins.

| Function | Configuration |
|---|---|
| Display | ESPHome `mipi_dsi`, model `JC1060P470`, 1024 × 600 landscape |
| Display reset | GPIO5; horizontal sync pulse 20 |
| DSI supply | ESP32-P4 LDO channel 3, 2.5 V |
| Backlight PWM | GPIO23 |
| I²C SDA / SCL | GPIO7 / GPIO8 |
| GT911 touch | Address 0x5D, reset GPIO22, interrupt GPIO21 |
| ES8311 codec | I²C address 0x18 |
| I²S MCLK / BCLK / LRCLK | GPIO13 / GPIO12 / GPIO10 |
| Microphone input | GPIO48, left slot |
| Speaker data / amplifier enable | GPIO9 / GPIO11 |
| Audio format | 16 kHz, 16-bit, mono |
| Ethernet PHY | IP101, PHY address 1 |
| Ethernet management / reset | MDC GPIO31, MDIO GPIO52, PHY reset GPIO51 (`power_pin`) |
| Ethernet clock | External RMII reference input GPIO50 |
| Ethernet data | ESP32-P4 RMII defaults: TXD0 34, TXD1 35, TXEN 49, RXD0 29, RXD1 30, RXDV 28 |
| C6 Wi-Fi link | SDIO; reset 54, command 19, clock 18, data 14–17 |

**`use_microphone: false` is intentional.** In ESPHome's ES8311 driver, this option selects the codec's PDM digital input when true; false selects its analogue input. It does not disable microphone capture. The referenced schematic connects an analogue microphone to the ES8311. No ES7210 component is configured.

The microphone, speaker and codec all use 16 kHz to keep their shared I²S clock consistent. This favours dependable speech audio over high-fidelity music playback. Microphone gain starts at 24 dB. If speech clips, reduce `mic_gain` to `18DB` or `12DB`; if it is consistently too quiet, try `30DB` after checking the microphone is working.

## Troubleshooting

| Symptom | Check |
|---|---|
| Build rejects a component or option | Update ESPHome to 2026.9.0 or later; validate again. |
| Illegal-instruction boot failure | Check the P4 silicon revision and `p4_engineering_sample`. |
| Startup assertion `esp_task_stack_is_sane_cache_disabled()` with stack pointer `0x3010....` | Retain `CONFIG_ESP_MAIN_TASK_STACK_SIZE: "16384"` under `esp32.framework.sdkconfig_options`, rebuild and flash over USB. See the startup workaround above. |
| Display lit but blank | Check the exact board suffix, display reset wiring and boot log; use the native `JC1060P470` driver, not a Waveshare profile. |
| Touch fails | Check for GT911 at 0x5D in the I²C log. This configuration uses reset 22 and interrupt 21. |
| Screen orientation wrong | The supplied layout is landscape. A 180-degree mounting can be handled with `lvgl: rotation: 180`; use LVGL rotation so its touch mapping follows. |
| Light tile says NOT LINKED | Correct its entity ID, make sure it is a light, and check Home Assistant is connected. |
| Tile shows state but tapping has no effect | Enable permission for the device to perform Home Assistant actions. |
| Voice says Connect Assist in HA | Check the ESPHome integration and the pipeline selected for this device. |
| Release logs `Signaling stop`, then `Assist Pipeline ended`, with no recognised speech or intent | Use the revised PTT scripts. Normal release must mute the microphone and let silence detection finish, rather than call `voice_assistant.stop`. See `SKILLS.md`. |
| Microphone is silent | Confirm the codec at 0x18 and GPIO48; retain the analogue input setting. The codec's default slot is left; `microphone_channel` is exposed for boards/firmware needing a different slot. |
| No response sound | Check Panel audio volume/mute, the speaker connector, GPIO11 amplifier control and TTS pipeline. Confirm the panel can fetch the TTS URL provided by Home Assistant. |
| `Parent bus is busy`, followed by `Cannot receive audio, buffer is full` | The speaker cannot acquire the shared I²S bus. Use the explicit microphone-source release revision; muting alone does not stop capture. Check DEBUG logs for `audio_handoff` driver shutdown and send the complete turn if the error remains. |
| Media URL fails to play | Check `speaker_source_media_player` / `audio_http` logs, URL reachability and the FLAC/16 kHz/mono format. Prefer Home Assistant media-source playback for transcoding. |
| Talk button remains blocked during music | Tap Stop/Reset to stop local playback, then wait for the I²S speaker to shut down. Pausing or muting alone is not a reliable bus release. |
| Sendspin player not discovered | Check mDNS, TCP 8928, server compatibility and the Music Assistant provider discovery settings. |
| Sendspin stutters | Check Network interface first. On Ethernet, check cable/link and server load; on Wi-Fi, check reception and congestion. The earlier -78 dBm reading indicated a weak wireless link. |
| Second voice request remains blocked | Wait for playback to finish; tap Stop or hold it three seconds for a restart. Inspect logs for an audio or Assist error. |
| Ethernet has no address | Check cable/switch link, DHCP, PHY address 1 and the Ethernet boot log. Wi-Fi remains available as a fallback if it connects. |
| Cable removal loses the API/audio session | Wait for reconnection and restart playback if needed; an established socket cannot migrate between the two IP addresses. Check the Home Assistant entry is not tied to the now-unreachable IP. |
| ESP-Hosted/C6 startup errors | Check the C6 co-processor firmware compatibility. Wi-Fi credentials cannot fix a failed P4-to-C6 SDIO link. |

For C6 firmware maintenance, ESPHome provides an [ESP32 Hosted update component](https://esphome.io/components/update/esp32_hosted/). It is not enabled to perform automatic co-processor updates in this file. If the C6 is incompatible before networking starts, follow that component's embedded-update procedure or the supplier's C6 recovery instructions. Keep the P4 and C6 firmware targets distinct.

## Validation

The startup-stack workaround was added after reviewing your v1.3 crash log. Your subsequent log reaches the Assist pipeline, showing that the device has progressed past the earlier startup failure.

The PTT release revision also compiled successfully with ESPHome 2026.9.0. The new recognition/intent behaviour still needs confirmation on your device. Expected logs after release include `Released; microphone muted, waiting for silence detection`, `STT by VAD end`, `Speech recognised as` and `Intent started`; backend failures may instead produce an explicit Assist error.

Your subsequent 28 September log receives response audio but shows a busy I²S bus preventing speaker startup. The microphone-source release revision addresses that handoff and compiled successfully with ESPHome 2026.9.0 / ESP-IDF 5.5.5. Source-level checks passed for repeated stop calls and restarting capture for the next request. DEBUG logging is enabled, with routine light and sensor messages suppressed. After flashing, capture a full request from button press through the spoken response, including `audio_handoff` messages, and test a second request. Hardware confirmation of this revision is pending.

The Ethernet-priority revision passed ESPHome 2026.9.0 schema validation and the complete ESP32-P4 firmware build with ESP-IDF 5.5.5, including the Ethernet driver, network diagnostics, LVGL widgets and C++ lambdas. The display preview has been visually checked. The build used temporary test credentials; compile your own firmware after entering your credentials. No test firmware binary is included. No physical Guition board or your Home Assistant instance was available for hardware testing; audio, touch, Ethernet/Wi-Fi failover and entity integration still need checking on your unit.

## References

- [Guition product specifications](https://www.guition.com/esp32p4-display-module/secondary-portable-monitor).
- [Manufacturer schematic and example-code mirror for the exact model](https://github.com/wegi1/ESP32P4-JC1060P470C-I_W_Y), especially `5-Schematic/JC1060P470C_I_W_Y-V1.0.pdf` and the BSP header under the IDF common components.
- [ESPHome MIPI DSI display driver](https://esphome.io/components/display/mipi_dsi/).
- [ESPHome voice assistant and push-to-talk](https://esphome.io/components/voice_assistant/).
- [ESPHome speaker-source media player](https://esphome.io/components/media_player/speaker_source/).
- [ESPHome Sendspin hub](https://esphome.io/components/sendspin/).
- [ESPHome Sendspin media source](https://esphome.io/components/media_source/sendspin/).
- [ESPHome Sendspin group controller](https://esphome.io/components/media_player/sendspin/).
- [Music Assistant Sendspin setup](https://www.music-assistant.io/player-support/sendspin/).
- [Home Assistant text-to-speech](https://www.home-assistant.io/integrations/tts/).
- [ESPHome ES8311 codec](https://esphome.io/components/audio_dac/es8311/).
- [ESPHome LVGL](https://esphome.io/components/lvgl/).
- [ESPHome network priority and multi-interface support](https://esphome.io/components/network/#multi-interface-support).
- [ESPHome Ethernet](https://esphome.io/components/ethernet/).
- [ESPHome Ethernet diagnostics](https://esphome.io/components/text_sensor/ethernet_info/).
- [ESPHome native API and Home Assistant actions](https://esphome.io/components/api/).
