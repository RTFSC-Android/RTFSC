# Space 源码分析


## 简介

	public final class Space extends View

  Space是一个轻量的View，可以在布局中被用来创建间隙；常用于布局优化；

介于可能很多人根本不知道Space的存在！所以稍微提一下它的使用场景，比如以下场景的右侧小三角，就可以使用Space：  

![Space使用场景](http://ww2.sinaimg.cn/large/98900c07jw1f6pifwudkgj201k0110mp.jpg)

在两个三角之间放置一个`Space`，两三角分别位于它的上下，控制它的高度就能控制三角之间的间隔。

```
<RelativeLayout
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:layout_alignParentRight="true"
    android:layout_marginRight="2dp"
    >

    <ImageView
        android:id="@+id/iv_price_up"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_above="@+id/divider"
        android:background="@drawable/price_up"
        android:layout_marginBottom="1dp"
        />

    <Space
        android:layout_width="wrap_content"
        android:layout_height="1dp"
        android:id="@+id/divider"
        android:layout_centerInParent="true"
        />

    <ImageView
        android:id="@+id/iv_price_down"
        android:layout_marginTop="1dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/divider"
        android:background="@drawable/price_down"
        />
</RelativeLayout>
```


## 构造方法分析

看第一个构造方法即可。  

```
// 最终都会调用这个构造方法
public Space(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);
    // 如果是VISIBLE 则改为 INVISIBLE
    if (getVisibility() == VISIBLE) {
        setVisibility(INVISIBLE);
    }
}

public Space(Context context, AttributeSet attrs, int defStyleAttr) {
    this(context, attrs, defStyleAttr, 0);
}


public Space(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
}

public Space(Context context) {
    //noinspection NullableProblems
    this(context, null);
}
``` 

跟ViewStub类似，Space在构造方法里做了一些操作：**当可见性为VISIBLE的时候，把它改为INVISIBLE了**。

由于Space方法非常少，接下去直接都分析了。  

## 其余方法分析


```
/**
 * Draw nothing.
 *
 * @param canvas an unused parameter.
 */
@Override
public void draw(Canvas canvas) {
	//空方法
}

/**
 * Compare to: {@link View#getDefaultSize(int, int)}
 * If mode is AT_MOST, return the child size instead of the parent size
 * (unless it is too big).
 */
private static int getDefaultSize2(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST: //wrap_content 返回更小的值
            result = Math.min(size, specSize);
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	// getSuggestedMinimumWidth() 根据minWidth以及背景的宽度来返回
    setMeasuredDimension(
            getDefaultSize2(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize2(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

Space跟ViewStub一个套路，`draw()`都为空方法，然后重写`onMeasure`，相比ViewStub，Space代码更加比较简单。

另外一般我们使用Space都是会指定宽高，大部分走的是 EXACTLY的流程。  

## 要点

0. Space 用来做间隙非常有用
1. Space 默认为不可见（invisible），但是有宽高，会占据空间。
2. 布局文件中设置 VISIBLE 无效。

## 小结

ViewStub跟Space作为Android布局优化的常用手段，有着一些同样的思路值得我们去学习：

- 不绘制(减少overDraw)  
- 优化或者不参与测量与布局（提高整体布局的渲染速度） 


[ViewStub源码分析](./ViewStub.md)

	2016年8月11日下午2:41


