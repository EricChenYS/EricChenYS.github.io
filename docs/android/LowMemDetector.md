---
layout: default
title: LowMemDetector
description: 低内存检测
---
# 低内存检测
## 1. 实现原理
依赖Linux Kernel的PSI，LowMemThread会等待PSI会上报event

## 2. PSI
Pressure Stall Information（压力失速信息），这是 Linux 内核中的一种机制，用于实时监测系统资源（CPU、内存、I/O）的压力状态。当系统资源不足时，PSI 会通过指标反映资源争用情况，帮助检测问题。

### 关于内存检测：
当 PSI 监测到内存压力时，会报告以下信息：

"some" 指标：部分任务因内存不足而延迟的百分比时间。
"full" 指标：整个系统因内存不足而完全停滞的百分比时间。
例如，cat /proc/pressure/memory 的输出可能类似：

```java
some avg10=5.2% avg60=2.1% avg300=0.8% total=123456789  
full avg10=0.1% avg60=0.0% avg300=0.0% total=12345
```

avg10：过去 10秒 的平均压力百分比。

avg60：过去 1分钟（60秒） 的平均压力百分比。

avg300：过去 5分钟（300秒） 的平均压力百分比。

total： 表示自系统启动以来，因资源（如内存、CPU、I/O）不足导致任务阻塞或停滞的累计时间（单位：微秒，1秒=1,000,000微秒）。它直接反映系统在生命周期内经历的总压力时长，是绝对值而非百分比

### 数值解读
avg10=5.2% 表示过去10秒内，有5.2%的时间系统因资源不足导致任务延迟。

total=123456789 表示：从系统启动到当前时刻，因内存不足导致部分任务停滞的总时间为 123,456,789 微秒（即约 123.46 秒）

### 高低意义：
avg10 高：短期压力大（需立即关注，可能影响当前任务）。
avg60/avg300 高：中长期压力大（需优化资源配置或排查泄漏）
total：若total随运行时间快速增加，可能暗示内存泄漏或资源配置不足

参考：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/am/LowMemDetector.java


