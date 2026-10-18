# vpn-manager

A small Bash CLI to list, start and stop WireGuard VPN connections through
NetworkManager — without `sudo`. It also drives an Omarchy bar widget with a
status icon, a tooltip and a click menu to switch tunnels.

## Why

Previously all tunnels were managed with `sudo wg-quick up/down <name>`.
`wg-quick` needs root, so a bar widget could not toggle tunnels.
NetworkManager runs as a system daemon and lets the logged-in user switch
connections via polkit, so `nmcli connection up <name>` works as a normal user.

## Requirements

- NetworkManager (`nmcli`)
- WireGuard support in the kernel (Linux ≥ 5.6)
- WireGuard configs imported into NetworkManager (see Setup)
- `jq` — builds the JSON for the bar widget
- `notify-send` (libnotify) — desktop notifications
- Omarchy (`omarchy-menu-select`, Omarchy shell bar) — for the widget and menu

## Setup

Import each WireGuard config into NetworkManager once. `sudo` is only needed
because `/etc/wireguard/` is readable by root only:

```bash
sudo nmcli connection import type wireguard file /etc/wireguard/<name>.conf
nmcli connection modify <name> connection.autoconnect no
nmcli connection down <name>    # NM may activate it right after import
```

Notes:

- The connection and interface name come from the filename.
- Autoconnect is on by default after import — turn it off, otherwise
  NetworkManager brings the tunnel up on every network connect.
- The import is a **copy**. Changes to `/etc/wireguard/<name>.conf` are not
  picked up; delete and re-import the connection instead.
- Never run the same tunnel through `wg-quick` and NetworkManager at the same
  time (both use the same interface name).

## Install

```bash
cp vpn ~/.local/bin/vpn
chmod +x ~/.local/bin/vpn
```

`~/.local/bin` must be in `$PATH`. Re-run the `cp` after every change to the
project copy.

## Usage

```bash
vpn list              # all WireGuard connections with their state
vpn up <name>         # bring a connection up
vpn down <name>       # take a connection down
vpn menu              # pick a tunnel from the Omarchy menu and toggle it
vpn bar               # print status JSON for the Omarchy bar
```

Example:

```
$ vpn list
office inactive
proton-BE-30 inactive
proton-DE-210 active
work active

$ vpn bar
{"text":"󰴴","tooltip":"proton-DE-210, work","class":"active"}
```

Behavior:

- `up` / `down` are **idempotent**: bringing up an already active connection
  (or taking down an inactive one) prints a message and exits with `0`.
- Unknown names print an error to stderr and exit with `1`.
- Output is sorted by name, so the order stays stable when states change.
- `menu` shows a checkmark next to active tunnels; choosing one toggles it.
  Pressing Esc cancels and exits with `0`.

## Notifications

`up` and `down` send a desktop notification via `notify-send`:

- success: "VPN connected" / "VPN disconnected" with the tunnel name
- failure: a **critical** "VPN error" notification that includes nmcli's
  error message (also printed to stderr)

Notifications only cover changes made through `vpn` itself. A tunnel that
drops for other reasons is not reported (see Roadmap: event-driven updates).

If no notifications appear, check whether Omarchy's Do Not Disturb mode is on
(`omarchy toggle notification silencing`).

## Bar widget

The Omarchy bar supports **command modules**: it runs a command every few
seconds and reads Waybar-style JSON (`text`, `tooltip`, `class`) from stdout.
Add this to `bar.layout.right` in `~/.config/omarchy/shell.json` (it
hot-reloads on save):

```json
{ "id": "vpn", "type": "command", "exec": "~/.local/bin/vpn bar", "interval": 5,
  "onClick": "~/.local/bin/vpn menu", "onRightClick": "~/.local/bin/vpn menu" }
```

- **Icon:** Nerd Font glyph U+F0D34 (written as `$'\U000f0d34'` in Bash).
- **Colour:** `"class": "active"` switches the icon to the theme's active
  colour, which the bar calls `urgent` (red in some themes). `class` is
  `active` when any tunnel is up.
- **Tooltip:** names of all active tunnels, or `VPN off`.
- **Click:** left or right click opens `vpn menu`.
- The icon updates on the next poll, so it can take up to `interval` seconds
  after a click.

Validate `shell.json` after editing:

```bash
jq . ~/.config/omarchy/shell.json > /dev/null && echo valid
```

## Proton exclusivity

Connections whose name starts with `proton-` are full tunnels and exclude each
other: `vpn up proton-DE-210` first takes down any other active `proton-*`
connection, then brings up the new one. All other connections (e.g. `work`,
`office`) are independent and stay untouched.

The grouping is purely a naming convention — keep the `proton-` prefix when
importing new Proton servers.

## Implementation notes

- `vpn_list` parses `nmcli -t` (terse, colon-separated) output. nmcli escapes
  colons inside values as `\:`. `read` is used **without** `-r` on purpose, so
  it treats `\:` as a literal colon instead of a field separator.
  (shellcheck will warn about this — SC2162.)
- `vpn_state <name>` returns the state on stdout and signals "not found" via
  exit code `1`, so callers get both answers from one call:
  `state=$(vpn_state "$1") || …`
- Loops that need to keep variables use process substitution
  (`done < <(vpn_list)`) instead of a pipe, because a pipe runs the loop in a
  subshell.
- All function variables are declared `local` (Bash variables are dynamically
  scoped). Variables assigned from `$(…)` are declared on a separate line,
  because `local x=$(cmd)` always returns `0` and hides `cmd`'s exit code.
- `err=$(nmcli … 2>&1 > /dev/null)` captures **only stderr**: redirections
  are applied left to right, so stderr is pointed at the capture pipe first,
  then stdout is sent to `/dev/null`.
- `vpn bar` builds JSON with `jq --arg`, never by string concatenation, so
  names with quotes or backslashes cannot break the output.
- `vpn menu` collects options in an array and passes them as
  `"${options[@]}"`, so every option stays one argument. Each option is
  `<icon><TAB><name>`; `omarchy-menu-select` shows the icon but returns only
  the name.

## Limitations

- Any NetworkManager connection name is accepted by nmcli, but `vpn` only
  operates on connections of type `wireguard`.
- Whether two tunnels can run at the same time (e.g. `work` + Proton) depends
  on their `AllowedIPs`, DNS and address ranges — not on this script.
- Switching Proton servers currently sends two notifications ("disconnected"
  for the old server, "connected" for the new one).
- The bar command module only offers one alternate colour (the theme's active
  colour); custom colours need a QML plugin (see Roadmap).

## Roadmap

- [x] Omarchy bar widget: status icon, tooltip with active tunnels
      (command module)
- [x] Server selection via the Omarchy menu
- [x] Notifications for connect / disconnect / errors
- [ ] Single "switched from X to Y" notification when changing Proton servers
- [ ] Version B: own QML plugin (`~/.config/omarchy/plugins/<user>.vpn/`)
      with custom colours (e.g. dimmed when off) and a popup panel
- [ ] Extra info: tunnel IP, latest handshake, public IP
- [ ] Event-driven updates: react to NetworkManager state changes
      (`nmcli monitor` / D-Bus signals) instead of polling every few seconds,
      e.g. as a `vpn watch` systemd user service that also sends
      notifications when a tunnel drops
- [ ] Health check: detect tunnels that are "active" but not working.
      WireGuard is connectionless, so a dead server leaves the interface up;
      warn when the latest handshake is older than ~3 minutes
      (check every 30–60 s)
