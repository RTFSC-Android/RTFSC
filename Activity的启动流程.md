# Activity的启动流程


妈的 老子不信搞不懂。

Activity的启动流程涉及到 进程的创建 Application的创建 ActivityStack 的创建 Activity 的创建


涉及到的类有:

- Context 通常是我们 startActivity 的第一步，具体的实现类是 ContextImpl。
- ActivityManagerService 
- ActivityThread 
- ActivityThread.H 是一个运行在主线的Handler，负责把 ApplicationThread 从 AMS 拿来的通知转换到主线程，并调用 ActivityThread 相应的方法。  
- ActivityThread.ApplicationThread 是一个实现了 IApplicationThread 的Binder类，是 ActivityThread 与 AMS 交互通信的媒介。AMS 通过它来告知 ActivityThread 该做什么。它最后会把 AMS 的通知都交给 ActivityThread.H 转换成 Message，转到 ActivityThread 去执行。
- Instrumentation  负责创建Activity，并调用 Activity 的生命周期 eg: mInstrumentation.newActivity ,mInstrumentation.callActivityOnCreate
- ActivityStackSupervisor 


//todo 详细的分析


大致内容记录一下 

太复杂了，吃不消。。。。。。


ContextImpl.startActivity/Activity.startActivity
 => ActivityThread.mInstrumentation.execStartActivity
  => ActivityManagerNative.startActivity  (IActivityManagerNative 的代理ActivityManagerProxy) 
   => ActivityManagerProxy.startActivity
    => ActivityManagerService.startActivity
     => ActivityManagerService.startActivityAsUser
      => ActivityStackSupervisor.startActivityMayWait
       => ActivityStackSupervisor.startActivityLocked
        => ActivityStackSupervisor.startActivityUncheckedLocked
         => ActivityStack.startActivityLocked
          => ActivityStackSupervisor.resumeTopActivitiesLocked
           => ActivityStack.resumeTopActivitiesLocked
            => ActivityStack.resumeTopActivityInnerLocked
            .
            .
            . 
            .
            => ActivityStackSupervisor.realStartActivityLocked
             => app.thread.scheduleLaunchActivity
              => ApplicationThreadProxy.scheduleLaunchActivity
              	=> ActivityThread.sendMessage
              	 => ActivityThread.mH.sendMessage & handleMessage
              	  => ActivityThread.handleLaunchActivity
              	   => performLaunchActivity
              	    => handleResumeActivity

差不多大致是这样的流程。。。。

心好累，还是有很多很多细节没有理清楚。

ActivityManagerService 与 ActivityThread 之间用 ApplicationThread 来交互。


ActivityStack或ActivityStackSupervisor 里会调用 ActivityRecod.app.thread (IApplicationThread)的 `scheduleXXXXActivity`比如 `schedulePauseActivity`



[](http://blog.csdn.net/luoshengyang/article/details/6689748)

