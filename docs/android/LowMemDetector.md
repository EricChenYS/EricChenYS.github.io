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


some avg10=5.2% avg60=2.1% avg300=0.8% total=123456789  
full avg10=0.1% avg60=0.0% avg300=0.0% total=12345  

avg10：过去 10秒 的平均压力百分比。
avg60：过去 1分钟（60秒） 的平均压力百分比。
avg300：过去 5分钟（300秒） 的平均压力百分比。
total： 表示自系统启动以来，因资源（如内存、CPU、I/O）不足导致任务阻塞或停滞的累计时间（单位：微秒，1秒=1,000,000微秒）。它直接反映系统在生命周期内经历的总压力时长，是绝对值而非百分比


