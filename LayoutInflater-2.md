# LayoutInflater 源码分析（二）

继上篇[LayoutInflater 源码分析（一）](./LayoutInflater.md)  

本篇继续对`LayoutInflater`进行源码分析，目标为分析`LayoutInflater`对`include`、`merge`、`fragment`等标签的处理原理。  


上篇讲到

```
if (TAG_MERGE.equals(name)) {
    if (root == null || !attachToRoot) {
        throw new InflateException("<merge /> can be used only with a valid "
                + "ViewGroup root and attachToRoot=true");
    }
    // 调用
    rInflate(parser, root, inflaterContext, attrs, false);
}
```

