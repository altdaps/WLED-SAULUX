# SAULUX firmware — WLED 16.0.1

[Detailed Russian user guide](SAULUX_RU.md).

The firmware base is the official **WLED v16.0.1** tag
(`29b389df1c1aaec6ff53aea742d17063b985906c`), with its ESP-IDF 4.4 / Arduino 2
platform. SAULUX features are ported onto that source; WLED 17 development changes
are not included. Fork releases use versions such as `16.0.1-saulux.10`.

This fork builds firmware for ESP32-WROOM-32 modules with **4 MB or 8 MB flash and
no PSRAM**. The three variants share a **4 MB flash layout** so the
same model-specific image can be installed on either flash capacity.
Each release contains three application images with fixed hardware profiles:
**SAULUX 2I2O**, **SAULUX 5I4O** and **SAULUX ETHERNET**. A fresh installation
configures its board immediately. The installed firmware determines the model;
there is no model selector in the web interface. OTA updates for the same variant
preserve GPIO settings, Wi-Fi credentials and presets. Upgrading a universal build
or restoring a configuration cannot select another model. If the previously applied
profile differs, this firmware initializes its own hardware defaults while keeping
Wi-Fi credentials and presets.

| Profile | Input GPIOs, in connector order | LED output GPIOs | Relay GPIO | Status GPIO |
| --- | --- | --- | --- | --- |
| SAULUX 2I2O | 18, 4 | 17, 16 | 2 | None |
| SAULUX 5I4O | 27, 26, 25, 33, 32 | 18, 19, 21, 22 | 12 | 23 |
| SAULUX ETHERNET | 32, 33 | 16 | None | 2 |

The Ethernet profile uses IP101GRI: RX0=25, RX1=26, CRS_DV=27, external 50 MHz
REFCLK input=0, RESET=5, MDIO=18, TX0=19, TX_EN=21, TX1=22, MDC=23.
It reuses WLED's `ESP32-ETHERNET-KIT-VE` configuration. The PHY address defaults
to **1** and can be changed to 0–31 in the SAULUX settings.
Ethernet remains disabled by default on the other two profiles.

## Device identity and LED defaults

The device name and product brand are **SAULUX**. The access point name and mDNS
hostname are generated from the last six hexadecimal characters of the board's
STA/base MAC address, without colons, in lowercase. For MAC `AA:BB:CC:12:34:56`:

- Access point: `saulux-123456`
- mDNS URL: `http://saulux-123456.local/`

These network names are derived from the current board at boot and cannot be edited
in Wi-Fi settings. Restoring another board's configuration retains this board's
unique network identity. Upgrading an earlier build renames its default `WLED`
display name to `SAULUX`; explicitly customized display names are kept.

All three profiles default to **SK6812 RGBW**, color order **GRBW**, 30 LEDs per
output. The first upgrade from earlier SAULUX defaults changes existing WS2812
outputs to SK6812 RGBW while keeping their GPIOs, counts, color order and power
limits. This migration runs once; later explicit LED type changes remain saved.
WLED project credits, license information and protocol names remain in the interface.

## Device configuration

Install the firmware matching your board: 2I2O, 5I4O or ETHERNET. The model is shown
as read-only information in **Config → LED Preferences → SAULUX device**.
Changing the model requires uploading the corresponding firmware image. When the
previously applied model differs, LED outputs, inputs, relay and global I2C/SPI pins
reset to that variant's defaults: 30 SK6812 RGBW/GRBW LEDs per output, with consecutive
logical LED ranges. Set actual LED lengths, color order and current limits in LED
Preferences. Wi-Fi settings and stored presets are retained; presets containing
segment ranges may need adjustment after a hardware variant change.

