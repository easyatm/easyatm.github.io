---
title: "windows-BBR2造成vscode连不上远程ss"
date: 2026-03-24
draft: false
---

## windows-BBR2造成vscode连不上远程ss

查看状态:

    #powershell命令
    Get-NetTCPSetting | Select SettingName, CongestionProvider
.

    #已开启状态
    PS R:\TerminalTemp> Get-NetTCPSetting | Select SettingName, CongestionProvider

    SettingName      CongestionProvider
    -----------      ------------------
    Automatic
    InternetCustom   BBR2
    DatacenterCustom BBR2
    Compat           BBR2
    Datacenter       BBR2
    Internet         BBR2

    #已关闭状态
    PS R:\TerminalTemp> Get-NetTCPSetting | Select SettingName, CongestionProvider

    SettingName      CongestionProvider
    -----------      ------------------
    Automatic
    InternetCustom   CUBIC
    DatacenterCustom CUBIC
    Compat           NewReno
    Datacenter       CUBIC
    Internet         CUBIC

## 禁用BBR2

    netsh int tcp set supplemental Template=Internet CongestionProvider=cubic
    netsh int tcp set supplemental Template=Datacenter CongestionProvider=cubic
    netsh int tcp set supplemental Template=Compat CongestionProvider=newreno
    netsh int tcp set supplemental Template=DatacenterCustom CongestionProvider=cubic
    netsh int tcp set supplemental Template=InternetCustom CongestionProvider=cubic