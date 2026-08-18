[中文文档](README_zh.md) | English

---

# SteamDeck iOS AirPlay Setup

A containerized deployment solution for iOS/iPadOS/Mac screen mirroring on SteamOS, based on Distrobox and UxPlay.
This workflow is deeply optimized for both SteamOS Gaming Mode and Desktop Mode, resolving common issues found in default environments, such as hardware decoding failures, incorrect adaptive scaling, network connection freezing, and accidental sleep interruptions.

## 🌟 Core Features & Optimizations

* **Smart Environment Detection & UI**: Automatically detects the current running mode. In Gaming Mode, it launches silently in fullscreen; in Desktop Mode, it opens a dedicated terminal with a **D-Pad interactive menu**, allowing you to choose between "Standalone Window" or "Immersive Fullscreen".
* **Global Low-Latency Lock (30 FPS)**: Tuned specifically for the Apple AirPlay Mirroring protocol, it forces a 30 FPS lock. This drastically reduces network congestion during video streaming, completely resolving the "ghost latency" and audio desync issues often seen on older devices (e.g., iPad 6) or 2.4GHz Wi-Fi networks.
* **Lossless Containerized Deployment**: Powered by Distrobox and a stripped-down Ubuntu image. All underlying dependencies are safely stored in the `/home` directory, making the setup immune to major SteamOS system updates. Deploy once, maintain never.
* **AMD Hardware Decoding**: Passes through SteamOS underlying drivers (`-avdec`) and disables timestamp buffering (`-vsync no`), significantly reducing input and decoding latency.
* **Aggressive Anti-Sleep Daemon**: Suspends a background daemon during casting that sends a `dbus` active heartbeat every 45 seconds, completely bypassing KDE's and Gaming Mode's aggressive power-saving policies and preventing the screen from dimming/sleeping during long videos.

---
<img width="3024" height="4032" alt="IMG_1897" src="https://github.com/user-attachments/assets/17e74014-dadc-4751-908c-8ec539530231" />
<img width="1280" height="800" alt="20260817133348_1" src="https://github.com/user-attachments/assets/83c2ff27-9f68-47ad-bad4-0be9d6a3bc84" />
<img width="1280" height="800" alt="20260817133405_1" src="https://github.com/user-attachments/assets/798ce1dd-c0f3-4ac2-baf8-6f5c2630f04c" />
<img width="1280" height="800" alt="20260817133838_1" src="https://github.com/user-attachments/assets/87317102-5a47-4cfe-a07b-038117132333" />

## 🛠️ Deployment Process

### 1. Create Ubuntu Container Environment
Enter Steam Deck's Desktop Mode, open Konsole (Terminal), pull the base image, and create a container named `uxplay-env`:

```bash
distrobox create --image ubuntu:latest --name uxplay-env
```

### 2. Install Dependencies and Hardware Decoding Plugins
Enter the newly created container:

```bash
distrobox enter uxplay-env
```

Execute the following command inside the container terminal to install local network broadcast services, GStreamer video rendering components, and AMD hardware decoding drivers:

```bash
sudo apt update
sudo apt install -y dbus avahi-daemon uxplay gstreamer1.0-vaapi mesa-va-drivers libva-drm2 gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-base gstreamer1.0-gl
```
Once the installation is complete, type `exit` to leave the container.

### 3. Generate the Adaptive Launch Script
While still in the Desktop Mode Konsole terminal, **copy and run the entire block of code below**. This will generate the optimized `Start_AirPlay.sh` executable script on your desktop:

