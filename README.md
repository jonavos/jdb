# jdb

Small bash wrappers around `adb` and `scrcpy` that add friendly mDNS names and an interactive device picker when more than one device is connected.

## Scripts

- **`jdb`** — `adb` drop-in. Resolves friendly names (via `mdns-resolve`) anywhere an IP would normally go, annotates `jdb devices` output with the friendly name, and prompts you to pick a device when more than one is attached and you didn't pass `-s`.
- **`jscrcpy`** — `scrcpy` drop-in with the same friendly-name resolution and device-picker behavior.
- **`mdns-resolve`** — Resolves a friendly name (or bare hostname) to an IP via `avahi-resolve`, using a user-defined alias table. IP literals and unresolvable inputs pass through unchanged.
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

Names without an alias entry are tried as `<name>.local` directly. If `avahi-resolve` fails, the input is passed through to `adb` so things like IP literals still work.

## Multi-device prompt

When more than one device is connected and no `-s`/`--serial` is given, both `jdb` and `jscrcpy` invoke `adb-pick-device`, which lists devices (with friendly names where known) and prompts on `/dev/tty`. Subcommands that don't target a single device (`start-server`, `kill-server`, `pair`, `mdns`, etc.) pass through untouched.
