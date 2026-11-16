# SAULUX firmware — WLED 16.0.1

The firmware base is the official **WLED v16.0.1** tag
(`29b389df1c1aaec6ff53aea742d17063b985906c`), with its ESP-IDF 4.4 / Arduino 2
platform. SAULUX features are ported onto that source; WLED 17 development changes
are not included. Fork releases use versions such as `16.0.1-saulux.1`.

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
   git tag saulux-v16.0.1-saulux.1
   git push origin saulux-v16.0.1-saulux.1
   ```

The tag suffix (`16.0.1-saulux.1` here) becomes the firmware and asset version.
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
