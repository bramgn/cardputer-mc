# cardputer-mc

A MeshCore chat client for the **M5 CardputerZero with the Cap-LoRa-1262** module. One Python file,
no separate radio program: it starts [meshcore-pi](https://github.com/brianwiddas/meshcore-pi) as a
child process (which drives the SX1262 over SPI), switches the module's 5V supply on, switches the Cap's antenna
(pin P0 of its I/O expander at I2C 0x43, which meshcore-pi does not touch; without it the link was 10 to 15 dB weaker), and runs an
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
cardputer-mc            # connects to the local radio on start; F4 (or F1) shows the keys
cardputer-mc --version
```

Marks after your own messages: ` ·` handed to the radio, ` ✓` delivered (private message, ACK received) or
heard being repeated by at least one repeater (channel message), ` ✗` failed. Select a message and press
Enter to see the details, including how many repeats were heard and through which repeaters.

Keys: the arrow keys always navigate; the `>` prompt only becomes active when you press Enter on a channel or
chat contact (Enter on a repeater, room, Log or Radio opens its menu instead, F6 opens Actions for the selected item from anywhere).
Esc goes back one step, to where you came from (input, messages or key bar to the list); a question or window opened from Options or the Actions menu goes back to that menu (Options > Send advert > Esc is Options again). Tab goes through every row of the list (headings and conversations alike), then to the messages
> each key of the function-key bar in turn > list. Esc is Tab backwards: on the function-key bar one key to the left, from the first key to the messages, from the messages to the list, and in the list one row up (the CardputerZero's Tab and Esc keys stick out above the rest, so they work under the thumbs). In menus, Tab picks the next entry (round to the first), also in the search results; in Help and detail windows it is the next page. PageUp/PageDown move a page in the list and in the Options and Actions menus. Quitting clears the screen. The headings in the list (Channels, Contacts, Repeaters,
Console) take the cursor too: Enter (or Left/Right) folds and unfolds them, so 350 contacts can be put away; a
folded heading still shows the unread count, and the choice is remembered. `F6` (Actions) with the cursor on a heading opens a menu for the whole group: Contacts has *Sort* (newest advert, name A-Z, last message; favorites always first), *Favorites only*, *Show repeaters* and *Remove stale contacts...* (asks for a number of days, then for a yes; favorites and contacts you have talked to are kept), Channels has *Add channel...*, Console has *Clear Log* and *Clear Radio packets*, every group has *Mark all read*, and each has Fold / Unfold. Favorites are shown in yellow. The Help and detail windows scroll sideways with Left/Right (`h`/`l`) when a line is wider than the screen. Left/Right switch between the
list and the messages; on the function-key bar they pick a key (highlighted) and Enter presses it, so the F-keys
are usable without reaching for the tiny Fn row. Typing in the list, the messages or the key bar does nothing.

The arrow keys sit on the CardputerZero's Fn layer, so whenever you are not typing (not in the prompt, a question
or a setting being edited) `z` `c` `f` `x` act as left, right, up and down, in the lists, menus and windows.

Quitting: `Ctrl-C`, `Alt-X`, `Alt-Q` and `F8` ask `Quit? [y/n]` first (the same key again, `y` or Enter says yes, `n`
or Esc cancels), also while a window is open; `Ctrl-Q` and `/q` quit at once. Alt is read as ESC followed by the
key, which is what terminals send; in the Cardputer's own CLI app `cat -v` should show `^[x` for Alt-X.

In the prompt, Up and Down walk through what you sent before, messages and commands alike (kept across restarts in `~/.config/cardputer-mc/input_history.json`, 200 lines; Down past the newest brings back what you were typing). Ctrl-P / Ctrl-N switch conversation. Space or `/` start typing from anywhere: Space continues what you were writing, `/` starts a new command (a draft
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
prefix (Enter shows the time in the details); and the function-key bar is five slots,
each centred above the keyboard key that presses it (`Fn`+4 to 8, the five keys that sit fully under the
screen: Help, Options, Actions, Advert, Quit; Tab and Enter work too). Only the Cardputer's own terminal gets that
layout (and the plain `4`-`8` digit shortcuts); an ssh session, even in a small window, or any terminal of 70+ columns
gets a left-aligned bar (`F1 Help  F2 Options  F6 Actions  F7 Advert  F10 Quit`, shorter names under 50 columns).
The program decides by `SSH_CONNECTION`/`SSH_TTY` and the Cardputer's 5V LED; `CARDPUTER_MC_KEYBOARD=1` (or `0`) forces it. `F1`/`F2`/`F10` work on the Cardputer too, `F3` connects or disconnects, `F9` shows the radio log.

The list on the left takes a quarter of the width (10 to 24 columns): a wide terminal keeps the full list, the
built-in 45-column screen gives the messages 33 columns instead of 20.

`/connect` reconnects, `/disconnect` stops the radio, F3 toggles. Chat history and UI settings are in
`~/.config/cardputer-mc/`.

## One radio, several windows

The radio is owned by a small background process, the radio daemon (`cardputer-mc radio daemon`), that the chat starts by
itself when it is not running. Any number of chat windows connect to it, for example the Cardputer's own CLI and an
SSH session at the same time: each window has its own cursor and input line, they share the radio, the contacts and
the message history (merged through `history.json`, so a message sent in one window shows up in the other within
a few seconds; `/clear` clears it everywhere). `mcorectl --tcp 127.0.0.1:5001` can use the same daemon.

- When you quit the last window it asks a second question, "Keep radio running?" (y or Enter keeps it, n switches it off). If it
  keeps running, messages keep arriving while no window is open, and the next window to connect gets everything that came
  in. Ctrl-Q quits at once without asking; then `/keep on|off` (or `cardputer-mc radio keep`) decides whether the radio stays on.
- **`/stop`** (or `cardputer-mc radio stop`) switches the radio off for good; `cardputer-mc radio status` tells whether it runs.
  `check` and `uninstall` need the radio to themselves, so stop it first.
- Not shared between windows: the cursor, the unread counts, the Log and Radio lists, and the UI settings file
  (the last window to change a setting wins).

## From the command line

The same radio daemon serves commands without a screen, so scripts (and other programs) can use the radio:

```
cardputer-mc send Yoshi "on my way"     # waits for the acknowledgement: exit 0 delivered, 1 not, 2 bad target
cardputer-mc send '#test' hello all     # channel (no acknowledgement exists; exit 0 once the radio took it)
echo "from a script" | cardputer-mc send Yoshi -
cardputer-mc read Yoshi -n 5            # the stored conversation (--json, --dates); no target: overview
cardputer-mc listen                     # print incoming messages as they arrive (--count N, --seconds N)
cardputer-mc contacts [part-of-name]    # --json
cardputer-mc channels
cardputer-mc info                       # name, key, frequency, battery (--json)
cardputer-mc advert [--flood]
cardputer-mc ping Repeater78 [-c 4] [-i 2]  # (-c 0: until Ctrl-C) zero-hop trace to a repeater or room server: SNR + round trip; exit 0 if any answered
cardputer-mc noise [-w] [-c N] [-i 2]   # the radio noise floor in dBm; -w: a live sparkline per line (Ctrl-C ends)
cardputer-mc radio up                   # start the daemon and exit
```

A target is a contact or channel name: exact, else the start, else any part of it (`#name` = channel).
Everything lands in the shared history, so a running chat window shows messages sent from the command line
(with the delivery mark) and `listen`/`read` also collect what arrived while no window was open.
Repeaters are not a target for `send`: they take commands, which stay in the chat (Enter on the repeater).

## Radio commands and tab completion

The commands that deal with the radio daemon are grouped under `radio`, like `nmcli radio`:

```
cardputer-mc radio                # same as: radio status
cardputer-mc radio up [--keep]    # start the daemon (and keep it running with --keep)
cardputer-mc radio stop           # stop it and switch the radio off
cardputer-mc radio status         # running? pid, uptime, windows, messages waiting, sound
cardputer-mc radio keep [on|off]  # stay running when the last window closes
cardputer-mc radio sound [off|private|all|test]
cardputer-mc radio check          # wiring test of the Cap-LoRa (no airtime)
```

The old names (`cardputer-mc stop`, `up`, `status`, `keep`, `sound`, `check`, `daemon`) still work, but are no longer listed.

`install` sets up tab completion for bash, generated from the program's own options: commands, `radio` actions, options and
the choices of `radio keep` and `radio sound`; where a contact is expected (`send`, `read`, `ping`) it offers the names of the
contacts and channels, from a radio daemon that is already running (completing a word never starts the radio). For a
machine that is already set up: `cardputer-mc completion --install` (or `source <(cardputer-mc completion)` for this shell).
`cardputer-mc update` refreshes it when it was installed.

## Updating

```
cardputer-mc update --check    # is there something newer on GitHub? (exit 10 when there is)
cardputer-mc update            # fast-forward this git checkout, re-apply the meshcore-pi fixes, restart an idle daemon
```

`update` only moves forward (`git merge --ff-only`) and refuses when the checkout has local changes. It restarts the radio
daemon only when no window is open; otherwise it says what to run when you are done (`cardputer-mc radio stop; cardputer-mc radio up`;
a daemon started with `--serial` needs that option again). It is never automatic: the chat looks once a day (a `git ls-remote`
of the public repository, 10 seconds after it starts) and puts `update available: run cardputer-mc update` in the status
line. Turn the look off with `CARDPUTER_MC_NO_UPDATE_CHECK=1` or `"update_check": false` in `~/.config/cardputer-mc/config.json`.

## Using a MeshCore USB companion (a Heltec ...) instead of the Cap-LoRa

A board running MeshCore's USB companion firmware speaks the same frames as meshcore-pi, over a serial line, so the
radio daemon can use it as its radio. On the machine it is plugged into:

```
cardputer-mc --serial /dev/ttyUSB0                  # the chat, with that board as the radio
cardputer-mc --serial /dev/ttyUSB0 up --keep        # or only the daemon; then the commands work as usual
cardputer-mc --serial /dev/ttyUSB0 ping NAME        # send, read, contacts, info ... all of them
```

(`CARDPUTER_MC_SERIAL` sets it for good; the daemon is started once with it and keeps using that port until `cardputer-mc radio stop`.)
Not available with `--remote`, and `check` is for the Cap-LoRa wiring only. The board's firmware decides what it can do:
TX power, add contact and status requests have been used with meshcore-pi, so test them on your board. Tested with a
Heltec V3 (v1.18.0): info, contacts, send with acknowledgement, zero-hop ping to a repeater.

## Using the radio from another computer

On a PC, install only the client part and point it at the Cardputer:

```
./cardputer-mc install --client                 # a venv with the meshcore library, and the command
cardputer-mc --remote bram@cardputer            # the chat, with the Cardputer's radio
cardputer-mc --remote bram@cardputer send Yoshi hi    # every command above works too
```

`--remote` (or `CARDPUTER_MC_REMOTE=user@host`) logs in over ssh (key based, no password prompt), starts the radio
daemon there with `cardputer-mc radio up` if needed, and forwards its port to the PC for as long as the command runs. The
daemon has no password of its own and only listens on the Cardputer's localhost: do not make it listen on the network,
let the ssh login be the protection. The Cardputer needs `cardputer-mc` installed and `~/.local/bin/cardputer-mc`
in place. The PC keeps its own chat history (it is not shared with the Cardputer's windows), and the title bar
shows no battery (the PC's battery says nothing about the radio's).

## Without a window: the daemon in the background

```
cardputer-mc radio up --keep      # start the radio daemon, detached, and keep it running when no window is open
cardputer-mc radio status         # running? pid, uptime, open windows, messages waiting, sound
cardputer-mc radio keep off       # back to: the daemon stops a minute after the last window
cardputer-mc radio stop           # switch the radio off for good
```

With `keep on` the Cardputer terminal is free for other things while the radio keeps listening; messages that arrive
meanwhile wait in the daemon's inbox and show up in the next window, in `cardputer-mc read` or `cardputer-mc listen`.
The price is some power: the radio's 5V rail stays on and the radio keeps listening. `/keep on|off` and `/stop` do the same inside the chat.

**Sound.** The daemon plays a chime on the Cardputer's speaker when a message arrives, also with no window open:
two rising tones for a private message, one lower tone for a channel message. `cardputer-mc radio sound off|private|all|test`
(or `/sound` in the chat); the default is `private` (channels can be busy). Repeater replies and bursts within two
seconds stay quiet. It needs `aplay` (alsa-utils) and uses the first sound card that is not HDMI; set
`CARDPUTER_MC_SOUND_DEVICE=plughw:1,0` to pick another.

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

## Routes

The details of a received message (Enter on it) show its route: `flood, 1 hop(s): NO-4517-RPT1 (78ac)`, taken from the
radio-log packet the message arrived in. Channel messages (always flooded) show the repeaters they passed. A message
that arrived on a direct route shows no repeaters: each one removes itself from the path while forwarding. The
message is shown for the first copy that reaches you; a later copy through a repeater is a duplicate and is dropped,
but the Radio view still lists it with its path.

Actions (F6) > Set path sets the route to a contact by hand: hops are hex, 1 to 3 bytes each and all the same
length, e.g. `78`, `78,64` or `78ac,64e1` (use the length your repeaters' hashes have; the Radio view shows it).

## Noise floor

`/noise` (or `/n`) shows the radio's noise floor as a bar graph in the message pane: a reading every 5 seconds
while connected, the last 30 minutes kept, and the graph follows the size of the pane (a bigger terminal shows a
taller graph and more minutes). Higher bars = more noise, so a bad antenna connection or interference shows up at
once. `cardputer-mc noise` prints one reading, `noise --watch` a live sparkline. On a MeshCore USB companion the
firmware reports it; on the Cap-LoRa meshcore-pi gets it from the SX1262 (see below).

## Fixes applied to meshcore-pi

`install` patches the pinned meshcore-pi checkout (the patch is embedded in the script; re-running
`install` applies it to an existing install):

- **Multi-byte path hashes.** MeshCore 1.11+ puts the hash size in the top two bits of the path-length
  byte. Stock meshcore-pi rejects those packets as `Bad packet: Path length too long`, so anything
  sent by newer phones, repeaters and companions was dropped. Now they parse.
- **Receive race.** LoRaRF's interrupt handler updates the packet length and buffer start after it
  sets the IRQ status, over the same SPI bus the receive loop uses. Packets could come out with the
  wrong length or bytes (and zero-length packets). The receive loop now waits for the handler to finish.

- **Noise floor.** While idle the receive thread reads the chip's instantaneous RSSI four times a second; the
  median of the last 20 readings is answered to `CMD_GET_STATS` (radio), which stock meshcore-pi does not know.
- **Transmit led.** The CardputerZero's green ACT led lights up for the length of every transmission. Set
  `txled = ""` under `[interface.caplora]` in the meshcore-pi config to switch it off (or another name from
  `/sys/class/leds`). Writing to the led switches its kernel trigger to `none`; a reboot brings the old one back.
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
cardputer-mc radio check
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
