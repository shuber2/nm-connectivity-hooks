This script runs your own hook scripts whenever the connectivity state of
NetworkManager changes, for instance, when you join a WiFi that has no uplink
to the internet, like a train WiFi, or when you are behind a captive portal.

NetworkManager's own dispatcher has a `connectivity-change` event, but its
scripts must be owned by root and run as root, outside your session. This
script runs as a systemd user unit instead, so the hooks run as you, with access
to the session D-Bus, desktop notifications, and so on. It needs nothing but
`bash` and `gdbus` (glib).

## How it works

The script listens on the system D-Bus for changes of NetworkManager's
`Connectivity` property. Each time the state changes, and once at startup, it
runs all executable files in `~/.config/nm-connectivity-hooks/hooks.d/` in
alphabetical order:

```
HOOK NEW_STATE OLD_STATE
```

The states are `unknown`, `none`, `portal`, `limited`, and `full`. `OLD_STATE`
is empty at startup. They are also exported as `NM_CONNECTIVITY_STATE` and
`NM_CONNECTIVITY_OLD_STATE`. A new state must persist for `SETTLE_TIME` seconds
before the hooks run, so a short glitch does not trigger them. Hooks should be
idempotent.

NetworkManager only knows a state other than `unknown` if connectivity checking
is enabled, which is the default on most distributions. Check with:
```sh
nmcli networking connectivity check
```

## Installation

1. Put `nm-connectivity-hooks` to `~/.local/bin`.

2. Put `nm-connectivity-hooks.service` to `~/.config/systemd/user`. It launches
   `~/.local/bin/nm-connectivity-hooks`. If you install the script elsewhere,
   adapt `ExecStart=` accordingly: systemd does not search `$PATH` for it.

3. Put your hooks into `~/.config/nm-connectivity-hooks/hooks.d/` and make them
   executable.

Enable the service:
```sh
systemctl --user daemon-reload
systemctl --user enable --now nm-connectivity-hooks.service
```

## Configuration

The optional config file `~/.config/nm-connectivity-hooks.conf` is sourced by
the script. These are the defaults:
```sh
HOOK_DIR="$HOME/.config/nm-connectivity-hooks/hooks.d"
SETTLE_TIME=3
```

Variables you `export` there are visible to the hooks.

## Usage

Watch what happens:
```
journalctl --user -u nm-connectivity-hooks.service
```

To test your hooks without leaving the network, run them once by hand:
```sh
nm-connectivity-hooks --run-hooks limited full   # pretend the uplink is gone
nm-connectivity-hooks --run-hooks full limited   # pretend it is back
```

## Hooks

### akonadi-offline

The Akonadi EWS resource, which syncs Exchange and Office 365 mail and
calendars in KDE PIM, does not back off when the network is unreachable. On a
WiFi without uplink it retries in a tight loop, logging thousands of `Failed to
process EWS request: Network unreachable` messages per second and keeping the
CPU busy. See KDE [bug 422410](https://bugs.kde.org/show_bug.cgi?id=422410).

This is fixed in kdepim-runtime 26.12 (KDE Gear 26.12.0) by
[MR !384](https://invent.kde.org/pim/kdepim-runtime/-/merge_requests/384): the
EWS resource now backs off and goes offline by itself without full
connectivity. The hook is only needed for older versions.

The hook `hooks/akonadi-offline` takes the matching Akonadi resources offline
while connectivity is `none`, `portal`, or `limited`, and brings them back
online at `full`. While offline, the calendar is still readable from Akonadi's
local cache, and your changes are synced once the resource is online again.

The resources are selected by glob patterns, by default all EWS resources. To
change that, put into `~/.config/nm-connectivity-hooks.conf`, e.g.:
```sh
export AKONADI_RESOURCES="akonadi_ews_resource_* akonadi_davgroupware_resource_*"
```

The hook only brings back those resources that it took offline itself, which it
remembers in `~/.local/state/nm-connectivity-hooks/akonadi-offline`. A resource
you took offline by hand therefore stays offline. If Akonadi is not running
when the uplink comes back, the hook retries at the next state change.
