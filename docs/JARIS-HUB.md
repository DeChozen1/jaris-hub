# Jaris Hub on your Samsung S90D

Jaris Hub is this personal fork of [Seagey/couch-tv](https://github.com/Seagey/couch-tv). It keeps the upstream TV interface, five photographic Night scenes, large frosted clock, weather display, app navigation and remote controls. This fork supplies the Jaris Hub name and Singapore-first defaults.

The upstream MIT licence and attribution are retained. The Samsung app's internal Tizen application ID remains `CouchTVApp.Home` so the existing Samsung bridge can launch it without a compatibility change.

## Defaults in this fork

- Location and weather: Singapore, using Open-Meteo.
- Time zone: `Asia/Singapore`; clock: 24-hour.
- Language: English.
- Home look: Night, with the original Night, Golden, Water, Fog and Dawn scenes.
- Glass panels and glass app icons: enabled.
- Clock style: the original frosted style.
- Samsung TV profile: `living-room`, with the IP left as a placeholder.
- Home Assistant: off until local entity IDs and a dedicated token are configured.
- Proxmox, Grafana and Prometheus: intentionally deferred.

## What you need

- A Windows, macOS or Linux computer on the same home network as the TV.
- For automatic launch on power-on, that computer must stay on and run the bridge.
- Python 3.11 or newer for the bridge.
- Apps2Samsung v3 or Tizen Studio to sign and install the Samsung `.wgt` package.
- A stable/reserved IP address for the TV and for the bridge computer.

## 1. Clone the fork and prepare Python (Windows PowerShell)

```powershell
git clone https://github.com/DeChozen1/jaris-hub.git
cd jaris-hub
py -3.11 --version
py -3.11 -m venv .venv
.venv\Scripts\python.exe -m pip install --upgrade pip
.venv\Scripts\python.exe -m pip install -r bridge\requirements.txt
```

On macOS or Linux, use the upstream commands in the main README, replacing the repository URL with this fork.

## 2. Make the private configuration

Copy the example file. The resulting `config.json` is ignored by Git and should stay local:

```powershell
Copy-Item config.example.json config.json
```

Open `config.json` and replace `SET_YOUR_TV_IP` with the Samsung S90D's current LAN IP. Keep the TV entry's `kind` set to `samsung`. Set `bridge.host` to the computer's LAN hostname if you have one; when building the TV app, use the computer's numeric IP address in the `--bridge` URL.

Reserve both IPs in your router's DHCP settings so they do not change.

Do not put a Samsung signing certificate, bridge pairing key, Home Assistant token, or other secret into a tracked file.

## 3. Start the bridge

From the repository folder:

```powershell
.venv\Scripts\python.exe bridge\server.py
```

The bridge listens on port 5400. Keep this terminal open while testing. From another device on the same Wi-Fi, check that `http://<bridge-ip>:5400` loads.

For permanent use, install the bridge as a Windows startup service/task or run it on an always-on Linux host. The host must not sleep. The upstream README documents the Linux systemd and Docker options.

## 4. Enable Developer Mode on the S90D

1. Make sure the TV and bridge/build computer are on the same network.
2. On the TV open Apps, then App Settings.
3. Enter `12345` using the remote or the on-screen keypad.
4. Turn Developer Mode on and enter the computer's LAN IP address (the computer used for installation).
5. Save and restart the TV.

Samsung's current developer instructions use Apps → App Settings → 12345 → Developer Mode, host computer IP, then restart. The exact menu labels can vary slightly by firmware.

## 5. Build the Jaris Hub Samsung package

Use the bridge computer's numeric IP address, not a `.local` hostname:

```powershell
$BRIDGE_IP = "192.168.50.x"  # Replace with the bridge computer's reserved LAN IP
.venv\Scripts\python.exe samsung\build.py --bridge "http://$BRIDGE_IP:5400" --tv living-room
```

The unsigned package is created at:

```
build\samsung\JarisHub.wgt
```

The package includes the home page, backgrounds, fonts, Samsung app mappings and a copy of the public settings. It does not contain your Home Assistant token or the bridge's Samsung pairing token.

## 6. Sign and install

Install Apps2Samsung on the computer, open it, select the S90D and choose the local `JarisHub.wgt` file as the app to install. The first installation requires a Samsung sign-in so the installer can create a certificate for your TV. The certificate is retained by Apps2Samsung for later updates.

The command-line route is also supported:

```powershell
$TV_IP = "192.168.50.x"  # Replace with the S90D's reserved LAN IP
apps2samsung-cli tv $TV_IP
apps2samsung-cli install build\samsung\JarisHub.wgt --tv $TV_IP
```

## 7. Pair the bridge and test automatic launch

1. With the TV on, send a key through the bridge or use the phone remote. The TV should ask whether to allow the Jaris Hub bridge to connect.
2. Press Allow on the TV. Samsung stores a pairing token on the bridge computer under `bridge/data/`.
3. Restart the TV. With the bridge running and `open_on_power_on: true`, Jaris Hub should open a few seconds after the TV becomes active.
4. Test the arrow keys, OK, Back, sleep timer and Look control.
5. Open Home settings and arrange the installed apps. Put beIN SPORTS and mewatch near the front if they are installed. Add HDMI input shortcuts only after confirming which physical ports correspond to each device.

Samsung does not permit a third-party app to replace the TV's native Home screen. The Samsung Home button still opens Samsung's menu; use Back or the app row to return to Jaris Hub. The bridge is what reopens Jaris Hub automatically on power-on.

## 8. Optional Home Assistant integration

Home Assistant remains disabled in the tracked example. To enable it locally:

1. Edit the private `config.json` and set `home_assistant.enabled` to `true`.
2. Change the URL to the Home Assistant address reachable from the TV.
3. Replace the sample climate, scene/script and doorbell entity IDs with real IDs from Home Assistant.
4. Create a dedicated non-administrator Home Assistant user and create its long-lived access token.
5. Put the token in the ignored `.env` file as `HOME_ASSISTANT_TOKEN=...`.
6. Rebuild the WGT and reinstall it.

Never paste the token into chat or commit it. The current doorbell integration displays a camera still image, not a live video stream.

## Remaining after the repository changes

- Confirm the S90D IP and reserve it in DHCP.
- Choose the always-on bridge host and reserve its IP.
- Enable Developer Mode on the TV and set the host IP.
- Install Apps2Samsung (or use Tizen Studio), sign the package and install it.
- Approve the bridge pairing prompt on the TV.
- Test on the actual S90D; upstream's Samsung real-TV test was on a 2021 Frame, so S90D-specific behaviour still needs a hands-on check.
- Arrange beIN SPORTS, mewatch and confirmed HDMI inputs.
- Configure Home Assistant entity IDs and token separately if you want the smart-home panel.
- Add Proxmox/Grafana/Prometheus only in a later phase.
