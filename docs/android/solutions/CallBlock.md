# 拦截电话呼出实现

## 方案一：（无需修改系统）
注：当NEW_OUTGOING_CALL广播不是有序广播时，该方案会失效
### 1. 监听NEW_OUTGOING_CALL广播
参考：
```
<receiver android:name="pl.agilevision.semm.semmdpc.SecretCodeReceiver" android:exported="false">
    <intent-filter android:priority="999">
        <action android:name="android.intent.action.NEW_OUTGOING_CALL"/>
    </intent-filter>
</receiver>
```

### 2. 在接收到广播(android.intent.action.NEW_OUTGOING_CALL)后，调用BroadcastReceiver的setResultData(null)
参考：
```
public class SecretCodeReceiver extends BroadcastReceiver {

    @Override // android.content.BroadcastReceiver
    public void onReceive(Context context, Intent intent) {
        Log.entry();
        String action = intent.getAction();
        if (!action.equals("android.intent.action.NEW_OUTGOING_CALL")) {
            Log.w("Unhandled intent action: " + intent.getAction());
        } else {
            String stringExtra = intent.getStringExtra("android.intent.extra.PHONE_NUMBER");
            if (LockActivity.isLocked.get()) {
                if (!BuildConfig.CUSTOMER_CARE_NUMBERS.contains(stringExtra)) {
                    Log.d("Denying call to " + stringExtra);
                    setResultData(null);
                } else {
                    Log.d("Allowing call to " + stringExtra);
                }
            }
        }
    }
}
```

### 3. 授予接受广播(android.intent.action.NEW_OUTGOING_CALL)所需要的Runtime permission(android.permission.PROCESS_OUTGOING_CALLS)

### 原理参考文档：
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/content/Intent.java;l=3728;bpv=1;bpt=1?q=ACTION_NEW_OUTGOING_CALL
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/content/BroadcastReceiver.java;l=477?q=setResultData
