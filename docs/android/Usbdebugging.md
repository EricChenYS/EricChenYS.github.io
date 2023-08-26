---
layout: default
title: Usedebugging
description: Android Usedebugging
---

# Settings开发者选项里面开启Usb debugging
行为整理
## 1. Settings行为
往Settings.Global的 ADB_ENABLED("adb_enabled")里面写入1

参考：
frameworks/base/packages/SettingsLib/src/com/android/settingslib/development/AbstractEnableAdbPreferenceController.java

```
protected void writeAdbSetting(boolean enabled) {
    Settings.Global.putInt(mContext.getContentResolver(),
            Settings.Global.ADB_ENABLED, enabled ? ADB_SETTING_ON : ADB_SETTING_OFF);
    notifyStateChanged();
}

public static final int ADB_SETTING_ON = 1;
public static final int ADB_SETTING_OFF = 0;
```


## 2. AdbService行为（SystemServer）

a. AdbService系统服务在onStart的时候会new一个AdbService对象，在AdbService的构造方法里面会注册ContentObserver监听Settings.Global.ADB_ENABLED的变化
参考：
frameworks/base/services/core/java/com/android/server/adb/AdbService.java

```
private void registerContentObservers() {
    try {
        // register observer to listen for settings changes
        mObserver = new AdbSettingsObserver();
        mContentResolver.registerContentObserver(
                Settings.Global.getUriFor(Settings.Global.ADB_ENABLED),
                false, mObserver);
        mContentResolver.registerContentObserver(
                Settings.Global.getUriFor(Settings.Global.ADB_WIFI_ENABLED),
                false, mObserver);
    } catch (Exception e) {
        Slog.e(TAG, "Error in registerContentObservers", e);
    }
}
```

## b. AdbSettingsObserver在监听到Settings.Global.ADB_ENABLED变化后会

b1) Start（或Stop）native serivce（adbd）

b2) 会回调AdbTransport(UsbDeviceManager)的onAdbEnabled方法

b3) 会调用AdbDebuggingManager的setAdbEnabled
参考：
frameworks/base/services/core/java/com/android/server/adb/AdbService.java

