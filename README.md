# GT7 Telemetry Relay

A small WebSocket <-> UDP bridge server. Flutter Web cannot open raw UDP
sockets, so the dashboard's web build (and any other client that would
rather not deal with UDP/Salsa20 directly) connects to this relay instead
of talking to the PS4/PS5 directly.

The relay must run on a machine on the **same local network** as the
console (e.g. your desktop, laptop, or a Raspberry Pi) - it does the UDP
heartbeat/decrypt/parse work and forwards decoded telemetry as JSON text
frames to any number of connected WebSocket clients.

## Running

```bash
dart pub get
dart run bin/relay.dart --ps-ip 192.168.1.100
```

Omit `--ps-ip` to attempt automatic discovery of a PlayStation on the
local network.

Options:

```
--ps-ip <ip>       IP address of the PlayStation running GT7.
-b, --heartbeat    Heartbeat/packet type: A (default), B (motion data), ~ (extended).
-p, --port         Port to serve the WebSocket bridge on (default 33750).
--version          Print the relay version and exit.
```

On startup the relay prints this computer's LAN IP (preferring the one on
the PlayStation's network) and hostname, and the dashboard addresses built
from them, e.g. `ws://192.168.1.50:33750` and `ws://My-Computer.local:33750`. Clients
receive a JSON-encoded telemetry object (matching `Gt7Telemetry.toJson()`)
for every decoded packet.

## Prebuilt executables

Standalone executables that don't need a Dart SDK are published as GitHub
Release assets for tags named `relay-v<version>`:

| Target | Archive |
| --- | --- |
| Windows x64 | `pit-wall-relay-<version>-windows-x64.zip` |
| macOS Apple Silicon | `pit-wall-relay-<version>-macos-arm64.zip` |
| macOS Intel | `pit-wall-relay-<version>-macos-x64.zip` |
| Linux x64 | `pit-wall-relay-<version>-linux-x64.tar.gz` |
| Linux arm64 (e.g. Raspberry Pi, 64-bit OS) | `pit-wall-relay-<version>-linux-arm64.tar.gz` |

Each archive holds `pit-wall-relay` (`pit-wall-relay.exe` on Windows) and a
`README.txt` generated from [dist_readme.txt](dist_readme.txt). A
`SHA256SUMS.txt` covers all archives. The binaries are unsigned, so macOS
Gatekeeper and Windows SmartScreen warn on first run (README.txt explains
what to do).

### Building the archives

[tool/package.sh](tool/package.sh) compiles with `dart compile exe` and
packages into `dist/` (executables go to `build/`):

```bash
tool/package.sh local                # every target this machine can build + SHA256SUMS.txt
tool/package.sh build linux-arm64    # specific targets
DART=/path/to/flutter/bin/dart tool/package.sh local   # if dart isn't on PATH
```

A machine can build its own platform plus Linux x64/arm64 (cross-compiled).
Windows and the other macOS architecture need a native machine, which the
[relay-release workflow](../.github/workflows/relay-release.yml) provides.

To publish a release: update `version:` in `pubspec.yaml` and `relayVersion`
in `bin/relay.dart` (the script checks they match), commit, push, then push a
`relay-v<version>` tag. The workflow builds all five targets on GitHub-hosted
runners and attaches the archives and `SHA256SUMS.txt` to the release.

GitHub adds "Source code" archives of the repository to every release and
they can't be removed. To publish downloads without this repository's code,
create a separate public repository (e.g. `durfu/pit-wall-relay`) holding
only a README, then in this repository's Settings → Secrets and variables →
Actions set the variable `RELEASE_REPO` to its name and the secret
`RELEASES_TOKEN` to a fine-grained token with Contents read/write on that
repository only. Releases then go there, and its source archives contain
just that README.

## Control API - setting/changing the PlayStation IP without a terminal

A browser tab can't launch or restart this process for you (browsers can't
spawn processes, same restriction as the raw-UDP one), so the relay exposes
a tiny HTTP control API on the **same port** as the WebSocket bridge, which
the dashboard's Settings screen uses under the hood:

```
GET  /status   -> { "psIp": "...", "heartbeat": "A", "connectedClients": 0, "hasTelemetry": false }
POST /target   body: {"psIp": "192.168.1.100"}  -> re-points the relay at a different console
```

This means you only have to start the relay **once** (ideally with no
`--ps-ip` at all, letting it auto-discover) - after that, the console IP can
be read or changed from the dashboard app itself (any platform, including
Web), and existing WebSocket clients stay connected through the switch. You
can also call it directly, e.g.:

```bash
curl http://192.168.1.50:33750/status
curl -X POST -H 'Content-Type: application/json' \
  -d '{"psIp":"192.168.1.100"}' http://192.168.1.50:33750/target
```

