# cardputer-mc

A MeshCore chat client for the **M5 CardputerZero with the Cap-LoRa-1262** module. One Python file,
no separate radio program: it starts [meshcore-pi](https://github.com/brianwiddas/meshcore-pi) as a
child process (which drives the SX1262 over SPI), switches the module's 5V supply on, and runs an
ncurses chat on top. Quitting stops meshcore-pi and switches the 5V supply off again.

The chat UI is derived from `mcorechat` (ncurses MeshCore client for companion radios); this
version only talks to the local radio, so BLE, serial and TCP are gone.

## Install

On the Cardputer (Raspberry Pi OS / Debian 13, SPI enabled, Cap-LoRa-1262 fitted):

```bash
git clone https://github.com/bramgn/cardputer-mc.git ~/git/cardputer-mc
~/git/cardputer-mc/cardputer-mc install --name YourName
```

It is a single file, so you can also just download it:
`curl -O https://raw.githubusercontent.com/bramgn/cardputer-mc/master/cardputer-mc && chmod +x cardputer-mc && ./cardputer-mc install`.
Make sure `~/.local/bin` is on your `PATH` (the CardputerZero's built-in CLI app starts a non-login shell
that does not read `~/.profile`; add it to `~/.bashrc`).

`install` is safe to repeat. It:

1. installs the apt packages `git python3-venv python3-rpi-lgpio python3-lgpio` (asks first; `-y` skips the question),
2. creates a venv in `~/.local/share/cardputer-mc/venv` and pip-installs `meshcore`, `LoRaRF` and friends,
3. clones meshcore-pi, checks out the tested commit (`85de8dc`) and applies two small fixes (see below),
4. writes `~/.local/share/cardputer-mc/config.toml` with a fresh node identity (mode 600),
5. links `~/.local/bin/cardputer-mc` to the script.

Options: `--name NAME` (node name, default `Cardputer`), `--key HEX` (reuse an existing 32-byte
private key instead of generating one), `-y`.

## Renamed from cardputer-mesh

The project used to be called `cardputer-mesh`. Running `cardputer-mc install` on a machine that has the old
install moves `~/.local/share/cardputer-mesh` and `~/.config/cardputer-mesh` to the new names (keeping the node
identity, contacts and history) and removes the old `~/.local/bin/cardputer-mesh` link. The environment variable
is now `CARDPUTER_MC_HOME`.

## Uninstall

```bash
cardputer-mc uninstall                  # asks first, then removes the venv, meshcore-pi, config, logs and the link
cardputer-mc uninstall --keep-identity  # keeps config.toml (the node key), contacts.mesh and channels.json
cardputer-mc uninstall --purge          # also removes chat history and UI settings in ~/.config/cardputer-mc
```

It refuses to run while the app is open. The apt packages stay installed. Without `--keep-identity` the
private key is deleted, so the next install is a new node; to keep the identity, copy the `privatekey` line
from `config.toml` first and pass it to `install --key`.

## Use

```bash
cardputer-mc            # connects to the local radio on start; F1 shows the keys
cardputer-mc --version
```

Marks after your own messages: ` ·` handed to the radio, ` ✓` delivered (private message, ACK received) or
heard being repeated by at least one repeater (channel message), ` ✗` failed. Select a message and press
Enter to see the details, including how many repeats were heard and through which repeaters.

Keys: the arrow keys always navigate; the `>` prompt only becomes active when you press Enter on a channel or
chat contact (Enter on a repeater, room, Log or Radio opens its menu instead, F7 opens the menu from anywhere).
Esc goes back to the list. Tab goes list > messages > function-key bar > list. Left/Right switch between the
list and the messages; on the function-key bar they pick a key (highlighted) and Enter presses it, so the F-keys
are usable without reaching for the tiny Fn row. Typing in the list, the messages or the key bar does nothing.

The arrow keys sit on the CardputerZero's Fn layer, so whenever you are not typing (not in the prompt, a question
or a setting being edited) `z` `c` `f` `x` act as left, right, up and down, in the lists, menus and windows.