Initial input configuration is momentary buttons, **active LOW with internal pull-ups**.
Change the button types and preset actions to match the connected equipment.
The relay defaults to **active HIGH**; its polarity is adjustable in Hardware setup.
The status LED defaults to active HIGH and has an **active LOW** option in the SAULUX section.
The status LED blinks at 1 Hz when connected and at 0.5 Hz in AP mode.
PHY address and status polarity changes also restart the device, preserving the LED/input settings.
The **Apply settings** button applies only the PHY/status fields; **Save** applies other LED preferences.

**Hardware verification is still required:** confirm input/relay polarity, PHY address,
external REFCLK, actual flash capacity and boot strap levels (including GPIO12)
on the real boards before use. Local tests and firmware compilation do not establish
correct electrical operation.

## Time settings

**Config → Time & Macros → Time zone** includes **Moscow (UTC+3)** (ID 25)
and **Saratov (UTC+4)** (ID 26), both without daylight saving time. Enable NTP,
select the city and save. **UTC offset** is an additional correction in seconds;
set it to **0** if it previously supplied the city's offset manually.
Existing timezone selections and manual offsets are preserved on upgrade.

The fixed offsets follow [IANA Europe/Moscow and Europe/Saratov](https://data.iana.org/time-zones/tzdb/europe).

## Independent directional light lines

Create non-overlapping segments in the web interface, then save endpoint commands
as custom API presets and assign them to the corresponding buttons in Time & Macros:

```json
{"sauluxLine":"start","seg":{"id":0},"tt":20}
{"sauluxLine":"end","seg":{"id":0},"tt":20}
{"sauluxLine":"start","seg":{"id":1},"tt":20}
{"sauluxLine":"end","seg":{"id":1},"tt":20}
```

`seg.id` is the existing segment ID, starting at zero. Live API commands also
accept the numeric shorthand `"seg":1`. Omit `seg` to use the current main segment;
existing one-line presets remain usable, including their old expanded fields.
Each command addresses one segment. It does not create segments or apply other
cached scene or master fields that an older firmware added to a preset.
Saving a custom API preset keeps its command rather than expanding its scene.
Starting with `.10`, numeric segment selectors are saved as `{"id":N}` so
Home Assistant's WLED client can parse and list them. Existing numeric presets
need to be resaved; an omitted selector still uses the main segment dynamically.
See [python-wled's preset model](https://github.com/frenck/python-wled/blob/v1.1.1/src/wled/models.py).

The following additional API presets select symmetric passes:

```json
{"sauluxLine":"center","seg":{"id":0},"tt":20}
{"sauluxLine":"edges","seg":{"id":0},"tt":20}
```

| Direction | New ON pass | OFF pass |
| --- | --- | --- |
| `start` | First LED to last LED | Last LED to first LED |
| `end` | Last LED to first LED | First LED to last LED |
| `center` | Center to both edges | Both edges to center |
| `edges` | Both edges to center | Center to both edges |

The center is local to the addressed segment. Odd visible/dark counts may give
one side one extra LED. These are `sauluxLine` preset commands, not cycling
entries in the Effects list. Assign any direction to a button or call it via API.

`start` is the first LED of that segment, and `end` is the opposite end. `tt` sets
the full-pass time in tenths of a second (0–655); 20 gives a two-second full pass.
Assign the same preset to short, long and double press on a button for immediate
response on its rising edge. A held button acts once until released.

A new ON pass selects Solid white, using the dedicated W channel on RGBW strips
(RGB white otherwise). An OFF pass keeps the running effect, palette, colors,
opacity and geometry, while the visible region shrinks in the reverse of its ON direction.
During a pass, any direction reverses the existing boundary or boundaries without
moving the illuminated area. The current shape remains until the pass finishes;
a later command then chooses a fresh direction. Its speed stays constant, and the return time is proportional
to the distance. Other segments retain their effects and independent timers;
several segments can fill, fade out or reverse simultaneously.

**Master power stays on even when every segment is off.** This lets Home Assistant
control the segment lights independently. Their ON flags report their targets
immediately while the outgoing pixels finish their curtain. If the master was
off, a line ON command restores master brightness and enables only its segment;
other segments remain off. An explicit master OFF still turns everything off.

Ordinary segment commands from the UI or Home Assistant override only that
segment's curtain. Changing master power or brightness overrides all curtains.
Segment selection and names do not interrupt them. A nightlight is cancelled by
a line command. Deleted or resized segments cannot retain a stale curtain.
A separate ordinary OFF preset may still be selected as the boot preset.

## Button states over MQTT

Configure the broker and **Device Topic** in **Config → Sync Interfaces → MQTT**,
enable MQTT and **Publish on button press**, save and reboot. The SAULUX publisher
uses the configured device topic; the AP/mDNS hostname does not replace this setting.

Each physical input publishes a logical state in its connector order:

| Hardware | State topics |
| --- | --- |
| SAULUX 2I2O | `<Device Topic>/input/1/state` through `/input/2/state` |
| SAULUX 5I4O | `<Device Topic>/input/1/state` through `/input/5/state` |
| SAULUX ETHERNET | `<Device Topic>/input/1/state` through `/input/2/state` |

Payload **`1` means pressed**, **`0` means released**. The default active-LOW wiring
therefore publishes `1` when the input is pulled to ground. A configured active-HIGH
button uses its corresponding logical polarity. States follow the first configured
button entries, in the same order as the board connectors by default.

States are debounced for 50 ms, published at QoS 1 and retained independently of the
brightness/color retain option. The device sends an initial snapshot and refreshes
all input states after every MQTT connection. It publishes subsequent changes
without repeating messages while a button is held. Rejected queue entries are
retried with a bounded interval using the latest state.

Disabling a button, changing it to an unsupported type such as analog, or disabling
**Publish on button press** clears the corresponding retained input states when the
broker is connected. Existing `button/0`, `button/1`, etc. click events (`short`,
`long`, `double`) and local button actions are preserved; their numbering starts
at zero. These event topics are separate from the new one-based input state topics.

Retained input values describe the last known state. Consumers should also use
`<Device Topic>/status` (`online` / `offline`) for device availability.

## Build locally

Install Node.js 20+ and PlatformIO 6.1.17, then:

```sh
npm ci
npm run build
pio run
```

The default environments are `saulux_2i2o`, `saulux_5i4o` and `saulux_ethernet`.
Build one with, for example, `pio run -e saulux_5i4o`.
Outputs are written to `build_output/release/`:

| Board | OTA application |
| --- | --- |
| SAULUX 2I2O | `WLED_<version>_SAULUX_2I2O.bin` |
| SAULUX 5I4O | `WLED_<version>_SAULUX_5I4O.bin` |
| SAULUX ETHERNET | `WLED_<version>_SAULUX_ETHERNET.bin` |

The shared partition table is `tools/WLED_ESP32_4MB_1MB_FS.csv`: two 1.5 MiB OTA
application slots and a 960 KiB filesystem, all within the first 4 MiB. On an 8 MB
module installed with the factory image, the additional 4 MiB is unused. A larger
flash chip alone does not enlarge the firmware slots or filesystem. No separate
8 MB binaries or flash-size menu selection are needed for this layout.

Ordinary OTA application updates do not replace the installed partition table.
Using the full 8 MB would require an 8 MB layout and a separate installation/migration
plan; it is not enabled by these shared images.

Each has a corresponding `_factory.bin` image with bootloader and partitions for
first serial flashing at offset `0x0`.

Use the ordinary `.bin` file for OTA. Never upload a factory image through OTA.
For an empty module, flash the factory image with esptool, for example:

```sh
python3 -m esptool --chip esp32 --port /dev/ttyUSB0 write-flash 0x0 WLED_<version>_SAULUX_2I2O_factory.bin
```

Replace the serial port and filename. Factory images include the actual bootloader,
partition table and application selected by PlatformIO.

## Transition from SAULUX 17

**The first transition from the previous `17.0.0-saulux.*` / ESP-IDF 5 images requires
serial flashing of the matching 16.x factory image.** Save configuration and presets
from the device's backup page first. This replaces the newer bootloader together
with the application; uploading only the application by OTA is not a supported
ESP-IDF downgrade. After serial installation, restore compatible settings/presets
and verify LED lengths, GPIOs and effects on the board.

The 16.x builds use the separate firmware compatibility identifier **`SAULUX_16`**.
Normal firmware validation rejects direct OTA between the previous `SAULUX` family
and this family. **Do not bypass this with Ignore firmware validation.** Normal
validation remains enabled for subsequent 16.x Auto updates from this repository.
The three OTA asset filenames still identify the hardware model, as listed above.

Ordinary OTA retains the installed partition table. A factory image written at
`0x0` also includes a fresh OTA-selection record for a consistent first boot.
See [Espressif bootloader compatibility](https://docs.espressif.com/projects/esp-idf/en/v5.2/esp32/api-guides/bootloader.html#bootloader-compatibility).

## GitHub Actions releases and OTA

1. Enable Actions in this fork if GitHub disabled inherited workflows.
2. In **Settings → Pages → Build and deployment → Source**, select **GitHub Actions**.
   If the `github-pages` environment restricts deployments, allow the `saulux-v*` tags.
3. Commit the source and push a release tag, for example:

   ```sh
   git tag saulux-v16.0.1-saulux.10
   git push origin saulux-v16.0.1-saulux.10
   ```

The tag suffix (`16.0.1-saulux.10` here) becomes the firmware and asset version.
If a tag push does not start a run, use **Actions → SAULUX Release → Run workflow**
on `main` and enter the existing tag. Manual runs build the exact tagged source.
Use a new, increasing version for each release. The workflow compiles and tests the
three firmware variants, publishes their OTA/factory images and SHA256 checksums, and deploys
the same files plus `manifest.json` to:

`https://altdaps.github.io/WLED-SAULUX/`

After the **SAULUX Release** workflow completes, open **Config → Security & Updates
→ Manual OTA Update → Auto update** on a device. Its browser downloads the matching
binary from this fork's GitHub Pages and uploads it to the device. Keep the browser
open and connected to the device until the upload completes. Internet access is
needed on the browser; the ESP32 itself does not need outbound internet access.
Normal firmware validation stays enabled. Interrupted/failed downloads are not uploaded.

Each board downloads the application matching its compiled hardware variant, exposed
as `otaRelease` in `/json/info`. The shared `release: "SAULUX_16"` field enables normal
firmware validation across these three 16.x variants while separating them from
the previous ESP-IDF 5 family. Saved or restored model fields cannot redirect OTA to another variant.
No firmware is downloaded from the upstream repository or the WLED proxy in SAULUX
builds. Inherited upstream nightly publishing is disabled in this fork.

The latest successfully deployed tag is the OTA channel, including tags with suffixes.
A release can exist before its Pages deployment finishes; OTA stays on the previous
successfully deployed version until deployment succeeds. Re-run a failed deployment
job rather than creating the same release again.

## Interface loading

The interface has no development-build overlay. Controls and presets become
available before palette previews finish downloading. The WebSocket control channel
opens after the initial state is received, and an unavailable optional `skin.css`
file does not prevent control from starting. HTTP API reads remain sequential.

## Validation

```sh
npm test
python3 -m unittest discover -s test -p 'test_saulux.py'
pio run
pio run -e esp32dev
```

Host tests exercise the actual SAULUX implementation with hardware shims: all GPIO
maps for each fixed variant, migration from other profiles, retained settings, input
validation and delayed reboot.
Updater tests cover matching variants, missing images, failed requests and truncated
downloads. Native OTA validation tests cover family separation, valid names and
unterminated names. Successful compilation checks the real ESP32 interfaces. Flashing, relay
polarity, input behavior and Ethernet link operation must be tested on the boards.

References: [WLED Ethernet profiles](wled00/network.cpp),
[Espressif Ethernet documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/network/esp_eth.html),
[GitHub Pages custom workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages).
