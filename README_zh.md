中文文档 | [English](README.md)

---

# SteamDeck iOS AirPlay Setup

基于 Distrobox 和 UxPlay 的 SteamOS iOS/iPadOS/Mac 屏幕镜像投屏部署方案。
该工作流专为 SteamOS 游戏模式 (Gaming Mode) 与桌面模式 (Desktop Mode) 深度优化，彻底解决了默认环境下的硬件解码失效、自适应缩放异常、网络连接假死以及休眠断连等痛点。

## 核心特性与优化

* **智能环境识别与交互**: 自动判定当前运行模式。游戏模式下静默全屏启动；桌面模式下自动拉起专属终端，并提供**手柄十字键交互菜单**，可自由选择“独立窗口”或“沉浸全屏”。
* **全局低延迟锁定 (30 FPS)**: 针对苹果 AirPlay 镜像协议进行调优，强制锁定 30 帧，大幅降低视频推流时的局域网拥堵，彻底解决老旧设备（如 iPad 6）及 2.4G Wi-Fi 环境下的“画面延迟与音画不同步”问题。
* **容器化无损部署**: 依托 Distrobox 与精简版 Ubuntu 镜像，环境数据储存在不受系统大版本更新影响的 `/home` 目录，一次部署，终身免维护。
* **AMD 硬件解码**: 透传 SteamOS 底层驱动 (`-avdec`) 并关闭时间戳缓冲 (`-vsync no`)，大幅降低输入延迟。
* **激进防休眠守护**: 投屏期间后台挂起守护进程，每 45 秒循环发送 `dbus` 活跃心跳，彻底击穿系统激进的省电策略，阻止长视频播放期间自动息屏。

---
<img width="3024" height="4032" alt="IMG_1897" src="https://github.com/user-attachments/assets/17e74014-dadc-4751-908c-8ec539530231" />

<img width="1280" height="800" alt="20260817133348_1" src="https://github.com/user-attachments/assets/83c2ff27-9f68-47ad-bad4-0be9d6a3bc84" />
<img width="1280" height="800" alt="20260817133405_1" src="https://github.com/user-attachments/assets/798ce1dd-c0f3-4ac2-baf8-6f5c2630f04c" />
<img width="1280" height="800" alt="20260817133838_1" src="https://github.com/user-attachments/assets/87317102-5a47-4cfe-a07b-038117132333" />

## 部署流程

### 1. 创建 Ubuntu 容器环境
进入 Steam Deck 的桌面模式 (Desktop Mode)，打开 Konsole（终端），拉取基础镜像并创建名为 `uxplay-env` 的容器：

```bash
distrobox create --image ubuntu:latest --name uxplay-env
```
> 注：国内网络环境如遇 TLS 握手超时，可将源替换为国内镜像加速，例如
```bash
distrobox create --image docker.1panel.live/library/ubuntu:latest --name uxplay-env
```

### 2. 安装底层依赖与硬解插件
进入已创建的容器：

```bash
distrobox enter uxplay-env
```

在容器终端内执行以下命令，安装局域网广播服务、GStreamer 视频渲染组件以及 AMD 硬件解码驱动：

```bash
sudo apt update
sudo apt install -y dbus avahi-daemon uxplay gstreamer1.0-vaapi mesa-va-drivers libva-drm2 gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-base gstreamer1.0-gl
```
安装完成后，执行 `exit` 退出容器。

### 3. 生成一键自适应启动脚本
在桌面模式的 Konsole 终端中，直接复制并运行以下整段代码。此操作会在桌面生成调优后的 `Start_AirPlay.sh` 可执行脚本：

