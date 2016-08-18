# LayoutInflater源码分析（三）之 fragment 标签的处理


[LayoutInflater 源码分析（一）之 inflate 深度分析](./LayoutInflater.md)  
[LayoutInflater 源码分析（二）之 include 以及 merge 标签的处理](./LayoutInflater-2.md)  
[LayoutInflater 源码分析（三）之 fragment 标签的处理](LayoutInflater-3.md)  
[LayoutInflater 源码分析（四）之 闪耀的彩蛋](./BlinkLayout.md)

## 前言

上一篇[LayoutInflater 源码分析（二）](./LayoutInflater-2.md)中分析了`LayoutInflater`对`include`以及`merge`标签的处理，但是并没有找到对`fragment`的处理痕迹。

本文将继续探索以求揭晓答案。

提一下`fragment`标签的使用方式： 

```
<fragment
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    class="me.yifeiyuan.MainFragment"
    android:tag="Main"
    android:id="@+id/main"
    />
```

## 分析

在第一篇分析中提到了 Factory 的 Hook 机制，代码如下:

```
View view;
if (mFactory2 != null) {
    view = mFactory2.onCreateView(parent, name, context, attrs);
} else if (mFactory != null) {
    view = mFactory.onCreateView(name, context, attrs);
} else {
    view = null;
}
// mPrivateFactory 是个 FactoryMerger 
if (view == null && mPrivateFactory != null) {
    view = mPrivateFactory.onCreateView(parent, name, context, attrs);
}

```

大致的画了一张类图

