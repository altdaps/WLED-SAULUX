# SAULUX firmware

This fork builds firmware for ESP32-WROOM-32 modules with **4 MB flash and no PSRAM**.
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

Install Node.js 20+ and PlatformIO 6.1.19, then:

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

Each has a corresponding `_factory.bin` image with bootloader and partitions for
first serial flashing at offset `0x0`.

Use the ordinary `.bin` file for OTA. Never upload a factory image through OTA.
For an empty module, flash the factory image with esptool, for example:

```sh
python3 -m esptool --chip esp32 --port /dev/ttyUSB0 write-flash 0x0 WLED_<version>_SAULUX_2I2O_factory.bin
```

Replace the serial port and filename. Factory images include the actual bootloader,
partition table and application selected by PlatformIO. For a device already running
upstream WLED, back up its configuration and presets first. A first manual OTA
installation may require WLED's **Ignore firmware validation** checkbox because
SAULUX has a different release name. Keep validation enabled for subsequent updates.
When upgrading from universal versions `saulux.4`–`saulux.6`, manually upload the
ordinary `.bin` for your board once. **Leave firmware validation enabled:** all three
variants retain `SAULUX` as the firmware compatibility identifier. The earlier
universal update page cannot select the new variant filenames automatically.
If the saved applied profile matches the firmware, GPIO settings are retained.
A different or unconfigured profile is replaced with this firmware's hardware defaults.

Earlier `saulux.1`–`saulux.3` builds use model-specific compatibility identifiers;
their first manual upgrade to these variants may require **Ignore firmware validation**.
Keep validation enabled after that transition.

## GitHub Actions releases and OTA

1. Enable Actions in this fork if GitHub disabled inherited workflows.
2. In **Settings → Pages → Build and deployment → Source**, select **GitHub Actions**.
   If the `github-pages` environment restricts deployments, allow the `saulux-v*` tags.
3. Commit the source and push a release tag, for example:

   ```sh
   git tag saulux-v17.0.0-saulux.8
   git push origin saulux-v17.0.0-saulux.8
   ```

The tag suffix (`17.0.0-saulux.8` here) becomes the firmware and asset version.
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
as `otaRelease` in `/json/info`. The shared `release: "SAULUX"` field keeps normal
firmware validation compatible across all three variants and the earlier universal
build. Saved or restored model fields cannot redirect OTA to another variant.
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
downloads. Successful compilation checks the real ESP32 interfaces. Flashing, relay
polarity, input behavior and Ethernet link operation must be tested on the boards.

References: [WLED Ethernet profiles](wled00/network.cpp),
[Espressif Ethernet documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/network/esp_eth.html),
[GitHub Pages custom workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages).
