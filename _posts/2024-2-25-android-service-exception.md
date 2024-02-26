---
title: startForegroundService() did not then call startForeground() Crash崩溃排查记录
author: HeroKince
date: 2024-02-25 20:00:00 +0800
categories: [Crash]
tags: [Crash,Android]
---

首先看堆栈：
```
android.app.RemoteServiceException

Context.startForegroundService() did not then call Service.startForeground(): ServiceRecord{413a500 u0 com.xx.xx/com.xx.baseUi.service.BackgroundKeepService}

android.app.ActivityThread$H.handleMessage(ActivityThread.java:1796)
android.os.Handler.dispatchMessage(Handler.java:106)
android.os.Looper.loop(Looper.java:213)
android.app.ActivityThread.main(ActivityThread.java:6954)
java.lang.reflect.Method.invoke(Native Method)
com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
com.android.internal.os.ZygoteInit.main(ZygoteInit.java:909)
```

大概是说 startForegroundService 但是没有调用 Service.startForeground()。
回到业务代码里一看，BackgroundKeepService 的 onCreate 方法里确实已经调用了。这一下就很迷惑了...

思来想去都觉得这不会是AOSP的一个Bug吧... 有疑问，看源码！

在Framework的 ActiveService 里面有两个地方会抛出 Context.startForegroundService() did not then call Service.startForeground() 异常，分别是：

```
   void serviceForegroundTimeout(ServiceRecord r) {
        ProcessRecord app;
        synchronized (mAm) {
            if (!r.fgRequired || !r.fgWaiting || r.destroying) {
                return;
            }

            app = r.app;
            if (app != null && app.isDebugging()) {
                // The app's being debugged; let it ride
                return;
            }

            if (DEBUG_BACKGROUND_CHECK) {
                Slog.i(TAG, "Service foreground-required timeout for " + r);
            }
            r.fgWaiting = false;
            stopServiceLocked(r, false);
        }

        if (app != null) {
            final String annotation = "Context.startForegroundService() did not then call "
                    + "Service.startForeground(): " + r;
            Message msg = mAm.mHandler.obtainMessage(
                    ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_ANR_MSG);
            SomeArgs args = SomeArgs.obtain();
            args.arg1 = app;
            args.arg2 = annotation;
            msg.obj = args;
            mAm.mHandler.sendMessageDelayed(msg,
                    mAm.mConstants.mServiceStartForegroundAnrDelayMs);
        }
    }
```

和

```
 void serviceForegroundCrash(ProcessRecord app, String serviceRecord,
            ComponentName service) {
        mAm.crashApplicationWithTypeWithExtras(
                app.uid, app.getPid(), app.info.packageName, app.userId,
                "Context.startForegroundService() did not then call " + "Service.startForeground(): "
                    + serviceRecord, false /*force*/,
                ForegroundServiceDidNotStartInTimeException.TYPE_ID,
                ForegroundServiceDidNotStartInTimeException.createExtrasForService(service));
    }
```

由于第一个抛出的是ANR不是Crash，不是我们的目标，那么就重点分析 serviceForegroundCrash 方法。定位到调用的地方 ActivityManagerService 中有一段这样的代码：

```
         case SERVICE_FOREGROUND_CRASH_MSG: {
                SomeArgs args = (SomeArgs) msg.obj;
                mServices.serviceForegroundCrash(
                        (ProcessRecord) args.arg1,
                        (String) args.arg2,
                        (ComponentName) args.arg3);
                args.recycle();
            } break;
```

Crash并且符合堆栈抛出的特征。查找调用的地方在 ActiveService 的 bringDownServiceLocked 方法：
```
private void bringDownServiceLocked(ServiceRecord r, boolean enqueueOomAdj) {
           //....
            mAm.mAppOpsService.finishOperation(AppOpsManager.getToken(mAm.mAppOpsService),
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, null);
            mAm.mHandler.removeMessages(
                    ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG, r);
            if (r.app != null) {
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_FOREGROUND_CRASH_MSG);
                SomeArgs args = SomeArgs.obtain();
                args.arg1 = r.app;
                args.arg2 = r.toString();
                args.arg3 = r.getComponentName();

                msg.obj = args;
                mAm.mHandler.sendMessage(msg);
            }
        }

       //...
    }
```

