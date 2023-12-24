---
layout: default
title: UserRestriction
description: Android UserRestriction
---


# 1. 简介
对于手机里面的一些功能进行限制

# 2. 设置方法
## 2.1 通过pm的命令来设置
adb shell pm set-user-restriction [--user USER_ID] RESTRICTION VALUE

pm命令的实质是通过UserManger的setUserRestriction接口来设置（需要权限）

例如：
| 指令描述        | 指令          |
|:-------------|:------------------|
| 添加限制 | adb shell pm  set-user-restriction no_install_unknown_sources 1 |
| 移除限制 | adb shell pm  set-user-restriction no_install_unknown_sources 0 |



## 2.2 通过DeviceOwner/ProfileOwner来设置
DevicePolicyManager的addUserRestriction接口来设置

## 2.3 行为分析
1）adb shell pm  set-user-restriction no_install_unknown_sources 1
2）do设置no_install_unknown_sources
3）adb shell dumpsys user

![image](https://github.com/EricChenYS/EricChenYS.github.io/assets/5690448/a3c3b4bb-d458-4c65-9e0d-3bceebba23c1)

分析：
Restrictions:
UserManager接口设置的限制策略会显示在这里，在UserManagerService里面是有mBaseUserRestrictions来保存

Device policy global restrictions:
DO或PO设置的限制策略会显示在这里，在UserManagerService里面是有mDevicePolicyGlobalUserRestrictions来保存；保存应用到所有user的策略，包含子用户，guest


Device policy local restrictions:
DO或PO设置的限制策略会显示在这里，在UserManagerService里面是有mDevicePolicyLocalUserRestrictions来保存；保存应用到当前user的策略

Effective restrictions:
UserManager接口或DO设置的限制策略会显示在这里，在UserManagerService里面是有mCachedEffectiveUserRestrictions来保存。该值也显示所有生效的策略（不关心是调用DO/PO还是UserManager的接口来设置）


# 3. 接口介绍
## 3.1 UserManager hasUserRestriction
实现描述：从mCachedEffectiveUserRestrictions的策略来判断是否有限制，简而言之就是DO/PO或通过UserManager接口设置的策略，都可以通过UserManager的hasUserRestriction来判断是否有设置相应策略
```java
@Override
public boolean hasUserRestriction(String restrictionKey, @UserIdInt int userId) {
    if (!UserRestrictionsUtils.isValidRestriction(restrictionKey)) {
        return false;
    }
    Bundle restrictions = getEffectiveUserRestrictions(userId);
    return restrictions != null && restrictions.getBoolean(restrictionKey);
}
```

## 3.2 UserManager hasBaseUserRestriction
实现描述：从mBaseUserRestrictions的策略来判断是否有限制，简而言之就是通过UserManager接口设置的策略，可以通过UserManager的hasBaseUserRestriction来判断是否有设置相应策略；DO/PO接口设置的策略，不能通过UserManager的hasBaseUserRestriction来判断是否有设置相应策略
```java
@Override
public boolean hasBaseUserRestriction(String restrictionKey, @UserIdInt int userId) {
    checkManageOrCreateUsersPermission("hasBaseUserRestriction");
    if (!UserRestrictionsUtils.isValidRestriction(restrictionKey)) {
        return false;
    }
    synchronized (mRestrictionsLock) {
        Bundle bundle = mBaseUserRestrictions.getRestrictions(userId);
        return (bundle != null && bundle.getBoolean(restrictionKey, false));
    }
}
```
