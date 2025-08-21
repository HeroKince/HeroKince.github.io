---
title: Android 后台启动Activity适配
author: HeroKince
date: 2023-08-30 22:55:06 +0800
categories: [Android]
tags: [Android,Activity]
---

在Android 9及以下版本，后台启动Activity相对自由，但是如果在Activity上下文之外启动Activity会有限制。
```
Calling startActivity() from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag
```
所以此时需要给intent添加flag：FLAG_ACTIVITY_NEW_TASK。

在Android版本10及以后版本， 引入了后台执行限制，限制了应用在后台执行操作的能力。非核心任务的后台启动 Activity 可能会受到限制。详情可参见官方文档：[从后台启动 Activity 的限制](https://developer.android.com/guide/components/activities/background-starts?hl=zh-cn)。

根据文档可知，大致有两种方案可实现从后台启动Activity。

##### 方案一：设置全屏Notification

设置Notification时通过setFullScreenIntent添加一个全屏Intent对象，可以在Android 10上从后台启动一个Activity界面，需要在Manifest.xml清单文件中加上：
```
<uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT"/>
```
示例代码如下：
```
private fun getChannelNotificationQ(context: Context, title: String?, content: String?): Notification {
        val fullScreenPendingIntent = PendingIntent.getActivity(
            context,
            0,
            DemoActivity.genIntent(context),
            PendingIntent.FLAG_UPDATE_CURRENT
        )
        val notificationBuilder = NotificationCompat.Builder(context, ID)
            .setSmallIcon(R.drawable.ic_launcher_foreground)
            .setContentTitle(title)
            .setContentText(content)
            .setPriority(NotificationCompat.PRIORITY_MAX)
            .setCategory(Notification.CATEGORY_CALL)
            .setOngoing(true)
            .setFullScreenIntent(fullScreenPendingIntent, true)
        return notificationBuilder.build()
    }
```

##### 方案二：获取SYSTEM_ALERT_WINDOW权限

如果用户已向应用授予SYSTEM_ALERT_WINDOW权限，则可以在后台启动Activity。在 Android 10 Go 版本中，应用已经无法直接获得SYSTEM_ALERT_WINDOW权限。不过Android引入了一种称为"Display over other apps"（在其他应用上层显示）的新权限体系。这种新的权限体系允许应用请求"TYPE_APPLICATION_OVERLAY"类型的窗口权限。申请步骤如下：

在Manifest.xml清单文件中加上：
```
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```
代码中发起请求权限申请：
```
if (!Settings.canDrawOverlays(this)) {
     Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
     intent.setData(Uri.parse("package:" + getPackageName()));
     startActivityForResult(intent, 0);
}
```

##### 定制化ROM新权限

有些机型增加了一项权限——后台弹出界面，比如在华为、 小米等设备上便新增了这项权限，且默认是关闭的，除非加入了它们的白名单。而且如果权限是关闭的，那么前面所说的两种方案将无效。所以在这些机型上，必须获取后台弹出界面权限，才能够从后台启动Activity。

判断是否获取弹出界面权限：

```
object PopBackgroundPermissionUtil {
    private const val TAG = "PopPermissionUtil"

    private const val HW_OP_CODE_POPUP_BACKGROUND_WINDOW = 100000
    private const val XM_OP_CODE_POPUP_BACKGROUND_WINDOW = 10021

    /**
     * 是否有后台弹出页面权限
     */
    fun hasPopupBackgroundPermission(): Boolean {
        if (isHuawei()) {
            return checkHwPermission()
        }
        if (isXiaoMi()) {
            return checkXmPermission()
        }
        if (isVivo()) {
            checkVivoPermission()
        }
        if (isOppo() && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            return Settings.canDrawOverlays(sContext)
        }
        return true
    }

    fun isHuawei(): Boolean {
        return checkManufacturer("huawei")
    }

    fun isXiaoMi(): Boolean {
        return checkManufacturer("xiaomi")
    }

    fun isOppo(): Boolean {
        return checkManufacturer("oppo")
    }

    fun isVivo(): Boolean {
        return checkManufacturer("vivo")
    }

    private fun checkManufacturer(manufacturer: String): Boolean {
        return manufacturer.equals(Build.MANUFACTURER, true)
    }

    private fun checkHwPermission(): Boolean {
        val context = sContext
        try {
            val c = Class.forName("com.huawei.android.app.AppOpsManagerEx")
            val m = c.getDeclaredMethod(
                "checkHwOpNoThrow",
                AppOpsManager::class.java,
                Int::class.javaPrimitiveType,
                Int::class.javaPrimitiveType,
                String::class.java
            )
            val result = m.invoke(
                c.newInstance(),
                *arrayOf(
                    context.getSystemService(Context.APP_OPS_SERVICE) as AppOpsManager,
                    HW_OP_CODE_POPUP_BACKGROUND_WINDOW,
                    Binder.getCallingUid(),
                    context.packageName
                )
            ) as Int
            Log.d(
                TAG,
                "PopBackgroundPermissionUtil checkHwPermission result:" + (AppOpsManager.MODE_ALLOWED == result)
            )
            return AppOpsManager.MODE_ALLOWED == result
        } catch (e: Exception) {
            //ignore
        }
        return false
    }

    private fun checkXmPermission(): Boolean {
        val context = sContext
        val ops = context.getSystemService(Context.APP_OPS_SERVICE) as AppOpsManager
        try {
            val method = ops.javaClass.getMethod(
                "checkOpNoThrow", *arrayOf<Class<*>?>(
                    Int::class.javaPrimitiveType, Int::class.javaPrimitiveType, String::class.java
                )
            )
            val result = method.invoke(
                ops,
                XM_OP_CODE_POPUP_BACKGROUND_WINDOW,
                Process.myUid(),
                context.packageName
            ) as Int
            Log.d(
                TAG,
                "PopBackgroundPermissionUtil checkXmPermission result:" + (AppOpsManager.MODE_ALLOWED == result)
            )
            return result == AppOpsManager.MODE_ALLOWED
        } catch (e: Exception) {
            //ignore
        }
        return false
    }

    private fun checkVivoPermission(): Boolean {
        val context = sContext
        val uri =
            Uri.parse("content://com.vivo.permissionmanager.provider.permission/start_bg_activity")
        val selection = "pkgname = ?"
        val selectionArgs = arrayOf(context.packageName)
        var result = 1
        val contentResolver = context.contentResolver
        try {
            contentResolver.query(uri, null, selection, selectionArgs, null).use { cursor ->
                if (cursor!!.moveToFirst()) {
                    result = cursor.getInt(cursor.getColumnIndex("currentstate"))
                }
            }
        } catch (exception: Exception) {
            //ignore
        }
        Log.d(
            TAG,
            "PopBackgroundPermissionUtil checkVivoPermission result:" + (AppOpsManager.MODE_ALLOWED == result)
        )
        return result == AppOpsManager.MODE_ALLOWED
    }

}
```
跳转弹出界面权限界面：
```
class SystemAlertWindow(private val mSource: Activity) {
    
    fun start(requestCode: Int) {
        var intent: Intent?
        intent = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (MARK.contains("meizu")) {
                meiZuApi(mSource)
            } else {
                MdefaultApi(mSource)
            }
        } else {
            if (MARK.contains("huawei")) {
                huaweiApi(mSource)
            } else if (MARK.contains("xiaomi")) {
                xiaomiApi(mSource)
            } else if (MARK.contains("oppo")) {
                oppoApi(mSource)
            } else if (MARK.contains("vivo")) {
                vivoApi(mSource)
            } else if (MARK.contains("meizu")) {
                meizuApi(mSource)
            } else {
                LdefaultApi(mSource)
            }
        }
        try {
            mSource.startActivityForResult(intent, requestCode)
        } catch (e: Exception) {
            intent = appDetailsApi(mSource)
            mSource.startActivityForResult(intent, requestCode)
        }
    }

    private fun huaweiApi(context: Context): Intent? {
        val intent = Intent()
        intent.setClassName(
            "com.huawei.systemmanager",
            "com.huawei.permissionmanager.ui.MainActivity"
        )
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName(
            "com.huawei.systemmanager",
            "com.huawei.systemmanager.addviewmonitor.AddViewMonitorActivity"
        )
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName(
            "com.huawei.systemmanager",
            "com.huawei.notificationmanager.ui.NotificationManagmentActivity"
        )
        return if (hasActivity(context, intent)) {
            intent
        } else MdefaultApi(context)
    }

    private fun xiaomiApi(context: Context): Intent? {
        val intent = Intent("miui.intent.action.APP_PERM_EDITOR")
        intent.putExtra("extra_pkgname", context.packageName)
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName(
            "com.miui.securitycenter",
            "com.miui.permcenter.permissions.AppPermissionsEditorActivity"
        )
        return if (hasActivity(context, intent)) {
            intent
        } else MdefaultApi(context)
    }

    private fun oppoApi(context: Context): Intent? {
        val intent = Intent()
        intent.putExtra("packageName", context.packageName)
        intent.setClassName(
            "com.color.safecenter",
            "com.color.safecenter.permission.floatwindow.FloatWindowListActivity"
        )
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName(
            "com.coloros.safecenter",
            "com.coloros.safecenter.sysfloatwindow.FloatWindowListActivity"
        )
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName("com.oppo.safe", "com.oppo.safe.permission.PermissionAppListActivity")
        return if (hasActivity(context, intent)) {
            intent
        } else MdefaultApi(context)
    }

    private fun vivoApi(context: Context): Intent? {
        val intent = Intent()
        intent.setClassName(
            "com.iqoo.secure",
            "com.iqoo.secure.ui.phoneoptimize.FloatWindowManager"
        )
        intent.putExtra("packagename", context.packageName)
        if (hasActivity(context, intent)) {
            return intent
        }
        intent.setClassName(
            "com.iqoo.secure",
            "com.iqoo.secure.safeguard.SoftPermissionDetailActivity"
        )
        return if (hasActivity(context, intent)) {
            intent
        } else MdefaultApi(context)
    }

    private fun meizuApi(context: Context): Intent? {
        val intent = Intent("com.meizu.safe.security.SHOW_APPSEC")
        intent.putExtra("packageName", context.packageName)
        intent.component = ComponentName("com.meizu.safe", "com.meizu.safe.security.AppSecActivity")
        return if (hasActivity(context, intent)) {
            intent
        } else MdefaultApi(context)
    }

    companion object {
        private val MARK = Build.MANUFACTURER.lowercase(Locale.getDefault())
        const val REQUEST_OVERLY = 7562
        private fun LdefaultApi(context: Context): Intent {
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
            intent.data = Uri.fromParts("package", context.packageName, null)
            return intent
        }

        private fun appDetailsApi(context: Context): Intent {
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
            intent.data = Uri.fromParts("package", context.packageName, null)
            return intent
        }

        private fun MdefaultApi(context: Context): Intent? {
            var intent: Intent? = null
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                intent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION)
            }
            intent!!.data = Uri.fromParts("package", context.packageName, null)
            return if (hasActivity(context, intent)) {
                intent
            } else appDetailsApi(context)
        }

        private fun meiZuApi(context: Context): Intent? {
            val intent = Intent("com.meizu.safe.security.SHOW_APPSEC")
            intent.putExtra("packageName", context.packageName)
            intent.setClassName("com.meizu.safe", "com.meizu.safe.security.AppSecActivity")
            return if (hasActivity(context, intent)) {
                intent
            } else MdefaultApi(context)
        }

        private fun hasActivity(context: Context, intent: Intent?): Boolean {
            val packageManager = context.packageManager
            return packageManager.queryIntentActivities(
                intent!!,
                PackageManager.MATCH_DEFAULT_ONLY
            ).size > 0
        }
    }
}
```

##### 权限说明

"Draw Over Other Apps"（在其他应用上层绘制）权限：

这个权限允许应用在其他应用的上层绘制悬浮窗口，例如悬浮通知、悬浮工具栏、聊天头像等。通过这个权限，应用可以在其他应用的界面上显示自己的内容，但是这些窗口通常会有一定的限制，不会覆盖系统级别的UI元素（如状态栏、导航栏等）。

"Background Pop-ups"（后台弹窗）权限：

这个权限控制应用在后台是否允许弹出窗口，即使应用处于后台运行状态。这意味着即使应用不在前台，它仍然可以显示一些弹窗、通知或者提醒。这可以让应用在后台运行时继续向用户展示重要的信息。

##### 总结

- 先判断是否是特殊机型，如果是则需要申请后台弹出界面权限
- 如果不是特殊机型，则有两种方案，一是全屏通知，二是申请在其他应用上层绘制权限

参考：

- [Android后台启动的实践之路](https://ljd1996.github.io/2020/12/18/Android%E5%90%8E%E5%8F%B0%E5%90%AF%E5%8A%A8%E7%9A%84%E5%AE%9E%E8%B7%B5%E4%B9%8B%E8%B7%AF/)
- https://github.com/zhoulinxue/BGStart