---
layout: default
title: AppLock
description: Android AppLock解决方案
---
## 方案一：
DeviceOwner通过调用DPMS的setPackagesSuspended方法可以让App处于Suspend状态，可以现在App Activity的启动

原理：
ActivityStarter.java
ActivityStarter.executeRequest -> ActivityStartInterceptor.intercept -> ActivityStartInterceptor.interceptSuspendedPackageIfNeeded(return true进行拦截，并在拦截后，替换掉了原来启动的Activity)
关键代码（ActivityStarter.java）
```
if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, inTaskFragment,
                callingPid, callingUid, checkedOptions)) {
    // activity start was intercepted, e.g. because the target user is currently in quiet
    // mode (turn off work) or the target application is suspended
    intent = mInterceptor.mIntent;
    rInfo = mInterceptor.mRInfo;
    aInfo = mInterceptor.mAInfo;
    resolvedType = mInterceptor.mResolvedType;
    inTask = mInterceptor.mInTask;
    callingPid = mInterceptor.mCallingPid;
    callingUid = mInterceptor.mCallingUid;
    checkedOptions = mInterceptor.mActivityOptions;

    // The interception target shouldn't get any permission grants
    // intended for the original destination
    intentGrants = null;
}
```

注：
App Suspend状态介绍：
处于Suspend状态的App，进程依然可以正常运行，只是不允许启动App的Activity



## 方案二：
特殊权限(android.Manifest.permission.SET_ACTIVITY_WATCHER protectionLevel="signature")的App设置ActivityTaskManagerService中的Controller(调用IActivityManager.aidl 中的setActivityController)，然后重写Controller中的activityStarting方法就可以实现拦截Activity的启动。

原理：
Activity在启动的时候，如果有设置过Controller，则会先调用Controller的activityStarting来判断是否允许启动
ActivityStarter.java
ActivityStarter.executeRequest -> ActivityTaskManagerService.mController.activityStarting


关键代码
```
if (mService.mController != null) {
    try {
        // The Intent we give to the watcher has the extra data stripped off, since it
        // can contain private information.
        Intent watchIntent = intent.cloneFilter();
        abort |= !mService.mController.activityStarting(watchIntent,
                aInfo.applicationInfo.packageName);
    } catch (RemoteException e) {
        mService.mController = null;
    }
}
```
注：该方案可以监听到app的多种状态
```
interface IActivityController
{
    /**
     * The system is trying to start an activity.  Return true to allow
     * it to be started as normal, or false to cancel/reject this activity.
     */
    boolean activityStarting(in Intent intent, String pkg);
    
    /**
     * The system is trying to return to an activity.  Return true to allow
     * it to be resumed as normal, or false to cancel/reject this activity.
     */
    boolean activityResuming(String pkg);
    
    /**
     * An application process has crashed (in Java).  Return true for the
     * normal error recovery (app crash dialog) to occur, false to kill
     * it immediately.
     */
    boolean appCrashed(String processName, int pid,
            String shortMsg, String longMsg,
            long timeMillis, String stackTrace);
    
    /**
     * Early call as soon as an ANR is detected.
     */
    int appEarlyNotResponding(String processName, int pid, String annotation);

    /**
     * An application process is not responding.  Return 0 to show the "app
     * not responding" dialog, 1 to continue waiting, or -1 to kill it
     * immediately.
     */
    int appNotResponding(String processName, int pid, String processStats);

    /**
     * The system process watchdog has detected that the system seems to be
     * hung.  Return 1 to continue waiting, or -1 to let it continue with its
     * normal kill.
     */
    int systemNotResponding(String msg);
}
```

## 方案三：
Framework埋点通知App activity切换
