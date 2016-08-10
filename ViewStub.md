# ViewStub 源码分析

	public final class ViewStub extends View 


简介：ViewStub 是一个宽高为0，不参与measure与layout，不绘制任何东西，可以用来做懒加载的View；常用于布局优化；  

本文涉及到两个角色，一个是 ViewStub本身，另外一个是用来做懒加载的View，是ViewStub的作用对象，称之为`StubbedView`（本文用此称呼来替代）。  


## `ViewStub`的简单使用教程 

`ViewStub` 的使用非常非常简单，只需要两步~  

Step 1. 在XML里配置使用：

```xml
<ViewStub
	android:id="@+id/stub"  // 这个id是ViewStub的id
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout="@layout/mySubTree" //StubbedView的资源id（跟include一样）
    android:visibility="gone"
    android:inflatedId="@+id/subTree" // StubbedView的id
    />
```

Step 2. 调用ViewStub的`inflate`

```
ViewStub stub = (ViewStub)findViewById(R.id.stub);
View stubbedView = stub.inflate();
//...初始化StubbedView
```

非常简单的两步，就能做到View的懒加载，非常方便，其原因是什么呢？  

接下去深入源码分析一下。  

## 构造方法分析

首先分析一下构造方法，了解一下它是如何创建的。  

```java
public ViewStub(Context context, @LayoutRes int layoutResource) {
    this(context, null);
    // StubbedView的资源id
    mLayoutResource = layoutResource;
}

public ViewStub(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
}

public ViewStub(Context context, AttributeSet attrs, int defStyleAttr) {
    this(context, attrs, defStyleAttr, 0);
}

public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context);

    final TypedArray a = context.obtainStyledAttributes(attrs,
            R.styleable.ViewStub, defStyleAttr, defStyleRes);
    // mInflatedId 存储StubbedView的id
    mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
    // mLayoutResource 为StubbedView的resourceId
    mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
    // viewStub 自己的id
    mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
    a.recycle();
    // 设置为不可见 
    setVisibility(GONE);
    // 不绘制本身
    setWillNotDraw(true);
}
```

`ViewStub`在构造方法里不仅仅获取赋值属性，比较关键的是，还 ** 设置为不可见（跳过onMeasure与onLayout），不绘制**。

这里有一个要点：**在XML里配置ViewStub的可见性是没有用的**。    


## 测量 与 绘制

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	// 写死的宽高为0
    setMeasuredDimension(0, 0);
}

@Override
public void draw(Canvas canvas) {
	//空方法，不draw任何东西
}

@Override
protected void dispatchDraw(Canvas canvas) {
	//空方法，不draw任何东西
}
```

## inflate 分析

之前在简单教程里有提到 `inflate`方法，它是`ViewStub`实现懒加载的最为关键的方法，接下去去分析一下。  


```
// 返回 StubbedView
public View inflate() {
	// 尝试去获取 viewParent 第一次调用的时候不为null，而后则为null
    final ViewParent viewParent = getParent();
    // 当 viewParent 不为null的时候
    if (viewParent != null && viewParent instanceof ViewGroup) {
    	// 我们在xml里配置的layout的资源id 如果id无效，则会报错
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
            // 实例化 LayoutInflater
            final LayoutInflater factory;
            if (mInflater != null) {
                factory = mInflater;
            } else {
                factory = LayoutInflater.from(mContext);
            }
            // inflate，StubbedView在这里被实例化
            final View view = factory.inflate(mLayoutResource, parent,
                    false);
            // 可以看到，这里如果我们在XML里写了inflateId，则会设置给StubbedView
            if (mInflatedId != NO_ID) {
                view.setId(mInflatedId);
            }
            // 注意：这两步步 ViewSutb 找到自己的位置，并从父View中移除了自己
            // 这会导致 以后调用inflate的时候 再也获取不到 viewParent了
            final int index = parent.indexOfChild(this);
            parent.removeViewInLayout(this);
            // 拿出ViewStub的LayoutParamas，不为null 则会赋值给 StubbedView
            final ViewGroup.LayoutParams layoutParams = getLayoutParams();
            // 把 StubbedView 添加到ViewStub的父View里
            if (layoutParams != null) {
                parent.addView(view, index, layoutParams);
            } else {
                parent.addView(view, index);
            }
            //使用一个弱引用来保存StubbedView
            mInflatedViewRef = new WeakReference<View>(view);
            //回调listener
            if (mInflateListener != null) {
                mInflateListener.onInflate(this, view);
            }
            // 返回 StubbedView
            return view;
        } else {
        	// id无效，则throw一个 IllegalArgumentException
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
    	// inflate被调用一次后 就没有了ViewParent，就会报这个错
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

我在每行代码上都加上了详细的注释，其实思路非常清晰，非常简单。  

总结来说，其实`inflate`方法是做了一个『偷梁换柱』的操作，把 `StubbedView`动态的添加到自己原来的位置上，也因此实现了懒加载功能。  


另外ViewStub还重写了View的`setVisibility`方法，让我们来分析一下：

```
public void setVisibility(int visibility) {
	// mInflatedViewRef 保存了 StubbedView还记得吗？ inflate过后它就不是null了 
    if (mInflatedViewRef != null) {
        View view = mInflatedViewRef.get();
        // 操作 StubbedView
        if (view != null) {
            view.setVisibility(visibility);
        } else {
            throw new IllegalStateException("setVisibility called on un-referenced view");
        }
    } else {
        // 操作ViewStub自己，构造方法里的GONE记得么？
        super.setVisibility(visibility);
        // 如果是 VISIBLE INVISIBLE 则会去调用 inflate方法！！！！
        if (visibility == VISIBLE || visibility == INVISIBLE) {
            inflate();
        }
    }
}
```

注意点： `setVisibility`方法中也可能会调用`inflate()`方法，所以需要注意的是 不要即调用setvisibility来设置可见，再接着调用`inflate`方法！  


## 要点小结

源码分析完毕，可以看到，ViewStub的源码还是非常简单的，但是作用还是挺大的。

稍微总结一下要点：  

1. 在XML里配置ViewStub的可见性是没有用的  
2. ViewStub 主要原来在`inflate()`方法，是它把真正要加载的View给加载了进来  
3. `inflate()`方法只能调用一次
4. ViewStub调用`inflate()`后就不要再用它了（让它功成身退！）
5. 小心`setVisibility`方法
6. 在XML里给ViewStub设置的LayoutParamas(宽高margin等)会传递给StubbedView,所以我们如果要控制StubbedView的LayoutParamas，则需要写在ViewStub里而不是StubbedView！  
6. 期待补充！  


