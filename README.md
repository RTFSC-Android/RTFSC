#　RTFSC

> Read The Fucking Source Code

还有什么比阅读源码更好的学习方式吗？

并没有！！！

开启阅读源码之路（包括但是不限于framework源码，开源项目源码）  


## 阅读源码的好处

『所有的知识其实都来自源码』是我最深的感悟。  

通过阅读源码，对知识点的掌握不再流于表面，而能够做到知其然以及所以然，极大地提升判断力，不再人云亦云。

阅读源码还能极大的扩大知识面，通常在阅读源码的时候你会发现很多你根本不知道，或者看文章博客根本不会获取得到的知识，经常会遇到各种『彩蛋』。

Android 源码是学习设计模式的最佳途径之一，Android 团队遇到的坑，比我写过的代码还多，Android 源码中到处可见设计模式的影子，阅读它，可以加深对设计模式的理解。  

好处绝不止我所说的，自己去体会。  

## 哪里可以看Android源码

Android 源码的查看一般有以下几种方式：

- 在在线网站上查看,如：[grepcode](http://grepcode.com/),[androidxref](http://androidxref.com/)  
- 获取Android Framework源码查看，clone [frameworks_base](https://github.com/android/platform_frameworks_base) ，在 Mac 端可以使用 Sublime 配合 CTAG 查看。  
- 使用 AndroidStudio 看

取合适自己的。

## 阅读源码的姿势

源码数量庞大，如果漫无目的地去阅读很容易迷失自己，所以阅读源码要有一定的目标。

比如针对某一个问题去查看源码，eg. invalidate和postInvalidate的关系与区别是什么？

这样才能有目标性的寻找答案。

另外阅读源码不是容易的事情，可以从简单的类开始阅读，培养阅读习惯以及技巧，增加信心，一层一层深入，不宜在刚开始就非常深入，容易打击自信，怀疑猿生。

另外这些资料可能对你有帮助：

- [大牛们是怎么阅读 Android 系统源码的？](https://www.zhihu.com/question/19759722)  
- [阅读 ANDROID 源码的一些姿势](http://kaedea.com/2016/02/09/android-about-source-code-how-to-read/)  

## 加入我们

可以看到 这是一个组织，如果你有兴趣加入，请联系我。


## 版权信息

todo


## 目录


[ViewStub 源码分析](./ViewStub.md)  
[Space 源码分析](./Space.md)   

[LayoutInflater 源码分析（一）之 inflate 深度分析](./LayoutInflater.md)  
[LayoutInflater 源码分析（二）之 include 以及 merge 标签的处理](./LayoutInflater-2.md)  
[LayoutInflater 源码分析（三）之 fragment 标签的处理](LayoutInflater-3.md)  
[LayoutInflater 源码分析（四）之 闪耀的彩蛋](./BlinkLayout.md)

[invalidate和postInvalidate的关系与区别](invalidate-and-postinvalidate.md)  
[Android应用的程序入口是哪里？](where-is-app's-entrance.md)