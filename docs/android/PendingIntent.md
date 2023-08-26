---
layout: default
title: PendingIntent
description: Android PendingIntent
---

PendingIntent pendingIntent = PendingIntent.getBroadcast(mContext, 0, intent,
PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_MUTABLE);


flag包含PendingIntent.FLAG_MUTABLE，intent中的参数才能带过去(带到BroadcastReceiver的intent中)
