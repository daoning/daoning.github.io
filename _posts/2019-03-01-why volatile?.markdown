---
layout: post
title:  why volatile?
date:   2019-02-28 20:18:29 +0800
categories: jekyll update
---
最近在看`java.util.concurrent`包的源码时，总是能看到volatile修饰的属性。以前脑海里只是隐约有个声音“并发编程中最好使用这个关键字”，便再无更深的理解。今天在某个成员变量的doc里出现了“Java内存模型”字眼，寻思着得把这个关键字搞清楚。

*什么时候必须用这个关键字？* or *被它修饰的变量能给程序带来什么？*

带着疑问，先百度了一番，搜到的文章鱼龙混杂。不行，这种微妙的机制，还是直接看[JavaSpecification](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.4)吧！文档这样解释volatile：  
A field may be declared volatile, in which case the Java Memory Model ensures that all threads see a consistent value for the variable.  
关键在于**consistent**这个词。先说结果，其在此处的含义为“一致的，不矛盾的”。然后文档举了一个很赞的例子，之后的一句话非常清晰、准确，我的意译是：对于volatile修饰的变量i，任何线程在运行到*读取i的值*的代码时，会严格地读取i所在的内存地址***当前***的值。 反过来说，若i没有volatile修饰，Java语言不能***保证***，*读取i的值*的代码得到的就是i的内存地址在那个时刻的值。


{% highlight java %}
class TestVolatile {
    private int i = 0, j = 0;
    void add() { i++; j++; }
    void read() {
        int readj = j;
        int readi = i;
        if (readi < readj)
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


order|volatile|total (K)|normal|total (K)|synchronized|total (K)
:-:|  :-:   |  :-:   |  :-: |  :-:   |    :-:     |  :-:  
1  |     0  |   353  |   1  |   218  |    0       |    116
2  |     0  |   142  |   6  |   172  |    0       |    74
3  |     0  |   258  |  19  |   164  |    0       |    103
4  |     0  |   201  |   7  |   256  |    0       |    67
5  |     0  |   151  |  14  |   171  |    0       |    74
average| 0  |   221  | 9.4  | 196.2  |    0       |    86.8

最大一次gap为j-i:331…惊呆了