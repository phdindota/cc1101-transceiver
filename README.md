# CC1101 Dual-Band RF Transceiver for ESPHome

Control **433.92 MHz and 315 MHz** devices from Home Assistant with a **single**
CC1101 module and an ESP32, using ESPHome's native `cc1101` component.

Learn signals from your remotes, replay them, and expose blinds, ceiling fans,
outlets and other fixed-code RF devices as Home Assistant entities.

---

## Features

- **Two bands, one radio** — the CC1101 retunes per transmission; no second module
- **Per-button frequency** — each button declares its own band, so 433 and 315 devices coexist
- **Learning mode** — capture a signal and replay it on the band it arrived on
- **Copy-paste logging** — every received burst is logged as a ready-to-paste `code:` line
- **Band scanning** — optionally alternate between bands while listening
- **Fan entity** — a template fan with speeds and Low/Medium/High presets
- **Home Assistant native** — buttons, a fan, a select, sensors, plus a web interface

---

## Requirements

| Component | Notes |
|-----------|-------|
| ESP32 board | Any ESP32; `esp32dev` by default |
| CC1101 module | Buy the version tuned for your **main** band |
| Antenna | Sized for your main band (17.3 cm at 433 MHz, 23.8 cm at 315 MHz) |
| ESPHome | **2026.1 or newer** — earlier versions can't retune while running |

> The CC1101 chip covers 300–928 MHz, but each module's matching network and
> antenna are tuned for one band. A 433 MHz module still works at 315 MHz, with
> reduced range. If both bands must reach far, use two modules.

---

## Wiring — Dual Pin Mode

Separate pins for transmit and receive, so no mode switching is needed.

```
 ESP32                       CC1101
 ┌──────────┐              ┌──────────────┐
 │     3V3  ├──── Red ─────┤ VCC          │
 │     GND  ├──── Black ───┤ GND          │
 │  GPIO23  ├──── Orange ──┤ MOSI (SI)    │
 │  GPIO19  ├──── Yellow ──┤ MISO (SO)    │
 │  GPIO18  ├──── Green ───┤ SCK (SCLK)   │
 │   GPIO5  ├──── Purple ──┤ CSN (CS)     │
 │   GPIO4  ├──── Blue ────┤ GDO0  (TX)   │
 │   GPIO2  ├──── Cyan ────┤ GDO2  (RX)   │
 └──────────┘              └──────────────┘
```

| ESP32 | CC1101 | Function |
|-------|--------|----------|
| 3V3 | VCC | Power — **3.3 V only** |
| GND | GND | Ground |
| GPIO23 | MOSI | SPI data out |
| GPIO19 | MISO | SPI data in |
| GPIO18 | SCK | SPI clock |
| GPIO5 | CSN | Chip select |
| GPIO4 | GDO0 | Transmit data |
| GPIO2 | GDO2 | Receive data |

> ⚠️ The CC1101 is a **3.3 V** device. 5 V will damage it.

> The transmit data line **must** go to GDO0 — the chip accepts transmit data on
> no other pin. Do **not** set `gdo0_pin:` in the `cc1101:` block when using
> `remote_transmitter` on ESP32: it re-routes the pad away from the RMT
> peripheral and transmission silently stops.

> Using a LAN8720 Ethernet board? It occupies GPIO23/18/17, so move the SPI bus
> (for example to GPIO13/14/15). The config has a commented-out Ethernet block.

See `cc1101_esp32_wiring.svg` for a full diagram.

---

## Setup

1. **Clone and enter the repo**

   ```bash
   git clone https://github.com/YOUR_USERNAME/esphome-cc1101-transceiver.git
   cd esphome-cc1101-transceiver
   ```

2. **Create your secrets file**

   ```bash
   cp secrets.yaml.example secrets.yaml
   ```

   Edit it, generating the API key with:

   ```bash
   python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
   ```

   `secrets.yaml` is git-ignored. Keep it that way.

3. **Flash**

   ```bash
   esphome run cc1101-transceiver.yaml
   ```

4. **Add to Home Assistant** — auto-discovered under **Settings → Devices & Services → ESPHome**.

---

## Capturing your own codes

The example buttons in the config are **placeholders** and control nothing. Replace
them with codes from your own remotes.

1. Set **RF Receive Band** to the band your remote uses.
2. Press **Learn Signal**, then hold the remote button for about a second.
3. Read the log:

   ```
   [raw_code]: 49 pulses @ 315.00 MHz. For a button:
   [raw_code]: freq_mhz: 315.00
   [raw_code]: code: [450, -500, 900, -900, 450, -450, ...]
   ```

4. Paste it into a button:

   ```yaml
     - platform: template
       name: "Blinds Up"
       icon: "mdi:blinds-open"
       on_press:
         - script.execute:
             id: rf_send
             freq_mhz: ${mhz_433}   # or ${mhz_315}
             times: 10
             wait_ms: 0
             code: [PASTE_HERE]
   ```

**Which band?** If nothing is captured on 433.92, try 315. Looking up the remote's
FCC ID (printed on the back) tells you outright.

