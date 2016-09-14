# ViewRootImpl的创建时机


//todo 详细的分析


大致流程记录如下

ActivityThread.performLaunchActivity
 => activity.attach
  => ActivityThread.handleResumeActivity
   => WindowManager.addView
    => WindowManagerImpl.addView
     => WindowManagerGlobal.addView
      => root = new ViewRootImpl





