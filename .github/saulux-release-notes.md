SAULUX firmware based on the official **WLED 16.0.1** source, with its ESP-IDF 4.4 / Arduino 2 platform.

Three fixed hardware variants: **SAULUX 2I2O**, **SAULUX 5I4O** and **SAULUX ETHERNET**. Each has an ordinary OTA `.bin` and a corresponding `_factory.bin` for serial flashing. The web interface shows the installed model without a model selector.

Two-ended white light lines can use API presets `{"sauluxLine":"start","tt":20}`
and `{"sauluxLine":"end","tt":20}`. Each toggles from the current light state,
with white filling from that endpoint and darkness returning towards it. Assign
presets to the endpoint buttons; `tt:20` is a 2-second full pass. During a pass,
either button reverses the existing boundary without moving the illuminated side or
changing its speed. Repeated presses reverse it again, with no queued action.
Switching off preserves the running effect and its palette instead of replacing it
with white. A reversal reveals the same effect without restarting its runtime.
A new ON pass from fully off still selects white. The visible portion keeps its
brightness, avoiding a full-line flash or fade.

Button state publishing is available at `<Device Topic>/input/1/state` and subsequent
one-based input numbers: `1` = pressed, `0` = released. Enable MQTT and **Publish on
button press** in Sync Interfaces. States are debounced, retained and refreshed on
MQTT reconnect; existing click events and local actions are preserved.

**First transition from the previous SAULUX 17 / ESP-IDF 5 builds: back up configuration and presets, then install your board's 16.x `_factory.bin` by USB/serial at offset `0x0`.** This also replaces the newer bootloader. Installing only the 16.x application over a newer bootloader is not a supported ESP-IDF downgrade. Do not bypass firmware validation to force that OTA transition.

After the first 16.x factory installation, **Auto update** continues to download the matching hardware variant from this repository. Leave normal firmware validation enabled. The 16.x firmware compatibility family is `SAULUX_16`.

SAULUX branding, `saulux-<last six MAC characters>` AP/mDNS names, SK6812 RGBW defaults and the interface loading improvements are retained. The shared 4 MB flash layout works on 4 MB and 8 MB modules; the additional 4 MB on 8 MB modules is unused.

[Hardware, installation and update instructions](https://github.com/altdaps/WLED-SAULUX/blob/main/SAULUX.md)