![Factory](http://ww4.sinaimg.cn/large/98900c07jw1f6wndhrquvj21560lsad0.jpg)

前面代码中的mPrivateFactory 是个 FactoryMerger 对象。


回想我们在使用 Fragment 的时候都会继承 FragmentActivity，所以去 FragmentActivity 寻找答案比较靠谱。

接下去开始分析。

## 寻找踪迹 

查看了源码后发现，FragmentActivity的继承结构如下 

![FragmentActivity](http://ww3.sinaimg.cn/large/98900c07jw1f6wnvk60gkj20fq0r0wfs.jpg)

其实 Activity 就已经实现了 LayoutInflater.Factory2 接口，具体实现如下：

```
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    // 如果不是 fragment 则调用另一个onCreateView，而它返回的是 null
    if (!"fragment".equals(name)) {
        return onCreateView(name, context, attrs);
    }
    // 如果是 fragment 标签 则交给了 mFragments
    return mFragments.onCreateView(parent, name, context, attrs);
}

public View onCreateView(String name, Context context, AttributeSet attrs) {
    return null;
}
```

可以看到如果是 fragment 标签，则会交给一个叫 mFragments 的FragmentController类的对象。

但是需要注意的是Activity 没有调用 LayoutInflater.setFactory 所以 Activity 并不支持 fragment 的解析。

继续它的子类的实现。

```
abstract class BaseFragmentActivityDonut extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        if (Build.VERSION.SDK_INT < 11 && getLayoutInflater().getFactory() == null) {
            // On pre-HC devices we need to manually install ourselves as a Factory.
            // On HC and above, we are automatically installed as a private factory
            getLayoutInflater().setFactory(this);
        }
        super.onCreate(savedInstanceState);
    }
    //重写 onCreateView
    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
    	//　优先调用 dispatchFragmentsOnCreateView
        final View v = dispatchFragmentsOnCreateView(null, name, context, attrs);
        if (v == null) {
            return super.onCreateView(name, context, attrs);
        }
        return v;
    }

    abstract View dispatchFragmentsOnCreateView(View parent, String name,
            Context context, AttributeSet attrs);

}

abstract class BaseFragmentActivityHoneycomb extends BaseFragmentActivityDonut {
    @Override
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        final View v = dispatchFragmentsOnCreateView(parent, name, context, attrs);
        if (v == null && Build.VERSION.SDK_INT >= 11) {
            // If we're running on HC or above, let the super have a go
            return super.onCreateView(parent, name, context, attrs);
        }
        return v;
    }

}
// FragmentActivity 并没有重写 onCreateView
public class FragmentActivity extends BaseFragmentActivityHoneycomb{
    @Override
    final View dispatchFragmentsOnCreateView(View parent, String name, Context context,
            AttributeSet attrs) {
        return mFragments.onCreateView(parent, name, context, attrs);
    }
}
```

可以看到，在`BaseFragmentActivityDonut`中调用了`setFactory`，并定义了一个`dispatchFragmentsOnCreateView`抽象方法，在`onCreateView`里调用了它。

接着，在 FragmentActivity 重写`dispatchFragmentsOnCreateView`，又把它交给了`mFragments`，咦，又回去了。

其实到这里已经可以知道，`LayoutInflater`把处理`fragment`的事情交给了`FragmentManagerImpl`。

而对于`FragmentManagerImpl`的分析其实已经超过了本文的界限。  

但是我想再深入，看看`Fragment`的`onCreateView`方法究竟是什么时候调用的！！

所以继续跟下去：

```
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    //mFragmentManager 是 FragmentManagerImpl 的实例
    return mHost.mFragmentManager.onCreateView(parent, name, context, attrs);
}

// #FragmentManagerImpl 
@Override
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    // 不为 fragment 直接 return null 说明确实只处理 fragment
    if (!"fragment".equals(name)) {
        return null;
    }
    // xml 中的 各种 属性 fname为 Fragemnt 的全路径
    String fname = attrs.getAttributeValue(null, "class");
    TypedArray a =  context.obtainStyledAttributes(attrs, FragmentTag.Fragment);
    if (fname == null) {
        fname = a.getString(FragmentTag.Fragment_name);
    }
    int id = a.getResourceId(FragmentTag.Fragment_id, View.NO_ID);
    String tag = a.getString(FragmentTag.Fragment_tag);
    a.recycle();

    if (!Fragment.isSupportFragmentClass(mHost.getContext(), fname)) {
        // Invalid support lib fragment; let the device's framework handle it.
        // This will allow android.app.Fragments to do the right thing.
        return null;
    }
    // 如果配置的信息不够 则会抛异常
    int containerId = parent != null ? parent.getId() : 0;
    if (containerId == View.NO_ID && id == View.NO_ID && tag == null) {
        throw new IllegalArgumentException(attrs.getPositionDescription()
                + ": Must specify unique android:id, android:tag, or have a parent with an id for " + fname);
    }

    // If we restored from a previous state, we may already have
    // instantiated this fragment from the state and should use
    // that instance instead of making a new one.
    // 尝试着先去找 fragment
    Fragment fragment = id != View.NO_ID ? findFragmentById(id) : null;
    if (fragment == null && tag != null) {
        fragment = findFragmentByTag(tag);
    }
    if (fragment == null && containerId != View.NO_ID) {
        fragment = findFragmentById(containerId);
    }

    if (FragmentManagerImpl.DEBUG) Log.v(TAG, "onCreateView: id=0x"
            + Integer.toHexString(id) + " fname=" + fname
            + " existing=" + fragment);
    // 如果没有 则去实例化
    if (fragment == null) {
    	// 调用 instantiate 其实也是反射来实例化的。
        fragment = Fragment.instantiate(context, fname);
        fragment.mFromLayout = true;
        fragment.mFragmentId = id != 0 ? id : containerId;
        fragment.mContainerId = containerId;
        fragment.mTag = tag;
        fragment.mInLayout = true;
        fragment.mFragmentManager = this;
        fragment.mHost = mHost;
        fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
        addFragment(fragment, true);

    } else if (fragment.mInLayout) {
        // A fragment already exists and it is not one we restored from
        // previous state.
        throw new IllegalArgumentException(attrs.getPositionDescription()
                + ": Duplicate id 0x" + Integer.toHexString(id)
                + ", tag " + tag + ", or parent id 0x" + Integer.toHexString(containerId)
                + " with another fragment for " + fname);
    } else {
        // This fragment was retained from a previous instance; get it
        // going now.
        fragment.mInLayout = true;
        fragment.mHost = mHost;
        // If this fragment is newly instantiated (either right now, or
        // from last saved state), then give it the attributes to
        // initialize itself.
        if (!fragment.mRetaining) {
            fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
        }
    }

    // If we haven't finished entering the CREATED state ourselves yet,
    // push the inflated child fragment along.
    if (mCurState < Fragment.CREATED && fragment.mFromLayout) {
        moveToState(fragment, Fragment.CREATED, 0, 0, false);
    } else {
        moveToState(fragment);
    }

    if (fragment.mView == null) {
        throw new IllegalStateException("Fragment " + fname
                + " did not create a view.");
    }
    if (id != 0) {
        fragment.mView.setId(id);
    }
    if (fragment.mView.getTag() == null) {
        fragment.mView.setTag(tag);
    }
    return fragment.mView;
}
```

可以看到，在处理 xml 中的属性后，会先去寻找要加载的 fragment 是否已经加载过了，如果没有则会调用`fragment = Fragment.instantiate(context, fname);`，这个方法也是
反射，这点其实跟 View 的处理是一样的。  

接着会去调用`moveToState`方法，而这个方法里，我看了`onCreateView`方法的调用时机。

伪代码如下（太复杂，删减了绝大部分代码）：

```
// # FragmentManager
void moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive) {
	//...
	f.mView = f.performCreateView(xxx);
	//....
}
// 真正的调用时机在这里！！
// # Fragment
View performCreateView(LayoutInflater inflater, ViewGroup container,
        Bundle savedInstanceState) {
    if (mChildFragmentManager != null) {
        mChildFragmentManager.noteStateNotSaved();
    }
    // 调用onCreateView
    return onCreateView(inflater, container, savedInstanceState);
}
```

可以看到，`FragmentManager`的`moveToState`中会去调用`Fragment`的`performCreateView`方法，而它里面，调用了`onCreateView`！！

终于找到啦！！  

呼，藏得真深。  

不过功夫不负有心人！~~~  

爽！~  

## 小结

`FragmentActivity`通过 `setFactory`把对`fragment`标签的处理委托给了 `FragmentManageImpl`的`onCreateView`方法。

最终通过反射，实例化指定的 `Fragment`，并最终调用了`Fragment.performCreateView`。  

另外要说的是，`LayoutInflater.Factory`的作用其实非常强大，还可以实现夜间模式等功能，Support 包的主题切换原理就是这了，有机会再讲吧。  

