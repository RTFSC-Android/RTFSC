# ActivityThread

在这里记录我对 ActivityThread 的认知。

## 简介

	package android.app;
	public final class ActivityThread 


在[Android应用程序的入口是哪里？](./where-is-app's-entrance.md)中已经有过简单介绍。


ActivityThread 在很多文章中常被称为 主线程。

这个说法并不准确，因为其实它并不是一个线程，只是主线程调用了AactivityThread的main方法，所以我觉得把 ActivityThread 称作『主线程的入口』会更加合适一些。

	PS: ActivityThread的main方法是在 ZygoteInit.invokeStaticMain 中通过反射调用。



## ActivityThread.main 分析

来看一下`main`方法：  

```
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    SamplingProfilerIntegration.start();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
    Environment.initForCurrentUser();
    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());
    AndroidKeyStoreProvider.install();
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");
    // 开始看得懂了对不对？  
    // 准备主线程的 Looper
    Looper.prepareMainLooper();
    // 实例化 ActivityThread 并调用 attach 并传入了 false 
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    // 主线程的 Handler
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    // 开始循环
    Looper.loop();
    
    throw new RuntimeException("Main thread loop unexpectedly exited");
}

// 从 main 方法过来的是 false PS：另外一个方法 systemMain()中会传入 true
private void attach(boolean system) {
    
    sCurrentActivityThread = this;
    mSystemThread = system;

    if (!system) {
        // false 走这里
        // ... 
        // 重要的在这里 
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            // Ignore
        }
        //...BinderInternal.addGcWatcher(new Runnable() {});
            
    } else {
        //... 不走这里 省略
    }

    //... ViewRootImpl.addConfigCallback  onConfigurationChanged

}
```

暂时先不管那些个看不懂的方法，挑我们看得懂的。    

ActivityThread 的 main 方法中，主要做了以下步骤(挑了重点)：

- 1.为主线程准备了 Looper 
- 2.实例化 ActivityThread 并调用它的 attach 方法
- 3.在 attach 方法中，又做了如下几件事
	- 1.获取 IActivityManager mgr 
	- 2.调用 mgr.attachApplication (涉及到 AMS与 IApplicationThread 的交互)
- 4. 把主线程的Handler `sMainThreadHandler` 赋值为 ActivityThread.mH 
- 5. Looper.loop(); 循环消息


从以上步骤来看，ActivityThread 对 App 的重要程度可见一斑。

再看 ActivityThread 的方法，有一系列的`performXXXActivity`和`handleXXXActivity`。

其实在 Activity 的启动流程中，ActivityThread的IApplicationThread 与 AMS 的交互最后都会走到 ActivityThread 来。

比如启动一个 Activity，会调用到 ActivityThread.performLaunchActivity 方法，已这个方法为例。

## performLaunchActivity 分析


```
//ActivityThread
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
    //...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        // 创建 activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        //...
    }

    try {
        // 创建 App 方法的实现看后面 是个 单例实现
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        //...
        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            //...
            // 调用 activity.attach
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor);

            //...
            //...调用Activity 的生命周期等  mInstrumentation.callActivityOnCreate
            
        }
        r.paused = true;

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
       //...
    }
    return activity;
}

// LoadedApk.makeApplication
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    //保证了 Application 单例
    if (mApplication != null) {
        return mApplication;
    }

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            initializeJavaContextClassLoader();
        }
        // 创建 ContextImpl
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 创建我们的 Application
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        //...
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            // 调用 Application.OnCreate
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            //...
        }
    }
    //....  Rewrite the R 'constants' for all library apks.
    // Rewrite the R 'constants' for all library apks.

    return app;
}

// Instrumentation
public void callApplicationOnCreate(Application app) {
    app.onCreate();
}
```

代码稍微有点多，删减了大部分暂时不关心的代码。

可以看到

Instrumentation.newApplication 创建了 Application，Instrumentation.callApplicationOnCreate 里调用了 Application 的 onCreate。

看到上面所说的步骤都非常重要，主线程Looper的创建、ContextImpl的创建、Application的创建以及onCreate 回调，这些都跟 App 息息相关。  


## 小结

ActivityThread
Instrumentation
LoadedApk
IApplicationThread