```bash
cat << 'EOF' > ~/Desktop/Start_AirPlay.sh
#!/bin/bash
unset LD_PRELOAD

TARGET_FPS=30
FPS_NAME="30 FPS (影音低延迟稳定版)"

# 1. 智能环境判定
if [ -n "$SteamTenfoot" ]; then
    IS_GAMEMODE=1
else
    IS_GAMEMODE=0
fi

# 桌面模式拉起终端窗口
if [ $IS_GAMEMODE -eq 0 ] && { [ "$XDG_SESSION_TYPE" = "x11" ] || [ "$XDG_SESSION_TYPE" = "wayland" ]; }; then
    if [ -z "$_RUNNING_IN_KONSOLE" ]; then
        export _RUNNING_IN_KONSOLE=1
        exec konsole --nofork -e "$0" "$@"
    fi
fi

# 宿主机级幽灵进程清理
pkill -9 -f uxplay 2>/dev/null

# -------------------------------------------------------------
# 🎮 手柄十字键菜单函数 (仅在桌面模式触发)
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
        echo -e "\033[1;37m        📺 请选择桌面显示模式 (当前锁定 30 FPS) \033[0m"
        echo -e "\033[1;36m=========================================================\033[0m"
        echo -e "\033[1;33m 🎮 [操作] 十字键/左摇杆 ↑↓ 切换  |  🟢 A键 / Enter 确认\033[0m"
        echo -e "\033[1;90m ⏱️  倒计时 ${remain} 秒后自动进入默认的 [独立窗口]...\033[0m"
        echo -e "\033[1;90m ---------------------------------------------------------\033[0m"

        if [ "$sel" -eq 1 ]; then
            echo -e " \033[1;32m➔ [ ▶ 1. 独立窗口 (默认) - 可拖拽缩放，带 X 关闭按钮 ]\033[0m"
            echo -e " \033[0;37m     2. 强制全屏 (沉浸) - 画面铺满屏幕，遮挡底部任务栏\033[0m"
        else
            echo -e " \033[0;37m     1. 独立窗口 (默认) - 可拖拽缩放，带 X 关闭按钮\033[0m"
            echo -e " \033[1;32m➔ [ ▶ 2. 强制全屏 (沉浸) - 画面铺满屏幕，遮挡底部任务栏 ]\033[0m"
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

# 2. 核心逻辑分流
if [ $IS_GAMEMODE -eq 1 ]; then
    # 游戏模式：静默跳过菜单，强制全屏
    UXPLAY_FS="-fs"
    WIN_NAME="游戏模式全屏"
    EXIT_TIP="* 提示：退出投屏时，请直接按 STEAM 键选择“退出游戏”"
else
    # 桌面模式：弹出交互式选择菜单
    interactive_menu
    if [ "$MENU_CHOICE" -eq 2 ]; then
        UXPLAY_FS="-fs"
        WIN_NAME="桌面全屏"
        EXIT_TIP="* ⚠️ 提示：桌面全屏下若快捷键失效，在手机端断开投屏即可退出"
    else
        UXPLAY_FS=""
        WIN_NAME="独立窗口"
        EXIT_TIP="* 提示：您可以随时点击窗口右上角的 X 按钮关闭投屏"
    fi
fi

# 3. 防息屏心跳 (45秒对抗休眠)
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

# 4. 进入容器并运行主程序
distrobox enter uxplay-env -- bash -l -c "
clear
echo -e '\033[1;36m=========================================================\033[0m'
echo -e '\033[1;37m        📺 Steam Deck AirPlay 专属屏幕镜像终端 \033[0m'
echo -e '\033[1;36m=========================================================\033[0m'
echo -e '\033[1;32m [状态] 🟢 服务已就绪 ($FPS_NAME | $WIN_NAME)\033[0m'
echo -e '\033[1;33m [操作] 请在您的苹果设备控制中心点击【屏幕镜像】进行连接\033[0m'
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

### 4. 导入游戏模式 (Gaming Mode)
为绕过 Gamescope 对无窗口进程的强杀机制，需通过终端程序唤醒该脚本。

1. 在桌面模式打开 Steam 客户端，点击左下角 **添加游戏** -> **添加非 Steam 游戏**，任意勾选一个常规程序并添加。
2. 在 Steam 库中右键该程序，选择 **属性 (Properties)**。
3. **名称**: 修改为 `AirPlay`（或任意自定义名称）。
4. **目标 (Target)**: 填写 `"/usr/bin/konsole"` （需包含双引号）。
5. **启动选项 (Launch Options)**: 填写 `--fullscreen -e /home/deck/Desktop/Start_AirPlay.sh`。
<img width="1280" height="800" alt="20260817133326_1" src="https://github.com/user-attachments/assets/c9a303c4-a06b-459e-90f1-0e801a826852" />

---

## 使用说明与常见问题

* **启动投屏**: 切换至游戏模式，运行配置好的 `AirPlay` 快捷方式。等待出现绿字终端界面后，在 iOS/iPadOS 控制中心点击“屏幕镜像”并选择 Steam Deck 设备。退出时按 `STEAM 键` 选择退出游戏即可释放所有后台进程。
* **在桌面模式投屏**: 直接双击桌面的 `Start_AirPlay.sh`，会弹出专属交互菜单，可用手柄方向键选择“独立窗口”或“全屏模式”，3秒不操作自动进入默认窗口模式。
* **画面被终端窗口遮挡**: 极少数情况下在游戏模式遇到有声音无画面，按 `STEAM 键` 呼出左侧菜单，点击当前运行的 AirPlay 程序，在弹出的多窗口列表中手动切换至 `UxPlay` 视频窗口即可。
* **关于后台推送限制**: 本方案基于 AirPlay Mirroring 屏幕镜像协议逆向，因缺乏官方 DRM 证书支持，不支持通过应用内（如 Bilibili、YouTube）的 TV 图标进行 DLNA 视频流后台推送。投屏期间需保持 iOS 端屏幕常亮。
* **屏幕变暗（Dim Screen）问题**: 防息屏脚本已在后台维持 45 秒心跳机制。若插电播放期间屏幕依旧变暗，请在游戏模式按 `STEAM 键` -> 设置 -> 显示 -> 将“插电时屏幕变暗”设置为“从不”。

---
*Based on the open-source implementation by the [UxPlay](https://github.com/FDH2/UxPlay) community.*
