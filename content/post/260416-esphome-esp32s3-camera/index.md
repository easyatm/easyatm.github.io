---
title: "ESP32-S3 OV2640 ESPHome 摄像头配置指南"
date: 2026-03-19
draft: false
---


## 硬件准备

- **主控板**：Freenove ESP32-S3 WROOM（N16R8，16MB Flash，8MB PSRAM）
- **摄像头**：OV2640
- **连接方式**：OV2640 FPC 排线插入板载 FPC 插槽，金属触点朝下，锁扣锁紧

> ⚠️ **注意**：如果出现画面卡顿或连接不稳定，请切换到另一个 USB 口（原生串口 USB），不要使用 JTAG USB 口（靠近摄像头那侧）。供电和烧录统一使用原生串口 USB 口。

---

## 软件环境

- Home Assistant（已安装）
- ESPHome Builder Add-on（已安装）
- 用于本地编译的 Debian 12 虚拟机（可选，编译速度更快）

---

## 第一步：刷入 Bootstrap 固件

首次刷写使用 ESPHome Web，无需安装任何软件。

1. 用 USB 连接 ESP32-S3 到电脑
2. 打开 [https://web.esphome.io](https://web.esphome.io)（Chrome 或 Edge）
3. 点击 **Connect** → 选择串口
4. 点击 **Prepare for first use** → 输入 Wi-Fi 名称和密码 → 刷入
5. 刷完后设备自动连接 Wi-Fi

> 如果串口无法识别，按住 **BOOT** 键再插 USB。

---

## 第二步：在 HA ESPHome Builder 中接管设备

Bootstrap 刷入后，设备会出现在 ESPHome Builder 中，显示 **DISCOVERED** 状态（见下图）。

> ℹ️ **注意**：新版 ESPHome 已将 `Adopt` 按钮改为 **`TAKE CONTROL`**，点击后输入设备名称确认即可，ESPHome 会自动生成基础配置。

![ESPHome Builder 截图](screenshot.png)

---

## 第三步：配置 secrets.yaml

在 ESPHome Builder 左上角点击 **Secrets**，填入：

```yaml
wifi_ssid: "你的WiFi名称"
wifi_password: "你的WiFi密码"
```

---

## 第四步：完整 YAML 配置

将以下内容覆盖设备的 YAML 配置：

```yaml
esphome:
  name: "esphome-camera"
  friendly_name: ESPHome Camera
  min_version: 2025.11.0
  name_add_mac_suffix: false

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework:
    type: esp-idf
  flash_size: 16MB
  cpu_frequency: 240MHz

psram:
  mode: octal
  speed: 80MHz

logger:

api:

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
  ap:
    ssid: "Camera Fallback"

captive_portal:

i2c:
  - id: camera_i2c
    sda: GPIO4
    scl: GPIO5

esp32_camera:
  name: "ESP32-S3 Camera"
  external_clock:
    pin: GPIO15
    frequency: 20MHz
  i2c_id: camera_i2c
  data_pins: [GPIO11, GPIO9, GPIO8, GPIO10, GPIO12, GPIO18, GPIO17, GPIO16]
  vsync_pin: GPIO6
  href_pin: GPIO7
  pixel_clock_pin: GPIO13
  frame_buffer_location: PSRAM   # PSRAM（快）或 DRAM（慢）
  frame_buffer_count: 2
  resolution: 800x600
  jpeg_quality: 6                # 6=最佳画质，10=默认，63=最差
  max_framerate: 15 fps          # 默认 10fps
  idle_framerate: 0.1 fps
  vertical_flip: false
  horizontal_mirror: false       # 水平镜像

esp32_camera_web_server:
  - port: 8080
    mode: stream
  - port: 8081
    mode: snapshot
```

---

## 引脚对照表（Freenove ESP32-S3 WROOM）

| 信号 | GPIO |
|------|------|
| CAM_SIOD (SDA) | GPIO4 |
| CAM_SIOC (SCL) | GPIO5 |
| CAM_XCLK | GPIO15 |
| CAM_Y2 (D0) | GPIO11 |
| CAM_Y3 (D1) | GPIO9 |
| CAM_Y4 (D2) | GPIO8 |
| CAM_Y5 (D3) | GPIO10 |
| CAM_Y6 (D4) | GPIO12 |
| CAM_Y7 (D5) | GPIO18 |
| CAM_Y8 (D6) | GPIO17 |
| CAM_Y9 (D7) | GPIO16 |
| CAM_VYSNC | GPIO6 |
| CAM_HREF | GPIO7 |
| CAM_PCLK | GPIO13 |

---

## 第五步：编译与刷写

> ✅ **推荐优先使用本地 Linux 系统编译**，速度远快于 HA ESPHome Builder。每次修改配置后都需要重新编译并上传固件。

### 方式 A：在本地 Linux/Debian 虚拟机编译（推荐）

```bash
# 首次安装 ESPHome
pip3 install esphome --break-system-packages

# 创建工作目录
mkdir ~/esphome && cd ~/esphome

# 创建 secrets.yaml
nano secrets.yaml
# 填入以下内容：
# wifi_ssid: "你的WiFi名称"
# wifi_password: "你的WiFi密码"

# 创建 camera.yaml，粘贴上方完整配置
nano camera.yaml

# 编译并 OTA 上传（每次修改配置后执行）
esphome compile camera.yaml && esphome upload camera.yaml
```

### 方式 B：在 HA ESPHome Builder 中编译（较慢，不推荐）

点击设备 → **Install** → **Wirelessly**

---

## 第六步：查看摄像头画面

**浏览器直接访问：**

| 地址 | 说明 |
|------|------|
| `http://<设备IP>:8080` | MJPEG 实时流 |
| `http://<设备IP>:8081` | 单帧快照 |

**在 HA 仪表板添加摄像头卡片：**

仪表板 → 编辑 → 添加卡片 → **Camera** → 选择 `camera.esp32_s3_camera`

---

## 常见问题

**Q：画面卡顿/延迟高**
- 确认 `power_save_mode: none` 已配置
- 运行期间断开 JTAG USB 口
- 确认 `frame_buffer_location: PSRAM`，不要用 DRAM

**Q：`Got invalid frame from camera!`**
- 检查 FPC 排线是否插紧，锁扣是否锁上
- 确认排线金属触点朝下

**Q：编译报 PSRAM 错误**
- N16R8 必须使用 `mode: octal`，不能用 quad

**Q：Setup Failed: ESP_ERR_NOT_FOUND**
- I2C 引脚接错，检查 SDA/SCL 对应 CAM_SIOD/CAM_SIOC

---

## 分辨率参考

| 分辨率 | 适用场景 |
|--------|---------|
| 320x240 | 最流畅，低带宽 |
| 640x480 | 均衡推荐 |
| 800x600 | 较清晰（当前配置） |
| 1600x1200 | 最清晰，仅适合单张快照 |

> OV2640 最大支持 1600x1200（UXGA）
