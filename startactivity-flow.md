# Activity的启动流程


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

