---
title: "pve相关"
date: 2026-03-19
draft: false
---

# 让终端画面旋转180度

因为我显示器是倒过来安装的

    echo 2 | tee /sys/class/graphics/fbcon/rotate

# 让PVE宿主机禁止加载驱动

#自定义一个配置文件,写入需要禁止的驱动

    nano /etc/modprobe.d/disable-list.conf

文件内容

    #这是禁用usb串口驱动
    blacklist ch341
    blacklist cp210x
    blacklist pl2303
    blacklist ftdi_sio

    # --- 禁用 Intel 无线网卡 (Wi-Fi) ---
    # iwlmvm 是核心管理模块，iwlwifi 是底层驱动
    blacklist iwlmvm
    blacklist iwlwifi
    blacklist mac80211
    blacklist cfg80211

    # --- 禁用 蓝牙 (Bluetooth) ---
    # btusb 是 USB 接口的蓝牙驱动，其他是特定厂商的固件补丁
    blacklist btusb
    blacklist btintel
    blacklist btrtl
    blacklist btbcm
    blacklist btmtk
    blacklist bluetooth

更新

    update-initramfs -u

然后重启