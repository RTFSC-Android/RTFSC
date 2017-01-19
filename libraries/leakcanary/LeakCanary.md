# LeakCanary内存泄漏检测原理分析



## 什么是内存泄漏





### 内存泄漏的危害





## 什么是LeakCanary







```java
public interface GcTrigger {
    GcTrigger DEFAULT = new GcTrigger() {
        public void runGc() {
            Runtime.getRuntime().gc();
            this.enqueueReferences();
            System.runFinalization();
        }

        private void enqueueReferences() {
            try {
                Thread.sleep(100L);
            } catch (InterruptedException var2) {
                throw new AssertionError();
            }
        }
    };

    void runGc();
}
```





## 延伸阅读

[LeakCanary: Detect all memory leaks!](https://medium.com/square-corner-blog/leakcanary-detect-all-memory-leaks-875ff8360745#.kqaeapqlx)

[LeakCanary-FAQ](https://github.com/square/leakcanary/wiki/FAQ)

[LeakCanary 内存泄露监测原理研究](http://www.jianshu.com/p/5ee6b471970e)

[Android应用内存泄漏的定位、分析与解决策略](http://www.jianshu.com/p/96c55ea3446e#)

