# Activity.setContentView 分析

todo 尚未完成

## 引言

通常，我们在使用 Activity 的时候，都会在 onCreate 里调用setContentView这个方法，这样才能让我们的 UI 显示出来。 

那么问题来了 setContentView 里做了什么呢？为什么这个方法叫 setContentView 呢？

## 探索 Activity.setContentView

看一下setContentView的实现(它有三个重载，代码差不多，就只列举一个最常用的):

```
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

public Window getWindow() {
    return mWindow;
}
```

首先调用了 window 的 setContentView，接着调用 initWindowDecorActionBar，它是用来创建 ActionBar 的，这里就不多关心了。

接下去的关键是 mWindow 。

mWindow 是什么，它又从何而来？

这要看一下 Activity.attach 方法了：

	NOTE：至于为什么是 attach，这涉及到 Activity 的启动过程，以后会讲，现在只要知道就好。

```
//省略了很多参数
//Activity
final void attach(Context,ActivityThread,Instrumentation...){
	mWindow = new PhoneWindow(this);
	mWindow.setCallback(this);
	mWindow.getLayoutInflater().setPrivateFactory(this);
}
```

偶，原来它是个 `com.android.internal.policy.PhoneWindow` 的实例。

## 揭秘 PhoneWindow.setContentView

看一下setContentView的实现。

```
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    // mContentParent 是 contentView 的父容器
    if (mContentParent == null) {
    	//第一次走这里肯定为 null 啦。
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
    	//第二次就先把所有 View 都移除
        mContentParent.removeAllViews();
    }
    // 处理 TRANSITIONS
    // 咱们按常规路走 
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
    	// 填充 xml布局到mContentParent
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    // 处理 Insets
    mContentParent.requestApplyInsets();
    // 回调 Activity.onContentChanged
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

步骤:

1. installDecor()
2. 将布局填充到 mContentParent
3. 回调 Activity.onContentChanged

	PS：关于 LayoutInflater 如果你不懂它的原理，可以看看[LayoutInflater源码分析系列](./LayoutInflater.md)  

接下去需要看 installDecor 方法。

问自己一下，这个 Decor 是什么？

### installDecor 解析

```
// 在installDecor中调用，为方便查看，挪前面了。
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}

private void installDecor() {
	// private DecorView mDecor;
    if (mDecor == null) {
    	// 实例化一个 DecorView ，哦，原来是个 View 啊~
        mDecor = generateDecor();
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    // 为 null 才走，也就是说只会执行一遍~~ 
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);

        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeOptionalFitsSystemWindows();

        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            // 省略大段代码
        } else {
        	// 标题
            mTitleView = (TextView)findViewById(R.id.title);
            if (mTitleView != null) {
                mTitleView.setLayoutDirection(mDecor.getLayoutDirection());
                // 如果 NO_TITLE 则隐藏
                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                    View titleContainer = findViewById(
                            R.id.title_container);
                    if (titleContainer != null) {
                        titleContainer.setVisibility(View.GONE);
                    } else {
                        mTitleView.setVisibility(View.GONE);
                    }
                    if (mContentParent instanceof FrameLayout) {
                        ((FrameLayout)mContentParent).setForeground(null);
                    }
                } else {
                    mTitleView.setText(mTitle);
                }
            }
        }
        //处理背景
        if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
            mDecor.setBackgroundFallback(mBackgroundFallbackResource);
        }

        // Only inflate or create a new TransitionManager if the caller hasn't
        // already set a custom one.
        if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
            //处理 Transition 这里不关心，就省略代码了。
        }
    }
}

这边可以看到 Decor 其实是 DecorView，瞥一眼它的定义：

```
// PhoneWindow$DecorView
// top-level view of the window, containing the window decor
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker
```

它是一个 Window 最最最顶层的 View。至于它为什么叫做`Decor`View，后面解释。  

接下去看看 DecorView 的生成。


### generateLayout 方法解析

generateLayout 方法的实现：

```
// 该方法有 317行，不把重要的东西筛选出来看不懂 
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.

    TypedArray a = getWindowStyle();
    // 处理各种 FEATURE Window 属性，如 FEATURE_NO_TITLE   Floating statusBarColor 啊 CloseOnTouchOutside啊 各种
    mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
    // ...
    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
        // Don't allow an action bar if there is no title.
        requestFeature(FEATURE_ACTION_BAR);
    }
    //...
    if (!mForcedStatusBarColor) {
        mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
    }
    // ...
    WindowManager.LayoutParams params = getAttributes();
    // 处理各种属性 
    //.....

    // The rest are only done if this window is not embedded; otherwise,
    // the values are inherited from our container.
    // 如果没有父 Window 那么获取一些属性 比如windowBackground
    // 想想 PopupWindow 这些需要依附到 Activity 的
    if (getContainer() == null) {
        //... 获取一些属性
    }
    // Inflate the window decor.
    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    //... 根据 features 赋值 layoutResource 
    // 取值有一些如 R.layout.screen_simple R.layout.screen_action_bar
    // ...
    layoutResource = R.layout.XXXXXX;
    // ...

    mDecor.startChanging();
    // 把 layoutResource 实例化成 View
    View in = mLayoutInflater.inflate(layoutResource, null);
    // 把 in 添加到 decor 
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in;
    // 提示一下 ID_ANDROID_CONTENT = com.android.internal.R.id.content
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }

    //...

    // Remaining setup -- of background and title -- that only applies
    // to top-level windows.
    // 去设置 背景和 title
    if (getContainer() == null) {
        final Drawable background;
        if (mBackgroundResource != 0) {
            background = getContext().getDrawable(mBackgroundResource);
        } else {
            background = mBackgroundDrawable;
        }
        // 设置背景
        mDecor.setWindowBackground(background);

        final Drawable frame;
        if (mFrameResource != 0) {
            frame = getContext().getDrawable(mFrameResource);
        } else {
            frame = null;
        }
        mDecor.setWindowFrame(frame);
        mDecor.setElevation(mElevation);
        mDecor.setClipToOutline(mClipToOutline);
        if (mTitle != null) {
            setTitle(mTitle);
        }
        if (mTitleColor == 0) {
            mTitleColor = mTextColor;
        }
        setTitleColor(mTitleColor);
    }

    mDecor.finishChanging();
    // 返回了 R.id.content 的那个 View
    return contentParent;
}




另外提一下，我们知道 requestFeature 需要在 setContentView之前调用就是因为在setContentView里把mContentParent实例化了。

