[中文文档](README_zh.md) | English

---

# SteamDeck iOS AirPlay Setup

A SteamOS iOS/iPadOS screen mirroring deployment solution based on Distrobox and UxPlay. 
This workflow is specifically optimized for SteamOS Gaming Mode (and also works in Desktop Mode), resolving default environment issues such as hardware decoding failures, adaptive scaling anomalies, and network connection freezing.

## Features

* **Containerized Deployment**: Relies on Distrobox and a minimal Ubuntu image. Environment data is stored in the `/home` directory, which is unaffected by major SteamOS updates.
* **AMD Hardware Decoding**: Passthrough SteamOS underlying drivers (`-avdec`) and disable timestamp buffering (`-vsync no`) to significantly reduce input latency.
* **Adaptive Fullscreen**: Fully compatible with the Gamescope window manager. Forces the screen to stretch and fill the display (`-fs`), solving border cropping and scaling issues during full-screen video playback.
* **Connection Stability**: Automatically clears Avahi broadcast cache and dead ports before startup, adjusts the default heartbeat disconnection threshold (`-reset 15`), and fixes freezing errors caused by repeated reconnections.
* **Anti-Sleep Daemon**: Suspends a daemon in the background during screen mirroring, continuously sending `dbus` activity signals to prevent the system from automatically sleeping during long video playback.

---
<img width="3024" height="4032" alt="IMG_1897" src="https://github.com/user-attachments/assets/17e74014-dadc-4751-908c-8ec539530231" />

<img width="1280" height="800" alt="20260817133348_1" src="https://github.com/user-attachments/assets/83c2ff27-9f68-47ad-bad4-0be9d6a3bc84" />
<img width="1280" height="800" alt="20260817133405_1" src="https://github.com/user-attachments/assets/798ce1dd-c0f3-4ac2-baf8-6f5c2630f04c" />
<img width="1280" height="800" alt="20260817133838_1" src="https://github.com/user-attachments/assets/87317102-5a47-4cfe-a07b-038117132333" />

## Installation

### 1. Create Ubuntu Container Environment
Enter **Desktop Mode** on your Steam Deck, open Konsole (Terminal), pull the base image, and create a container named `uxplay-env`:

```bash
distrobox create --image ubuntu:latest --name uxplay-env
```
> Note: If a TLS handshake timeout occurs (e.g., in mainland China network environments), you can replace the source with a domestic mirror accelerator, for example:
```bash
distrobox create --image docker.1panel.live/library/ubuntu:latest --name uxplay-env
```

### 2. Install Dependencies and Decoding Plugins
Enter the newly created container:

```bash
distrobox enter uxplay-env
```

Run the following commands inside the container terminal to install the local network broadcast service, GStreamer video rendering components, and AMD hardware decoding drivers:

```bash
sudo apt update
sudo apt install -y dbus avahi-daemon uxplay gstreamer1.0-vaapi mesa-va-drivers libva-drm2 gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-base gstreamer1.0-gl
```
Once the installation is complete, execute `exit` to leave the container.

### 3. Generate One-Click Startup Script
In the Konsole terminal of Desktop Mode, copy and run the entire code block below. This will generate an executable script named `Start_AirPlay.sh` on your desktop:

```bash
cat << 'EOF' > ~/Desktop/Start_AirPlay.sh
#!/bin/bash
unset LD_PRELOAD

# [NEW] Host-level double insurance: Force kill any residual AirPlay zombie processes on SteamOS before entering the container
pkill -9 -f uxplay 2>/dev/null

# Suspend anti-sleep heartbeat mechanism
(
    while true; do
        dbus-send --session --dest=org.freedesktop.ScreenSaver --type=method_call /org/freedesktop/ScreenSaver org.freedesktop.ScreenSaver.SimulateUserActivity 2>/dev/null
        sleep 180
    done
) &
AWAKE_PID=$!

distrobox enter uxplay-env -- bash -l -c "
echo 'Cleaning network cache and dead ports...'
# [FIX] Use pkill instead of killall to completely resolve port occupation (Socket 98) errors
sudo pkill -9 avahi-daemon 2>/dev/null
sudo pkill -9 uxplay 2>/dev/null
sudo rm -f /var/run/dbus/pid /run/avahi-daemon/pid

echo 'Starting D-Bus...'
sudo service dbus start
sleep 1

echo 'Starting Avahi daemon...'
sudo avahi-daemon --daemonize
sleep 1

echo 'Starting UxPlay...'
uxplay -p -fps 60 -vsync no -reset 15 -fs
"

# Clean up processes and restore native power policy
kill -9 $AWAKE_PID 2>/dev/null
sleep 3
EOF
chmod +x ~/Desktop/Start_AirPlay.sh
```

### 4. Import to Gaming Mode
To bypass Gamescope's strict kill mechanism for windowless processes, the script must be launched via the terminal.

1. Open the **Steam** client in Desktop Mode, click **Add a Game** in the bottom left corner -> **Add a Non-Steam Game**, check any random program, and add it.
2. Find the newly added program in your Steam Library, right-click, and select **Properties**.
3. **Name**: Change to `AirPlay` (or any custom name).
4. **Target**: Enter `"/usr/bin/konsole"` (Make sure to include the double quotes).
5. **Launch Options**: Enter `--fullscreen -e /home/deck/Desktop/Start_AirPlay.sh`.
<img width="1280" height="800" alt="20260817133326_1" src="https://github.com/user-attachments/assets/c9a303c4-a06b-459e-90f1-0e801a826852" />

---

## Usage & FAQ

* **Starting AirPlay**: Switch to Gaming Mode and launch the `AirPlay` shortcut. Wait for the black terminal window to appear, then open the Control Center on your iOS/iPadOS device, click "Screen Mirroring," and select your Steam Deck. When finished, press the `STEAM Button` and select "Exit Game" to release all background processes. In Desktop Mode, simply double-click the `Start_AirPlay.sh` script on your desktop to start mirroring (there might be no visual feedback on launch, but mirroring will work normally).
* **Screen Blocked by Terminal Window**: In Gaming Mode, if you hear audio but see no video, press the `STEAM Button` to bring up the left menu. Click on the running AirPlay program, and manually switch to the `UxPlay` video window from the multi-window list.
* **Background Pushing Limitations**: This solution is based on reverse engineering of the AirPlay Mirroring protocol. Lacking official DRM certificate support, it does not support DLNA video stream background pushing via the TV icon in apps (e.g., YouTube, Bilibili). Your iOS screen must remain on during casting.
* **Dim Screen Issue**: The anti-sleep script only prevents the device from going into deep sleep. If the screen dims while playing plugged in, press the `STEAM Button` in Gaming Mode -> Settings -> Display -> set "Dim screen while plugged in" to "Never".

---
*Based on the open-source implementation by the [UxPlay](https://github.com/FDH2/UxPlay) community.*
