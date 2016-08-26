# Android应用的程序入口是哪里？

## 引言 

一般我们都会认为Application的 onCreate 方法就是入口了，毕竟它算是第一个回调，我们通常在那做初始化。

不过 Application 的 onCreate 真的是程序的入口吗？它是什么时候调用被谁调用的呢？  

我们知道 Android 基于 Java，而 Java 的入口其实是一个 `main` 方法，那么App的 `main` 方法在哪里呢？  

所以，显然，其实入口并不是Application.onCreate。

那么又是哪里呢？  

这里要涉及到一个类`ActivityThread`，它拥有这个`main`方法。  

让我们一探究竟。 

## ActivityThread的main方法

ActivityThread 位于`android.app`包下。

来看一下`main`方法：  

```
// android.app.ActivityThread
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

ActivityThread 的 main 方法中，主要做了以下步骤(挑重点)：

- 1.为主线程准备 Looper 
- 2.实例化 ActivityThread 并调用它的 attach 方法
- 3.在 attach 方法中，又做了如下几件事
	- 1.获取 IActivityManager mgr
	- 2.调用 mgr.attachApplication
- 4. 把主线程的Handler `sMainThreadHandler` 赋值为 ActivityThread.mH 
- 5. Looper.loop(); 循环消息


在3.2步骤中进行了非常复杂的操作，各种交互，这里不展开，最后在启动我们的启动页的时候，最终会调用到 ActivityThread.performLaunchActivity 方法。


```
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
        // 创建 App 方法的实现看后面
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
            // 调用 ApplicationOnCreate
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

也解答了我们之前的疑问。

## 小结

**对于一个 App 来说其实 ActivityThread.main 才是真正的入口。**(Java层面)  

**Application 的创建以及 onCreate 的回调，都由 Instrumentation掌控。**(ActivityThread.LoadedApk.makeApplication 方法中)

BONUS：另外我们还顺带发现了另外一个问题的答案：**主线程的 Looper 是什么时候实例化的？** 

答案显而易见： ActivityThread.main 。

ActivityThread 非常重要，它更是我们常说的`mainThread`！后续还会讲更多关于它的知识。  
