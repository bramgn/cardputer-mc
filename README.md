# cardputer-mesh

A MeshCore chat client for the **M5 CardputerZero with the Cap-LoRa-1262** module. One Python file,
no separate radio program: it starts [meshcore-pi](https://github.com/brianwiddas/meshcore-pi) as a
child process (which drives the SX1262 over SPI), switches the module's 5V supply on, and runs an
ncurses chat on top. Quitting stops meshcore-pi and switches the 5V supply off again.

The chat UI is derived from `mcorechat` (ncurses MeshCore client for companion radios); this
version only talks to the local radio, so BLE, serial and TCP are gone.

## Install

On the Cardputer (Raspberry Pi OS / Debian 13, SPI enabled, Cap-LoRa-1262 fitted):

```bash
git clone /srv/git/cardputer-mesh.git ~/git/cardputer-mesh
~/git/cardputer-mesh/cardputer-mesh install
```

`install` is safe to repeat. It:

1. installs the apt packages `git python3-venv python3-rpi-lgpio python3-lgpio` (asks first; `-y` skips the question),
2. creates a venv in `~/.local/share/cardputer-mesh/venv` and pip-installs `meshcore`, `LoRaRF` and friends,
3. clones meshcore-pi and checks out the tested commit (`85de8dc`),
4. writes `~/.local/share/cardputer-mesh/config.toml` with a fresh node identity (mode 600),
5. links `~/.local/bin/cardputer-mesh` to the script.

Options: `--name NAME` (node name, default `Cardputer`), `--key HEX` (reuse an existing 32-byte
private key instead of generating one), `-y`.

## Use

```bash
cardputer-mesh            # connects to the local radio on start; F1 shows the keys
cardputer-mesh --version
```

`/connect` reconnects, `/disconnect` stops the radio, F3 toggles. Chat history and UI settings are in
`~/.config/cardputer-mesh/`.

## Radio settings

Edit `~/.local/share/cardputer-mesh/config.toml` (frequency, bandwidth, spreading factor, coding
rate, TX power). Defaults are the EU/UK narrow preset: 869.618 MHz, 62.5 kHz, SF8, CR8, **14 dBm**.
The pins are the CardputerZero's: SPI0 CS1, RESET = GPIO26, BUSY = GPIO22, DIO1 = GPIO23, TCXO 3.0 V
on DIO3, RF switch on DIO2. In the EU, 869.4–869.65 MHz allows up to 27 dBm ERP; most of the rest of
863–870 MHz only 14 dBm. Check your local rules before raising `txpower`.

Changing TX power or the radio clock from inside the chat has no effect: meshcore-pi ignores those
commands. Edit `config.toml` and reconnect.

## Files

State is in `~/.local/share/cardputer-mesh` (override with `CARDPUTER_MESH_HOME`): `config.toml`,
`meshcore-pi/`, `venv/`, `contacts.mesh`, `channels.json`, `meshcore.log`, `local-radio.log`.
Back up `config.toml`: it holds the node's private key.

## Known limits

- If the process is killed outright (`kill -9`, power loss) the 5V supply stays on. Switch it off with
  `echo 0 | sudo tee /sys/class/leds/ext_5v_out/brightness` (or just start and quit the app).
- meshcore-pi accepts one client at a time, so `mcorectl` and this app cannot be connected together.
- meshcore-pi is a third-party project; its receive loop keeps one CPU core busy while the radio is on.
- Tested on a real CardputerZero: install, start, connect to the radio, quit and cleanup (5V off, no stray process), in an 80x24 terminal over SSH. Not checked: the 320x170 built-in display, sending and receiving chat messages from inside the UI.