继续分析，在 ActiveService 里共有9处调用这个方法的，排除掉应用启动、杀进程等场景的调用，其中 private String bringUpServiceLocked() 中两处调用，private final void bringDownServiceIfNeededLocked() 中一处调用。

OK，问题简化。进一步分析 private String bringUpServiceLocked() 中两处和进程启动和权限相关，那么可以确定不是这个问题。
问题再简化，针对 private final void bringDownServiceIfNeededLocked() 分析即可。 ActiveService 里共有三处调用的地方，分别是：

```
  private void stopServiceLocked(ServiceRecord service, boolean enqueueOomAdj) {
        //...

        bringDownServiceIfNeededLocked(service, false, false, enqueueOomAdj); //这里
    }
    
        boolean stopServiceTokenLocked(ComponentName className, IBinder token,
            int startId) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "stopServiceToken: " + className
                + " " + token + " startId=" + startId);
        ServiceRecord r = findServiceLocked(className, token, UserHandle.getCallingUserId());
        if (r != null) {
            //...
            r.callStart = false;
            final long origId = Binder.clearCallingIdentity();
            bringDownServiceIfNeededLocked(r, false, false, false); //这里
            Binder.restoreCallingIdentity(origId);
            return true;
        }
        return false;
    }
    
     void removeConnectionLocked(ConnectionRecord c, ProcessRecord skipApp,
            ActivityServiceConnectionsHolder skipAct, boolean enqueueOomAdj) {
                //...
                bringDownServiceIfNeededLocked(s, true, hasAutoCreate, enqueueOomAdj); //这里
            }
        }
    }
    
```

最后一个是connection相关的，由于这个Service并没有Binder connection排除。再此分析下来竟然和stopService有关系。啊！这...

先来把代码里stopService的代码注掉，跑一遍Monkey，果然没事了。

那么究竟为什么stopService会导致这个Crash？让我们先追踪一下stopService的源码。先看 ActivityManagerService 的 stopService()

```
   @Override
    public int stopService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) {
        enforceNotIsolatedCaller("stopService");
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            return mServices.stopServiceLocked(caller, service, resolvedType, userId);
        }
    }
```

再追踪 mServices.stopServiceLocked() 到 ActiveServices的 int stopServiceLocked(IApplicationThread caller, Intent service,String resolvedType, int userId) 方法：

```
    int stopServiceLocked(IApplicationThread caller, Intent service,
            String resolvedType, int userId) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "stopService: " + service
                + " type=" + resolvedType);

        final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
        if (caller != null && callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when stopping service " + service);
        }

        // If this service is active, make sure it is stopped.
        ServiceLookupResult r = retrieveServiceLocked(service, null, resolvedType, null,
                Binder.getCallingPid(), Binder.getCallingUid(), userId, false, false, false, false);
        if (r != null) {
            if (r.record != null) {
                final long origId = Binder.clearCallingIdentity();
                try {
                    stopServiceLocked(r.record, false);//这里
                } finally {
                    Binder.restoreCallingIdentity(origId);
                }
                return 1;
            }
            return -1;
        }

        return 0;
    }
```

到这里，就可以确定和上面的 private void stopServiceLocked(ServiceRecord service, boolean enqueueOomAdj) 对上了。细看这里面的逻辑:

```
        // Check to see if the service had been started as foreground, but being
        // brought down before actually showing a notification.  That is not allowed.
        if (r.fgRequired) {
            Slog.w(TAG_SERVICE, "Bringing down service while still waiting for start foreground: "
                    + r);
            r.fgRequired = false;
            r.fgWaiting = false;
            synchronized (mAm.mProcessStats.mLock) {
                ServiceState stracker = r.getTracker();
                if (stracker != null) {
                    stracker.setForeground(false, mAm.mProcessStats.getMemFactorLocked(),
                            SystemClock.uptimeMillis());
                }
            }
            mAm.mAppOpsService.finishOperation(AppOpsManager.getToken(mAm.mAppOpsService),
                    AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, null);
            mAm.mHandler.removeMessages(
                    ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG, r);
            if (r.app != null) {
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_FOREGROUND_CRASH_MSG);
                SomeArgs args = SomeArgs.obtain();
                args.arg1 = r.app;
                args.arg2 = r.toString();
                args.arg3 = r.getComponentName();

                msg.obj = args;
                mAm.mHandler.sendMessage(msg);
            }
        }
```

