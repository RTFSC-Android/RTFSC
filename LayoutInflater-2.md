# LayoutInflater 源码分析（二）

继上篇[LayoutInflater 源码分析（一）](./LayoutInflater.md)  

本篇继续对`LayoutInflater`进行源码分析，目标为分析`LayoutInflater`对`include`、`merge`、`fragment`等标签的处理原理。  

## merge 标签
[上篇](./LayoutInflater.md)讲到`inflate`方法中出现 Merge 的踪迹，代码如下：

```
if (TAG_MERGE.equals(name)) {
    if (root == null || !attachToRoot) {
        throw new InflateException("<merge /> can be used only with a valid "
                + "ViewGroup root and attachToRoot=true");
    }
    // 调用 rInflate 注意最后的参数是 false
    rInflate(parser, root, inflaterContext, attrs, false);
}
```

所以需要看 `rInflate`方法才行。

### rInflate 深入解析

```
/**
 * Recursive method used to descend down the xml hierarchy and instantiate
 * views, instantiate their children, and then call onFinishInflate().
 * <p>
 * <strong>Note:</strong> Default visibility so the BridgeInflater can
 * override it.
 * 递归方法 实例化 View 以及它的子 View,并且调用 onFinishInflate
 */
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    // while 循环 parser.next  遍历整个 XML 
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }
        //获取标签名
        final String name = parser.getName();
        //如果是 requestFocus
        if (TAG_REQUEST_FOCUS.equals(name)) {
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {
        	// 处理 tag
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            // 处理 include
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
        	// merge 里不能再有 merge 标签的
            throw new InflateException("<merge /> must be the root element");
        } else {
        	// 如果不是特殊的标签那么走 createViewFromTag
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }
    // 注意这里从传递inflate传递过来的是 false！！
    // 从 rInflateChildren 过来的是 true！！！
    if (finishInflate) {
        parent.onFinishInflate();
    }
}	
```

从注释来看`rInflate`方法，是个 **递归方法** 实例化 View 以及它的子 View,并且调用 `onFinishInflate` 

从源码来看，`rInflate`方法先判断特殊的标签名，优先处理：
- 针对`requestFocus`标签,调用parseRequestFocus方法
- 针对`tag`标签，调用 parseViewTag 方法
- 针对`merge`标签则直接抛了异常，因为`merge`标签不能是子元素

很奇怪，并没有看到 `fragment` 标签的处理逻辑!

处理完特殊标签之后走到最后一个else 块中，这块代码需要注意：

```
// 如果不是特殊的标签那么走 createViewFromTag 获得 view
// createViewFromTag 方法的流程已经分析过了，不再多说。  
final View view = createViewFromTag(parent, name, context, attrs);
// 注意注意 这边的 parent 是之前inflate传入的 root
final ViewGroup viewGroup = (ViewGroup) parent;
// 生成 paramas
final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
// 调用 rInflateChildren 把 view传递过去 并传了一个 true 过去
rInflateChildren(parser, view, attrs, true);
// 注意
// 注意
// 注意
// 这里直接调用 viewGroup 的 addView 这也是 merge 能减少层级的根本原因
viewGroup.addView(view, params);
```

这里涉及到一个方法`rInflateChildren`，其实它在上一篇中的`inflate`方法中出现过，不过我并没有去分析，所以这里讲一下。  

方法实现如下：