Quitting: `Ctrl-C`, `Alt-X`, `Alt-Q` and `F10` ask `Quit? [y/n]` first (the same key again, `y` or Enter says yes, `n`
or Esc cancels), also while a window is open; `Ctrl-Q` and `/q` quit at once. Alt is read as ESC followed by the
key, which is what terminals send; in the Cardputer's own CLI app `cat -v` should show `^[x` for Alt-X.

Space or `/` start typing from anywhere: Space continues what you were writing, `/` starts a new command (a draft
is parked on Ctrl-Y), so `/q` always quits.

Going from the list to the messages (Right, `c` or Tab) puts the cursor on the message that is on the same
screen row as the list's cursor, or on the last message when that row is empty.

Vim-style navigation works the same way: `h j k l` for left/down/up/right, `gg` and `G` for the top and bottom
(list, messages, help, options, menus, search results), `Ctrl-D` / `Ctrl-U` for half a page. In the prompt these
are ordinary letters.

`:` also starts a command, like `/`: `:q`, `:quit`, `:help`, `:ping`, ... are the same as `/q`, `/quit`, `/help`,
`/ping`. A line that starts with `:` followed by anything else (a smiley such as `:D`, `:)`) is still sent as a
message; `::` at the start sends a literal `:`. Tab completes `:` commands too.

On a small screen (under about 60 columns) the title bar shows just the conversation, the battery of the
CardputerZero itself (`58%+`, the plus means charging) and the connection state; messages drop their `HH:MM`
prefix (Enter shows the time in the details); and the function-key bar uses short names (`Help Opts Conn Menu
Log RX Quit`, reachable with Tab and Enter) instead of `F1 Help  F2 Options ...`. On a wide terminal nothing
changes.

The list on the left takes a quarter of the width (10 to 24 columns): a wide terminal keeps the full list, the
built-in 45-column screen gives the messages 33 columns instead of 20.

`/connect` reconnects, `/disconnect` stops the radio, F3 toggles. Chat history and UI settings are in
`~/.config/cardputer-mc/`.

## One radio, several windows

The radio is owned by a small background process, the radio daemon (`cardputer-mc daemon`), that the chat starts by
itself when it is not running. Any number of chat windows connect to it, for example the Cardputer's own CLI and an
SSH session at the same time: each window has its own cursor and input line, they share the radio, the contacts and
the message history (merged through `history.json`, so a message sent in one window shows up in the other within
a few seconds; `/clear` clears it everywhere). `mcorectl --tcp 127.0.0.1:5001` can use the same daemon.

- The daemon switches the radio off 20 seconds after the last window closes. **`/keep on`** keeps it running, so
  messages keep arriving while no window is open; the next window to connect gets everything that came in.
- **`/stop`** (or `cardputer-mc stop`) switches the radio off for good; `cardputer-mc status` tells whether it runs.
  `check` and `uninstall` need the radio to themselves, so stop it first.
- Not shared between windows: the cursor, the unread counts, the Log and Radio lists, and the UI settings file
  (the last window to change a setting wins).

## Staying connected

meshcore-pi drops a client that is silent for 90 seconds (the phone app asks for the battery every minute). The radio
daemon does that request every 30 seconds, so the link stays up. If the link breaks anyway (meshcore-pi dies, the
radio hangs) the daemon stops and every window starts a fresh one by itself, up to five times with growing pauses,
showing `reconnecting ...` in the title bar; F3 or `/connect` tries again by hand.

## Radio settings

Edit `~/.local/share/cardputer-mc/config.toml` (frequency, bandwidth, spreading factor, coding
rate, TX power). Defaults are the EU/UK narrow preset: 869.618 MHz, 62.5 kHz, SF8, CR8, **14 dBm**.
The pins are the CardputerZero's: SPI0 CS1, RESET = GPIO26, BUSY = GPIO22, DIO1 = GPIO23, TCXO 3.0 V
on DIO3, RF switch on DIO2. In the EU, 869.4–869.65 MHz allows up to 27 dBm ERP; most of the rest of
863–870 MHz only 14 dBm. Check your local rules before raising `txpower`.

Changing TX power or the radio clock from inside the chat has no effect: meshcore-pi ignores those
commands. Edit `config.toml` and reconnect.

## Fixes applied to meshcore-pi

`install` patches the pinned meshcore-pi checkout (the patch is embedded in the script; re-running
`install` applies it to an existing install):

