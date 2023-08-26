---
layout: default
title: LeakCanary
description: LeakCanary原理
---
# 功能：
可以用来检测Activity, Fragment, ViewModel, RooView, Service的泄漏

# 原理：
## 1）LeakCanary启动时机
MainProcessAppWatcherInstaller为ContentProvider, 其onCreate在Application的attachBaseContext之后，Application的onCreate之前被调用，因此不再需要开发者集成任何代码。

```
internal class MainProcessAppWatcherInstaller : ContentProvider() {

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

```

## 2）LeakCanary行为
### A. 定义检测的对象
检测Activity，FragmentAndViewModel，RootView，Service

```
watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)

fun appDefaultWatchers(
  application: Application,
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ActivityWatcher(application, reachabilityWatcher),
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    RootViewWatcher(reachabilityWatcher),
    ServiceWatcher(reachabilityWatcher)
  )
}

val objectWatcher = ObjectWatcher(
  clock = { SystemClock.uptimeMillis() },
  checkRetainedExecutor = {
    check(isInstalled) {
      "AppWatcher not installed"
    }
    mainHandler.postDelayed(it, retainedDelayMillis)
  },
  isEnabled = { true }
)

```


### B.初始化GcTrigger，HeapDumpTrigger，监测ObjectRetained，定义Shortcut等
LeakCanaryDelegate.loadLeakCanary(application)
在被监测的对象出现泄漏（ObjectRetained）时，触发Gc，dump hprof（Debug.dumpHprofData）


### C. 安装要监测的对象

```
watchersToInstall.forEach {
  it.install()
}

```

C0）检测是否leak原理
使用带有ReferenceQueue的构造方法，构造被监测者T（例如：Activity，Fragment等）的WeakReference对象
注：被监测者T如果被 GC 回收时，JVM 就会把这个弱引用存入与之关联的ReferenceQueue之中，即ReferenceQueue中如果包含弱引用对象，则表示该对象已经被Gc回收，反之表示该对象没有被Gc回收（存在内存泄漏）
ObjectWatcher

```
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  removeWeaklyReachableObjects()
  val key = UUID.randomUUID()
    .toString()
  val watchUptimeMillis = clock.uptimeMillis()
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }

  watchedObjects[key] = reference
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}

private fun removeWeaklyReachableObjects() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  var ref: KeyedWeakReference?
  do {
    ref = queue.poll() as KeyedWeakReference?
    if (ref != null) {
      watchedObjects.remove(ref.key)
    }
  } while (ref != null)
}

@Synchronized private fun moveToRetained(key: String) {
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}

```


C1）ActivityWatcher
监测Activity生命周期，在Activity onDestroy 5s后，检测该Activity是否被回收，如果没有被回收，则认为是有泄漏，会通知执行dump hprof行为

```
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}

```


C2）FragmentAndViewModelWatcher
监测Activity生命周期，在Activity onCreate后，注册监听Fragment生命周期；在onFragmentViewDestroyed回调5s后会判断监测FragmentView是否会被回收，如果没有被回收，则认为是有内存泄漏；在onFragmentDestroyed回调5后会判断监测Fragment是否被回收，如果没有被回收，则认为是有内存泄漏。
会判断包里面是否有使用“androidx.fragment.app.Fragment”和“android.support.v4.app.Fragment”，如果有使用，也会监测这些Fragment是否有内存泄漏（原理同上）
如果有使用“androidx.fragment.app.Fragment”（应该是viewmode只存在于androidx包），在ViewModel的onCleared回调5s后会监测Activity和Fragment中viewmodel是否有被回收，如果没有被回收，则认为有内存泄漏


C3）RootViewWatcher
通过Curtains获取到rootview（windowType为TOOLTIP，TOAST，UNKNOWN或PHONE_WINDOW[但是不是Activity，Dialog]），在rootView的onViewDetachedFromWindow会被回调5s后监测rootView是否会被回收，如果没有被回收，则认为有内存泄漏
注：
Curtains源码：https://github.com/square/curtains


C4）ServiceWatcher
由于没有接口可以直接监测到Service的生命周期
通过获取currentActivityThread的mCallBack，可以在Msg.what为STOP_SERVICE的时候认为是onServicePreDestroy（该阶段会保存该Service的WeakReference）
通过获取IActivityManager的serviceDoneExecuting方法在被回调时，认为是onServiceDestroyed（该阶段会在5s后监测Service是否有被回收，如果没有被回收，则认为有内存泄漏）


# 源码：
https://github.com/square/leakcanary


# 集成：
App build.gradle

```
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.8.1'
}

```

# 参考文档：
https://www.jianshu.com/p/036a4ea420d6
https://blog.csdn.net/weixin_61845324/article/details/120356951
