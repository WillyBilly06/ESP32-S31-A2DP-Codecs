# ESP32-S31 A2DP Audio Sink — Multi-Codec

A high-resolution Bluetooth A2DP audio receiver for the **ESP32-S31** (N16R16V), built on a
modified **ESP-IDF v6.1** Bluedroid stack. Unlike stock ESP-IDF — which decodes only SBC — this
project decodes the full range of A2DP codecs **in-stack**, streams the PCM straight to I2S, and
runs **entirely on internal SRAM** (no PSRAM required).

## Supported codecs

| Codec | Rates / depth | Notes |
|-------|---------------|-------|
| **LDAC** | up to 96 kHz / 32-bit | 660 / 909 / 990 kbps |
| **LHDC v5** | up to 192 kHz / 32-bit | 400–1000 kbps |
| **aptX / aptX HD / aptX LL** | up to 48 kHz / 24-bit | |
| **Opus** | 48 kHz | |
| **LC3plus** | up to 96 kHz | |
| **AAC** | up to 48 kHz | Helix decoder |
| **SBC** | 44.1 / 48 kHz | stock baseline |

## Highlights

- **In-stack decoding.** Decoders are integrated into the AOSP-style Bluedroid A2DP path
  (`A2DP_GetDecoderInterface` / `tA2DP_DECODER_INTERFACE`), the same approach used on my v5.5.2
  build — not the v6.1 external-codec framework.
- **Internal-SRAM only.** The design fits in the ESP32-S31's internal RAM. Heavy codec tables live
  in flash (`.rodata`) and are lazily allocated to RAM only while that codec is active, then freed
  on codec switch or disconnect, so only the working set is ever resident.
- **Dual-core scheduling.** The audio decode task is pinned to core 1 (nearly alone), while the BT
  controller, host, and I2S render task share core 0. This keeps heavy paths (e.g. LHDC v5 at
  1000 kbps) from starving the controller or overflowing the decode queue.
- **Exact I2S pipeline.** 32-bit Philips slots, APLL clocking (with a graceful fallback) for the
  high sample rates, and a prefetch/processing ring buffer ported from my working v5.5.2 sink.
- **Clean stream teardown.** I2S output stops and the ring buffer is cleared the instant the phone
  pauses or disconnects — no residual audio tail.

## Hardware

- **MCU:** ESP32-S31 (RISC-V dual-core), N16R16V — 16 MB flash, 16 MB PSRAM (PSRAM unused).
- **Output:** external I2S DAC (BCLK / LRCK / DIN). Configure the pins in the app's Kconfig.

## Repository layout

```
ESP32-S31-A2DP-Codecs/
├── S31_A2DP_Sink/          # the application project (main, sdkconfig.defaults, partitions)
└── esp-idf-v6.1-codecs/    # the forked ESP-IDF v6.1 with the modified Bluedroid codec stack
```

The fork is based on ESP-IDF `release/v6.1`. All of my changes live under
`esp-idf-v6.1-codecs/components/bt/` (the decoders, codec libraries, and A2DP plumbing) and the
`a2dp_utils` example components (the I2S service).

## Building

```bash
# 1. Set up the forked IDF as your IDF_PATH
cd esp-idf-v6.1-codecs
./install.ps1            # or install.sh
. ./export.ps1           # dot-source so idf.py is wired into the shell

# 2. Build & flash the app
cd ../S31_A2DP_Sink
idf.py set-target esp32s31
idf.py build
idf.py -p <PORT> flash monitor
```

> The application **must** be built against this fork — the codec decoders are not present in a
> stock ESP-IDF checkout.

## Codec licensing

This repository redistributes vendor codec sources (LDAC, aptX, LHDC, and others). These are subject
to their respective owners' licenses. Ensure you have the necessary rights before using or
redistributing them.
