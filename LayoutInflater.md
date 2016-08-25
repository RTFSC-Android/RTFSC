# LayoutInflater 源码分析（一）之 inflate 深度分析


[LayoutInflater 源码分析（一）之 inflate 深度分析](./LayoutInflater.md)  
[LayoutInflater 源码分析（二）之 include 以及 merge 标签的处理](./LayoutInflater-2.md)  
[LayoutInflater 源码分析（三）之 fragment 标签的处理](LayoutInflater-3.md)  
[LayoutInflater 源码分析（四）之 闪耀的彩蛋](./BlinkLayout.md)


## 简介

	public abstract class LayoutInflater


从名字就可以看出它用于加载布局。  


我们常用的方式大概如下：  

```
// 方法定义：inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
View view = LayoutInflater.from(context).inflate(R.layout.resource,root,flase);	
```

PS：`LayoutInflater`的获取方式还有很多种，实际最终调用的是 Context.getSystemService 方法，关于getSystemService 的分析，有兴趣的可以看看[Context.getSystemService分析](context-getsystemservice.md)。  


上述代码我写过无数遍，但是心中一直有很多疑问：  

- 上述方法中的`root`、`attachToRoot`究竟有什么作用？
- 它究竟是在哪里实例化View又是如何去实例化 View 的？
- 为什么系统的View我们在Xml里不需要写全路径，而自定义View却需要？
- 它又是如何处理`fragment`以及各种标签如`include`、`merge`的？
- `View`的`onFinishInflate`是否跟它有关呢？

这一切都藏在源码里，所以深入源码一点点了解吧！  

## inflate深入解析 

上面的例子中可以看到，我们调用的是`inflate(in resource, in root, in attachToRoot)`方法，所以首先分析一下该方法。  

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
	//获取rescources
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }
    // 获取parser　这里我不关心parse是怎么来的
    final XmlResourceParser parser = res.getLayout(resource);
    try {
    	// 调用了另外一个inflate方法
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

上面代码可以看到该方法主要是获取一个`XmlResourceParser`对象parser（这里不关心它是如何来的）。

不过需要提一下的这里解析XML采用的是 **Pull**方法(不知道的自行Google)。

然后调用了另外一个`inflate`方法，所以我们还需要继续跟踪`inflate(parser, root, attachToRoot)`才能进一步理解。 

上代码！  

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        // 一些赋值 后续会有用到
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        // 先把result赋值为我们传递的root
        View result = root;

        try {
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            // 获取标签名字 比如FrameLayout
            final String name = parser.getName();
            
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }
            // 这里的 TAG_MERGE 为 merge 看到了merge的身影
            if (TAG_MERGE.equals(name)) {
            	// 不知道merge怎么用？ 这个异常教你做人。
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                //如果为 merge 调用rInflate方法 后面再具体分析merge的情况
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                // 这里调用了一个createViewFromTag 从名字来看，就是用来创建View的！
                // 注意，这里的temp其实是我们xml里的top view,具体暂时先不管 先把整个流程看了
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                //接下去处理LayoutParams
                ViewGroup.LayoutParams params = null;
				//如果我们传递进来的root不为null              
                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    // 那么调用 root的generateLayoutParams 来生成LayoutParamas
                    params = root.generateLayoutParams(attrs);
                    //如果attachToRoot为false，那么就把刚生成的params赋值给View
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // 源码的打印日志已经告诉我，这里是加载子View的~~ 后续再讲解
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);

                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                // root不为null 并且 attachToRoot 则直接把temp添加到root里去
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                // null或false 那么结果就是之前的top view了
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            InflateException ex = new InflateException(e.getMessage());
            ex.initCause(e);
            throw ex;
        } catch (Exception e) {
            InflateException ex = new InflateException(
                    parser.getPositionDescription()
                            + ": " + e.getMessage());
            ex.initCause(e);
            throw ex;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        // 返回结果
        return result;
    }
}
```

可以看到该方法是重点了。  

这里出现了`merge`的踪迹，可以看到遇到`merge`标签，**当root为null或者attachToRoot为false的时候，直接抛了异常！**
也可以看到如果是`merge`标签，走的是`rInflate`方法(不过这里暂时不分析`rInflate`方法)。


关键的是，我看到`inflate`方法处理了`root`与`attachToRoot`参数。

围绕`root`是否为`null`有两种处理分支：

第一种 当`root`不为`null`的时候：

1. 调用`root.generateLayoutParams`方法来生成`LayoutParamas`并赋值给`paramas`
2. 然后如果`attachToRoot`为`false`，则把`paramas`赋值给`createViewFromTag`解析出来的`temp`（XML里的根布局）
3. 而如果`attachToRoot`为`true`的话，则会 调用`root.addView(temp, params);` **直接把`temp`给加到`root`里去**。如果我们自己再调用`addView`则会报错！


这里再提一下`root` 对`topView`的`LayoutParamas`的影响：

需要先提一下 LayoutParamas 一般有3种来源：

1. 用户完全自定义 自己 `new` 出来
2. ViewGroup.generateLayoutParams 方法生成 上面已经提到
3. ViewGroup.generateDefaultLayoutParams 方法生成，在`addView`的时候 如果 `childView` 没有的话 LayoutParams 属性的话，会由这个方法生成。

来看一下 addView 里对 paramas 的操作就明白了：

```
public void addView(View child) {
    addView(child, -1);
}
public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
    	// 没有 params 则调用 generateDefaultLayoutParams 去生成
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}
```

所以当 root 不为 null 的时候，topview 的 paramas 是通过`generateLayoutParams`生成的。  

需要注意的是：`generateLayoutParams`与`generateDefaultLayoutParams`生成的 paramas 是不同的,会无视我们在 xml 里配置的属性，所以它会影响到布局效果。  


第二种 当`root`为`null`的时候：  

是`null`的时候会返回`temp` （XML里的根布局）

```
// null 或是 false 那么result=temp
if (root == null || !attachToRoot) {
    result = temp;
}
```

也可以看到，`root==null`如果成立，那么`attachToRoot`也就没有用了。

所以`attachToRoot`只有在`root`不为 null 的时候才有效。  

大致总结成流程图如下所示：  

![流程图](http://ww4.sinaimg.cn/large/98900c07jw1f6ul65ibh3j20g40qeta0.jpg)


搞清楚`root`以及`attachToRoot`参数的影响之后，来看View究竟是如何被创建的。  

进入`createViewFromTag`方法。

##　createViewFromTag　解析

上面提到的`createViewFromTag`方法如下：

```
/**
 * 提到了include，除了include，都会被使用
 * Convenience method for calling through to the five-arg createViewFromTag
 * method. This method passes {@code false} for the {@code ignoreThemeAttr}
 * argument and should be used for everything except {@code &gt;include>}
 * tag parsing.
 */
