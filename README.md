# Smart Ceiling Fan Bridge

Bringing a 15+ year old RF-controlled ceiling fan into Home Assistant using an ESP32 and ESPHome — no fan replacement required.

## The Problem

My ceiling fan predates smart home technology by well over a decade. It's controlled entirely by a handheld RF remote — no wall switch integration, no existing smart control path, and replacing the fan or its receiver wasn't a practical (or desirable) option. The goal was to bring it into Home Assistant without touching the fan itself.

## The Approach

An ESP32, flashed with [ESPHome](https://esphome.io/), acts as an RF bridge — capable of both receiving the original remote's signals and transmitting them on command. This lets Home Assistant fully control the fan (and still lets the original remote work alongside it).

Codes were captured directly with the ESP32 itself, before any transmit logic was built. A dedicated ESPHome config (see [`esphome/rf-capture.yaml`](esphome/rf-capture.yaml)) configured the CC1101's GDO0 pin as a `remote_receiver`, dumping both Pronto hex and raw microsecond pulse-timing output to the logs. From there, each button on the original remote was pressed individually and the resulting code was logged and documented, building up a full command map (fan on/off, each speed, light toggle, etc.) that the transmit side later replays on demand.

```
[Original Remote] --RF--> [ESP32 + RF Receiver/Transmitter] <--RF--> [Fan Receiver]
                                    |
                              ESPHome (YAML)
                                    |
                              Home Assistant
                                    |
                              Automations / Voice / Scheduling
```

## Hardware

| Component        | Details                          |
|-------------------|-----------------------------------|
| Microcontroller   | ESP-WROOM-32 (ESP32-S)            |
| RF Module         | CC1101 (SPI, sub-GHz transceiver) |
| Enclosure         | <!-- optional --> |

### Wiring

The CC1101 communicates with the ESP32 over SPI:

| CC1101 Pin   | ESP32 Pin           | Purpose                          |
|--------------|----------------------|-----------------------------------|
| VCC          | 3.3V                 | Power (never use 5V)              |
| GND          | GND                  | Ground                            |
| GDO0         | GPIO 4 (or free GPIO)| Digital I/O — data for TX/RX      |
| CSN (CS)     | GPIO 5 (SS/CS)       | SPI chip select                   |
| SCK (CLK)    | GPIO 18 (SCK)        | SPI clock                         |
| MOSI         | GPIO 23 (MOSI)       | SPI Master Out, Slave In          |
| MISO         | GPIO 19 (MISO)       | SPI Master In, Slave Out          |
| GDO2         | Leave disconnected   | Optional — unused for basic OOK   |

See [`docs/photos/`](docs/photos) for build photos and [`docs/wiring-diagram.png`](docs/wiring-diagram.png) for a visual reference of the table above.

### Secrets

This config references WiFi credentials, the Home Assistant API encryption key, the OTA password, network details, and the captured RF codes via ESPHome's `!secret` mechanism. Copy [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) to `esphome/secrets.yaml`, fill in your own values, and it'll be excluded from git automatically (see `.gitignore`).

## ESPHome Configuration

The full config is in [`esphome/fan-bridge.yaml`](esphome/fan-bridge.yaml) (secrets redacted). It defines:

- The RF receiver component, used to capture and log incoming codes from the original remote
- The RF transmitter component, used to replay captured codes on command
- Exposed entities for fan power and speed, visible directly in Home Assistant

The initial code-capture pass used a separate, minimal config ([`esphome/rf-capture.yaml`](esphome/rf-capture.yaml)) with the `remote_receiver` tuned specifically for reliable capture: tightened tolerance to cut ambient RF noise, a filter to drop high-frequency static, an idle threshold to mark packet boundaries, and a larger buffer to hold longer raw pulse-timing arrays.

## Home Assistant Integration

Once exposed via ESPHome's native API, the fan appears as a standard entity in Home Assistant. From there:

- <!-- e.g. "Controllable via the standard HA dashboard fan card" -->
- <!-- e.g. "Voice control via [Assistant/Alexa/etc.]" -->
- <!-- e.g. "Scheduled automations — e.g. auto-off after X hours" -->

See [`home-assistant/automations.yaml`](home-assistant/automations.yaml) for example automations built on top of this integration.

## What I'd Improve Next

- **Move the manual bit-decoding into a reusable component.** Right now the fan-specific decode logic (the `on_raw` lambda that turns pulse timings into a boolean array and matches known patterns) is hand-rolled per device. Generalizing that into an ESPHome external component would make it reusable for other legacy RF devices beyond just this fan.
- **Add a debug/learning mode.** The debounce and repeat-confirmation logic (requiring the same code 3x before accepting it) works well for reliability, but exposing a toggle to log raw pattern matches during setup would make it faster to onboard a new remote/fan combo.
- **Automated tests for the decode logic.** The bit-pattern matching is pure logic and could be pulled out and unit tested against captured raw sequences, rather than relying on live RF signals to verify correctness.

## License

MIT