```
private class AdbSettingsObserver extends ContentObserver {
    private final Uri mAdbUsbUri = Settings.Global.getUriFor(Settings.Global.ADB_ENABLED);
    private final Uri mAdbWifiUri = Settings.Global.getUriFor(Settings.Global.ADB_WIFI_ENABLED);

    AdbSettingsObserver() {
        super(null);
    }

    @Override
    public void onChange(boolean selfChange, @NonNull Uri uri, @UserIdInt int userId) {
        if (mAdbUsbUri.equals(uri)) {
            boolean shouldEnable = (Settings.Global.getInt(mContentResolver,
                    Settings.Global.ADB_ENABLED, 0) > 0);
            FgThread.getHandler().sendMessage(obtainMessage(
                    AdbService::setAdbEnabled, AdbService.this, shouldEnable,
                        AdbTransportType.USB));
        } else if (mAdbWifiUri.equals(uri)) {
            boolean shouldEnable = (Settings.Global.getInt(mContentResolver,
                    Settings.Global.ADB_WIFI_ENABLED, 0) > 0);
            FgThread.getHandler().sendMessage(obtainMessage(
                    AdbService::setAdbEnabled, AdbService.this, shouldEnable,
                        AdbTransportType.WIFI));
        }
    }
}

private void setAdbEnabled(boolean enable, byte transportType) {
    if (DEBUG) {
        Slog.d(TAG, "setAdbEnabled(" + enable + "), mIsAdbUsbEnabled=" + mIsAdbUsbEnabled
                + ", mIsAdbWifiEnabled=" + mIsAdbWifiEnabled + ", transportType="
                    + transportType);
    }

    if (transportType == AdbTransportType.USB && enable != mIsAdbUsbEnabled) {
        mIsAdbUsbEnabled = enable;
    } else if (transportType == AdbTransportType.WIFI && enable != mIsAdbWifiEnabled) {
        mIsAdbWifiEnabled = enable;
        if (mIsAdbWifiEnabled) {
            if (!AdbProperties.secure().orElse(false) && mDebuggingManager == null) {
                // Start adbd. If this is secure adb, then we defer enabling adb over WiFi.
                SystemProperties.set(WIFI_PERSISTENT_CONFIG_PROPERTY, "1");
                mConnectionPortPoller =
                        new AdbDebuggingManager.AdbConnectionPortPoller(mPortListener);
                mConnectionPortPoller.start();
            }
        } else {
            // Stop adb over WiFi.
            SystemProperties.set(WIFI_PERSISTENT_CONFIG_PROPERTY, "0");
            if (mConnectionPortPoller != null) {
                mConnectionPortPoller.cancelAndWait();
                mConnectionPortPoller = null;
            }
        }
    } else {
        // No change
        return;
    }

    if (enable) {
        startAdbd();
    } else {
        stopAdbd();
    }

    for (IAdbTransport transport : mTransports.values()) {
        try {
            transport.onAdbEnabled(enable, transportType);
        } catch (RemoteException e) {
            Slog.w(TAG, "Unable to send onAdbEnabled to transport " + transport.toString());
        }
    }

    if (mDebuggingManager != null) {
        mDebuggingManager.setAdbEnabled(enable, transportType);
    }

    if (DEBUG) {
        Slog.d(TAG, "Broadcasting enable = " + enable + ", type = " + transportType);
    }
    mCallbacks.broadcast((callback) -> {
        if (DEBUG) {
            Slog.d(TAG, "Sending enable = " + enable + ", type = " + transportType
                    + " to " + callback);
        }
        try {
            callback.onDebuggingChanged(enable, transportType);
        } catch (RemoteException ex) {
            if (DEBUG) {
                Slog.d(TAG, "Unable to send onDebuggingChanged:", ex);
            }
        }
    });
}


start或stop adbd（native service）的方法

/**
 * Adb native daemon.
 */
static final String ADBD = "adbd";

/**
 * Command to start native service.
 */
static final String CTL_START = "ctl.start";

/**
 * Command to start native service.
 */
static final String CTL_STOP = "ctl.stop";

private void startAdbd() {
    SystemProperties.set(CTL_START, ADBD);
}

private void stopAdbd() {
    if (!mIsAdbUsbEnabled && !mIsAdbWifiEnabled) {
        SystemProperties.set(CTL_STOP, ADBD);
    }
}

```


## 3. UsbDeviceManager行为

a. 系统服务UsbService在systemReady的时候会调用UsbDeviceManager的systemReady方法

b. UsbDeviceManager的systemReady里面会register AdbTransport

c. AdbService调用AdbTransport onAdbEnabled(UsbDeviceManager里面AdbTransport onAdbEnabled)的行为（USB连接才会走下面，wifi连接不会）

c1）修改property存在adb是enable还是disable

c2）启动USB

c3）更新AdbNotifiction

参考：
frameworks/base/services/usb/java/com/android/server/usb/UsbDeviceManager.java

