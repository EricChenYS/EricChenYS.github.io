---
layout: default
title: Selinux
description: Android selinux
---
# Selinux简介

## untrusted_app_x.te的定义
system/sepolicy/private/untrusted_app_x.te
1. untrusted_app.te
   This file defines the rules for untrusted apps running with targetSdkVersion >= 34(和代码有关系，Android U)
2. untrusted_app_30.te
   This file defines the rules for untrusted apps running with 29 < targetSdkVersion <= 31
3. untrusted_app_32.te
   This file defines the rules for untrusted apps running with 31 < targetSdkVersion <= 33
