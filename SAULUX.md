# SAULUX firmware

This fork builds firmware for ESP32-WROOM-32 modules with **4 MB flash and no PSRAM**.
All three binaries contain all three hardware profiles. Their only difference is the
profile selected on first installation. Subsequent OTA updates preserve the saved
profile, GPIO settings, Wi-Fi credentials and presets.

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

## Select a device

Open **Config → LED Preferences → SAULUX device**, select the model and save.
The device saves the selection and restarts before configuring the new hardware.
Changing the model resets LED outputs, inputs, relay and global I2C/SPI pins.
It creates 30 WS2812/GRB LEDs per output, with consecutive logical LED ranges.
Set the actual LED type, length, color order and current limits after selecting the model.
Wi-Fi settings and stored presets are retained. Presets containing segment ranges
may need adjustment to the new LED layout.

Initial input configuration is momentary buttons, **active LOW with internal pull-ups**.
Change the button types and preset actions to match the connected equipment.
The relay defaults to **active HIGH**; its polarity is adjustable in Hardware setup.
The status LED defaults to active HIGH and has an **active LOW** option in the SAULUX section.
The status LED blinks at 1 Hz when connected and at 0.5 Hz in AP mode.
PHY address and status polarity changes also restart the device, preserving the LED/input settings.
Save changes to the SAULUX section separately from other LED Preferences changes.

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

The default environments are `saulux_2i2o`, `saulux_5i4o`, and `saulux_ethernet`.
Build just one with `pio run -e saulux_ethernet`.
Outputs are written to `build_output/release/`:

- `WLED_<version>_SAULUX_2I2O.bin`
- `WLED_<version>_SAULUX_5I4O.bin`
- `WLED_<version>_SAULUX_ETHERNET.bin`
- The corresponding `*_factory.bin` images for serial flashing at offset `0x0`.

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
The first SAULUX boot replaces the hardware mapping with the selected profile.

## GitHub Actions releases and OTA

1. Enable Actions in this fork if GitHub disabled inherited workflows.
2. In **Settings → Pages → Build and deployment → Source**, select **GitHub Actions**.
   If the `github-pages` environment restricts deployments, allow the `saulux-v*` tags.
3. Commit the source and push a release tag, for example:

   ```sh
   git tag saulux-v17.0.0-saulux.3
   git push origin saulux-v17.0.0-saulux.3
   ```

The tag suffix (`17.0.0-saulux.3` here) becomes the firmware and asset version.
If a tag push does not start a run, use **Actions → SAULUX Release → Run workflow**
on `main` and enter the existing tag. Manual runs build the exact tagged source.
Use a new, increasing version for each release. The workflow compiles and tests all
three variants, publishes a release with OTA/factory binaries and SHA256 checksums,
and deploys the same files plus `manifest.json` to:

`https://altdaps.github.io/WLED-SAULUX/`

After the **SAULUX Release** workflow completes, open **Config → Security & Updates
→ Manual OTA Update → Auto update** on a device. Its browser downloads the matching
binary from this fork's GitHub Pages and uploads it to the device. Keep the browser
open and connected to the device until the upload completes. Internet access is
needed on the browser; the ESP32 itself does not need outbound internet access.
Normal firmware validation stays enabled. Interrupted/failed downloads are not uploaded.

The compiled variant determines the update filename. Changing the model in the menu
does not change that filename: each variant supports every model, and the saved model
survives updates. No firmware is downloaded from the upstream repository or the WLED
proxy in SAULUX builds. Inherited upstream nightly publishing is disabled in this fork.

The latest successfully deployed tag is the OTA channel, including tags with suffixes.
A release can exist before its Pages deployment finishes; OTA stays on the previous
successfully deployed version until deployment succeeds. Re-run a failed deployment
job rather than creating the same release again.

## Validation

```sh
npm test
python3 -m unittest discover -s test -p 'test_saulux.py'
pio run
pio run -e esp32dev
```

Host tests exercise the actual SAULUX implementation with hardware shims: all GPIO
maps, model migration, retained settings, input validation and delayed reboot.
Updater tests cover matching variants, missing images, failed requests and truncated
downloads. Successful compilation checks the real ESP32 interfaces. Flashing, relay
polarity, input behavior and Ethernet link operation must be tested on the boards.

References: [WLED Ethernet profiles](wled00/network.cpp),
[Espressif Ethernet documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/network/esp_eth.html),
[GitHub Pages custom workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages).