```
/**
 * The persistent property which stores whether adb is enabled or not.
 * May also contain vendor-specific default functions for testing purposes.
 */
protected static final String USB_PERSISTENT_CONFIG_PROPERTY = "persist.sys.usb.config";

/**
 * Name of the adb USB function.
 * Used in extras for the {@link #ACTION_USB_STATE} broadcast
 *
 * @hide
 */
public static final String USB_FUNCTION_ADB = "adb";

private void setAdbEnabled(boolean enable, int operationId) {
    if (DEBUG) Slog.d(TAG, "setAdbEnabled: " + enable);

    if (enable) {
        setSystemProperty(USB_PERSISTENT_CONFIG_PROPERTY, UsbManager.USB_FUNCTION_ADB);
    } else {
        setSystemProperty(USB_PERSISTENT_CONFIG_PROPERTY, "");
    }

    setEnabledFunctions(mCurrentFunctions, true, operationId);
    updateAdbNotification(false);
}


@Override
protected void setEnabledFunctions(long functions, boolean forceRestart, int operationId) {
    if (DEBUG) {
        Slog.d(TAG, "setEnabledFunctionsi " +
                "functions=" + functions +
                ", forceRestart=" + forceRestart +
                ", operationId=" + operationId);
    }
    if (mCurrentGadgetHalVersion < UsbManager.GADGET_HAL_V1_2) {
        if ((functions & UsbManager.FUNCTION_NCM) != 0) {
            Slog.e(TAG, "Could not set unsupported function for the GadgetHal");
            return;
        }
    }
    if (mCurrentFunctions != functions
            || !mCurrentFunctionsApplied
            || forceRestart) {
        Slog.i(TAG, "Setting USB config to " + UsbManager.usbFunctionsToString(functions));
        mCurrentFunctions = functions;
        mCurrentFunctionsApplied = false;
        // set the flag to false as that would be stale value
        mCurrentUsbFunctionsRequested = false;

        boolean chargingFunctions = functions == UsbManager.FUNCTION_NONE;
        functions = getAppliedFunctions(functions);

        // Set the new USB configuration.
        setUsbConfig(functions, chargingFunctions, operationId);

        if (mBootCompleted && isUsbDataTransferActive(functions)) {
            // Start up dependent services.
            updateUsbStateBroadcastIfNeeded(functions);
        }
    }
}

和USB hal层通信
private void setUsbConfig(long config, boolean chargingFunctions, int operationId) {
    if (true) Slog.d(TAG, "setUsbConfig(" + config + ") request:" + ++mCurrentRequest);
    /**
     * Cancel any ongoing requests, if present.
     */
    removeMessages(MSG_FUNCTION_SWITCH_TIMEOUT);
    removeMessages(MSG_SET_FUNCTIONS_TIMEOUT);
    removeMessages(MSG_SET_CHARGING_FUNCTIONS);

    synchronized (mGadgetProxyLock) {
        if (mUsbGadgetHal == null) {
            Slog.e(TAG, "setUsbConfig mUsbGadgetHal is null");
            return;
        }
        try {
            if ((config & UsbManager.FUNCTION_ADB) != 0) {
                /**
                 * Start adbd if ADB function is included in the configuration.
                 */
                LocalServices.getService(AdbManagerInternal.class)
                        .startAdbdForTransport(AdbTransportType.USB);
            } else {
                /**
                 * Stop adbd otherwise
                 */
                LocalServices.getService(AdbManagerInternal.class)
                        .stopAdbdForTransport(AdbTransportType.USB);
            }
            mUsbGadgetHal.setCurrentUsbFunctions(mCurrentRequest,
                    config, chargingFunctions,
                    SET_FUNCTIONS_TIMEOUT_MS - SET_FUNCTIONS_LEEWAY_MS, operationId);
            sendMessageDelayed(MSG_SET_FUNCTIONS_TIMEOUT, chargingFunctions,
                    SET_FUNCTIONS_TIMEOUT_MS);
            if (mConnected) {
                // Only queue timeout of enumeration when the USB is connected
                sendMessageDelayed(MSG_FUNCTION_SWITCH_TIMEOUT, chargingFunctions,
                        SET_FUNCTIONS_TIMEOUT_MS + ENUMERATION_TIME_OUT_MS);
            }
            if (DEBUG) Slog.d(TAG, "timeout message queued");
        } catch (Exception e) {//RemoteException e) {
            Slog.e(TAG, "Remoteexception while calling setCurrentUsbFunctions", e);
        }
    }
}

```




## 4. AdbDebuggingManager行为（主要负责auth相关行为）

a. 在AdbService的构造方法里面new了AdbDebuggingManager对象

b. AdbDebuggingThread和adbd进行socket通信（实现待确认），监听adb发送过来的指令，启动请求授权的SystemUI弹框


参考：
frameworks/base/services/core/java/com/android/server/adb/AdbDebuggingManager.java

