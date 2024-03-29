---
layout: default
title: Safe mode
description: Android safe mode
---
# Safe mode简介

Safe mode存在的目的 当Android处于严重crash状态时，可以进入safe mode将导致问题的app卸载掉，解决crash后再进入normal mode

## Safe mode的行为

1. FontManagerService不会读取UpdatableFontDir

2. DisplayManagerService不会注册ADDITIONAL_DISPLAY_ADAPTERS（即Safe mode不会注册非必需的Display adapter）

3. 开机自动进入飞行模式（目的：为了防止radio firmware导致的问题），进入Safe mode后，用户是可以关闭飞行模式的

4. ActivityManagerService控制展示Safe mode overlay(设备右下角的Safe mode字样)

5. ProcessList在startProc的时候，设置runtimeFlag(Zygote.DEBUG_ENABLE_SAFEMODE)（目的：将进程设置为debugable进程，即Safe mode下，release app也是可以debug的）

6. PackageManagerService在通过Intent查找Component的时候，仅匹配System app（即会增加flag：PackageManager.MATCH_SYSTEM_ONLY） 

7. PackageManagerService在checkPackageStartable时，如果是Safe mode，且不是system app时，会抛出SecurityException（即认为非system app不能被start）

8. Launcher disable widget


## 如何进入Safe mode

关机情况进入Safe mode 开机按“音量减”（注：需要进入Android后才开始按键）

