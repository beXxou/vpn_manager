# vpn-manager

A small Bash CLI to list, start and stop WireGuard VPN connections through
NetworkManager — without `sudo`. It is the backend for a planned Omarchy
bar widget.

## Why

Previously all tunnels were managed with `sudo wg-quick up/down <name>`.
`wg-quick` needs root, so a bar widget could not toggle tunnels.
NetworkManager runs as a system daemon and lets the logged-in user switch
connections via polkit, so `nmcli connection up <name>` works as a normal user.

## Requirements

- NetworkManager (`nmcli`)
- WireGuard support in the kernel (Linux ≥ 5.6)
- WireGuard configs imported into NetworkManager (see Setup)

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

`~/.local/bin` must be in `$PATH`.

## Usage

```bash
vpn list              # all WireGuard connections with their state
vpn up <name>         # bring a connection up
vpn down <name>       # take a connection down
```

Example:

```
$ vpn list
office inactive
proton-BE-30 inactive
proton-DE-210 active
work active
```

Behavior:

- `up` / `down` are **idempotent**: bringing up an already active connection
  (or taking down an inactive one) prints a message and exits with `0`.
- Unknown names print an error to stderr and exit with `1`.
- Output is sorted by name, so the order stays stable when states change.

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
  scoped).

## Limitations

- Any NetworkManager connection name is accepted by nmcli, but `vpn` only
  operates on connections of type `wireguard`.
- Whether two tunnels can run at the same time (e.g. `work` + Proton) depends
  on their `AllowedIPs`, DNS and address ranges — not on this script.

## Roadmap

- [ ] Omarchy bar widget (`~/.config/omarchy/plugins/<user>.vpn/`):
      status icon, tooltip with active tunnel, click to toggle
- [ ] Server selection panel or Omarchy menu entry
- [ ] Extra info: tunnel IP, latest handshake, public IP
