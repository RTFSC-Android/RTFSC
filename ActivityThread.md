# ActivityThread

This manages the execution of the main thread in an
application process, scheduling and executing activities,
broadcasts, and other operations on it as the activity
manager requests.


简单释义：ActivityThread 运行在 App 进程的主线程上，负责安排并执行 ActivityManagerService 对activities broadcasts 的操作请求。


显然ActivityThread是一个AMS的命令执行者。很多非常重要的操作比如 Activity 的创建，Activity 的生命周期，都在ActivityThread中执行。

ActivityThread 与 AMS 的通信是跨进程的，通过 ApplicationThread 这个 Binder 来通信的。


## ActivityThread 究竟是不是主线程？

ActivityThread 在很多文章中常被称为 主线程。

这个说法并不准确，因为其实它本身并不是一个线程，只是主线程调用了AactivityThread的main方法，所以我觉得把 ActivityThread 称作『主线程的入口』会更加合适一些。 

不过在 Activity的源码中有一个ActivityThread类型的成员：`/*package*/ ActivityThread mMainThread;` ，这个变量名表示了 ActivityThread 是`mainThread`。

细细想来主线程对于 App开发者来说是隐藏的，所以拿 ActivityThread 来表示 主线程倒也无可厚非。

所以我觉得虽然 ActivityThread本身并不是一个线程，但是用它来表示主线程倒也没什么，自己心里清楚即可。


