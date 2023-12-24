---
layout: default
title: PinnerService
description: Android PinnerService
---


# 1. 目的
为了将一些常用的apk和jar锁定在物理内存中，防止被交换到磁盘上。可以减少内存访问的延迟，提高应用程序的响应速度


# 2. Pin APP & File
Camera，Home，Assistant
    private static final int MAX_CAMERA_PIN_SIZE = 80 * (1 << 20); // 80MB max for camera app.
    private static final int MAX_HOME_PIN_SIZE = 6 * (1 << 20); // 6MB max for home app.
    private static final int MAX_ASSISTANT_PIN_SIZE = 60 * (1 << 20); // 60MB max for assistant app.




# 3. 原理
Os.mlock(address + pinStart, pinLength);
Os.mlock()是一个系统调用，用于将指定的内存区域锁定在物理内存中，防止被交换到磁盘上。这个函数通常用于提高性能，
特别是在需要频繁访问某些内存区域时。锁定内存区域可以减少内存访问的延迟，提高应用程序的响应速度。但是需要注意的是，
使用mlock()函数锁定的内存区域会一直占用物理内存，直到被显式地解锁。因此，在使用mlock()函数时需要谨慎，
避免锁定过多的内存导致系统内存不足。