private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
    return createViewFromTag(parent, name, context, attrs, false);//这里传了false进去
}
```

可以看到，它又调用了另外一个重载函数，并从注释中我们可以看到了`include`的信息。

该方法把`ignoreThemeAttr`属性赋值为了`false`，继续跟下去。 

```
/**
 * Creates a view from a tag name using the supplied attribute set.
 * @param ignoreThemeAttr {@code true} to ignore the {@code android:theme}
 *                        attribute (if set) for the view being inflated,
 *                        {@code false} otherwise
 */
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // Apply a theme wrapper, if allowed and one is specified.
    // 之前传递过来的为false
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        //如果设置了theme 那么context会被重新实例化为 ContextThemeWrapper
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }
    // 彩蛋后续 分析
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }
    // 这里开始去创建View了
    try {
        View view;
        //如果mFactory2不为null 那么mFactory2先解析
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
        	// 接着是 mFactory
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }
        // 如果 mFactory2 mFactory都返回null了 那么如果mPrivateFactory不为null，则交给它
        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }
        // 如果 那几个factory都返回null 即view还是null 那么继续
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
            	// 有. 代表不是系统自带的View,比如TextView、me.yifeiyuan.XXXLayout
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                	// 系统自带的View
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } catch (InflateException e) {
        throw e;

    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;

    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name);
        ie.initCause(e);
        throw ie;
    }
}
```

该`createViewFromTag`方法 先处理了主题属性，再走入创建View的流程。 

这里还涉及到了几个Factory，这其实是系统留给我们的Hook入口，我们可以人为的干涉系统创建View，可以添加更多功能，比如夜间模式。

`Factory`相关的知识后续再讲。

另外，我们可以看到该方法依然没有涉及到创建View的具体实现，而是又会去调用`onCreateView`以及`createView`方法，这俩方法总应该是View创建的具体地方了吧？！！

## onCreateView 与 createView

初步来看`onCreateView`方法负责创建自定义View，而`createView`方法负责创建系统自带的View。
但是感觉比较奇怪，因为不管是什么View，创建的套路应该是一样才对啊~ 
感觉有诈！

```
protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return createView(name, "android.view.", attrs);
}
```

咦！~ `onCreateView`调用了`createView`，到最后，其实都是调用`createView`方法啦！  

另外还传入了`android.view.`的一个参数，咦？这不是系统自带的View的包路径吗？  

继续深入`createView`。  

```
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    // 构造方法缓存
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;

    try {
    	// Trace
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
        //没有缓存 则去获取constructor 并存入缓存 注意这个缓存是静态的
        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            // 注意 注意 注意 在这里把 View的全称拼全，并loadClass 难怪要传递android.View.进来啊~
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            // 让 Filter 处理这个clazz能否被加载
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                	// 如果不允许加载 则failNotAllowed会抛出异常！
                    failNotAllowed(name, prefix, attrs);
                }
            }
            // 反射获取构造方法 并存入缓存
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
        	// 如果有缓存 就走filter流程，并把结果存入缓存（非静态）
            // If we have a filter, apply it to cached constructor
            if (mFilter != null) {
                // Have we seen this name before?
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // New class -- remember whether it is allowed
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);
                    // 获取allowed 并存入 缓存
                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }
        // mConstructorArgs存放了context 跟 args
        Object[] args = mConstructorArgs;
        args[1] = attrs;
        // 终于看到view实例化的地方了！！
        final View view = constructor.newInstance(args);
        // 如果是ViewStub 则设置LayoutInflater给它用 
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;

    } catch (NoSuchMethodException e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;

    } catch (ClassCastException e) {
        // If loaded class is not a View subclass
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Class is not a View "
                + (prefix != null ? (prefix + name) : name));
        ie.initCause(e);
        throw ie;
    } catch (ClassNotFoundException e) {
        // If loadClass fails, we should propagate the exception.
        throw e;
    } catch (Exception e) {
        InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class "
                + (clazz == null ? "<unknown>" : clazz.getName()));
        ie.initCause(e);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

