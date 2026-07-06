# jdb

Small bash wrappers around `adb` and `scrcpy` that add friendly mDNS names and an interactive device picker when more than one device is connected.

## Scripts

- **`jdb`** — `adb` drop-in. Resolves friendly names (via `mdns-resolve`) anywhere an IP would normally go, annotates `jdb devices` output with the friendly name, and prompts you to pick a device when more than one is attached and you didn't pass `-s`.
- **`jscrcpy`** — `scrcpy` drop-in with the same friendly-name resolution and device-picker behavior.
- **`mdns-resolve`** — Resolves a friendly name (or bare hostname/serial) to an IP. Tries `avahi-resolve` (mDNS) first; when that fails it falls back to a **Neat Pulse** API lookup by serial. IP literals and unresolvable inputs pass through unchanged.
- **`adb-pick-device`** — Prints the serial of the connected device, prompting via `/dev/tty` when more than one is attached.

## Install

Drop the scripts somewhere on your `PATH`:

```sh
ln -s "$PWD"/{jdb,jscrcpy,mdns-resolve,adb-pick-device} ~/.local/bin/
```

Requires `adb`, `avahi-resolve` (Debian/Ubuntu: `avahi-utils`), and optionally `scrcpy`.

## Friendly name aliases

Create `~/.config/mdns-resolve/hosts` as a shell file mapping aliases to mDNS hostnames:

```sh
pixel=pixel-8.local
tablet=galaxy-tab.local
kiosk=lobby-kiosk.local
```

Then:

```sh
jdb connect pixel              # resolves "pixel" → mDNS → IP, adb connect <ip>:4242
jdb connect pixel tablet kiosk # connect to several devices in one go
jdb -s tablet shell             # -s gets resolved too
jdb -s kiosk:5555 logcat        # explicit port honored; otherwise defaults to 4242
jdb devices                     # rows get prefixed with friendly names
```

Names without an alias entry are tried as `<name>.local` directly. If `avahi-resolve` fails, a serial-shaped input (two letters + digits) is looked up via the Pulse fallback; anything else — including IP literals — is passed through to `adb` unchanged so it can do its own resolution.

## Pulse fallback (working off-LAN / over Tailscale)

mDNS is link-local multicast and doesn't traverse a Tailscale subnet route, so `avahi-resolve` fails when you're not on the office LAN. As a fallback, `mdns-resolve` looks the device up by **serial** in the [Neat Pulse](https://api.pulse.neat.no/docs/) API and returns its current `localIpAddress`. The alias's mDNS hostname doubles as the serial (e.g. `b1=NB12249000245.local` → serial `NB12249000245`).

Configure credentials in `~/.config/mdns-resolve/pulse.conf` (chmod `600` — **keep the API key out of any repo**):

```sh
PULSE_API_BASE="https://api.pulse.neat.no"
PULSE_ORG="LXJ45kN"            # Pulse → Settings → organization ID
PULSE_API_KEY="…"             # Pulse → Settings → API keys (Read scope)
PULSE_TTL="3600"              # cache resolved IPs this many seconds
```

Resolved IPs (and the org endpoint list) are cached under `~/.cache/mdns-resolve/` for `PULSE_TTL` seconds. Requires `curl` and `jq`.

## Serial verification on connect

`jdb connect <name>` reads the device's serial via `getprop` immediately after connecting and **disconnects unless the expected Neat serial is present** — guarding against a stale or reused IP (from either mDNS or Pulse) pointing at the wrong device. Neat devices expose the serial several ways (`ro.serialno`, `ro.boot.serialno`, `net.hostname`, …), so a legitimate device always matches; a different device — or one that can't be read (offline/unauthorized) — is dropped. Verification is skipped for IP literals and plain hostnames that aren't serials.

## Multi-device prompt

When more than one device is connected and no `-s`/`--serial` is given, both `jdb` and `jscrcpy` invoke `adb-pick-device`, which lists devices (with friendly names where known) and prompts on `/dev/tty`. Subcommands that don't target a single device (`start-server`, `kill-server`, `pair`, `mdns`, etc.) pass through untouched.