- **Multi-byte path hashes.** MeshCore 1.11+ puts the hash size in the top two bits of the path-length
  byte. Stock meshcore-pi rejects those packets as `Bad packet: Path length too long`, so anything
  sent by newer phones, repeaters and companions was dropped. Now they parse.
- **Receive race.** LoRaRF's interrupt handler updates the packet length and buffer start after it
  sets the IRQ status, over the same SPI bus the receive loop uses. Packets could come out with the
  wrong length or bytes (and zero-length packets). The receive loop now waits for the handler to finish.

- **ACKs with extra bytes.** Newer firmware appends two bytes after the 4-byte ack hash. Stock meshcore-pi
  rejected those (`Ack packet payload is not 4 bytes`), so a message that did arrive was never shown as
  delivered. Now the first four bytes are used.

## First contact

The node can only decrypt messages from contacts whose advert it has received. After a fresh install
(new `contacts.mesh`), send an advert from your phone app (flood advert) and wait for the contact to
appear in the Contacts tab before you expect private messages to arrive.

## Files

State is in `~/.local/share/cardputer-mc` (override with `CARDPUTER_MC_HOME`): `config.toml`,
`meshcore-pi/`, `venv/`, `contacts.mesh`, `channels.json`, `meshcore.log`, `local-radio.log`.
Back up `config.toml`: it holds the node's private key.

## Known limits

- If the process is killed outright (`kill -9`, power loss) the 5V supply stays on. Switch it off with
  `echo 0 | sudo tee /sys/class/leds/ext_5v_out/brightness` (or just start and quit the app).
- meshcore-pi accepts one client at a time, so `mcorectl` and this app cannot be connected together.
- meshcore-pi is a third-party project; its receive loop keeps one CPU core busy while the radio is on.
- Tested on a real CardputerZero: install, start, connect to the radio, quit and cleanup (5V off, no stray process), in an 80x24 terminal over SSH. Not checked: the 320x170 built-in display, sending and receiving chat messages from inside the UI.

## Check the wiring

```bash
cardputer-mc check
```

Tests this board's SX1262 wiring in a few seconds, without sending anything: it switches the module's 5V supply on
(and restores it), pulses RESET and checks that BUSY follows, reads the chip's status and the sync-word register over
SPI, finds the I2C expander at 0x43 (read only) and checks that DIO1 is free. It uses the pins from `config.toml` and
exits with 0 when everything matches. Run it on a new board or revision, with the app closed.

## Before you use it

- The SX1262 pin mapping is **RESET GPIO26, BUSY GPIO22, DIO1/IRQ GPIO23, NSS = SPI0 CE1 (GPIO7, `/dev/spidev0.1`)**.
  It matches the labels on the Cap-LoRa-1262 and on the CardputerZero's 2x7 header, column by column, and it was
  verified on a running CardputerZero (`cardputerzero-v5-overlay`): RESET low makes BUSY go high, and packets are
  received and sent.

  | Cap-LoRa-1262 | RST | IRQ | BUSY | SCK | MOSI | MISO | NSS |
  |---|---|---|---|---|---|---|---|
  | CardputerZero header | G26 | G23 | G22 | G11 | G10 | G9 | G7 (CS1) |

  The Cap's PI4IOE5V6408 expander (I2C address 0x43, on G2/G3) switches something in the RF path through its P0
  pin ("BYPASS: EP0" on the label); cardputer-mc does not touch it. The GNSS module sits on the UART (G14/G15).
  The pins are verified for this board only: another board revision (v3, ...) might differ.
- Pick the frequency and TX power that are legal where you live, and that match the mesh you want to join.
- A fresh install is a new node with its own private key (in `config.toml`). Never share that file.

## Credits and license

MIT, see `LICENSE`. The chat UI is derived from mcorechat. The script installs and patches
[meshcore-pi](https://github.com/brianwiddas/meshcore-pi) (MIT, Copyright (c) 2025 Brian Widdas), which does
the actual radio work; the embedded patch is a small modification of that project's code and falls under its
license. It uses the [meshcore](https://pypi.org/project/meshcore/) Python library to talk to it, and
[LoRaRF](https://pypi.org/project/LoRaRF/) for the SX1262.
