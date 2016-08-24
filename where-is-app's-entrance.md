# Android应用的程序入口是哪里？

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
        // 因为是 flase，所以省略了这里的代码
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die. 哈哈 这么想 die
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
        	// 实例化 Instrumentation
            mInstrumentation = new Instrumentation();
            // 创建 contextImpl
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            // 在这里创建 Application 并且调用它的 onCreate
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
        	// 有没有在重复多次点击 run 跑 app 的时候遇到过这个错？
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }

    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());

    ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
        @Override
        public void onConfigurationChanged(Configuration newConfig) {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources
                // immediately, because upon returning the view
                // hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                    // This actually changed the resources!  Tell
                    // everyone about it.
                    if (mPendingConfiguration == null ||
                            mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                        mPendingConfiguration = newConfig;

                        sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                    }
                }
            }
        }
        @Override
        public void onLowMemory() {
        }
        @Override
        public void onTrimMemory(int level) {
        }
    });
}
```

暂时先不管那些个看不懂的方法，挑我们看得懂的。    

ActivityThread 的 main 方法中，主要做了以下步骤(挑重点)：

- 为主线程准备 Looper 
- 实例化 ActivityThread 并调用它的 attach 方法
- 在 attach 方法中，又做了如下几件事
	- 创建 Instrumentation 实例
	- 创建 ContextImpl
	- 创建 Application，并调用它的 onCreate
- 把主线程的Handler `sMainThreadHandler` 赋值为 ActivityThread.mH 
- Looper.loop();


可以看到上面所说的步骤都非常重要，主线程Looper的创建、ContextImpl的创建、Application的创建以及onCreate 回调，这些都跟 App 息息相关。  

也解答了我们之前的疑问。

## 小结

**对于一个 App 来说其实 ActivityThread.main 才是真正的入口。**(Java层面)  

**Application 的创建以及 onCreate 都在 ActivityThread.attach 方法中。

BONUS：另外我们还顺带发现了另外一个问题的答案：**主线程的 Looper 是什么时候实例化的？** 

答案显而易见： ActivityThread.main 。

ActivityThread 非常重要，后续还会讲更多关于它的知识。  