### Getting repeats right

`times` and `wait_ms` matter as much as the code itself:

- **`times`** — how many times the frame is sent. Remotes typically send 3–12.
- **`wait_ms`** — silence between repeats. **This is the setting most likely to
  stop a device responding.** Many ceiling fans repeat every frame roughly 12
  times with only ~10 ms of gap, and ignore anything sent further apart.

To measure a remote's real gap, raise `idle:` under `remote_receiver` above the
expected gap (say `30ms`). A whole button press then arrives as one capture with
the remote's own gaps intact, instead of being split into separate frames.

> Learning keeps the **longest** burst it hears within 1.5 s, because the first
> repeat of a press is often clipped while the receiver's gain settles.

---

## Home Assistant entities

| Entity | Type | Purpose |
|--------|------|---------|
| RF Receive Band | Select | 433.92 MHz / 315 MHz / Scan both |
| Learning Mode | Switch | Arm capture |
| Learn Signal | Button | Arm capture (one press) |
| Replay Learned Signal / Replay x3 | Button | Retransmit on the learned band |
| Clear Learned Signal | Button | Forget it |
| TX Test | Button | Send a test pattern on the current band |
| Learn Status / Last Received Signal / Learned Signal | Text sensor | What happened |
| Learned Signal Length | Sensor | Pulse count |
| Signal Learned | Binary sensor | Whether something is stored |
| Example Ceiling Fan | Fan | Speeds plus Low/Medium/High presets |

---

## How band switching works

Each button calls the `rf_send` script with a frequency, a code, and the repeat
timing. The script stores the band, then `remote_transmitter` fires:

- **`on_transmit`** — retune to the burst's band, fully re-initialise the radio, enter TX
- **`on_complete`** — retune back to the listening band, enter RX

The full re-initialisation before each transmission costs about 15 ms and makes
the chip start from its power-on state on that band, with fresh calibration and
the band's power table written while idle.

The `rf_send` script is `mode: queued`, so two buttons pressed at once can't send
on each other's frequency.

---

## The fan entity

`Example Ceiling Fan` is a template fan: on/off, a 3-step speed slider, and a
Low/Medium/High preset dropdown, all driving the same RF codes.

It is **optimistic** — remotes are one-way, so the entity shows what it last sent.
Using the handheld remote makes it wrong until the next command from Home Assistant.

All sending is dispatched from `on_state` with a de-duplication guard, because
Home Assistant often sets state and speed together and the code must go out once.
Speed and preset stay in sync in both directions. `restore_mode: ALWAYS_OFF` means
nothing transmits at boot.

---

## Troubleshooting

### "FF0F was found", or chip ID 0x0000 / 0xFFFF
SPI wiring. Check MISO, MOSI, SCK and CSN, use short leads, confirm 3.3 V.

### "PLL lock failed" after a frequency change
`set_frequency()` takes **hertz**. In a lambda, `315.0` means 315 Hz, far below
the chip's range. Use `315.0 * 1e6f`. In YAML, `cc1101.set_frequency: 315MHz` is fine.

### Nothing is received
- Wrong band — try the other one, or look up the remote's FCC ID.
- Antenna missing or badly sized.
- `filter:` too high, `idle:` too low.

### Captured fine, but the device ignores the replay
In order of likelihood:

1. **Repeat timing.** Raise `times`, and match `wait_ms` to the remote (often ~10 ms).
2. **Clipped code.** A short capture may be missing the frame's start — compare
   pulse counts across several captures and keep the longest.
3. **Range.** A module tuned for another band transmits weakly; test up close.
4. **Rolling codes.** Car fobs and most garage openers can't be replayed at all.

To check whether the radio itself is transmitting, read the chip's state during a
transmission with a second `spi_device` on the same CS pin: `MARCSTATE` `0x13`
means it really is in TX.

### Too much noise in the log
Raise `filter:` to `500us`, lower `filter_bandwidth`, or raise `min_pulses`.

---

## Compatible devices

| ✅ Works (fixed code) | ❌ Won't work |
|----------------------|---------------|
| Motorised blinds and shades | Car key fobs |
| Ceiling fan remotes | Most garage door openers |
| RF outlets and light switches | Alarm sensors with rolling codes |
| Doorbells, fixed-code gates | Anything encrypted |

---

## Files

```
├── cc1101-transceiver.yaml   # main config (example codes - replace them)
├── secrets.yaml.example      # copy to secrets.yaml and fill in
├── cc1101_esp32_wiring.svg   # wiring diagram
├── .gitignore
└── README.md
```

---

## References

- [ESPHome CC1101 component](https://esphome.io/components/cc1101/)
- [ESPHome Remote Transmitter](https://esphome.io/components/remote_transmitter/)
- [ESPHome Remote Receiver](https://esphome.io/components/remote_receiver/)
- [ESPHome Template Fan](https://esphome.io/components/fan/template/)
- [CC1101 datasheet (TI)](https://www.ti.com/lit/ds/symlink/cc1101.pdf)
- [FCC ID lookup](https://fccid.io/) — find a remote's frequency

---

## License

MIT
