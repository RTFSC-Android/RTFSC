# 心得

看源码的心得。



## 命名


XXXXXNative ,都是 Binder 对象，它们相当于AIDL默认生成的代码中的 Stub 类：

比如 ActivityManagerNative ,ApplicationThreadNative


XXXRecord 类通常是保存 XXX的信息的：

比如 ProcessRecord 是用来保存进程信息的；

ActivityRecord 是用来保存 Activity 信息的。
