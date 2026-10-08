# tclac — ESPHome component for TCL (and compatible) mini-split air conditioners

Replaces the stock WiFi dongle on TCL and compatible mini-splits with a plain
ESP module running ESPHome, so the AC becomes a first-class Home Assistant
climate entity.

**This repo is a fork of
[I-am-nightingale/tclac](https://github.com/I-am-nightingale/tclac)** with
additions for English-speaking users and Fahrenheit-friendly behavior:

- Reads and writes **0.5°C target setpoints**. The AC's remote has always
  supported 0.5°C steps (encoded as a +0.5°C flag in status byte 9, bit 0);
  the upstream component read only whole °C and the extra half degree was
  silently lost.
- Default visual step is **0.5°C** (matches the hardware's native
  resolution).
- Writes include a half-flag so setpoints like 23.5°C land on the AC
  cleanly. Setpoints that fall on a whole °C still produce byte-identical
  frames to the upstream behavior — no regression for existing deployments.
- Setpoints are snapped to the 0.5°C grid in `control()` before publishing,
  so Home Assistant doesn't flicker between the raw request and the
  snapped-ack value on each click (noticeable in Fahrenheit, where 1°F ≈
  0.56°C).
- `fahrenheit_display` config option toggles the AC's **indoor-unit panel**
  between °C and °F display (byte 12 bit 7 of the TX frame).
- Documentation and user-facing strings translated to English. The
  original Russian source comments are preserved to keep merges with
  upstream clean.

## Credits

Full credit to the upstream author
([I-am-nightingale](https://github.com/I-am-nightingale)) and prior
contributors ([xaxexa](https://github.com/xaxexa),
[junkfix](https://github.com/junkfix),
[Pommel4711](https://github.com/Pommel4711)) for the component itself.
Original Russian README and the author's write-ups:
<https://dzen.ru/a/ZmdoyUNswXWnulhg>.

## Confirmed-working units

The upstream project has reports from these models. Compatibility is hard to
guarantee from the model name alone — the same model can ship with or
without a native WiFi module, a USB power lead, or even the UART header
soldered. If yours isn't listed, it still has a good chance of working as
long as there's a UART header on the control board.

- Axioma ASX09H1/ASB09H1
- Ballu BSAI-12HN1_15Y
- Ballu Discovery DC BSVI-07HN8
- Ballu Discovery DC BSVI-09HN8
- Ballu Discovery DC BSVI-12HN8
- Daichi AIR20AVQ1/AIR20FV1
- Daichi AIR25AVQS1R-1/AIR25FVS1R-1
- Daichi AIR35AVQS1R-1/AIR35FVS1R-1
- Daichi DA35EVQ1-1/DF35EV1-1
- Dantex RK-12SATI/RK-12SATIE
- Ecostar Radium KVS-RAD09CH
- iFFALCON F1 18
- Royal Clima Gloria Inverter
- Royal Clima Pandora RC-PDC28HN
- Tesla TT27TP61S-0932IAWUV
- TCL ELI ONF 12
- TCL Liferise ONF 09
- TCL TAC-CT09INV/R
- TCL One Inverter TACM-09HRID/E1 (pin order may differ)
- TCL TAC-07CHSA/TPG-W
- TCL TAC-09CHSA/TPG
- TCL TAC-09CHSA/DSEI-W
- TCL TAC-09HRID/E1
- TCL TAC-12CHSA/TPG
- TCL TAC-12CHSA/TPGI
- TCL TAC-XAL24I
- TCL TPG31IHB

Status frames of 61, 65 and 68 bytes are all parsed, but only the 61-byte
variant has been exercised extensively.

## Requirements

Home Assistant with ESPHome **2026.4.0 or newer**.

If you need an MQTT-based solution instead,
[pavel211/TCL-TAC-07-WiFi](https://github.com/pavel211/TCL-TAC-07-WiFi) is
a well-known alternative.

## Quickstart

The repo ships two sample ESPHome configurations:

- `TCL-Conditioner.yaml` — fully commented reference config.
- `Sample_conf.yaml` — minimal config.

Download one, edit the substitutions, and flash it. The reusable climate
logic lives in `packages/core.yaml` and is pulled from GitHub automatically
at build time, so you only maintain the device-specific bits locally
(hostname, Wi-Fi, pin assignments, optional add-ons).

### Platform selection

Set the platform block for your module. Only one block should be
uncommented at a time.

ESP-01S:
```yaml
esp8266:
  board: esp01_1m
```

Hommyn HDN/WFN-02-01 (ESP32-C3):
```yaml
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino
```

ESP32 WROOM32 (contributed by
[kai-zer-ru](https://github.com/kai-zer-ru)):
```yaml
esphome:
  platform: ESP32
  board: nodemcu-32s
```

Wemos D1 Mini (ESP12-F):
```yaml
esphome:
  platform: ESP8266
  board: esp12e
```

### Static IP (optional)

By default the device gets a DHCP lease. For a static IP, append this at
the bottom of the config:

```yaml
wifi:
  manual_ip:
    static_ip: 192.168.1.4
    gateway: 192.168.1.1
    subnet: 255.255.255.0
```

### Add-on packages

Optional pieces live in separate files under `packages/` and are included
by listing them under `files:` in your `packages:` block. The core package
is required; everything else is opt-in.

```yaml
packages:
  remote_package:
    url: https://github.com/luthcody/tclac.git
    ref: master
    files:
    # v - keep every file line indented to this column, otherwise ESPHome parses it as nested
      - packages/core.yaml       # required — the climate component
      # - packages/leds.yaml       # RX/TX activity LEDs on the pins named by receive_led / transmit_led
      # - packages/bad_connect.yaml  # repeats each command 3× for noisy links
      # - packages/uart_speed.yaml   # exposes a UART baud-rate selector in the device settings
    refresh: 0s
```

Alignment matters — every file line must be indented exactly as shown. The
wrong indentation produces confusing ESPHome parse errors. Example of what
**not** to do:

```yaml
packages:
  remote_package:
    url: https://github.com/luthcody/tclac.git
    ref: master
    files:
      - packages/core.yaml
        - packages/leds.yaml     # WRONG — extra indent breaks the parse
    refresh: 30s
```

## Fahrenheit display mode

If the indoor unit's panel should match a Home Assistant UI set to °F,
add this substitution to your device yaml:

```yaml
substitutions:
  fahrenheit_display: "true"
```

This flips bit 7 of TX byte 12 (`//fahrenheit...80=f 0=c`) and only
changes the physical display on the indoor unit. The over-the-wire
protocol stays in °C with 0.5°C granularity regardless of the display
mode.

## Attribution

Thanks to the original author, [I-am-nightingale](https://github.com/I-am-nightingale),
who created and maintains the upstream project. If this fork helps you,
please star the upstream repo too.

Upstream author's Steam account for thanks:
<https://steamcommunity.com/id/solovey-iron/>.
