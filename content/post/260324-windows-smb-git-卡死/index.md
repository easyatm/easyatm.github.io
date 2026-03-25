---
title: "windows在smb共享驱动器中使用git卡死"
date: 2026-03-24
draft: false
---

检测文件:
 
    C:\Users\Administrator\.gitconfig
.

    [filter "lfs"]
        clean = git-lfs clean -- %f
        smudge = git-lfs smudge -- %f
        process = git-lfs filter-process
        required = true
    [safe]
        directory = %(prefix)///192.168.2.20/files/root/easy_esp
        directory = *
    [pull]
        rebase = false
    [core]
        filemode = false
        trustctime = false

safe中有个不存在的网络路径,删除即可

    directory = %(prefix)///192.168.2.20/files/root/easy_esp
