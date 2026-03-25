---
title: "intel NUC j4005 debian12 播放RTSP"
date: 2026-03-19
draft: false
---

## 使用CPU播放

    mpv --vo=drm --hwdec=vaapi --ao=alsa "rtsp://192.168.3.2:8554/onvif_8_3"

参数说明（本行常用参数）：

- **--vo**: 指定视频输出驱动，这里 `drm` 表示通过 Direct Rendering Manager 输出到显示设备。
- **--hwdec**: 硬件解码类型，`vaapi` 代表使用 Intel/VA-API 硬件加速解码，可减轻 CPU 负担。
- **--ao**: 指定音频输出，`alsa` 为 Linux ALSA 音频驱动。

---

### 使用GPU播放

    mpv --vo=gpu \
        --gpu-context=drm \
        --hwdec=vaapi \
        --ao=alsa \
        "rtsp://192.168.3.2:8554/onvif_8_3"

参数说明：

- **--vo=gpu**: 使用 mpv 的 GPU 渲染输出，通常配合 `--gpu-context` 使用。
- **--gpu-context**: 指定 GPU 上下文类型，`drm` 用于直接在 DRM 环境下创建上下文。

---

### 加快响应

    mpv --vo=gpu \
        --gpu-context=drm \
        --hwdec=vaapi \
        --profile=low-latency \
        --cache=no \
        --demuxer-readahead-secs=0 \
        --rtsp-transport=tcp \
        "rtsp://192.168.3.2:8554/onvif_8_3"

参数说明（用于降低延迟）：

- **--profile=low-latency**: 启用 mpv 的低延迟预设，调整若干内部参数以减少延迟。
- **--cache=no**: 关闭 mpv 的播放缓存，能进一步降低延迟，但可能增加丢帧风险。
- **--demuxer-readahead-secs=0**: 取消 demuxer 的预读秒数，配合 `cache=no` 使用以减少缓冲。
- **--rtsp-transport=tcp**: 强制使用 TCP 作为 RTSP 传输协议（也可用 udp），TCP 更稳定但可能更高延迟。

---

### 水平反转180度

    mpv --vo=gpu --gpu-context=drm --hwdec=vaapi \
    --video-rotate=180 \
    --profile=low-latency --cache=no --demuxer-readahead-secs=0 \
    --rtsp-transport=tcp "rtsp://192.168.3.2:8554/onvif_8_3"


参数说明：

- **--video-rotate=180**: 将视频旋转 180 度（适用于摄像头安装倒置等情况）。

---

### 声音加大

    mpv --vo=gpu \
        --gpu-context=drm \
        --hwdec=vaapi \
        --video-rotate=180 \
        --profile=low-latency \
        --cache=no \
        --demuxer-readahead-secs=0 \
        --rtsp-transport=tcp \
        --volume=200 \
        --ao=alsa \
        "rtsp://192.168.3.2:8554/onvif_8_3"

参数说明（音量相关）：

- **--volume=300**: 设置播放音量，mpv 默认 100 为基准，300 表示放大到 3 倍，注意可能导致失真或硬件限制。

---

### 去掉噪音

    mpv --vo=gpu --gpu-context=drm --hwdec=vaapi \
    --video-rotate=180 \
    --profile=low-latency --cache=no --demuxer-readahead-secs=0 \
    --rtsp-transport=tcp \
    --volume=150 --ao=alsa \
    --af="highpass=f=200,afftdn=nf=-30,agate=threshold=0.01:range=0:ratio=2" \
    "rtsp://192.168.3.2:8554/onvif_8_3"

参数说明（音频滤镜）：

- **--af**: 音频滤镜链，本例中包含：
    - `highpass=f=200`: 高通滤波器，去除 200Hz 以下低频噪声；
    - `afftdn=nf=-30`: 频域降噪滤镜，`nf` 控制噪声门限（dB）；
    - `agate=threshold=0.01:range=0:ratio=2`: 门限/增益滤镜，用于抑制突发噪声并平滑音量。

---

### 断开自动重连

    while true; do
        echo "正在启动监控流..."
        mpv --vo=gpu \
            --gpu-context=drm \
            --hwdec=vaapi \
            --video-rotate=180 \
            --profile=low-latency \
            --cache=no \
            --demuxer-readahead-secs=0 \
            --rtsp-transport=tcp \
            --volume=300 \
            --ao=alsa \
            "rtsp://192.168.3.2:8554/onvif_8_3"

        echo "检测到连接断开，1秒后尝试重新连接..."
        sleep 1
    done

参数说明（脚本要点）：

- 该循环脚本用于检测 mpv 播放退出后自动重连；当 mpv 断开或退出时，循环将在 1 秒后重启播放器。
- 可根据需要将 `sleep` 时间调整为更短或更长以控制重连频率。

## 在ssh中关闭/开启显示器
    setterm --term linux --blank force < /dev/tty1

    setterm --term linux --blank poke < /dev/tty1
    printf "\033[c" > /dev/tty1

## 查看GPU状态工具

    apt install intel-gpu-tools -y
    intel_gpu_top
