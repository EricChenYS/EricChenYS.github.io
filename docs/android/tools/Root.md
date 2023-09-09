---
layout: default
title: Android Root
description: 如何Root Android设备
---

# 前提
PC有安装adb，fastboot

# 步骤
## 1）OemUnlock（Bootloader解锁）
MTK平台解锁Bootloader
1. 开发者选项开启OEM Unlocking开关
2. 进入fastboot
     命令：adb reboot-bootloader
3. 解锁Bootloader
     命令：fastboot flashing unlock
4. 长按音量+，等待完成解锁
5. 重启手机
     命令：fastboot reboot


## 2）安装Magisk
1. 安装Magisk Apk
     命令：adb install -r Magisk.apk
2. Push rom的boot.img到手机
     命令：adb push boot.img sdcard/
3. Magisk安装boot.img
     步骤：手机进入Magisk的Home tab，然后点击install，选择Select and patch a File，找到sdcard里面的boot.img, 点击Let's Go。等待执行完成，会在sdcard/Download/目录下面生成magisk_patched-xxx.img
4. 将生成的magisk_patched-xxx.img从手机里面pull出来到PC上
      命令：adb pull sdcard/Download/magisk_patched-xxx.img .
5. 进入fastboot
     命令：adb reboot-bootloader
6. 关闭安全验证
     命令：fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
7. 刷入magisk_patched-xxx.img
     命令：fastboot flash boot magisk_patched-xxx.img
8. 重启手机
    命令：fastboot reboot
