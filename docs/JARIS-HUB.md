# Jaris Hub on your Samsung S90D

Jaris Hub is this personal fork of [Seagey/couch-tv](https://github.com/Seagey/couch-tv). It keeps the upstream TV interface, five photographic Night scenes, large frosted clock, weather display, app navigation and remote controls. This fork supplies the Jaris Hub name and Singapore-first defaults.

The upstream MIT licence and attribution are retained. The Samsung app's internal Tizen application ID remains `CouchTVApp.Home` so the existing Samsung bridge can launch it without a compatibility change.

## Defaults in this fork

- Location and weather: Singapore, using Open-Meteo.
- Time zone: `Asia/Singapore`; clock: 24-hour.
- Language: English.
- Home look: Night, with the original Night, Golden, Water, Fog and Dawn scenes.
- Glass panels and glass app icons: enabled; original frosted clock style retained.
- Samsung TV profile: `living-room`; confirm the local IP before deployment.
- Home Assistant: off until local entity IDs and a dedicated token are configured.
- Proxmox, Grafana and Prometheus: intentionally deferred.

## Deployment layout

Use a small **Ubuntu 24.04 LXC container on the Beelink's Proxmox host** for the always-on bridge. Do not install the bridge's Python packages directly on the Proxmox host. Build and install the Samsung app from your Windows PC, where Apps2Samsung or Tizen Studio can sign and install it.

The bridge container and the S90D must be on the same home LAN. In your previous network layout that is likely Proxmox's `vmbr0`, but check the bridge's physical/network attachment before selecting it. The bridge listens on TCP port 5400. Do not port-forward that port to the internet.

## 1. Create the bridge LXC in Proxmox

In the Proxmox web interface:

1. Open the node on the Beelink and go to **local storage → CT Templates → Templates**. Download the latest **Ubuntu 24.04 standard** template.
2. Select **Create CT**. Give it CT ID `120` (or the next free ID) and hostname `jaris-bridge`.
3. Use an **unprivileged** container. A password is needed for the root console; an SSH key is fine too.
4. Select the Ubuntu 24.04 template, an 8 GB root disk, 1 CPU core, 1024 MB RAM and 512 MB swap.
5. On Network, select the Proxmox bridge connected to the same LAN as the S90D (likely `vmbr0` in the earlier layout), set IPv4 to DHCP, and leave VLAN tag empty unless your LAN actually uses one.
6. Enable **Start at boot**. Do not enable nesting or privileged mode; this bridge does not need Docker or device passthrough.
7. Finish, start the container and open its Console.

After it starts, find its LAN address:

```sh
ip -4 addr show
ip route
```

Reserve that address against the container's MAC address in your router's DHCP reservation page. Also reserve the S90D's address. This keeps the bridge address stable after reboots. Use the same home LAN as the TV, not a guest Wi-Fi or isolated IoT network.

If the Proxmox firewall is enabled for this container, allow **TCP 5400 from your home LAN**. The bridge makes outbound connections to the Samsung TV on the TV's local network ports; no internet port forwarding is needed.

## 2. Install the bridge in the LXC

Run these commands in the LXC Console as `root`:

```sh
apt update
apt full-upgrade -y
apt install -y git python3 python3-venv python3-pip ca-certificates

useradd --system --no-create-home --shell /usr/sbin/nologin --user-group jaris
install -d -o jaris -g jaris /opt/jaris-hub

runuser -u jaris -- git clone --single-branch --branch feature/jaris-hub-branding https://github.com/DeChozen1/jaris-hub.git /opt/jaris-hub
runuser -u jaris -- python3 -m venv /opt/jaris-hub/.venv
runuser -u jaris -- /opt/jaris-hub/.venv/bin/pip install --upgrade pip
runuser -u jaris -- /opt/jaris-hub/.venv/bin/pip install -r /opt/jaris-hub/bridge/requirements.txt
```

The bridge will run as the unprivileged Linux service account `jaris`, not as root.

## 3. Make the private bridge configuration

Create the private config from the example:

```sh
cd /opt/jaris-hub
cp config.example.json config.json
nano config.json
```

Edit these values in `config.json`:

- `bridge.host`: the container's reserved numeric IPv4 address, for example `192.168.50.X`.
- `bridge.port`: leave as `5400`.
- `bridge.pairing_key`: generate a private key with the command below and paste the result between the JSON quotation marks.
- `tvs[0].ip`: the S90D's current/reserved IPv4 address.
- `tvs[0].kind`: keep as `samsung`.
- `tvs[0].open_on_power_on`: keep as `true`.

Generate a key locally (do not send it in chat or commit it):

```sh
/opt/jaris-hub/.venv/bin/python -c "import secrets; print(secrets.token_urlsafe(24))"
```

Use the same key in the private Windows `config.json` in step 7, so the WGT can authenticate to this bridge. The WGT contains that key; keep the WGT private and do not upload it to a public release.

Set file permissions:

```sh
chown jaris:jaris /opt/jaris-hub/config.json
chmod 600 /opt/jaris-hub/config.json
```

The `config.json` and `.env` files are ignored by Git. Never commit them.

## 4. Run the bridge automatically with systemd

Create the system service:

```sh
cat > /etc/systemd/system/jaris-hub-bridge.service <<'EOF'
[Unit]
Description=Jaris Hub bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=jaris
Group=jaris
WorkingDirectory=/opt/jaris-hub
EnvironmentFile=-/opt/jaris-hub/.env
Environment=PYTHONUNBUFFERED=1
ExecStart=/opt/jaris-hub/.venv/bin/python /opt/jaris-hub/bridge/server.py
Restart=always
RestartSec=5
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now jaris-hub-bridge
systemctl status jaris-hub-bridge --no-pager
```

Check the log if it does not start:

```sh
journalctl -u jaris-hub-bridge -n 100 --no-pager
```

From a computer on the same home LAN, check the health endpoint:

```sh
curl http://<bridge-container-ip>:5400/api/health
```

It should return a small JSON health response. The bridge should remain active after the LXC is rebooted.

## 5. Enable Developer Mode on the S90D

The **host PC IP in Developer Mode must be the Windows PC running Apps2Samsung/Tizen Studio**, not the LXC bridge IP.

1. Make sure the Windows PC and TV are on the same LAN.
2. On the S90D, open Apps, then App Settings.
3. Enter `12345` using the remote or on-screen keypad.
4. Turn Developer Mode on and enter the Windows PC's LAN IPv4 address.
5. Save and restart the TV.

Samsung's official [Tizen device setup guide](https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-device.html) shows Apps → App Settings → 12345 → Developer mode → host PC IP → restart. Menu wording can differ slightly by firmware.

## 6. Build the personalised WGT on Windows

On the Windows PC, install Python 3.11 or newer if it is not already available. Open PowerShell:

```powershell
git clone --single-branch --branch feature/jaris-hub-branding https://github.com/DeChozen1/jaris-hub.git
cd jaris-hub
Copy-Item config.example.json config.json
```

Edit this Windows copy of `config.json` with the same bridge IP, the same private pairing key, and the S90D's current IP used in the LXC config. Keep the pairing key private.

Build the package with the bridge address:

```powershell
$BRIDGE_IP = "192.168.50.X"  # Container's reserved IP
py -3.11 samsung\build.py --bridge "http://$BRIDGE_IP:5400" --tv living-room
```

The personalised package is:

```
build\samsung\JarisHub.wgt
```

This is the package to install. The generic GitHub Actions artifact is only a build check; it does not contain your local bridge address or private pairing key.

## 7. Sign and install with Apps2Samsung

Apps2Samsung is the simplest route from Windows:

1. Install it with `winget install Apps2Samsung.Apps2Samsung`, or download the Windows release from [Apps2Samsung releases](https://github.com/Apps2Samsung/Apps2Samsung/releases).
2. Open Apps2Samsung and scan/select the S90D.
3. Choose the option to install a custom/local WGT and select `build\samsung\JarisHub.wgt`.
4. On the first install, complete the Samsung sign-in so Apps2Samsung can create the certificate for this TV. Later installs reuse it.
5. Confirm the app appears in the TV's Apps list as **Jaris Hub**.

The alternative CLI route, with `apps2samsung-cli` installed, is:

```powershell
$TV_IP = "192.168.50.X"  # S90D's reserved IP
apps2samsung-cli tv $TV_IP
apps2samsung-cli install build\samsung\JarisHub.wgt --tv $TV_IP
```

## 8. Pair and test the TV

1. Leave the bridge service running and open Jaris Hub once from the TV's Apps list.
2. Use the app or the phone remote to send a key. When Samsung asks to allow the Jaris Hub bridge to connect, press **Allow**.
3. Restart the TV. With `open_on_power_on: true`, the bridge should reopen Jaris Hub a few seconds after the TV becomes active.
4. Test the remote arrows, OK, Back, Sleep and Look controls.
5. Open Home settings and arrange installed apps. Put beIN SPORTS and mewatch near the front if installed. The Input quick button sends Samsung's Source command through the bridge so you can choose an HDMI port from the TV's own Source menu.

Samsung does not permit a third-party app to replace its native Home screen. The Samsung Home button still opens Samsung's menu; use Back, the app row or the bridge's power-on hook to return to Jaris Hub.

## 9. Optional Home Assistant integration (later)

Home Assistant remains disabled for the first TV deployment. When ready:

1. Set `home_assistant.enabled` to `true` in the private configuration.
2. Use the HA URL reachable from the TV.
3. Replace sample climate, scene/script, doorbell and camera entity IDs with the actual IDs from Home Assistant.
4. Create a dedicated non-administrator Home Assistant user and create its long-lived access token.
5. Put the token in the ignored `.env` file as `HOME_ASSISTANT_TOKEN=...`.
6. Rebuild and reinstall the WGT.

Never paste the token into chat or commit it. The current doorbell integration displays a camera still image, not a live video stream.

## Remaining checklist

- [ ] Create the Ubuntu LXC on the Beelink Proxmox node.
- [ ] Give it a stable LAN IP and ensure it shares the S90D's LAN.
- [ ] Install the bridge, create the private config and enable the systemd service.
- [ ] Enable S90D Developer Mode using the Windows PC's IP.
- [ ] Build the personalised WGT on Windows, sign it and install it.
- [ ] Approve the Samsung remote connection and verify power-on launch on the S90D.
- [ ] Arrange beIN SPORTS, mewatch and the Source shortcut.
- [ ] Configure Home Assistant later if wanted.
- [ ] Add Proxmox/Grafana/Prometheus in a separate later phase.
