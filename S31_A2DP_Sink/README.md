# bt_audio_sink_s31 — Bluetooth A2DP receiver → I2S (ESP32-S31)

A minimal Bluetooth **Classic A2DP sink** for the **ESP32-S31-N16R16V**. A phone/PC
connects over Bluetooth and streams music; the chip decodes it (SBC) and outputs
PCM over an external **I2S** DAC. Runs **entirely on internal SRAM** — PSRAM is
intentionally left unused.

This is **Stage A**: a simple, working receiver using the stock SBC codec.
The external codecs (LDAC / aptX / LHDC-v5 / Opus / LC3) are a later stage that
plugs into the `a2dp_sink_ext_codec_utils` path.

## Target & memory model

| Item | Value |
|------|-------|
| Chip | `esp32s31` |
| Flash | 16 MB (`N16`) |
| PSRAM | present on board (`R16V`) but **disabled** — `CONFIG_SPIRAM=n` |
| RAM model | everything in the 512 KB internal SRAM |
| BT mode | Classic only (BR/EDR), BLE off |
| Codec | SBC (internal decode) |
| Output | external I2S, 16-bit stereo |

Building against the **forked IDF** `../esp-idf-s31-audio` (v6.1 + S31 support).
Do **not** build this with the old v5.5.2 IDF.

## Build & flash

> **Submodules:** the fork's git submodules must be checked out or the build
> fails (e.g. `components/mbedtls/mbedtls/include is not a directory`). This has
> already been done for `esp-idf-s31-audio`. On a fresh IDF clone run:
> `git submodule update --init --recursive` first.

```powershell
# 1. Activate the forked IDF (installs S31 tools on first run)
D:\ESP32-S31\esp-idf-s31-audio\install.ps1 esp32s31      # first time only
D:\ESP32-S31\esp-idf-s31-audio\export.ps1

# 2. Build
cd D:\ESP32-S31\bt_audio_sink_s31
idf.py set-target esp32s31
idf.py build

# 3. Flash + monitor (replace COMx with your port)
idf.py -p COMx flash monitor
```

Then pair from your phone/PC with the device named **`ESP32-S31-Speaker`** and play audio.

## I2S pin wiring

Default pins (set in `sdkconfig.defaults.esp32s31`) — change to match your DAC,
then rebuild:

| Signal | Config | Default GPIO |
|--------|--------|--------------|
| Bit clock (BCLK) | `CONFIG_EXAMPLE_I2S_BCK_PIN` | 2 |
| Word select (LRCK/WS) | `CONFIG_EXAMPLE_I2S_LRCK_PIN` | 3 |
| Data (DIN) | `CONFIG_EXAMPLE_I2S_DATA_PIN` | 4 |

Or run `idf.py menuconfig` → *A2DP Example Configuration* / *Audio Output*.

## What's where

| File | Purpose |
|------|---------|
| `main/main.c` | GAP + A2DP sink setup, registers the internal-codec I2S data path |
| `sdkconfig.defaults` | BT Classic + A2DP sink, BLE off, PSRAM off, device name |
| `sdkconfig.defaults.esp32s31` | 16 MB flash, `SPIRAM=n`, I2S output + pins, internal codec |
| `CMakeLists.txt` | project `bt_audio_sink_s31`; pulls tested A2DP-sink util components from the fork |

The A2DP-sink helper components (`bt_app_core_utils`, `bredr_app_common_utils`,
`a2dp_sink_common_utils`, `a2dp_sink_int_codec_utils`) resolve from
`${IDF_PATH}/examples/...` in the fork — they are Espressif's tested components,
reused unmodified. The I2S service reconfigures the I2S clock to the negotiated
SBC sample rate (16/32/44.1/48 kHz) automatically.

## Notes / caveats

- **Not yet build-verified in the authoring environment** (the v6.1 toolchain
  was not installed there). The configuration has been checked for correctness
  against the fork's sources; run the build steps above on your machine.
- The S31 has no internal DAC; the example's DAC service compiles to an empty
  stub on this target (I2S output only).
- Next stage: enable `CONFIG_EXAMPLE_A2DP_SINK_USE_EXTERNAL_CODEC` and add the
  vendored LDAC/aptX/LHDC/Opus/LC3 decoders into the fork's `bt` component.
