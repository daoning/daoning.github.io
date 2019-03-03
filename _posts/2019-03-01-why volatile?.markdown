---
layout: post
title:  why volatile?
date:   2019-03-01 20:18:29 +0800
categories: jekyll update
---
最近在看`java.util.concurrent`包的源码时，总是能看到volatile修饰的属性。以前脑海里只是隐约有个声音“并发编程中最好使用这个关键字”，便再无更深的理解。今天在某个成员变量的doc里出现了“Java内存模型”字眼，寻思着得把这个关键字搞清楚。

*什么时候必须用这个关键字？* or *被它修饰的变量能给程序带来什么？*

带着疑问，先百度了一番，搜到的文章鱼龙混杂。不行，这种微妙的机制，还是直接看[JavaSpecification](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.4)吧！文档这样解释volatile：  
A field may be declared volatile, in which case the Java Memory Model ensures that all threads see a consistent value for the variable.  
关键在于*consistent*这个词。先说结果，其在此处的含义为“一致的，不矛盾的”。然后文档举了一个很赞的例子，之后的一句话非常清晰、准确，我的意译是：对于volatile修饰的变量i，任何线程在运行到*读取i的值*的代码时，会严格地读取i所在的内存地址***当前***的值。 反过来说，若i没有volatile修饰，Java语言不能***保证***，*读取i的值*的代码得到的就是i的内存地址在那个时刻的值。

现在，先忘了上面的一切，看看下面的程序，直觉上觉得它会输出怎样的结果？  
{% highlight java %}
class TestVolatile {
    private int i = 0, j = 0;
    void add() { i++; j++; }
    void read() {
        int readj = j;
        int readi = i;
        if (readi < readj)
//intuition: true is impossible
            System.out.println("j-i:" + (readj - readi));
    }

    abstract class DoTest implements Runnable {
        @Override
        public void run() {
            int round = 0;
            long start = System.currentTimeMillis();
            do {
                exe();
                round++;
            } while (System.currentTimeMillis() - start < 20);
            System.out.println(name + " executed " + round + " times.");
        }
        abstract void exe();
        DoTest(String name) { this.name = name; }
        String name;
    }
}

/*following main function*/
        TestVolatile tv = new TestVolatile();
        Thread add = new Thread(tv.new DoTest("add") {
            @Override
            void exe() { tv.add(); }
        });
        Thread read = new Thread(tv.new DoTest("read") {
            @Override
            void exe() { tv.read(); }
        });
        add.start();
        read.start();
{% endhighlight %}
我把文档里的例子优化了下，在read方法中先取得j的值。由add()方法可知：任意时刻，内存中j的值都不大于i的值。那么不论读取线程在readj和readi之间，写入线程有没有执行自增语句，读取线程取得的i的值都***不可能***小于上一条语句读取的j的值。

然而，上述程序会输出形如`j-i:..`的行... 怎么会这样？？

现在回到上面对于volatile的解读，有那么点意思了。分别将代码作下述修改  
+ 属性i和j都用volatile修饰，其余不变
+ add()和read()方法都用synchronized修饰，其余不变

我把每个版本的代码各运行了5次，结果如下表。以normal列和它右边的total(K)列为例，normal列表示未修改的程序此次运行出现if为true的次数，它右边的列表示此次运行中add()和read()方法执行的次数之和（单位：千次）。

order|volatile|total (K)|normal|total (K)|synchronized|total (K)
:-:|  :-:   |  :-:   |  :-: |  :-:   |    :-:     |  :-:  
1  |     0  |   353  |   1  |   218  |    0       |    116
2  |     0  |   142  |   6  |   172  |    0       |    74
3  |     0  |   258  |  19  |   164  |    0       |    103
4  |     0  |   201  |   7  |   256  |    0       |    67
5  |     0  |   151  |  14  |   171  |    0       |    74
average| 0  |   221  | 9.4  | 196.2  |    0       |    86.8

最大一次gap为j-i:331，惊呆了！出现在normal为19的那次运行。所有j-i:..当中，出现频率最高的结果为j-i:1。从方法执行次数可以看出，同步方法执行前后的加锁、释放锁是有不少性能消耗的。

虽然我现在还无法解释*为什么*没有此关键字修饰时读取的j的值会大于i（应该和Java内存模型有关了，回头再研究），但是已经可以解释*为什么*有此关键字修饰时运行结果符合直觉了（对，就是上面定义中说的那样）。那么能否认为，volatile是在并发竞争共享变量的场景下，对于直觉的补救呢？并且对性能还没有什么影响。

至于synchronized版本不会出现true，文档这样解释：add()方法返回前，i和j的共享值被确保更新了。