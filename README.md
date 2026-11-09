# Plasma UPS Monitor

Plasma 6 widget for monitoring a UPS exposed through NUT (`upsc`).

> **Fork notice:** this is a fork of [maku2903/plasma-ups-monitor](https://github.com/maku2903/plasma-ups-monitor) (MIT license), adjusted to work with the **TS Shara 4452** UPS on **Fedora 45 KDE**. See [Fork notes](#fork-notes-ts-shara-4452-on-fedora-45-kde) for what changed and why.

The widget is built for Plasma Desktop and reads standard NUT fields instead of talking to vendor-specific software directly. It works best with a local NUT server, but it can also monitor a remote target if `upsc <name>@<host>` works on the machine running Plasma.

## Repository

- This fork: [https://github.com/rafaelruc/plasma-ups-monitor](https://github.com/rafaelruc/plasma-ups-monitor)
- Original project: [https://github.com/maku2903/plasma-ups-monitor](https://github.com/maku2903/plasma-ups-monitor)

## Screenshot

![UPS Monitor screenshot](https://github.com/rafaelruc/plasma-ups-monitor/raw/fix-non-numeric-runtime/screenshots/widget-default.png)

## Features

- battery percentage
- battery voltage (`battery.voltage`)
- input voltage (`input.voltage`) and output voltage (`output.voltage`)
- input frequency (`input.frequency`)
- nominal input current (`input.current.nominal`)
- UPS temperature (`ups.temperature`)
- load percentage (`ups.load`)
- runtime (`battery.runtime`), shown exactly as reported by NUT, with no formatting
- real power in watts when reported by NUT (card is hidden when the UPS does not report it)
- raw UPS alarm text
- last detected power-loss timestamp
- configurable refresh interval
- selectable detected local UPS target or manual NUT target entry

## Requirements

- Plasma 6
- `kpackagetool6`
- NUT client tools with `/usr/bin/upsc`
- a reachable NUT target such as `tsshara@localhost` (or `eaton@localhost`)
- GNU `date` for local timestamp formatting

## Install

Clone this fork **outside** the Plasma plasmoids folder (see the warning below) and install from the project directory:

```bash
git clone https://github.com/rafaelruc/plasma-ups-monitor.git ~/plasma-ups-monitor
cd ~/plasma-ups-monitor
git checkout fix-non-numeric-runtime
kpackagetool6 -t Plasma/Applet -i package
```

Update an existing local install:

```bash
kpackagetool6 -t Plasma/Applet -u package
```

Remove it:

```bash
kpackagetool6 -t Plasma/Applet -r io.maciek.plasma.upsmonitor
```

After installing or updating, restart Plasma and add the widget again if needed:

```bash
systemctl --user restart plasma-plasmashell
```

> **Warning:** do not clone this repository inside
> `~/.local/share/plasma/plasmoids/`. `kpackagetool6 -u` removes the installed
> plasmoid folder before reinstalling, which deletes the clone and makes the
> installation fail.

## Download

- This fork: clone the repository as shown in [Install](#install).
- Original releases: [https://github.com/maku2903/plasma-ups-monitor/releases](https://github.com/maku2903/plasma-ups-monitor/releases)

## Development

Run the widget in a standalone window:

```bash
plasmawindowed io.maciek.plasma.upsmonitor
```

Reinstall the current working copy after changes:

```bash
kpackagetool6 -t Plasma/Applet -u package
```

Test the helper script on its own (the output must be valid JSON):

```bash
bash package/contents/scripts/nut-status.sh tsshara@localhost | python3 -m json.tool
```

## Configuration

The widget exposes:

- `UPS name`: a detected local UPS target from `upsc -l`, or any manual NUT target (for this fork's tested setup: `tsshara@localhost`)
- `Refresh interval`: polling interval in seconds

Defaults:

- UPS name: `eaton@localhost`
- Refresh interval: `10`

## Fork notes: TS Shara 4452 on Fedora 45 KDE

### Tested setup

- UPS: TS Shara 4452 (reported by NUT as model `Senoi`)
- NUT driver: `nutdrv_qx` (Megatec protocol), port `/dev/ttyACM0`
- OS: Fedora 45 KDE (Plasma 6)
- NUT target: `tsshara@localhost`

### Problem

`upsc -l` listed the UPS and `upsc tsshara@localhost` returned data, but the widget
showed **"Unavailable - Invalid helper output"** with `N/A` in every field.

The cause is that this UPS reports a non-numeric value for the battery runtime:

```text
battery.runtime: d  22
```

The original helper (`package/contents/scripts/nut-status.sh`) copied this value
straight into its JSON output:

```text
"runtimeSeconds":d  22,
```

That is invalid JSON, so the widget discarded the whole output. The QML warnings
`Unable to assign [undefined] to bool / QString` seen in `plasmawindowed` are a
consequence of the widget receiving no data.

### Changes in this fork

**Helper script (`package/contents/scripts/nut-status.sh`)**

- `battery.runtime` is no longer emitted as a JSON number. It is now emitted as a
  string (`runtimeRaw`) containing exactly what NUT reports (for example `d  22`),
  so the widget can never break on a non-numeric value.
- All numeric fields now go through a validated `json_number_or_null`: anything
  that is not a number becomes `null` instead of producing invalid JSON.
- New fields read from NUT: `battery.voltage`, `input.frequency`, `input.voltage`,
  `input.current.nominal` and `ups.temperature` (in addition to the existing
  `output.voltage`, `ups.load` and `battery.charge`).
- The widget title uses `ups.model` and falls back to `device.model` when the UPS
  does not report `ups.model` (the TS Shara 4452 only reports `device.model`).

**Widget UI (`FullRepresentation.qml`, `main.qml`)**

- The metric cards are now: Input voltage, Output voltage, Input frequency,
  Input current (nominal), Battery voltage, Temperature, Load, Runtime and
  (only when the UPS reports it) Power.
- The Runtime card displays the raw `battery.runtime` value, with no unit or
  conversion. If the UPS reports nothing, it shows `N/A`.

### Fedora notes

- NUT configuration lives in `/etc/ups/` on Fedora (not `/etc/nut/`), for example `/etc/ups/ups.conf`:

  ```ini
  [tsshara]
      driver = nutdrv_qx
      port = /dev/ttyACM0
      desc = "Nobreak TS Shara 4452"
  ```

- Check that the driver is really delivering data before blaming the widget:

  ```bash
  upsc tsshara@localhost
  systemctl status nut-server nut-driver@tsshara
  ```

  `upsc -l` only lists names known to `upsd`; it works even when no data is arriving.

- This UPS reports a non-numeric `battery.runtime` (`d  22`), so the Runtime card
  shows that raw text. This is a driver/firmware limitation, not a widget bug.

- Alternative that does not touch the widget: set a fixed value in `ups.conf`
  with `override.battery.runtime = 600`. This displays a made-up runtime, so it is
  not recommended.

### Credits

Original widget by [maku2903](https://github.com/maku2903/plasma-ups-monitor),
MIT licensed. Only the helper script fix and this documentation were changed.

## Notes

- The widget shows what NUT reports. If a driver or UPS firmware exposes misleading alarm or status flags, the UI will reflect that.
- Some fields are optional in NUT. If your UPS does not expose wattage, runtime, or alarm text, those entries may show `N/A` or remain empty.
- `Last power loss` is tracked locally by the widget helper based on observed `OL -> OB` transitions.

## Project Layout

- `package/metadata.json`: Plasma package metadata
- `package/contents/ui/`: QML UI
- `package/contents/scripts/nut-status.sh`: helper that reads NUT and emits JSON (patched in this fork: raw runtime string and extra fields)
- `CMakeLists.txt`: optional install support for Plasma packaging workflows

## License

MIT. See [LICENSE](LICENSE).
