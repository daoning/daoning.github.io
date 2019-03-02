---
layout: post
title:  ThreadLocal 初探
date:   2019-02-14 22:53:37 +0800
categories: jekyll update
---
符号说明
+ tl: 一个ThreadLocal&lt;U&gt;对象
+ u: 一个U类的对象
+ m: Thread对象的threadLocalMap对象（ThreadLocalMap类是ThreadLocal类的静态内部类）

阅读源码后，总结下`tl.set(u)`运行时发生了什么。
1. 获取当前线程的m[注1]
2. 计算index = tl的哈希值 & m的tab的length-1，获取tab在index处的Entry对象e
3. 若e为null，则参照注1中，将index处指向new出来（以tl和u为参数）的Entry对象。方法结束。
4. 否则获取e弱引用指向的ThreadLocal对象key
    1. 若key==tl，将e的value指向u，方法结束。
    2. 若key为null（场景：之前另一个ThreadLocal对象在当前线程运行时也调用了本文讨论的set方法，后来（在此之前）被gc了。这也引导我们理解为什么Entry要弱引用ThreadLocal对象：即使某个线程中有引用指向ThreadLocal对象，后者也能被回收），调用staleEntry方法......，方法结束。
5. 步骤4的两个分支都不满足，循环给index加1重复步骤3、4，这边还没看懂.....

注
1. 线程对象生成时m为null。如果此时m为null，则以tl和u为参数，生成m。m拥有一个Entry[] tab，tab的初始长度为16；Entry继承了WeakReference&lt;ThreadLocal&lt;?&gt;&gt;，拥有一个Object value，Entry的构造函数将this弱引用到tl，将value指向u；生成m时，以tl和u为参数，new了一个Entry对象e，令（tl的哈希值 & 15）的结果为index，将tab[index]指向e。方法结束。
