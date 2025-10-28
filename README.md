# Drone Video Transport — README

**Repo:** https://github.com/Catra2/drone

**Goal (v0):** USB webcam on **Raspberry Pi 3** (on the drone) → **WebRTC** → **Windows** ground-station (**PyQt GUI**). Prefer **2.4 GHz Wi-Fi**; fall back to **LTE (Quectel EC25-A)**. Include **safety** (RF-mute, Return-to-Home, Emergency landing). **Auto-update** on launch (GUI & scripts).  
**Upgrade path:** Downward-facing + forward cameras, **1080p30+**, PC-VR (Quest 3 via Link/Air Link), optional on-ground **YOLO**, **TURN** relay on home RPi (Hawaiʻi).  
**Design tenets:** Config-driven, CLI first → C/C++ daemon later → GUI control; robust reconnection; Wi-Fi-first routing; Windows-only ground station for v0.

---

## Table of Contents
1. [System Overview](#system-overview)
2. [User-Visible Features & Controls (v0)](#user-visible-features--controls-v0)
3. [End-to-End Flow (Story of a Frame)](#end-to-end-flow-story-of-a-frame)
4. [Phased Milestones (Definition of Done)](#phased-milestones-definition-of-done)
5. [Repository Layout](#repository-layout)
6. [Install & Sanity Checks (Raspberry Pi 3)](#install--sanity-checks-raspberry-pi-3)
7. [Networking Policy: Wi-Fi First, LTE Fallback](#networking-policy-wi-fi-first-lte-fallback)
8. [Signaling & Receiver (Browser Proof on LAN)](#signaling--receiver-browser-proof-on-lan)
9. [Sender on Pi: CLI now → C/C++ Daemon later](#sender-on-pi-cli-now--ccc-daemon-later)
10. [Resilience, Recovery, and Latency Targets](#resilience-recovery-and-latency-targets)
11. [TURN Relay (Drop-in, Config-Only)](#turn-relay-drop-in-config-only)
12. [Ground-Station GUI (PyQt on Windows) & Packaging](#ground-station-gui-pyqt-on-windows--packaging)
13. [Auto-Update via GitHub Pages](#auto-update-via-github-pages)
14. [GitHub Basics (for this repo)](#github-basics-for-this-repo)
15. [Diagnostics & Measurement](#diagnostics--measurement)
16. [Hardware Notes & Upgrade Path](#hardware-notes--upgrade-path)
17. [Roadmap](#roadmap)
18. [Security, Safety, and Code Signing](#security-safety-and-code-signing)
19. [Glossary (Plain-English)](#glossary-plain-english)
20. [License](#license)

---

## System Overview

**Capture → Encode → Transport → Decode → Display**

- **Drone (Pi 3, v0):** USB webcam → **H.264 hardware encode** (start 720p30) → **GStreamer `webrtcbin`** → **WebRTC** (encrypted RTP). A **data channel** carries telemetry & commands (JSON).
- **Network:** **2.4 GHz Wi-Fi** as default route; **LTE (EC25-A)** as backup (`wwan0`). Routing decided by metrics/priority.
- **Ground (Windows):** Phase 1: **browser receiver**; Phase 2: **PyQt GUI** (.exe) with buttons & stats; Phase 3: PC-VR surface (Quest 3 tethered).
- **Config-driven:** `config/*.json` holds ICE/TURN, video, network, and app settings.
- **Auto-update:** GUI checks `updates/version.json` on GitHub Pages, verifies SHA-256, swaps binaries.

---

## User-Visible Features & Controls (v0)

- Connect / Disconnect
- Power-up & Initialize
- PID configuration (forwarded to FC; bus TBD UART or SPI)
- Manual waypoints (minimal map panel)
- Launch
- Video stream: Enable / Disable
- **RF-mute** (disable all radio TX)
- **Return-to-Home** (RTH)
- **Emergency landing**
- **Bitrate rungs:** Low / Medium / High
- **Black-and-white** toggle (bitrate saver, if encoder allows)
- **Manual control mode** (controller while wearing VR)
- **VR section on/off** (PC-VR; tethered)

---

## End-to-End Flow (Story of a Frame)

1. **Sensor → frame:** Webcam exposes frames to V4L2 on the Pi.
2. **Encode:** GStreamer reads `/dev/video0`, converts raw to **H.264** using Pi’s encoder, with GOP/keyframe settings tuned for fast recovery.
3. **WebRTC packeting:** `webrtcbin` encapsulates frames into **RTP**; DTLS/SRTP encrypts; ICE gathers candidate paths (Wi-Fi or LTE).
4. **Signaling:** Pi and PC swap SDP/ICE over a lightweight WebSocket server (on Pi for LAN proof).
5. **Network:** Packets ride Wi-Fi; if Wi-Fi dies, session re-ICEs and rides LTE (with TURN later if NATs misbehave).
6. **Receive & render:** Ground station sets remote description, decrypts SRTP, decodes H.264, displays video; data channel carries telemetry/commands.

---

## Phased Milestones (Definition of Done)

- **M0 Hardware sanity:** `v4l2-ctl --list-devices` shows webcam; Pi reachable on `wlan0`.
- **M1 Media basics:** GStreamer installed; `gst-inspect-1.0 webrtc` present; local preview runs smoothly.
- **M2 LAN proof:** Signaling + browser receiver show **live video** for 5+ minutes.
- **M3 Resilience:** Toggle `wlan0` → video recovers ≤ **5 s**; GOP keyint ≈ **1–2 s**; **no B-frames**.
- **M4 Net policy:** Wi-Fi default route; LTE fallback verified; Wi-Fi drop → LTE recovery ≈ **1–7 s**.
- **M5 GUI (.exe):** PyQt app with v0 controls & “Check for updates”.
- **M6 TURN (optional):** TURN in `config/ice.json`; relay candidate observed; recovery ≈ **1–3 s** typical.
- **M7 Packaging/Update:** PyInstaller one-folder build; app self-updates from `version.json`.

---

## Repository Layout

```
drone/
  README.md
  LICENSE
  config/
    app.json           # GUI prefs, signaling host/port, feature flags
    ice.json           # STUN now; TURN later
    video.json         # width/height/fps/bitrate/keyint, b/w toggle
    netprofile.json    # prefer wlan0/wlan1; LTE APN; route metrics
  scripts/
    00-sysinfo.sh
    10-gst-probe.sh
    20-preview-720p.sh
    net-apply.sh
    start-signal.sh
    start-receiver.sh
    start-sender-cli.sh   # placeholder; real sender next
    selfupdate.sh         # optional script updater
  signaling/
    signaling_server.py
    receiver.html
  sender/
    cli-examples.md
    c-daemon/             # (later) C/C++ backend with JSON IPC
  gui/
    pyqt-app/
      app.py
      updater.py
      version.py
      packaging/
        build.ps1
        DroneGS.spec
  services/
    drone-sender.service
    signaling.service
  updates/
    version.json          # template; host via GitHub Pages
```

---

## Install & Sanity Checks (Raspberry Pi 3)

```bash
sudo apt update
sudo apt install -y v4l-utils python3 python3-pip jq network-manager \
  gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav

v4l2-ctl --list-devices
gst-inspect-1.0 webrtc || echo "webrtcbin missing"
gst-inspect-1.0 v4l2h264enc || echo "HW H.264 missing (can fall back to x264enc)"

# Local preview (no network)
gst-launch-1.0 -v v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! autovideosink
```

---

## Networking Policy: Wi-Fi First, LTE Fallback

**`config/netprofile.json`**
```json
{
  "prefer_wifi_iface": "wlan0",
  "wifi": {
    "builtin": { "iface": "wlan0", "ssid": "HomeMesh", "psk": "REPLACE", "mac": "AA:BB:CC:DD:EE:FF", "metric": 60, "priority": 80 },
    "panda":   { "iface": "wlan1", "ssid": "LongRange2G", "psk": "REPLACE", "mac": "00:C0:CA:AA:BB:CC", "metric": 40, "priority": 100 }
  },
  "lte": { "apn": "fast.t-mobile.com", "metric": 500, "priority": 10 }
}
```

**`scripts/net-apply.sh`**
```bash
#!/usr/bin/env bash
set -euo pipefail
CFG="${1:-config/netprofile.json}"
j(){ jq -r "$1" "$CFG"; }
sudo systemctl enable NetworkManager --now || true

PREF_IF=$(j '.prefer_wifi_iface')
B_IF=$(j '.wifi.builtin.iface'); B_SSID=$(j '.wifi.builtin.ssid'); B_PSK=$(j '.wifi.builtin.psk')
B_MAC=$(j '.wifi.builtin.mac');  B_MET=$(j '.wifi.builtin.metric'); B_PRI=$(j '.wifi.builtin.priority')
P_IF=$(j '.wifi.panda.iface');   P_SSID=$(j '.wifi.panda.ssid');    P_PSK=$(j '.wifi.panda.psk')
P_MAC=$(j '.wifi.panda.mac');    P_MET=$(j '.wifi.panda.metric');   P_PRI=$(j '.wifi.panda.priority')
LTE_APN=$(j '.lte.apn'); LTE_MET=$(j '.lte.metric'); LTE_PRI=$(j '.lte.priority')

nmcli -t -f NAME con show | grep -q '^wifi-builtin$' || nmcli con add type wifi ifname "$B_IF" con-name wifi-builtin ssid "$B_SSID"
nmcli con modify wifi-builtin 802-11-wireless.mac-address "$B_MAC" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "$B_PSK" \
  ipv4.method auto ipv4.route-metric "$B_MET" ipv6.method auto ipv6.route-metric "$B_MET" \
  connection.autoconnect yes connection.autoconnect-priority "$B_PRI"

nmcli -t -f NAME con show | grep -q '^wifi-panda$'   || nmcli con add type wifi ifname "$P_IF" con-name wifi-panda ssid "$P_SSID"
nmcli con modify wifi-panda 802-11-wireless.mac-address "$P_MAC" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "$P_PSK" \
  ipv4.method auto ipv4.route-metric "$P_MET" ipv6.method auto ipv6.route-metric "$P_MET" \
  connection.autoconnect yes connection.autoconnect-priority "$P_PRI"

nmcli -t -f NAME con show | grep -q '^lte-con$'      || nmcli con add type gsm ifname "*" con-name lte-con apn "$LTE_APN"
nmcli con modify lte-con ipv4.method auto ipv4.route-metric "$LTE_MET" ipv6.method ignore \
  connection.autoconnect yes connection.autoconnect-priority "$LTE_PRI"

if [ "$PREF_IF" = "$P_IF" ]; then nmcli con up wifi-panda || true; nmcli con up wifi-builtin || true;
else nmcli con up wifi-builtin || true; nmcli con up wifi-panda || true; fi
nmcli con up lte-con || true
ip route
```

**Apply**
```bash
sudo bash scripts/net-apply.sh
ip route | head -n 3   # default via wlan0 expected
```

---

## Signaling & Receiver (Browser Proof on LAN)

**`signaling/signaling_server.py`**
```python
import asyncio, json, websockets
peers={}
async def handler(ws,_):
    me=None
    try:
        async for raw in ws:
            m=json.loads(raw)
            if m.get("type")=="register":
                me=m["id"]; peers[me]=ws
                await ws.send(json.dumps({"type":"registered","id":me}))
            else:
                to=m.get("to")
                if to in peers: await peers[to].send(json.dumps(m))
    finally:
        if me and peers.get(me) is ws: del peers[me]
async def main():
    print("Signaling on ws://0.0.0.0:8000")
    async with websockets.serve(handler,"0.0.0.0",8000): await asyncio.Future()
if __name__ == "__main__": asyncio.run(main())
```

**`signaling/receiver.html`** (minimal viewer)
```html
<!doctype html><meta charset="utf-8"><title>Receiver</title>
<video id="v" autoplay playsinline controls muted style="width:70vw"></video>
<pre id="log"></pre>
<script>
const id="pc"; let pc;
const ws=new WebSocket("ws://"+location.hostname+":8000");
const log=(...a)=>document.getElementById('log').textContent+=a.join(" ")+"\n";
ws.onopen=()=>{ws.send(JSON.stringify({type:"register",id})); log("registered",id);};
ws.onmessage=async ev=>{
  const m=JSON.parse(ev.data);
  if(m.type==="offer"){
    pc=new RTCPeerConnection({iceServers:[{urls:"stun:stun.l.google.com:19302"}]});
    pc.onicecandidate=e=>e.candidate&&ws.send(JSON.stringify({type:"ice",to:m.from,candidate:e.candidate}));
    pc.ontrack=e=>{document.getElementById('v').srcObject=e.streams[0];};
    await pc.setRemoteDescription(m.sdp);
    const ans=await pc.createAnswer(); await pc.setLocalDescription(ans);
    ws.send(JSON.stringify({type:"answer",to:m.from,sdp:pc.localDescription})); log("answered");
  } else if(m.type==="ice" && pc){ pc.addIceCandidate(m.candidate); }
};
</script>
```

**Run**
```bash
# on Pi
pip3 install --user websockets
python3 signaling/signaling_server.py &
cd signaling && python3 -m http.server 8001
# on Windows: open http://<PI_IP>:8001/receiver.html
```

---

## Sender on Pi: CLI now → C/C++ Daemon later

**ICE config (`config/ice.json`)**
```json
{ "iceServers": [ { "urls": "stun:stun.l.google.com:19302" } ] }
```

**Placeholder (`scripts/start-sender-cli.sh`)** — to be replaced with a working `webrtcbin` sender that exchanges SDP/ICE via WebSocket.
```bash
#!/usr/bin/env bash
set -euo pipefail
SIGNAL_IP="${1:-127.0.0.1}"
echo "TODO: build a GStreamer webrtcbin pipeline,"
echo "      create an SDP offer, signal via ws://$SIGNAL_IP:8000, handle ICE."
```

**Next step (plan):**
- Implement sender in **gst-python** or **C/C++** using `webrtcbin`.
- Expose JSON control commands: `start`, `stop`, `set_bitrate`, `force_keyframe`, `get_stats`.

---

## Resilience, Recovery, and Latency Targets

- Recreate PeerConnection on **failed** or **disconnected > ~2–3 s**.
- **Keyframe interval:** ~**1–2 s**; **no B-frames** for fast catch-up.
- **LAN glass-to-glass:** ~**120–250 ms** (tuned).
- **Wi-Fi → LTE recovery:** typically **~1–7 s**; with **TURN** closer to **1–3 s**.

---

## TURN Relay (Drop-in, Config-Only)

**Later `config/ice.json`:**
```json
{
  "iceServers": [
    { "urls": "stun:stun.l.google.com:19302" },
    { "urls": "turns:turn.yourdomain.com:5349", "username": "USER", "credential": "PASS" }
  ]
}
```

- Run **coturn** on home RPi (public IP + port-forward) or a $5 VPS.  
- Open UDP/TCP **3478**, TCP **5349**, and a small UDP relay port range.

---

## Ground-Station GUI (PyQt on Windows) & Packaging

**Skeleton `gui/pyqt-app/app.py`**
```python
import sys, json, pathlib, hashlib, urllib.request, zipfile, shutil
from PyQt6 import QtWidgets, QtCore

APPDIR = pathlib.Path(__file__).resolve().parent.parent.parent
CFG_APP = APPDIR / "config" / "app.json"
def load_json(p): return json.load(open(p, "r", encoding="utf-8"))

class Main(QtWidgets.QMainWindow):
    def __init__(self):
        super().__init__(); self.setWindowTitle("Drone Ground Station v0")
        w=QtWidgets.QWidget(); self.setCentralWidget(w); v=QtWidgets.QVBoxLayout(w)
        self.lbl=QtWidgets.QLabel("Disconnected")
        self.btnC=QtWidgets.QPushButton("Connect")
        self.btnD=QtWidgets.QPushButton("Disconnect")
        self.btnU=QtWidgets.QPushButton("Check for updates")
        hl=QtWidgets.QHBoxLayout(); [hl.addWidget(b) for b in (self.btnC,self.btnD,self.btnU)]
        v.addWidget(self.lbl); v.addLayout(hl)
        self.btnC.clicked.connect(self.on_connect); self.btnD.clicked.connect(self.on_disconnect)
        self.btnU.clicked.connect(self.on_update)
        self.cfg=load_json(CFG_APP)
    def on_connect(self): self.lbl.setText("Connecting...")   # TODO: start receiver/sender via QProcess
    def on_disconnect(self): self.lbl.setText("Disconnected") # TODO: stop processes
    def on_update(self): Updater.check_and_update(self, self.cfg.get("update_manifest"))

class Updater:
    @staticmethod
    def sha256(path):
        h=hashlib.sha256()
        with open(path,"rb") as f:
            for c in iter(lambda: f.read(1<<20), b""): h.update(c)
        return h.hexdigest()
    @staticmethod
    def dl(url, out):
        with urllib.request.urlopen(url) as r, open(out,"wb") as f: shutil.copyfileobj(r,f)
    @staticmethod
    def check_and_update(parent, manifest_url):
        if not manifest_url:
            QtWidgets.QMessageBox.information(parent,"Update","No update URL."); return
        try:
            with urllib.request.urlopen(manifest_url) as r: m=json.loads(r.read().decode("utf-8"))
        except Exception as e:
            QtWidgets.QMessageBox.warning(parent,"Update",f"Manifest fetch failed: {e}"); return
        local_ver="0.0.0"; remote_ver=m.get("version","0.0.0")
        if remote_ver<=local_ver:
            QtWidgets.QMessageBox.information(parent,"Update","Up to date."); return
        url=m["url"]; expected=m.get("sha256")
        zpath=pathlib.Path(QtCore.QStandardPaths.writableLocation(
            QtCore.QStandardPaths.StandardLocation.TempLocation))/ "update.zip"
        try:
            Updater.dl(url, zpath)
            if expected and Updater.sha256(zpath).lower()!=expected.lower():
                QtWidgets.QMessageBox.critical(parent,"Update","Checksum mismatch."); return
            install_root=APPDIR.parent.parent  # dist/DroneGS/
            with zipfile.ZipFile(zpath,"r") as z: z.extractall(install_root.parent)
            QtWidgets.QMessageBox.information(parent,"Update","Update installed. Restart app.")
        except Exception as e:
            QtWidgets.QMessageBox.critical(parent,"Update",f"Update failed: {e}")

if __name__=="__main__":
    app=QtWidgets.QApplication(sys.argv); m=Main(); m.resize(640,300); m.show(); sys.exit(app.exec())
```

**Packaging (PowerShell)**
```powershell
py -3.11 -m venv .venv
. .\.venv\Scripts\Activate.ps1
pip install --upgrade pip pyinstaller PyQt6
pyinstaller --noconfirm --windowed --name DroneGS `
  --add-data "..\..\config;config" app.py
```

> Bundle GStreamer later if you embed decode in the GUI, or require users to install the GStreamer runtime.

---

## Auto-Update via GitHub Pages

**`updates/version.json`**
```json
{
  "name": "DroneGS",
  "version": "0.1.0",
  "notes": "Initial v0 GUI; signaling scaffold; scripts",
  "url": "https://<your-username>.github.io/drone/DroneGS-0.1.0.zip",
  "sha256": "REPLACE_WITH_SHA256"
}
```

**`config/app.json`**
```json
{
  "signaling_url": "ws://192.168.1.50:8000",
  "receiver_url": "http://192.168.1.50:8001/receiver.html",
  "update_manifest": "https://<your-username>.github.io/drone/updates/version.json",
  "vr_enabled": true
}
```

**Enable Pages:** Repo → Settings → Pages → Source: `main` → publish path that serves `/updates/version.json`.

---

## GitHub Basics (for this repo)

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

git clone https://github.com/Catra2/drone
cd drone

git add -A
git commit -m "v0 skeleton: configs, scripts, signaling scaffold, GUI shell"
git push origin main

# daily work
git add -A
git commit -m "M2: signaling + receiver proof"
git push

# release for users
git tag v0.1.0
git push origin v0.1.0
# Build one-folder app → ZIP → upload to Releases/Pages, update version.json.
```

---

## Diagnostics & Measurement

- **Ping RTT:** Win `ping -n 50 <PI_IP>`; Pi `ping -c 50 <PC_IP>`.
- **WebRTC RTT:** `RTCPeerConnection.getStats()` → `currentRoundTripTime`.
- **Glass-to-glass:** screen record + blinking LED at 240 fps; count frames.
- **GStreamer logs:** add `-v`, and `GST_DEBUG=2,webrtc*:3` when needed.
- **Targets (LAN, tuned):** data RTT ≈ ping + **1–5 ms**; one-way media **~80–180 ms**; glass-to-glass **~120–250 ms**.

---

## Hardware Notes & Upgrade Path

- **v0 Camera:** USB webcam (Pi encodes H.264; 720p30 first, 1080p30 as HW allows).
- **Future cameras:** Downward-facing to see below; optional forward-facing. Consider 360° units that output a single H.264/H.265 stream (pass-through).
- **Antennas (2.4 GHz):** Drone = low-profile **puck** (2–4 dBi/elem) or small downward **patch** (6–9 dBi). Ground = **dual-pol directional panel** (12–15 dBi).
- **LTE:** Quectel **EC25-A** (no data cap noted). Prefer Wi-Fi; LTE backup.
- **Flight controller:** Low-level control on MCU (Teensy/Arduino). Pi handles transport, higher-level nav, and GUI commands.

---

## Roadmap

- **v0:** USB webcam → WebRTC → browser → PyQt GUI; Wi-Fi first/LTE fallback; safety; auto-update.
- **v1:** GUI-embedded video; PC-VR surface; waypoint list on minimal map; PID panel (choose UART vs SPI).
- **v2:** C/C++ sender daemon (JSON API), TURN config, richer telemetry & controller input.
- **v3:** Downward + forward cameras; **1080p30+**; optional YOLO on ground; LTE hardening.

---

## Security, Safety, and Code Signing

- **Safety UI always visible:** RTH, Emergency landing, RF-mute.
- **Regulatory:** HAM privileges ≠ Wi-Fi/LTE exemptions; follow local EIRP & flight rules.
- **Windows code signing:** obtain code-signing cert; sign to reduce SmartScreen prompts:
```powershell
signtool sign /fd SHA256 /a /tr http://timestamp.digicert.com /td SHA256 path\to\DroneGS.exe
```
- **Update integrity:** serve over HTTPS; verify **SHA-256** before swapping binaries.

---

## Glossary (Plain-English)

- **WebRTC:** real-time media/data between devices; encrypted; NAT-friendly.
- **RTP:** packet format for audio/video.
- **ICE/STUN/TURN:** ways to find a path (STUN = learn public address; TURN = relay).
- **GStreamer:** Lego-style media pipelines (source → encoder → network).
- **OpenXR:** PC-VR API (Quest 3 via Link/Air Link).
- **Keyframe/IDR:** full frame that enables quick recovery after packet loss.