```bash
cat << 'EOF' > ~/Desktop/Start_AirPlay.sh
#!/bin/bash
unset LD_PRELOAD

TARGET_FPS=30
FPS_NAME="30 FPS (Low Latency / Stable Edition)"

# 1. Smart Environment Detection
if [ -n "$SteamTenfoot" ]; then
    IS_GAMEMODE=1
else
    IS_GAMEMODE=0
fi

# Launch Terminal Window in Desktop Mode
if [ $IS_GAMEMODE -eq 0 ] && { [ "$XDG_SESSION_TYPE" = "x11" ] || [ "$XDG_SESSION_TYPE" = "wayland" ]; }; then
    if [ -z "$_RUNNING_IN_KONSOLE" ]; then
        export _RUNNING_IN_KONSOLE=1
        exec konsole --nofork -e "$0" "$@"
    fi
fi

# Clean up host-level ghost processes
pkill -9 -f uxplay 2>/dev/null

# -------------------------------------------------------------
# 🎮 D-Pad Interactive Menu Function (Desktop Mode Only)
# -------------------------------------------------------------
interactive_menu() {
    local sel=1
    local timeout_sec=3
    local start_time=$(date +%s)

    while true; do
        local now=$(date +%s)
        local elapsed=$((now - start_time))
        local remain=$((timeout_sec - elapsed))

        if [ $remain -le 0 ]; then
            break
        fi

        clear
        echo -e "\033[1;36m=========================================================\033[0m"
        echo -e "\033[1;37m      📺 Desktop Display Mode Selection (Locked at 30 FPS) \033[0m"
        echo -e "\033[1;36m=========================================================\033[0m"
        echo -e "\033[1;33m 🎮 [Ctrl] D-Pad / Left Stick ↑↓ to Switch  |  🟢 A Button / Enter to Confirm\033[0m"
        echo -e "\033[1;90m ⏱️  Auto-selecting [Standalone Window] in ${remain} seconds...\033[0m"
        echo -e "\033[1;90m ---------------------------------------------------------\033[0m"

        if [ "$sel" -eq 1 ]; then
            echo -e " \033[1;32m➔ [ ▶ 1. Standalone Window (Default) - Resizable, draggable, with X close button ]\033[0m"
            echo -e " \033[0;37m     2. Immersive Fullscreen - Covers entire screen, hides taskbar\033[0m"
        else
            echo -e " \033[0;37m     1. Standalone Window (Default) - Resizable, draggable, with X close button\033[0m"
            echo -e " \033[1;32m➔ [ ▶ 2. Immersive Fullscreen - Covers entire screen, hides taskbar ]\033[0m"
        fi
        echo -e "\033[1;90m ---------------------------------------------------------\033[0m"

        IFS= read -rsn1 -t 1 key
        if [ $? -eq 0 ]; then
            start_time=$(date +%s)
            timeout_sec=10 

            if [ "$key" = $'\x1b' ]; then
                read -rsn2 -t 0.1 rest
                key+="$rest"
            fi

            case "$key" in
                $'\x1b[A'|"[A"|"w"|"W") sel=1 ;;
                $'\x1b[B'|"[B"|"s"|"S") sel=2 ;;
                ""|$'\n'|" "|"a"|"A") break ;;
                "1") sel=1; break ;;
                "2") sel=2; break ;;
            esac
        fi
    done
    MENU_CHOICE=$sel
}

# 2. Core Logic Routing
if [ $IS_GAMEMODE -eq 1 ]; then
    # Gaming Mode: Skip menu, force fullscreen
    UXPLAY_FS="-fs"
    WIN_NAME="Gaming Mode Fullscreen"
    EXIT_TIP="* TIP: Press the STEAM button and select 'Exit Game' to stop casting."
else
    # Desktop Mode: Show interactive menu
    interactive_menu
    if [ "$MENU_CHOICE" -eq 2 ]; then
        UXPLAY_FS="-fs"
        WIN_NAME="Desktop Fullscreen"
        EXIT_TIP="* ⚠️ TIP: If shortcuts fail in fullscreen, disconnect casting from your phone to exit."
    else
        UXPLAY_FS=""
        WIN_NAME="Standalone Window"
        EXIT_TIP="* TIP: You can click the X button in the top right corner to close the window anytime."
    fi
fi

# 3. Anti-Sleep Heartbeat (45s cycle against sleep)
(
    while true; do
        dbus-send --session --dest=org.freedesktop.ScreenSaver --type=method_call /org/freedesktop/ScreenSaver org.freedesktop.ScreenSaver.SimulateUserActivity 2>/dev/null
        sleep 45
    done
) &
AWAKE_PID=$!

cleanup() {
    kill -9 $AWAKE_PID 2>/dev/null
    pkill -9 -f uxplay 2>/dev/null
}
trap cleanup EXIT INT TERM

# 4. Enter container and execute main program
distrobox enter uxplay-env -- bash -l -c "
clear
echo -e '\033[1;36m=========================================================\033[0m'
echo -e '\033[1;37m        📺 Steam Deck AirPlay Dedicated Terminal \033[0m'
echo -e '\033[1;36m=========================================================\033[0m'
echo -e '\033[1;32m [Status] 🟢 Service Ready ($FPS_NAME | $WIN_NAME)\033[0m'
echo -e '\033[1;33m [Action] Please tap [Screen Mirroring] in your Apple device Control Center to connect.\033[0m'
echo -e '\033[1;90m ---------------------------------------------------------\033[0m'
echo -e '\033[1;90m ${EXIT_TIP}\033[0m'
echo -e '\033[1;90m ---------------------------------------------------------\033[0m'

sudo pkill -9 avahi-daemon 2>/dev/null
sudo pkill -9 uxplay 2>/dev/null
sudo rm -f /var/run/dbus/pid /run/avahi-daemon/pid

sudo service dbus start >/dev/null 2>&1
sleep 1 
sudo avahi-daemon --daemonize >/dev/null 2>&1
sleep 1 

uxplay -p -fps $TARGET_FPS -vsync no -reset 15 ${UXPLAY_FS} > /tmp/uxplay.log
"
EOF
chmod +x ~/Desktop/Start_AirPlay.sh
```

