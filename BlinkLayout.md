# BlinkLayout源码分析

	private static class BlinkLayout extends FrameLayout


首先，我猜，你肯定不知道有这个Layout的存在！！！

因为它隐藏的非常深，是`LayoutInflater`的静态内部类，是我在看`LayoutInflater`源码的时候发现的！简直是个彩蛋！！    

在这里发现的：  

```
if (name.equals(TAG_1995)) {
    // Let's party like it's 1995!
    return new BlinkLayout(context, attrs);
}
```

oh,它其实还真算是个彩蛋，似乎是为了庆祝1995年的复活节，有兴趣可以看看
[reddit](https://www.reddit.com/r/androiddev/comments/3sekn8/lets_party_like_its_1995_from_the_layoutinflater/)上的讨论。  


blink 有 使...闪烁的意思，可以用来做一闪一闪的效果哦！！！ 

先上个简单的效果图看看它的效果：  

![效果图](http://ww2.sinaimg.cn/large/98900c07jw1f6pxwz67aqg207l0ckq30.gif)

是不是很闪？  

明明这么闪耀，为何要躲起来？

`BlinkLayout`的使用也有些特殊，它跟`merge`、`include`这些标签一样，用标签`blink`来表示。

贴一下上图效果的XML：

```
    <blink
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        >

        <ImageView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:src="@mipmap/ic_launcher"
            >
        </ImageView>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Hello World!"/>
    </blink>
```

非常简单！

那么它是怎么实现的呢？

接下去源码分析。

## 源码分析

`BlinkLayout`的源码非常简单，只有几十行！我就全部贴出来啦！  

```
private static class BlinkLayout extends FrameLayout {
    private static final int MESSAGE_BLINK = 0x42;
    private static final int BLINK_DELAY = 500;

    private boolean mBlink;
    private boolean mBlinkState;
    private final Handler mHandler;

    public BlinkLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        mHandler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                if (msg.what == MESSAGE_BLINK) {
                    if (mBlink) {
                        mBlinkState = !mBlinkState;
                        // 循环调用 makeBlink
                        makeBlink();
                    }
                    // 触发 dispatchDraw
                    invalidate();
                    return true;
                }
                return false;
            }
        });
    }
    // 发送blink指令
    private void makeBlink() {
        Message message = mHandler.obtainMessage(MESSAGE_BLINK);
        mHandler.sendMessageDelayed(message, BLINK_DELAY);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        // 设置为可以闪啦
        mBlink = true;
        mBlinkState = true;
        // 发送消息
        makeBlink();
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();

        mBlink = false;
        mBlinkState = true;
        // 移除消息 避免内存泄漏
        mHandler.removeMessages(MESSAGE_BLINK);
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
    	// 通过这个开关来控制是否分发绘制事件，来达到一闪一闪的效果
        if (mBlinkState) {
            super.dispatchDraw(canvas);
        }
    }
}
```

从源码中可以看出，`BlinkLayout`通过Handler循环调用`invalidate()`方法，触发并控制`dispatchDraw`来做到一闪一闪的效果，默认的闪烁间隔为Handler的DELAY时间，即500毫秒。  


##　小结

`BlinkLayout`的使用场景或许不多，但是它的代码还是非常漂亮哒！~  

如果有类似需求，仿造它的源码写一个功能更强的View也是非常简单的！  

深入源码阅读是一件相对枯燥的事情，能发现这么个小彩蛋也是棒棒的，心情也美丽了些，哈哈！~~  








