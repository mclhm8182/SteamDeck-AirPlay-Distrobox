中文文档 | [English](README.md)

---

# SteamDeck iOS AirPlay Setup

基于 Distrobox 和 UxPlay 的 SteamOS iOS/iPadOS/Mac 屏幕镜像投屏部署方案。
该工作流专为 SteamOS 游戏模式 (Gaming Mode) （也可以用于桌面模式）优化，解决了默认环境下的硬件解码失效、自适应缩放异常以及网络连接假死等问题。

## 核心特性

* **容器化部署**: 依托 Distrobox 与精简版 Ubuntu 镜像，环境数据储存在不受系统大版本更新影响的 `/home` 目录。
* **AMD 硬件解码**: 透传 SteamOS 底层驱动 (`-avdec`) 并关闭时间戳缓冲 (`-vsync no`)，大幅降低输入延迟。
* **自适应全屏**: 兼容 Gamescope 视窗管理器，强制画面拉伸铺满屏幕 (`-fs`)，解决视频全屏播放时的边界裁切与缩放异常。
* **连接稳定性**: 启动前自动清理 Avahi 广播缓存与死锁端口，调整默认心跳断连阈值 (`-reset 15`)，修复重复连接导致的假死报错。
* **防休眠进程**: 投屏期间后台挂起守护进程，循环发送 `dbus` 活跃信号，阻止系统在长视频播放期间自动息屏。

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

### 3. 生成一键启动脚本
在桌面模式的 Konsole 终端中，直接复制并运行以下整段代码。此操作会在桌面生成名为 `Start_AirPlay.sh` 的可执行脚本：

```bash
cat << 'EOF' > ~/Desktop/Start_AirPlay.sh
#!/bin/bash
unset LD_PRELOAD

# 【新增】宿主机级双保险：在进入容器前，强制在 SteamOS 最外层斩断所有残留的投屏幽灵进程
pkill -9 -f uxplay 2>/dev/null

# 挂起防息屏心跳机制
(
    while true; do
        dbus-send --session --dest=org.freedesktop.ScreenSaver --type=method_call /org/freedesktop/ScreenSaver org.freedesktop.ScreenSaver.SimulateUserActivity 2>/dev/null
        sleep 180
    done
) &
AWAKE_PID=$!

distrobox enter uxplay-env -- bash -l -c "
echo 'Cleaning network cache and dead ports...'
# 【修正】使用系统更底层的 pkill 替代 killall，彻底解决端口占用 (Socket 98) 报错
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

# 清理进程并恢复原生电源策略
kill -9 $AWAKE_PID 2>/dev/null
sleep 3
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

* **启动投屏**: 切换至游戏模式，运行配置好的 `AirPlay` 快捷方式。等待出现终端黑屏后，在 iOS/iPadOS 控制中心点击“屏幕镜像”并选择 Steam Deck 设备。退出时按 `STEAM 键` 选择退出游戏即可释放所有后台进程。在桌面模式，则直接双击桌面Start_AirPlay.sh打开即可投屏（可能没有任何反馈，但是可以正常投屏）。
* **画面被终端窗口遮挡**: 游戏模式下，若遇到有声音无画面，按 `STEAM 键` 呼出左侧菜单，点击当前运行的 AirPlay 程序，在弹出的多窗口列表中手动切换至 `UxPlay` 视频窗口即可。
* **关于后台推送限制**: 本方案基于 AirPlay Mirroring 屏幕镜像协议逆向，因缺乏官方 DRM 证书支持，不支持通过应用内（如 Bilibili、YouTube）的 TV 图标进行 DLNA 视频流后台推送。投屏期间需保持 iOS 端屏幕常亮。
* **屏幕变暗（Dim Screen）问题**: 防息屏脚本仅阻止设备进入睡眠状态。若插电播放期间屏幕变暗，请在游戏模式按 `STEAM 键` -> 设置 -> 显示 -> 将“插电时屏幕变暗”设置为“从不”。

---
*Based on the open-source implementation by the [UxPlay](https://github.com/FDH2/UxPlay) community.*