如果r.fgRequired 是true，那么就会出现这个crash。 所以 r.fgRequired 是什么逻辑？在此之前，先看一下 Service 的 startForeground() 到底干了啥？老规矩，看代码。通过ActivityManagerService最终跳到我们的老朋友 ActiveService 中：

```
    @GuardedBy("mAm")
    public void setServiceForegroundLocked(ComponentName className, IBinder token,
            int id, Notification notification, int flags, int foregroundServiceType) {
        final int userId = UserHandle.getCallingUserId();
        final long origId = Binder.clearCallingIdentity();
        try {
            ServiceRecord r = findServiceLocked(className, token, userId);
            if (r != null) {
                setServiceForegroundInnerLocked(r, id, notification, flags, foregroundServiceType);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

继续 setServiceForegroundInnerLocked：

```
@GuardedBy("mAm")
    private void setServiceForegroundInnerLocked(final ServiceRecord r, int id,
            Notification notification, int flags, int foregroundServiceType) {
//...
      if (r.fgRequired) {
                if (DEBUG_SERVICE || DEBUG_BACKGROUND_CHECK) {
                    Slog.i(TAG, "Service called startForeground() as required: " + r);
                }
                r.fgRequired = false;
                r.fgWaiting = false;
                alreadyStartedOp = stopProcStatsOp = true;
                mAm.mHandler.removeMessages(
                        ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG, r);
            }
//...
}
```

也就是 r.fgRequired = false; 这个代码没有执行，进而导致后面的Crash。可是为什么Service的onCreate()方法没有执行呢？这和Service的stop又有什么关系？难道说，我们的Servcie还没有执行onCreate方法就被stop了？思来想去也就只有这一个选项了！

由于Servcie调用onCreate方法是跨进程调用的，确实存在这种可能。写个代码验证下，在startForegroundService后立刻调用stopService，果然预期中的crash出现了：

```
android.app.RemoteServiceException$ForegroundServiceDidNotStartInTimeException: Context.startForegroundService() did not then call Service.startForeground(): ServiceRecord{1e9c4ea u0 com.example.test/.MyService}
    at android.app.ActivityThread.generateForegroundServiceDidNotStartInTimeException(ActivityThread.java:2042)
    at android.app.ActivityThread.throwRemoteServiceException(ActivityThread.java:2013)
    at android.app.ActivityThread.-$$Nest$mthrowRemoteServiceException(Unknown Source:0)
    at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2282)
    at android.os.Handler.dispatchMessage(Handler.java:106)
    at android.os.Looper.loopOnce(Looper.java:201)
    at android.os.Looper.loop(Looper.java:288)
    at android.app.ActivityThread.main(ActivityThread.java:8049)
    at java.lang.reflect.Method.invoke(Native Method)
    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:942)
```

对应代码:

```
 startForegroundService(Intent(this@MainActivity,MyService::class.java))
 stopService(Intent(this@MainActivity,MyService::class.java))
```

为什么会出现先后调用这两个方法呢？原来App做了保活，应用退到后台前会启动一个前台Service，防止应用在后台被kill。这个前后台监听是通过Application的ActivityLifecycleCallback实现的。也就是说可见页面小于0，就会startForegroundService，大于0就会stopService。问题是应用换皮肤的方案就是通过重建应用完成的，那么在系统切换黑/白模式的时候，应用会重建，相当于经历了一次快速前后台切换，导致Crash。

那么如何解决呢？要通过Service的stopSelf()进行Service销毁。具体代码：

```
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_UPDATE_INFO -> {
              
            }

            ACTION_STOP_SERVICE -> {
                stopSelf()
            }
        }

        return START_NOT_STICKY
    }
```

调用的地方：

```
val intent = Intent(this, BackgroundKeepService::class.java)
intent.action = BackgroundKeepService.ACTION_STOP_SERVICE
startService(intent)
```

至此，完成了Crash的分析、复现和验证。线上也没有相关Crash。

总结：这是AOSP对startForeground的一种限制，如果业务逻辑无法绕开短时间内启动暂停Service，上文提到的解决方案可以规避Crash问题。

转载自：https://www.jianshu.com/p/d974a7ff6cf8