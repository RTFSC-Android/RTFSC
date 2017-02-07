# ZygoteInit


	com.android.internal.os.ZygoteInit


Android 准备完 Native 环境后，在启动 zygote 进程过程中调用了 Java 世界的 ZygoteInit 的 main 方法，并且传入`com.anddroid.internal.os.ZygoteInit`和`true`两个参数。





```
at android.os.Handler.handleCallback(Handler.java:733)
at android.os.Handler.dispatchMessage(Handler.java:95)
at android.os.Looper.loop(Looper.java:212)
at android.app.ActivityThread.main(ActivityThread.java:5151)
at java.lang.reflect.Method.invokeNative(Method.java)
at java.lang.reflect.Method.invoke(Method.java:515)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:868)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:684)
at dalvik.system.NativeStart.main(NativeStart.java)
```


```
at android.app.Activity.performCreate(Activity.java:6049)
at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1106)
at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2294)
at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2403)
at android.app.ActivityThread.access$800(ActivityThread.java:154)
at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1317)
at android.os.Handler.dispatchMessage(Handler.java:102)
at android.os.Looper.loop(Looper.java:135)
at android.app.ActivityThread.main(ActivityThread.java:5298)
at java.lang.reflect.Method.invoke(Native Method)
at java.lang.reflect.Method.invoke(Method.java:372)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:910)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:705)
```







