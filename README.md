# wb-loxone — Wiren Board devices in Loxone

A bridge that runs directly on a [Wiren Board](https://wirenboard.com) controller
and exposes its devices to a [Loxone](https://www.loxone.com) Miniserver:
relays, inputs, sensors, meters, Zigbee via zigbee2mqtt, Z-Wave Long Range via
Z-Way, wb-rules virtual devices — anything the controller publishes to MQTT.

Channels are selected in the web UI. For Loxone Config the bridge generates
ready-made virtual input and output templates, one per device. Button presses
(single, double, triple, long) are detected by the bridge and delivered to
Loxone as separate pulses.

```
 WB devices ──► MQTT ──► wb-loxone ──UDP──► Virtual UDP input   ┐
 (RS-485, Zigbee,        (on WB)                                 │ Loxone
  Z-Wave LR, 1-Wire,                                             │ Miniserver
  wb-rules)    ◄────────────────◄──HTTP── Virtual output         ┘
```

![Bridge web UI: settings and channel selection](docs/img/web-ui.png)

* **States are pushed on change.** The bridge sends a UDP packet to Loxone as
  soon as a value changes in MQTT. Loxone polls nothing.
* **Commands take milliseconds.** A Loxone virtual output calls HTTP on the
  controller and the bridge publishes the value to MQTT. The bridge itself adds
  only a few milliseconds.
* **A template per device.** `VIU_<name>.xml` and `VO_<name>.xml`: the block in
  Loxone Config is named after the device, rows are "channel — device".
* **Buttons.** The press count is read from the module's hardware counter, so
  short presses are not lost even on a slow bus. Details in
  [docs/buttons.md](docs/buttons.md).
* **Nothing to install on the controller.** A single file in the standard
  Python 3.9 that ships with the firmware. No pip, no Node-RED, no extra server.

## How it differs from Modbus TCP (wb-mqtt-mbgate)

You can connect WB and Loxone through the controller's Modbus gateway, and many
people do. The bridge does not replace it; it covers what is awkward over Modbus:

| | Modbus TCP gateway | wb-loxone |
|---|---|---|
| States in Loxone | Loxone polls in a cycle | delivered immediately on change |
| Buttons | input state only; double/long press built in Loxone logic from polling | ready-made "single / double / triple / long" pulses |
| Templates | register addresses by hand | generated automatically, per device |
| Zigbee, Z-Wave LR, wb-rules, 1-Wire | a register map must be configured | everything that is in MQTT |
| Names | register numbers | channel and device names from WB |

They can be used together: for example, outputs via Modbus, inputs and buttons
via the bridge.

## Install

On the Wiren Board controller, as root:

```bash
wget -qO- https://raw.githubusercontent.com/megavoltt/wb-loxone/main/install.sh | sh
```

At the end the installer prints the web UI address — by default
`http://<controller IP>:8099`.

Running the same command again updates the bridge. Settings and channel
selection are kept.

Requirements: Wiren Board 7 or 8 with current firmware (tested on wb-2602,
Debian 11, Python 3.9), Loxone Miniserver Gen1 or Gen2, Loxone Config 16 or newer.

## The first five minutes

1. Open `http://<controller IP>:8099`, enter the **Miniserver IP** and click "Save".
2. Expand a device and tick the channels you need. The selection is saved
   immediately. Switches and buttons can be tested right from the page (⇆, ▶).
3. If needed, set a **name in Loxone** for the device — it names the block and
   the template files.
4. Download the templates: the **VIU / VO** buttons on a device, or
   **"All device templates (ZIP)"** at the top.
5. In Loxone Config: "Virtual Inputs" → "Import template" → `VIU_…xml`, then the
   same for "Virtual Outputs" → `VO_…xml`. Drag the inputs and outputs you need
   onto the program page and save to the Miniserver.

After import the blocks look like this — one block per device, rows are "channel — device":

![WB-MR6C relay virtual outputs in Loxone Config](docs/img/loxone-vo-relays.png)
![WB-MRM2-mini button: input state and press events](docs/img/loxone-buttons.png)

More, with examples of wiring to function blocks, in [docs/loxone.md](docs/loxone.md).

## Documentation

The documentation in `docs/` is in Russian.

* [docs/buttons.md](docs/buttons.md) — buttons: how presses are detected, settings, delays, debugging
* [docs/loxone.md](docs/loxone.md) — importing templates and wiring to Loxone blocks
* [docs/wirenboard.md](docs/wirenboard.md) — module input modes, Modbus events, the RS-485 bus
* [docs/protocol.md](docs/protocol.md) — UDP and HTTP formats, the settings file, the web UI API
* [CHANGELOG.md](CHANGELOG.md) — what changed in each version

## Where things live on the controller

| Path | What it is |
|---|---|
| `/mnt/data/wb-loxone/` | code and web UI |
| `/mnt/data/etc/wb-loxone.json` | settings and selected channels |
| `/etc/systemd/system/wb-loxone.service` | service |

`/mnt/data` survives firmware updates. After a full reflash (FIT) run the
installer again — the settings will be picked up.

```bash
systemctl status wb-loxone          # service status
journalctl -u wb-loxone -f          # live log
```

Uninstall:

```bash
systemctl disable --now wb-loxone && rm -rf /etc/systemd/system/wb-loxone.service /mnt/data/wb-loxone /mnt/data/etc/wb-loxone.json
```

## Security

The bridge is intended for a local network.

* The web UI has no password by default — set one in the settings if there are
  untrusted devices on the network.
* Commands from Loxone are protected by a key in the URL (`/set/<key>/…`). The
  bridge executes commands only for channels selected in the UI and only for
  writable channels. A new key can be generated with a button; after that the VO
  templates must be imported again.
* Do not expose the bridge port to the internet.

## License

[MIT](LICENSE) — free to use, including in commercial projects.

---

**Languages:** English · [Русский](README.ru.md)