调来调去，终于到真正实例化View的地方了。  

看到这方法的 `clazz = mContext.getClassLoader().loadClass(prefix != null ? (prefix + name) : name).asSubclass(View.class)` 步骤会把系统自带的`View`的路径拼起来，把类加载进来；

然后`clazz.getConstructor(mConstructorSignature);`获取`View`的构造方法，最终通过反射`constructor.newInstance(args);`实例化View。  

如果你足够机智，你会发现这里出来一个问题，WebView 怎么办？

它的路径可是`android.webkit`啊~
其实这里涉及到 LayoutInflater 的一个子类`com.android.internal.policy.PhoneLayoutInflater`，它处理了`android.widget.`、`android.webkit.`、`android.app.`这些路径。  

**事实上，我们最开始使用`LayoutInflater.from(cxt)`获取的就是`PhoneLayoutInflater`的实例。**  

另外这里又涉及到一个Hook入口，即`Filter`，但是我不知道它的使用场景。

`createView`方法里解答了我 *View是哪里实例化的*以及*XML中系统View为什么不需要写全路径* 这两个疑问。

## 小结

这一篇中分析了如下方法(省去了参数)：

- `inflate`：LayoutInflater对外开放的入口，这里分析了 root与attachToRoot 参数的作用。
- `createViewFromTag`：处理主题属性与Factory的Hook
- `onCreateView`： 处理系统自带View的路径，`android.view.`，实际调用的还是`createView`方法
- `createView`：  真正实例化View的地方，通过View的路径去加载类并获取构造方法，通过反射获取View的实例。

本篇解决了一些疑问：

- 上述方法中的`root`、`attachToRoot`究竟有什么作用？
    - 影响了`merge`标签，
    - View是否直接被 add 到 root
    - View 的 LayoutParams 从何而来
    - inflate 方法的返回值
- 为什么系统的View我们在Xml里不需要写全路径，而自定义View却需要？
    - 针对系统 View，会帮忙拼全路径,所以不需要写全
- 它究竟是在哪里实例化View又是如何实例化 View 的？
    - 在  createView 方法中，默认利用反射实例化 View
    - 也可通过 Factory hook 的方式实例化

但是还有好多疑问没有解决，也还有部分重要的方法没有解析，所以需要继续探索。  

篇幅太长了，所以先小结一下，换一篇继续。

下一篇着重分析`merge`、`include`等标签是如何处理的。

已经写好啦：[LayoutInflater 源码分析（二）之 include 以及 merge 标签的处理](./LayoutInflater-2.md)  