### 4. Import to Gaming Mode
To bypass Gamescope's strict background process killing mechanism, the script must be awakened via the terminal program.

1. Open the Steam client in Desktop Mode, click **Add a Game** -> **Add a Non-Steam Game** at the bottom left, check any standard application, and add it.
2. Right-click this newly added program in your Steam Library and select **Properties**.
3. **Name**: Change it to `AirPlay` (or any custom name).
4. **Target**: Enter `"/usr/bin/konsole"` (must include the double quotes).
5. **Launch Options**: Enter `--fullscreen -e /home/deck/Desktop/Start_AirPlay.sh`.
<img width="1280" height="800" alt="20260817133326_1" src="https://github.com/user-attachments/assets/c9a303c4-a06b-459e-90f1-0e801a826852" />

---

## 🎮 Usage Guide & FAQ

* **Casting in Gaming Mode**: Run the configured `AirPlay` shortcut in your Steam Library. Once the green-text terminal UI appears, open the Control Center on your iOS/iPadOS device, tap "Screen Mirroring," and select the Steam Deck. To exit, press the `STEAM button` and select "Exit Game" to release all background processes.
* **Casting in Desktop Mode**: Simply double-click `Start_AirPlay.sh` on your desktop. A **geeky graphical menu** will pop up, allowing you to use the D-Pad to choose between "Standalone Window" (resizable, perfect for multi-tasking) or "Immersive Fullscreen" (Auto-selects window mode if no action is taken within 3 seconds).
* **Screen Blocked by Terminal Window**: In rare cases within Gaming Mode, if you hear audio but see no video, press the `STEAM button` to bring up the left menu, click on the running AirPlay program, and manually switch to the `UxPlay` video window from the multi-window list.
* **Background Streaming Limitations**: This solution is based on reverse-engineering the AirPlay Mirroring protocol. Lacking official DRM certificates, it does not support independent DLNA background streaming via in-app TV icons (e.g., inside YouTube or Bilibili). The iOS device screen must remain on during casting.
* **Dim Screen Issue**: The script maintains a 45-second heartbeat to prevent system sleep. If your screen still dims while plugged in and playing, go to Gaming Mode, press the `STEAM button` -> Settings -> Display -> and manually set "Dim display when plugged in" to "Never."

---
*Based on the open-source implementation by the [UxPlay](https://github.com/FDH2/UxPlay) community.*
