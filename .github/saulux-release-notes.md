SAULUX firmware based on the official **WLED 16.0.1** source, with its ESP-IDF 4.4 / Arduino 2 platform.

Four fixed hardware variants: **SAULUX 2I2O**, **SAULUX 5I4O**, **SAULUX ETHERNET** and **WB-MGU**. Each has an ordinary OTA `.bin` and a corresponding `_factory.bin` for serial flashing. The web interface shows the installed model without a model selector.

**Three built-in line presets on every model:** `Line START`, `Line END` and
`Line CENTER` appear automatically at boot. Each uses segment 0, white startup,
live color controls and a one-second full pass; repeated calls reverse/toggle it.
Fresh installations use IDs 1–3. Upgrades use free IDs without replacing user
presets or applying a scene. Marked renames/edits persist; deleted built-ins return
at the next boot. Full or corrupt files are not overwritten.

**New WB-MGU build (WB-MGE v.3 pinout):** LED outputs GPIO10/14, permanently
asserted transmit enables GPIO4/15, RTL8201 Ethernet at address 0, and a reserved
TCA9535 bus on SDA32/SCL33. MIO reset uses P10/linear bit 8 and the production
firmware's LOW polarity; terminators, failsafe bias and Vout stay disabled.
Config button GPIO34 and Wi-Fi/Ethernet/Status indicators are supported.
The different WB-MGU v.1/WBE2 hardware is not targeted. First installation from
another WLED build initializes two 30-pixel SK6812 RGBW outputs; reconfigure
existing matrix geometry, LED types and lengths after migration. Wi-Fi and presets remain.

**Transitions during live power curtains:** keep-scene mode now renders both
old and new scenes, preserving native color/effect transitions such as Outside In
while the independent center power curtain fills, fades out or reverses.
Power reversals no longer terminate an ongoing native scene transition.
Use one second as HA `transition:1`, or WLED `tt:10` / `transition:10`.

**White startup with color controls:** use `"sauluxLineKeep":true` together with
`"sauluxLineWhite":true` to start a fresh ON pass in white and retain brightness.
Live color/brightness changes remain available. OFF and reversals keep their
current color, so an ongoing pass never jumps back to white.

**Color-preserving curtains:** add `"sauluxLineKeep":true` to resume the existing
color, effect and segment brightness on ON and accept live color/brightness changes
while filling. The default white-ON behavior is retained without the flag.
An HA Template Light can use this mode for center filling, idempotent power controls,
color and brightness through a single Yandex Smart Home light device.

**Home Assistant preset compatibility:** saved `sauluxLine` commands now convert
numeric `seg` selectors to the standard object form, such as `"seg":{"id":0}`.
The live API still accepts `"seg":0`. Resave older numeric presets to make them
appear in HA's Preset selector. Saving does not execute the command or expand
its scene, and an omitted selector remains dynamic.

**Time zone** now includes **Moscow (UTC+3)** and **Saratov (UTC+4)**, both without daylight saving time. Choose the city in **Config → Time & Macros**; set the additional **UTC offset** to **0** if you previously entered the city's offset manually. Upgrading preserves the existing time settings.

Two additional directional line presets are available:
`{"sauluxLine":"center","seg":{"id":0},"tt":20}` fills from the center to both edges;
`{"sauluxLine":"edges","seg":{"id":0},"tt":20}` fills from both edges to the center.
Calling either again switches off in reverse. During a pass, any line direction
reverses its current shape continuously, retaining its speed and running palette.
The center is calculated independently for each segment. Existing `start` / `end`
presets and button assignments are unchanged.

Directional curtains work independently per segment. Use
`{"sauluxLine":"start","seg":{"id":0},"tt":20}` and the corresponding `"end"` command;
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
