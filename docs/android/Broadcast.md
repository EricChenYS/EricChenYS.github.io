---
layout: default
title: Broadcast
description: Android Broadcast
---

## 1. Broadcast
分类：有序广播和无序广播

其中无序广播中有特殊的广播ParallelBroadcast，该类广播无需等待其他广播执行完成，并且需要是动态注册的广播（TODO）
```java
/**
 * Lists of all active broadcasts that are to be executed immediately
 * (without waiting for another broadcast to finish).  Currently this only
 * contains broadcasts to registered receivers, to avoid spinning up
 * a bunch of processes to execute IntentReceiver components.  Background-
 * and foreground-priority broadcasts are queued separately.
 */
final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
```

有序广播
mAlarmQueue
mOrderedBroadcasts

## 2. BroadcastQueue
### 2.1 mFgBroadcastQueue
```java
(intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0

/**
 * If set, when sending a broadcast the recipient is allowed to run at
 * foreground priority, with a shorter timeout interval.  During normal
 * broadcasts the receivers are not automatically hoisted out of the
 * background priority class.
 */
public static final int FLAG_RECEIVER_FOREGROUND = 0x10000000;
```


### 2.2 mBgBroadcastQueue
```java
(intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) == 0
```


### 2.3 mBgOffloadBroadcastQueue
```java
private boolean isOnBgOffloadQueue(int flags) {
    return (mEnableOffloadQueue && ((flags & Intent.FLAG_RECEIVER_OFFLOAD) != 0));
}
```
- 会进入到该BroadcastQueue的广播：
Intent.ACTION_LOCKED_BOOT_COMPLETED
Intent.ACTION_BOOT_COMPLETED

### 2.4 mFgOffloadBroadcastQueue
```java
private boolean isOnFgOffloadQueue(int flags) {                                                                                                                         
    return ((flags & Intent.FLAG_RECEIVER_OFFLOAD_FOREGROUND) != 0);
}
```
- 会进入到该BroadcastQueue的广播：
Intent.ACTION_CONFIGURATION_CHANGED
