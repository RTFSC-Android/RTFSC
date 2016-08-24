# invalidate和postInvalidate的关系与区别


从源码角度分析 invalidate 与 postInvalidate 之间的关系区别。  


`invalidate`的代码调用路径：

```
public void invalidate() {
    invalidate(true);
}

void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
```

可以看到 invalidate 最后调用 invalidateInternal 去刷新，这里暂时不关心它们的具体实现。

然后再看看`postInvalidate`的代码调用路径：

```
public void postInvalidate(int left, int top, int right, int bottom) {
    postInvalidateDelayed(0, left, top, right, bottom);
}
/**
 * This method can be invoked from outside of the UI thread
 * only when this View is attached to a window.
 */
public void postInvalidateDelayed(long delayMilliseconds) {
    // We try only with the AttachInfo because there's no point in invalidating
    // if we are not attached to our window
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}

// ViewRootImpl
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}

// ViewRootImpl$ViewRootHandler extends Handler
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
    case MSG_INVALIDATE:
        ((View) msg.obj).invalidate();
        break;
    case xxx:
    	break;
    }	
}
```

postInvalidate 其实是把 invalidate 这个操作封装成了一个 Message，post 到了 ViewRootImpl$ViewRootHandler 中去，最终在UI线程中调用了 View 的 invalidate。

我们知道，异步更新一个 View 会报`"Only the original thread that created a view hierarchy can touch its views."`（来自 ViewRootImplement$checkThread）的错误，

而 postInvalidate 可以在任意线程去调用，而不需要担心线程问题。  

PS：很多人说『一定要在主线程更新 UI』，其实不然，仔细看这报错信息指得是 original thread,而这个线程是创建 ViewRootImpl 的线程，而不是特指主线程，只不过是绝大部分情况下是主线程，仅此而已。  

## 小结

关系：
- postInvalidate 其实最终调用的就是 invalidate

差别：
- invalidate只能在 original thread 调用（一般就是主线程），而 postInvalidate 可以在任意线程调用。

这结论其实早就知道了，但是光知道这结论是远远不够的，要深入源码去理解，找出结论的由来；

慢慢地，会发现，其实那些知识点以及结论其实就是从源码里得来的；

自己看源码，会收获更多意想不到的知识。