```
/**
 * When {@code enabled} is {@code true}, this allows ADB debugging and starts the ADB handler
 * thread. When {@code enabled} is {@code false}, this disallows ADB debugging for the given
 * @{code transportType}. See {@link IAdbTransport} for all available transport types.
 * If all transport types are disabled, the ADB handler thread will shut down.
 */
public void setAdbEnabled(boolean enabled, byte transportType) {
    if (transportType == AdbTransportType.USB) {
        mHandler.sendEmptyMessage(enabled ? AdbDebuggingHandler.MESSAGE_ADB_ENABLED
                                          : AdbDebuggingHandler.MESSAGE_ADB_DISABLED);
    } else if (transportType == AdbTransportType.WIFI) {
        mHandler.sendEmptyMessage(enabled ? AdbDebuggingHandler.MSG_ADBDWIFI_ENABLE
                                          : AdbDebuggingHandler.MSG_ADBDWIFI_DISABLE);
    } else {
        throw new IllegalArgumentException(
                "setAdbEnabled called with unimplemented transport type=" + transportType);
    }
}

private void startAdbDebuggingThread() {
    ++mAdbEnabledRefCount;
    if (DEBUG) Slog.i(TAG, "startAdbDebuggingThread ref=" + mAdbEnabledRefCount);
    if (mAdbEnabledRefCount > 1) {
        return;
    }

    registerForAuthTimeChanges();
    mThread = new AdbDebuggingThread();
    mThread.setHandler(mHandler);
    mThread.start();

    mAdbKeyStore.updateKeyStore();
    scheduleJobToUpdateAdbKeyStore();
}

private void stopAdbDebuggingThread() {
    --mAdbEnabledRefCount;
    if (DEBUG) Slog.i(TAG, "stopAdbDebuggingThread ref=" + mAdbEnabledRefCount);
    if (mAdbEnabledRefCount > 0) {
        return;
    }

    if (mThread != null) {
        mThread.stopListening();
        mThread = null;
    }

    if (!mConnectedKeys.isEmpty()) {
        for (Map.Entry<String, Integer> entry : mConnectedKeys.entrySet()) {
            mAdbKeyStore.setLastConnectionTime(entry.getKey(), mTicker.currentTimeMillis());
        }
        sendPersistKeyStoreMessage();
        mConnectedKeys.clear();
        mWifiConnectedKeys.clear();
    }
    scheduleJobToUpdateAdbKeyStore();
}



private static final String ADBD_SOCKET = "adbd";

@VisibleForTesting
static class AdbDebuggingThread extends Thread {
    private boolean mStopped;
    private LocalSocket mSocket;
    private OutputStream mOutputStream;
    private InputStream mInputStream;
    private Handler mHandler;

    @VisibleForTesting
    AdbDebuggingThread() {
        super(TAG);
    }

    @VisibleForTesting
    void setHandler(Handler handler) {
        mHandler = handler;
    }

    @Override
    public void run() {
        if (DEBUG) Slog.d(TAG, "Entering thread");
        while (true) {
            synchronized (this) {
                if (mStopped) {
                    if (DEBUG) Slog.d(TAG, "Exiting thread");
                    return;
                }
                try {
                    openSocketLocked();
                } catch (Exception e) {
                    /* Don't loop too fast if adbd dies, before init restarts it */
                    SystemClock.sleep(1000);
                }
            }
            try {
                listenToSocket();
            } catch (Exception e) {
                /* Don't loop too fast if adbd dies, before init restarts it */
                SystemClock.sleep(1000);
            }
        }
    }

    private void openSocketLocked() throws IOException {
        try {
            LocalSocketAddress address = new LocalSocketAddress(ADBD_SOCKET,
                    LocalSocketAddress.Namespace.RESERVED);
            mInputStream = null;

            if (DEBUG) Slog.d(TAG, "Creating socket");
            mSocket = new LocalSocket(LocalSocket.SOCKET_SEQPACKET);
            mSocket.connect(address);

            mOutputStream = mSocket.getOutputStream();
            mInputStream = mSocket.getInputStream();
            mHandler.sendEmptyMessage(AdbDebuggingHandler.MSG_ADBD_SOCKET_CONNECTED);
        } catch (IOException ioe) {
            Slog.e(TAG, "Caught an exception opening the socket: " + ioe);
            closeSocketLocked();
            throw ioe;
        }
    }

    private void listenToSocket() throws IOException {
        try {
            byte[] buffer = new byte[BUFFER_SIZE];
            while (true) {
                int count = mInputStream.read(buffer);
                // if less than 2 bytes are read the if statements below will throw an
                // IndexOutOfBoundsException.
                if (count < 2) {
                    Slog.w(TAG, "Read failed with count " + count);
                    break;
                }

                if (buffer[0] == 'P' && buffer[1] == 'K') {
                    String key = new String(Arrays.copyOfRange(buffer, 2, count));
                    Slog.d(TAG, "Received public key: " + key);
                    Message msg = mHandler.obtainMessage(
                            AdbDebuggingHandler.MESSAGE_ADB_CONFIRM);
                    msg.obj = key;
                    mHandler.sendMessage(msg);
                } else if (buffer[0] == 'D' && buffer[1] == 'C') {
                    String key = new String(Arrays.copyOfRange(buffer, 2, count));
                    Slog.d(TAG, "Received disconnected message: " + key);
                    Message msg = mHandler.obtainMessage(
                            AdbDebuggingHandler.MESSAGE_ADB_DISCONNECT);
                    msg.obj = key;
                    mHandler.sendMessage(msg);
                } else if (buffer[0] == 'C' && buffer[1] == 'K') {
                    String key = new String(Arrays.copyOfRange(buffer, 2, count));
                    Slog.d(TAG, "Received connected key message: " + key);
                    Message msg = mHandler.obtainMessage(
                            AdbDebuggingHandler.MESSAGE_ADB_CONNECTED_KEY);
                    msg.obj = key;
                    mHandler.sendMessage(msg);
                } else if (buffer[0] == 'W' && buffer[1] == 'E') {
                    // adbd_auth.h and AdbTransportType.aidl need to be kept in
                    // sync.
                    byte transportType = buffer[2];
                    String key = new String(Arrays.copyOfRange(buffer, 3, count));
                    if (transportType == AdbTransportType.USB) {
                        Slog.d(TAG, "Received USB TLS connected key message: " + key);
                        Message msg = mHandler.obtainMessage(
                                AdbDebuggingHandler.MESSAGE_ADB_CONNECTED_KEY);
                        msg.obj = key;
                        mHandler.sendMessage(msg);
                    } else if (transportType == AdbTransportType.WIFI) {
                        Slog.d(TAG, "Received WIFI TLS connected key message: " + key);
                        Message msg = mHandler.obtainMessage(
                                AdbDebuggingHandler.MSG_WIFI_DEVICE_CONNECTED);
                        msg.obj = key;
                        mHandler.sendMessage(msg);
                    } else {
                        Slog.e(TAG, "Got unknown transport type from adbd (" + transportType
                                + ")");
                    }
                } else if (buffer[0] == 'W' && buffer[1] == 'F') {
                    byte transportType = buffer[2];
                    String key = new String(Arrays.copyOfRange(buffer, 3, count));
                    if (transportType == AdbTransportType.USB) {
                        Slog.d(TAG, "Received USB TLS disconnect message: " + key);
                        Message msg = mHandler.obtainMessage(
                                AdbDebuggingHandler.MESSAGE_ADB_DISCONNECT);
                        msg.obj = key;
                        mHandler.sendMessage(msg);
                    } else if (transportType == AdbTransportType.WIFI) {
                        Slog.d(TAG, "Received WIFI TLS disconnect key message: " + key);
                        Message msg = mHandler.obtainMessage(
                                AdbDebuggingHandler.MSG_WIFI_DEVICE_DISCONNECTED);
                        msg.obj = key;
                        mHandler.sendMessage(msg);
                    } else {
                        Slog.e(TAG, "Got unknown transport type from adbd (" + transportType
                                + ")");
                    }
                } else {
                    Slog.e(TAG, "Wrong message: "
                            + (new String(Arrays.copyOfRange(buffer, 0, 2))));
                    break;
                }
            }
        } finally {
            synchronized (this) {
                closeSocketLocked();
            }
        }
    }
}



    case MESSAGE_ADB_CONFIRM: {
        String key = (String) msg.obj;
        if ("trigger_restart_min_framework".equals(
                SystemProperties.get("vold.decrypt"))) {
            Slog.w(TAG, "Deferring adb confirmation until after vold decrypt");
            if (mThread != null) {
                mThread.sendResponse("NO");
                logAdbConnectionChanged(key, AdbProtoEnums.DENIED_VOLD_DECRYPT, false);
            }
            break;
        }
        String fingerprints = getFingerprints(key);
        if ("".equals(fingerprints)) {
            if (mThread != null) {
                mThread.sendResponse("NO");
                logAdbConnectionChanged(key, AdbProtoEnums.DENIED_INVALID_KEY, false);
            }
            break;
        }
        logAdbConnectionChanged(key, AdbProtoEnums.AWAITING_USER_APPROVAL, false);
        mFingerprints = fingerprints;
        startConfirmationForKey(key, mFingerprints);
        break;
    }

启动请求授权的SystemUI弹框
private void startConfirmationForKey(String key, String fingerprints) {
    List<Map.Entry<String, String>> extras = new ArrayList<Map.Entry<String, String>>();
    extras.add(new AbstractMap.SimpleEntry<String, String>("key", key));
    extras.add(new AbstractMap.SimpleEntry<String, String>("fingerprints", fingerprints));
    int currentUserId = ActivityManager.getCurrentUser();
    UserInfo userInfo = UserManager.get(mContext).getUserInfo(currentUserId);
    String componentString;
    if (userInfo.isAdmin()) {
        componentString = mConfirmComponent != null
                ? mConfirmComponent : Resources.getSystem().getString(
                com.android.internal.R.string.config_customAdbPublicKeyConfirmationComponent);
    } else {
        // If the current foreground user is not the admin user we send a different
        // notification specific to secondary users.
        componentString = Resources.getSystem().getString(
                R.string.config_customAdbPublicKeyConfirmationSecondaryUserComponent);
    }
    ComponentName componentName = ComponentName.unflattenFromString(componentString);
    if (startConfirmationActivity(componentName, userInfo.getUserHandle(), extras)
            || startConfirmationService(componentName, userInfo.getUserHandle(),
                    extras)) {
        return;
    }
    Slog.e(TAG, "unable to start customAdbPublicKeyConfirmation[SecondaryUser]Component "
            + componentString + " as an Activity or a Service");
}

/**
 * @return true if the componentName led to an Activity that was started.
 */
private boolean startConfirmationActivity(ComponentName componentName, UserHandle userHandle,
        List<Map.Entry<String, String>> extras) {
    PackageManager packageManager = mContext.getPackageManager();
    Intent intent = createConfirmationIntent(componentName, extras);
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    if (packageManager.resolveActivity(intent, PackageManager.MATCH_DEFAULT_ONLY) != null) {
        try {
            mContext.startActivityAsUser(intent, userHandle);
            return true;
        } catch (ActivityNotFoundException e) {
            Slog.e(TAG, "unable to start adb whitelist activity: " + componentName, e);
        }
    }
    return false;
}

/**
 * @return true if the componentName led to a Service that was started.
 */
private boolean startConfirmationService(ComponentName componentName, UserHandle userHandle,
        List<Map.Entry<String, String>> extras) {
    Intent intent = createConfirmationIntent(componentName, extras);
    try {
        if (mContext.startServiceAsUser(intent, userHandle) != null) {
            return true;
        }
    } catch (SecurityException e) {
        Slog.e(TAG, "unable to start adb whitelist service: " + componentName, e);
    }
    return false;
}



[frameworks/base/core/res/res/values/config.xml]
<!-- Name of the activity or service that prompts the user to reject, accept, or allowlist
     an adb host's public key, when an unwhitelisted host connects to the local adbd.
     Can be customized for other product types -->
<string name="config_customAdbPublicKeyConfirmationComponent"
        >com.android.systemui/com.android.systemui.usb.UsbDebuggingActivity</string>


```



## 5. adbd

## 6. usb

