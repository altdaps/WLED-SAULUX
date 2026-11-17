SAULUX firmware based on the official **WLED 16.0.1** source, with its ESP-IDF 4.4 / Arduino 2 platform.

Three fixed hardware variants: **SAULUX 2I2O**, **SAULUX 5I4O** and **SAULUX ETHERNET**. Each has an ordinary OTA `.bin` and a corresponding `_factory.bin` for serial flashing. The web interface shows the installed model without a model selector.

Endpoint curtains now work independently per segment. Use
`{"sauluxLine":"start","seg":0,"tt":20}` and the corresponding `"end"` command;
change `seg` for another line, or omit it for the main segment. Multiple segments
can animate and reverse concurrently, preserving their current effects and palettes
when switching off. A fresh ON pass still selects white.

Master power stays on when individual segments switch off. From master OFF,
only the requested segment is enabled. Standard segment controls override only
that segment; master power and brightness controls override all curtains.
Custom API presets are saved as commands, without runtime expansion.

Button state publishing is available at `<Device Topic>/input/1/state` and subsequent
one-based input numbers: `1` = pressed, `0` = released. Enable MQTT and **Publish on
button press** in Sync Interfaces. States are debounced, retained and refreshed on
MQTT reconnect; existing click events and local actions are preserved.

**First transition from the previous SAULUX 17 / ESP-IDF 5 builds: back up configuration and presets, then install your board's 16.x `_factory.bin` by USB/serial at offset `0x0`.** This also replaces the newer bootloader. Installing only the 16.x application over a newer bootloader is not a supported ESP-IDF downgrade. Do not bypass firmware validation to force that OTA transition.

After the first 16.x factory installation, **Auto update** continues to download the matching hardware variant from this repository. Leave normal firmware validation enabled. The 16.x firmware compatibility family is `SAULUX_16`.

SAULUX branding, `saulux-<last six MAC characters>` AP/mDNS names, SK6812 RGBW defaults and the interface loading improvements are retained. The shared 4 MB flash layout works on 4 MB and 8 MB modules; the additional 4 MB on 8 MB modules is unused.

[Hardware, installation and update instructions](https://github.com/altdaps/WLED-SAULUX/blob/main/SAULUX.md)
