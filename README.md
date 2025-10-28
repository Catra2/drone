```markdown
# Drone Video Transport — README

Repo: https://github.com/Catra2/drone

Target (v0): USB webcam on Raspberry Pi 3 → WebRTC → Windows ground-station (PyQt GUI). Prefer 2.4 GHz Wi-Fi; fall back to LTE (Quectel EC25-A). Include safety controls (RF-mute, Return-to-Home, Emergency land). Auto-update on launch (GUI + scripts).  
Upgrade path: Downward-facing + forward cameras; 1080p30+; PC-VR (Quest 3 via Link/Air Link); optional on-ground YOLO; TURN relay on home RPi (Hawaiʻi).

---

## 0) System at a Glance

Capture → Encode → Transport → Decode → Display

- Drone (Pi 3, v0): USB webcam → H.264 HW encode (start 720p30) → GStreamer webrtcbin → WebRTC (encrypted RTP). Data channel (JSON) for telemetry/commands.
- Network: 2.4 GHz Wi-Fi (default route) → LTE/EC25-A fallback (route metrics).
- Ground (Windows): Start with browser receiver → then PyQt GUI (.exe) with buttons (Connect, bitrate rungs, safety, VR on/off, etc.).
- Config-driven: config/*.json (ICE/TURN, video, network, app settings).
- Auto-update: GUI checks hosted updates/version.json; downloads ZIP; swaps files.

---

## 1) v0 Scope & Controls

- Cameras: USB webcam only for v0. Later: camera that sees below the drone; optional forward camera.
- GUI minimum controls: Connect/Disconnect; Power-up & Init; PID config (forward to flight controller); Manual waypoints (simple map); Launch; Video on/off; RF-mute (disable all radios); Return-to-Home; Emergency landing; Bitrate Low/Med/High (+ Black/White toggle to reduce bitrate if needed); Manual control mode (controller while in VR); VR on/off (PC-VR).
- OS: Windows ground station only (for v0).
- TURN: Add later via config (no code changes).
- Safety: Present in v0.
- AI: YOLO on ground; SLAM (if any) on ground.

---

## 2) Milestones (Definition of Done)

- M0 — Hardware sanity: Webcam listed by `v4l2-ctl --list-devices`; Pi reachable on wlan0.
- M1 — Media basics: GStreamer installed; `gst-inspect-1.0 webrtc` present; local preview runs cleanly.
- M2 — LAN proof: Signaling server + browser receiver → live video 5+ min.
- M3 — Resilience: Toggle wlan0 → stream resumes ≤ 5 s; keyframe interval ≈ 1–2 s; no B-frames.
- M4 — Net policy: Wi-Fi default route; LTE fallback; Wi-Fi drop → recovery over LTE ≈ 1–7 s.
- M5 — GUI (.exe): PyQt app with v0 controls; “Check for updates”.
- M6 — TURN (optional): TURN online; added via config/ice.json; relay candidate observed; faster recovery (~1–3 s).
- M7 — Packaging/Update: PyInstaller one-folder build; app self-updates from version.json.

---

## 3) Repository Layout

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
        start-sender-cli.sh   # placeholder; real sender added next
        selfupdate.sh         # (optional) script updater
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

---

## 4) Install (Pi) & Quick Tests

    sudo apt update
    sudo apt install -y v4l-utils python3 python3-pip jq network-manager \
      gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
      gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav

    v4l2-ctl --list-devices
    gst-inspect-1.0 webrtc || echo "webrtcbin plugin not found"
    gst-inspect-1.0 v4l2h264enc || echo "HW H.264 missing (use x264enc fallback)"

    # Local preview (no network)
    gst-launch-1.0 v4l2src device=/dev/video0 ! \
      video/x-raw,width=1280,height=720,framerate=30/1 ! autovideosink

---

## 5) Networking Policy (Wi-Fi first, LTE fallback)

config/netprofile.json

    {
      "prefer_wifi_iface": "wlan0",
      "wifi": {
        "builtin": { "iface": "wlan0", "ssid": "HomeMesh", "psk": "REPLACE", "mac": "AA:BB:CC:DD:EE:FF", "metric": 60, "priority": 80 },
        "panda":   { "iface": "wlan1", "ssid": "LongRange2G", "psk": "REPLACE", "mac": "00:C0:CA:AA:BB:CC", "metric": 40, "priority": 100 }
      },
      "lte": { "apn": "fast.t-mobile.com", "metric": 500, "priority": 10 }
    }

scripts/net-apply.sh

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

Run:

    sudo bash scripts/net-apply.sh
    ip route | head -n 3   # default via wlan0 expected

---

## 6) Signaling & Receiver (LAN proof)

signaling/signaling_server.py

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

signaling/receiver.html (simple viewer)

    <!doctype html><meta charset="utf-8"><title>Receiver</title>
    <video id="v" autoplay playsinline controls muted style="width:70vw"></video>
    <pre id="log"></pre>
    <script>
    const id="pc", to="pi"; let pc;
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

Run on Pi:

    pip3 install --user websockets
    python3 signaling/signaling_server.py
    cd signaling && python3 -m http.server 8001

Open on Windows: http://<PI_IP>:8001/receiver.html (waits for offer).

---

## 7) Pi Sender (CLI first → daemon later)

config/ice.json

    { "iceServers": [ { "urls": "stun:stun.l.google.com:19302" } ] }

scripts/start-sender-cli.sh (placeholder; replace with real webrtcbin sender that exchanges SDP/ICE via WebSocket)

    #!/usr/bin/env bash
    set -euo pipefail
    SIGNAL_IP="${1:-127.0.0.1}"
    echo "TODO: Implement sender that builds a GStreamer webrtcbin pipeline,"
    echo "creates an SDP offer, signals via ws://$SIGNAL_IP:8000, and handles ICE."

Path forward: keep CLI for speed; then promote to a C/C++ daemon exposing JSON commands: start, stop, set_bitrate, keyframe, get_stats.

---

## 8) Resilience & Latency Targets

- Rebuild PeerConnection on failed or > 2–3 s disconnected.
- Keyframe interval ≈ 1–2 s; no B-frames.
- LAN glass-to-glass ≈ 120–250 ms (tuned).
- Wi-Fi → LTE switch typical recovery ≈ 1–7 s (with TURN closer to 1–3 s).

---

## 9) TURN (drop-in later; config-only)

config/ice.json (later)

    {
      "iceServers": [
        { "urls": "stun:stun.l.google.com:19302" },
        { "urls": "turns:turn.yourdomain.com:5349", "username": "USER", "credential": "PASS" }
      ]
    }

Run coturn on home RPi (public IP + port-forward) or a $5 VPS. Open UDP/TCP 3478, TCP 5349, and a small UDP relay range.

---

## 10) GUI (Windows, PyQt) & Packaging

gui/pyqt-app/app.py (skeleton)

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
        def on_connect(self): self.lbl.setText("Connecting...")     # TODO: QProcess to start receiver/sender
        def on_disconnect(self): self.lbl.setText("Disconnected")   # TODO: stop processes
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
            if not manifest_url: QtWidgets.QMessageBox.information(parent,"Update","No update URL."); return
            try:
                with urllib.request.urlopen(manifest_url) as r: m=json.loads(r.read().decode("utf-8"))
            except Exception as e:
                QtWidgets.QMessageBox.warning(parent,"Update",f"Manifest fetch failed: {e}"); return
            local_ver="0.0.0"; remote_ver=m.get("version","0.0.0")
            if remote_ver<=local_ver: QtWidgets.QMessageBox.information(parent,"Update","Up to date."); return
            url=m["url"]; expected=m.get("sha256")
            zpath=pathlib.Path(QtCore.QStandardPaths.writableLocation(QtCore.QStandardPaths.StandardLocation.TempLocation))/ "update.zip"
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

Packaging (Windows, one-folder)

    py -3.11 -m venv .venv
    . .\.venv\Scripts\Activate.ps1
    pip install --upgrade pip pyinstaller PyQt6
    pyinstaller --noconfirm --windowed --name DroneGS ^
      --add-data "..\..\config;config" app.py

Note: Bundle GStreamer later if you embed decode in the GUI, or require users to install the GStreamer runtime once.

---

## 11) Auto-Update (GitHub Pages)

updates/version.json

    {
      "name": "DroneGS",
      "version": "0.1.0",
      "notes": "Initial v0 GUI; signaling scaffold; scripts",
      "url": "https://<your-username>.github.io/drone/DroneGS-0.1.0.zip",
      "sha256": "REPLACE_WITH_SHA256"
    }

config/app.json

    {
      "signaling_url": "ws://192.168.1.50:8000",
      "receiver_url": "http://192.168.1.50:8001/receiver.html",
      "update_manifest": "https://<your-username>.github.io/drone/updates/version.json",
      "vr_enabled": true
    }

Enable GitHub Pages: Repo → Settings → Pages → Source: main → serve /updates/version.json.

---

## 12) GitHub Basics

    git config --global user.name "Your Name"
    git config --global user.email "you@example.com"

    git clone https://github.com/Catra2/drone
    cd drone

    git add -A
    git commit -m "v0 skeleton: configs, scripts, signaling scaffold, GUI shell"
    git push origin main

    git add -A
    git commit -m "M2: signaling + receiver proof"
    git push

    git tag v0.1.0
    git push origin v0.1.0
    # Build one-folder app → ZIP → upload to GitHub Releases/Pages.
    # Update updates/version.json (version, URL, SHA-256).

---

## 13) Measurement & Diagnostics

- Ping RTT: Windows `ping -n 50 <PI_IP>` / Pi `ping -c 50 <PC_IP>`.
- WebRTC RTT: receiver RTCPeerConnection.getStats() → currentRoundTripTime.
- Glass-to-glass: record screen + blinking LED at 240 fps; count frames.
- GStreamer logs: `-v`; add `GST_DEBUG=2,webrtc*:3` when needed.
- Targets (LAN, tuned): data RTT ≈ ping + 1–5 ms; one-way media ≈ 80–180 ms; glass-to-glass ≈ 120–250 ms.

---

## 14) Hardware Notes

- v0 Camera: USB webcam (Pi encodes H.264; 720p30 first, 1080p30 as HW allows).
- Antennas (2.4 GHz): Drone = low-profile puck (2–4 dBi/elem) or small downward patch (6–9 dBi). Ground = dual-pol directional panel (12–15 dBi).
- LTE: Quectel EC25-A (T-Mobile). Prefer Wi-Fi; LTE as fallback.

---

## 15) Roadmap

- v0: USB webcam → WebRTC → browser → PyQt GUI; Wi-Fi first/LTE fallback; safety; auto-update.
- v1: GUI embeds video; PC-VR surface; waypoint list on minimal map; PID panel to FC (decide UART vs SPI).
- v2: C/C++ sender daemon (JSON commands), TURN config, richer telemetry & controller input.
- v3: Downward + forward cameras; 1080p30+; optional YOLO on ground; LTE hardening.

---

## 16) Security, Safety & Signing

- Safety: Keep RTH, Emergency landing, RF-mute always visible.
- Regulatory: Follow Wi-Fi/LTE flight & RF rules (HAM power ≠ Wi-Fi/LTE exemptions).
- Code signing (Windows): obtain a code-signing cert; sign builds to reduce SmartScreen prompts:

      signtool sign /fd SHA256 /a /tr http://timestamp.digicert.com /td SHA256 path\to\DroneGS.exe

- Updates: Serve version.json & ZIPs over HTTPS; verify SHA-256 before swap.

---

## 17) Glossary (one-liners)

WebRTC: real-time audio/video/data (encrypted). RTP: media packet format. ICE/STUN/TURN: path finding/relay. GStreamer: media pipelines. OpenXR: PC-VR API (Quest via Link/Air Link). MIMO: multi-antenna Wi-Fi. Keyframe/IDR: full frame for quick recovery.


```
