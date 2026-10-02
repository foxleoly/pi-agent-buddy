<p align="right">
  <a href="README.zh_CN.md">简体中文</a> · <strong>English</strong>
</p>

# Pi Agent Buddy

The **computer side** of Pi Agent Buddy: a wearable status board and remote for a
[pi](https://pi.dev) coding agent. The AI Passport device shows what your agent is
doing and lets you stop it with a button; everything in this repository is what runs
on your computer to feed it.

| Piece | What it does |
| --- | --- |
| [`extensions/pibuddy.ts`](extensions/pibuddy.ts) | pi extension: publishes the live session state, runs the device's acts, shows the link in pi's footer, and copies the pair code with `/pibuddy`. |
| [`sidecar/`](sidecar/) | Go program: reads the published state and the session JSONL, streams heartbeats to the device, and hands button presses back to the extension. |

## Requirements

- A running **pi** session.
- A running **pi** session with this package installed: the extension is the bridge
  between pi and the sidecar, not just a footer nicety. It creates `~/.pi-buddy`
  (mode 0700), which is the channel the two halves talk through.
- **Go 1.27+** only if you build the sidecar yourself (a prebuilt binary is available).

## Install the pi extension

```bash
pi install git:github.com/foxleoly/pi-agent-buddy@v0.1.0
```

Then `/pibuddy` reports the link state and offers to copy the pair code.

## Run the sidecar

Download the binary for your platform from the
[latest release](https://github.com/foxleoly/pi-agent-buddy/releases/latest)
(`pi-buddy-sidecar-<os>-<arch>`, with `checksums.txt`):

```bash
chmod +x pi-buddy-sidecar-darwin-arm64
./pi-buddy-sidecar-darwin-arm64 --ws --ws-addr :51820
```

Or build it yourself:

```bash
cd sidecar
go build -o pi-buddy-sidecar .
./pi-buddy-sidecar --ws --ws-addr :51820
```

The sidecar follows the most recently updated pi session, so no target is needed;
`--pi-pid <pid>` pins one process when several agents are running.

Useful flags:

```bash
./pi-buddy-sidecar --show-token                   # print the link code the device asks for
./pi-buddy-sidecar --ws --ws-addr :51820
./pi-buddy-sidecar --ws --ws-addr :51820 --pi-pid 41234     # pin one pi process
./pi-buddy-sidecar --usb --usb-path /dev/ttyACM0            # tethered instead of Wi-Fi
```

## Run it as a service

Both service definitions live in [`service/`](service/). Put the binary at
`~/.local/bin/pi-buddy-sidecar`.

### macOS (launchd)

launchd expands neither `~` nor environment variables, so the plist is rendered first:

```bash
mkdir -p ~/.local/bin ~/Library/LaunchAgents
mv pi-buddy-sidecar-darwin-arm64 ~/.local/bin/pi-buddy-sidecar
sed "s|@HOME@|$HOME|g" \
  service/dev.foxleoly.pi-buddy-sidecar.plist \
  > ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
launchctl print "gui/$(id -u)/dev.foxleoly.pi-buddy-sidecar" | head -20
```

Remove it again:

```bash
launchctl bootout "gui/$(id -u)/dev.foxleoly.pi-buddy-sidecar"
rm ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
```

### Linux (systemd user unit)

```bash
mkdir -p ~/.local/bin ~/.config/systemd/user
mv pi-buddy-sidecar-linux-amd64 ~/.local/bin/pi-buddy-sidecar
cp service/pi-buddy-sidecar.service \
  > ~/.config/systemd/user/pi-buddy-sidecar.service
systemctl --user daemon-reload
systemctl --user enable --now pi-buddy-sidecar
systemctl --user status pi-buddy-sidecar
loginctl enable-linger "$USER"    # keep it up with no open login session
```

Remove it again:

```bash
systemctl --user disable --now pi-buddy-sidecar
rm ~/.config/systemd/user/pi-buddy-sidecar.service
```

Both units restart the sidecar if it exits. The sidecar writes `~/.pi-buddy/ble.log`
itself; the launchd plist additionally captures stdout/stderr in
`~/.pi-buddy/sidecar.out.log` and `~/.pi-buddy/sidecar.err.log`.

## How the device finds the sidecar

The sidecar advertises `_pibuddy._tcp` over mDNS and listens on the port given to
`--ws-addr`. The device discovers it on the same network and dials `/<link code>`,
a 12-digit hex code generated once into `~/.pi-buddy/token` (print it again with
`--show-token`). Any other path is refused before the WebSocket upgrade, so an
unauthenticated peer on the network never reaches the control handler.

The sidecar also publishes `~/.pi-buddy/status.json` (`connected`, `device`, `pid`,
`updated`) so other local tools — such as the extension above — can show the link
state without IPC. The `pid` lets a reader tell a stale file left by a dead sidecar
from a live link.

## Notes

- Pure Go, no cgo: builds and runs on macOS, Linux, and Windows.
- The sidecar is the computer half only; it never talks to the device's hardware
  directly. The device firmware is distributed separately.
- MIT licensed — see [LICENSE](LICENSE).
