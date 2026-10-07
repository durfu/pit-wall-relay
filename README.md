# Pit Wall Relay

**[Website & download page](https://durfu.ro/pitwall/#relay)** ·
**[Get Pit Wall on Google Play](https://play.google.com/store/apps/details?id=com.durfu.pitwall)**

Pit Wall Relay connects to **Gran Turismo 7** on your PS4 or PS5, receives
its live telemetry over your local network, and passes it on to **Pit Wall**
dashboards on any device: a phone, a tablet, another computer or a web
browser.

- The **web version** of Pit Wall needs it: browsers can't receive the
  console's telemetry directly.
- It lets **several dashboards share one connection** to the console, so you
  can watch on a tablet in the rig and a laptop on the desk at the same time.

Run it on any computer or Raspberry Pi on the **same network as the
PlayStation** and leave it running while you drive. It's a single program
with nothing to install.

## Download

The easiest way is the **[download page](https://durfu.ro/pitwall/#relay)**:
it picks the right file for your computer and walks you through setup.

Or **[download the latest release](https://github.com/durfu/pit-wall-relay/releases/latest)**
here and pick the file for your computer:

| Your computer | File to download |
| --- | --- |
| Windows (64-bit) | `pit-wall-relay-<version>-windows-x64.zip` |
| Mac with Apple silicon (M1 and later) | `pit-wall-relay-<version>-macos-arm64.zip` |
| Mac with an Intel processor | `pit-wall-relay-<version>-macos-x64.zip` |
| Linux PC (64-bit) | `pit-wall-relay-<version>-linux-x64.tar.gz` |
| Raspberry Pi 3/4/5 with 64-bit Raspberry Pi OS | `pit-wall-relay-<version>-linux-arm64.tar.gz` |

Not sure which Mac you have? Apple menu → **About This Mac**: "Chip: Apple
M…" is Apple silicon, "Processor: … Intel" is Intel. On Linux, `uname -m`
prints `x86_64` (linux-x64) or `aarch64` (linux-arm64).

Each download contains the `pit-wall-relay` program (`pit-wall-relay.exe` on
Windows) and a `README.txt` with these instructions.

## Quick start

1. **Extract** the download.
2. **Start Gran Turismo 7** on the PlayStation.
3. **Run the relay** from a terminal in the extracted folder:

   | | Command |
   | --- | --- |
   | macOS / Linux (Terminal) | `./pit-wall-relay` |
   | Windows (Command Prompt or PowerShell) | `.\pit-wall-relay.exe` |

   It searches your network for the PlayStation. If it can't find it, give
   it the console's IP address (PS5: Settings → Network → Connection Status;
   PS4: Settings → Network → View Connection Status):

   ```
   ./pit-wall-relay --ps-ip 192.168.1.80
   ```

   The first time, your computer may ask for permission; see
   [First run](#first-run) below.

4. **Note the address it prints**, for example:

   ```
   This computer on the network:
     LAN IP:    192.168.1.50   (en0, same network as the PlayStation)
     Hostname:  My-Computer.local

   In the dashboard on another device, choose relay mode and enter:
     ws://192.168.1.50:33750
     ws://My-Computer.local:33750   (keeps working if the IP changes; needs .local name support)
   On this computer: ws://localhost:33750
   ```

5. **Connect Pit Wall:** open the connection settings (tap the status pill
   at the top), choose **Relay**, and enter one of those addresses. The
   hostname keeps working if the computer's IP address changes; if a device
   can't resolve it (some Android devices and networks can't), use the IP
   address instead.

6. **Get on track** in GT7. Live values only appear while you're in a
   session (practice, race, time trial and so on), not in the menus.

Press **Ctrl+C** in the terminal to stop the relay.

## First run

The relay isn't code-signed, so macOS and Windows ask before running it the
first time.

**macOS**
- Gatekeeper blocks the first run ("cannot be opened" / "Apple could not
  verify"). Either run this once in the extracted folder:
  ```
  xattr -d com.apple.quarantine pit-wall-relay
  ```
  or try to run it once, then go to **System Settings → Privacy & Security**
  and click **Open Anyway**.
- If macOS asks to let it find devices on your local network or accept
  incoming connections, click **Allow**.

**Windows**
- If SmartScreen shows "Windows protected your PC", click **More info**, then
  **Run anyway**.
- When Windows Defender Firewall asks, allow access on **Private networks**,
  and make sure your home network is set to Private, not Public. If you
  clicked Cancel, allow `pit-wall-relay.exe` under Windows Security →
  Firewall & network protection → Allow an app through firewall.
- Run it from Command Prompt or PowerShell rather than double-clicking, so
  the window stays open and you can see the address it prints.

**Linux**
- If the file isn't executable after extracting: `chmod +x pit-wall-relay`.
- If you use a firewall (e.g. ufw), allow `33750/tcp` and `33740/udp`.

## Options

| Option | What it does |
| --- | --- |
| `--ps-ip <ip>` | The PlayStation's IP address. Searched for automatically if omitted. |
| `-b, --heartbeat <A\|B\|~>` | Telemetry packet type. `A` (default) is the standard packet. `B` adds motion data (sway, heave, surge, wheel rotation). `~` adds filtered throttle/brake and energy recovery. |
| `-p, --port <port>` | Port dashboards connect to (default `33750`). |
| `--version` | Print the version and exit. |
| `-h, --help` | Show help. |

Once the relay is running you can switch to a different PlayStation from
Pit Wall's connection settings; there's no need to restart it.

## Network

- The relay and the PlayStation must be on the **same local network**.
  Dashboards connect to the relay, so they need to reach that computer too.
  Guest Wi-Fi networks often block devices from seeing each other.
- Ports: **TCP 33750** for dashboards (change with `--port`), and **UDP
  33739/33740** between the relay and the PlayStation.

## Keep it running on a Raspberry Pi (or any Linux machine)

To start the relay automatically at boot, extract it (for example to
`/home/pi/pit-wall-relay`) and create a systemd service:

```bash
sudo nano /etc/systemd/system/pit-wall-relay.service
```

```ini
[Unit]
Description=Pit Wall Relay
After=network-online.target
Wants=network-online.target

[Service]
User=pi
ExecStart=/home/pi/pit-wall-relay/pit-wall-relay
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Adjust `User` and the path to match your setup, and add `--ps-ip <ip>` to
`ExecStart` if the PlayStation has a fixed address. Then:

```bash
sudo systemctl enable --now pit-wall-relay
journalctl -u pit-wall-relay -f    # shows the address to enter in Pit Wall
```

If the PlayStation is off when the Pi boots, the relay exits and systemd
retries every 10 seconds until it finds the console.

## Troubleshooting

**"Could not find a PlayStation automatically"**
: Make sure the console is on and on the same network, then pass its
  address with `--ps-ip`.

**Pit Wall says "Waiting for telemetry" or "Waiting for the track"**
: Check the relay is still running and connected to the right PlayStation
  address. Live values only appear while your car is on track in GT7 (a
  race, practice, time trial and so on), not in the menus.

**A dashboard can't connect to the relay**
: Check the device is on the same network, the firewall prompt was allowed
  (on Windows, for Private networks), and try the IP address instead of the
  hostname.

**The web version of Pit Wall can't connect**
: Browsers block `ws://` connections from pages served over `https://`.
  Open the web dashboard over `http://` on your local network, or use the
  Pit Wall app.

**On Windows the window closes straight away**
: Run it from Command Prompt or PowerShell to see the message.

## Verify your download

Each release includes `SHA256SUMS.txt`. In the folder with your download:

```bash
shasum -a 256 -c SHA256SUMS.txt --ignore-missing          # macOS
sha256sum -c SHA256SUMS.txt --ignore-missing              # Linux
```

On Windows, run `Get-FileHash .\pit-wall-relay-<version>-windows-x64.zip`
in PowerShell and compare the hash with the line in `SHA256SUMS.txt`.

---

This repository only hosts release downloads. Pit Wall itself, the
dashboard app, is on
[Google Play](https://play.google.com/store/apps/details?id=com.durfu.pitwall);
more at [durfu.ro/pitwall](https://durfu.ro/pitwall).

Pit Wall is not affiliated with or endorsed by Sony Interactive
Entertainment or Polyphony Digital. Gran Turismo is a registered trademark
of Sony Interactive Entertainment Inc.
