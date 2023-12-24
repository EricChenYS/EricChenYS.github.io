---
layout: default
title: RoleManager
description: Android RoleManager
---


# 1. 简介
## 1.1 RoleManager介绍
Android RoleManager是Android 8.0（API level 26）引入的一个系统服务，它的作用是管理应用程序的角色和相关权限。通过RoleManager，应用可以请求成为设备上的角色，然后获取相关的权限。例如，如果您的应用程序需要访问设备的联系人信息，则可以请求成为设备上的联系人应用程序，并获得相应的权限，而无需请求联系人权限。这样，用户可以更好地控制他们的设备，并减少了应用程序需要的权限数量，从而增强了设备的安全性。

## 1.2 术语介绍
### 1.2.1 Role
应用程序角色，一个应用成为一个角色后会获取这个Role所具备的权限。多个应用可能都具备成为Role的特质，但是同一时间内，一个Role只能被一个应用占有（Holder），即Role的RoleHolder最多只能有一个app

### 1.2.2 RoleHolder
成为Role的应用

# 2. 流程介绍
## 2.1 addRole
![image](https://github.com/EricChenYS/EricChenYS.github.io/assets/5690448/7c146099-1dcd-4e68-9725-52bbdcbb9ad3)


## 2.2 Android支持哪些Role
packages/modules/Permission/PermissionController/res/xml/roles.xml

也可以dump手机里面的role来查看
adb shell dumpsys role

FINANCED_DEVICE_KIOSK举例
```java
    <permission-set name="notifications">                                                                                                                                   
        <permission name="android.permission.POST_NOTIFICATIONS" minSdkVersion="33" />
    </permission-set>
 
    <!--
     ~ A role assigned to the financing kiosk app
    -->
    <role
        name="android.app.role.FINANCED_DEVICE_KIOSK"
        exclusive="true"
        minSdkVersion="34"
        visible="false">
        <permissions>
            <permission-set name="notifications" />
            <permission name="android.permission.MANAGE_DEVICE_LOCK_STATE" />
        </permissions>
    </role>
```
当一个app成为了FINANCED_DEVICE_KIOSK Role，那么这个app会有下面权限MANAGE_DEVICE_LOCK_STATE和notifications(android.permission.POST_NOTIFICATIONS)


# 3. 附录
源码

packages/modules/Permission


RoleManager

packages/modules/Permission/framework-s/java/android/app/role/RoleManager.java

Role

packages/modules/Permission/PermissionController/role-controller/java/com/android/role/controller/model/Role.java