```
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

`rInflateChildren` 是个递归方法，被用来实例化不是 `root` 的子View，实际也是调用`rInflate`方法。(所以才是递归了嘛！)

这里需要注意的是`finishInflate`，按照之前所说的流程，`rInflate`方法传递过来的`finishInflate`参数为`false`,在上一篇中`inflate`传递的参数是`true`,这关系到`onFinishInflate`的回调。

了解完`rInflateChildren`方法后，继续分析。

可以看到，在处理`merge`标签的时候，是将`merge`标签里解析出来的 View 直接 add 到了传递进来的`root`中去了，而并不会多加一层 View，从而实现减少层级的效果，这就是`merge`标签的原理所在了。  
最后，由于`inflate`传递进来的`finishInflate`为 false，所以不会去调用`parent.onFinishInflate();`

到此也知晓了`LayoutInflater`是如何处理`merge`标签以及`merge`减少布局层次的原理了。  

## include 标签

```
private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    int type;

    if (parent instanceof ViewGroup) {
        // Apply a theme wrapper, if requested. This is sort of a weird
        // edge case, since developers think the <include> overwrites
        // values in the AttributeSet of the included View. So, if the
        // included View has a theme attribute, we'll need to ignore it.
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        final boolean hasThemeOverride = themeResId != 0;
        if (hasThemeOverride) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
        // If the layout is pointing to a theme attribute, we have to
        // massage the value to get a resource identifier out of it.
        int layout = attrs.getAttributeResourceValue(null, ATTR_LAYOUT, 0);
        if (layout == 0) {
        	// 没有 layout 属性则抛异常
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            if (value == null || value.length() <= 0) {
                throw new InflateException("You must specify a layout in the"
                        + " include tag: <include layout=\"@layout/layoutID\" />");
            }
            // Attempt to resolve the "?attr/name" string to an identifier.
            layout = context.getResources().getIdentifier(value.substring(1), null, null);
        }

        // The layout might be referencing a theme attribute.
        if (mTempValue == null) {
            mTempValue = new TypedValue();
        }
        if (layout != 0 && context.getTheme().resolveAttribute(layout, mTempValue, true)) {
            layout = mTempValue.resourceId;
        }
        // 之前的代码都是处理 theme layout 属性 不多说。
        if (layout == 0) {
        	// 必须指定有效的layout
            final String value = attrs.getAttributeValue(null, ATTR_LAYOUT);
            throw new InflateException("You must specify a valid layout "
                    + "reference. The layout ID " + value + " is not valid.");
        } else {
            final XmlResourceParser childParser = context.getResources().getLayout(layout);

            try {
                final AttributeSet childAttrs = Xml.asAttributeSet(childParser);

                while ((type = childParser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty.
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(childParser.getPositionDescription() +
                            ": No start tag found!");
                }

                final String childName = childParser.getName();

                if (TAG_MERGE.equals(childName)) {
                    // The <merge> tag doesn't support android:theme, so
                    // nothing special to do here.
                    rInflate(childParser, parent, context, childAttrs, false);
                } else {
                	// 获取 被 inlcude 的 topview
                    final View view = createViewFromTag(parent, childName,
                            context, childAttrs, hasThemeOverride);
                    final ViewGroup group = (ViewGroup) parent;

                    final TypedArray a = context.obtainStyledAttributes(
                            attrs, R.styleable.Include);
                    final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
                    final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                    a.recycle();

                    // We try to load the layout params set in the <include /> tag.
                    // If the parent can't generate layout params (ex. missing width
                    // or height for the framework ViewGroups, though this is not
                    // necessarily true of all ViewGroups) then we expect it to throw
                    // a runtime exception.
                    // We catch this exception and set localParams accordingly: true
                    // means we successfully loaded layout params from the <include>
                    // tag, false means we need to rely on the included layout params.
                    ViewGroup.LayoutParams params = null;
                    try {// 尝试对 include 标签生成 params
                        params = group.generateLayoutParams(attrs);
                    } catch (RuntimeException e) {
                        // Ignore, just fail over to child attrs.
                    }
                    // 如果失败 则对被 include 的 topview 处理
                    if (params == null) {
                        params = group.generateLayoutParams(childAttrs);
                    }
                    view.setLayoutParams(params);
                    // Inflate all children.  前面已经提到过了
                    rInflateChildren(childParser, view, childAttrs, true);
                    // 处理 id
                    if (id != View.NO_ID) {
                        view.setId(id);
                    }
                    // 处理可见性
                    switch (visibility) {
                        case 0:
                            view.setVisibility(View.VISIBLE);
                            break;
                        case 1:
                            view.setVisibility(View.INVISIBLE);
                            break;
                        case 2:
                            view.setVisibility(View.GONE);
                            break;
                    }
                    // 把 view 添加到 group 中
                    group.addView(view);
                }
            } finally {
                childParser.close();
            }
        }
    } else {
        throw new InflateException("<include /> can only be used inside of a ViewGroup");
    }

    LayoutInflater.consumeChildElements(parser);
}
```

首先判断 parent 是不是个 ViewGroup,如果不是则直接抛异常。  

如果是则接下去处理 `theme` 属性以及 `layout` 属性，我们知道使用`include`标签，`layout` 属性是必须要有的。  

其原因就是在源码中如果发现没有指定 `layout` 属性的话，那么会直接抛出异常。  

再接下去的步骤可以看出其实跟上篇`inflate`方法类似：  

1. 通过调用`createViewFromTag`解析获取被`include`的 `topview`
2. 生成 `params`，这里要注意，`include`标签可能没有宽高，会导致生成失败，所以如果失败则接着又对被`include`的 topview 做操作。所以使用`include`的时候，不对它设置宽高是没有关系的。
3. 调用`rInflateChildren`处理子View 之前已经分析过
4. 如果有的话，把 `include` 标签的 `id` 以及 `visibility`属性 设置给 `topview`
5. `topView` 被 add 进 group，这样被 include 的 topView 就被加到布局里去了。


## 小结

通过阅读源码，其实使用 merge 以及 include 等标签处理其实并不难，而且它们的使用方法在源码中皆有体现。

稍微总结一下要点：

1. 使用 LayoutInflater 去 inflate merge 标签的时候，root 一定不能为 null，attachToRoot 也不能为 false  
2. merge标签在 XML 中**必须是根元素**  
3. 与 merge 标签相反，include 绝对不能是根元素，必须需要在一个 ViewGroup 中使用  
4. 使用 include 标签必须指定有效的 layout 属性  
5. 使用 include 标签不写宽高是没有关系的（会去解析被 include 的 layout）  


到这里`merge`以及`include`已经分析完毕。  

同时也看到了其他标签如`tag`、`requestFocus`的处理，但是就是没看到`fragment`标签。

那究竟是在哪处理`fragment`标签的呢？

下一篇再讲解。  



