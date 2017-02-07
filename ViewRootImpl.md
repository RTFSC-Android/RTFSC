# ViewRootImpl

## 简介
	/**
	 * The top of a view hierarchy, implementing the needed protocol between View
	 * and the WindowManager.  This is for the most part an internal implementation
	 * detail of {@link WindowManagerGlobal}.
	 */
	public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks



## ViewRootImpl 的创建时机


//todo 详细的分析


大致流程记录如下

ActivityThread.performLaunchActivity
 => activity.attach
  => ActivityThread.handleResumeActivity
   => WindowManager.addView
    => WindowManagerImpl.addView
     => WindowManagerGlobal.addView
      => root = new ViewRootImpl





