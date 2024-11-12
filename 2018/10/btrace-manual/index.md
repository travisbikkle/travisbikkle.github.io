# Btrace 使用教程

**BTrace** 是 Java
的一个诊断工具，可以在不重启应用的情况下，对应用进行时间耗费、参数及结果跟踪、方法调用跟踪等分析。

BTrace 术语
===========

Probe Point  
位置

Trace Actions or Actions  
追踪语句

Action Methods  
追踪语句所在的静态方法

BTrace 程序结构
===============

一个 **BTrace 程序** 是一个 Java 类，包含数个由 `BTrace 注解` 注释的
`public static void` 方法。这些注解被用来指定被追踪程序的 **Probe
Point**. **Tracing Actions**
在这些静态方法内定义。这些静态方法也即上文提到的 **Action Methods**。

BTrace 的限制
=============

1.  不能 创建对象，数组。

2.  不能 抛出、捕获异常。

3.  不能 调用任意实例或静态方法，只能调用 BTraceUtils 中的方法。

4.  不能 修改目标程序的静态或实例变量，不过 BTrace 程序自己不做限制。

5.  不能 有实例变量或方法，方法不能有返回值类型，BTrace
    程序的所有方法必须是 public static 1. oid 的，所有的字段都必须是
    static 的。

6.  不能 有 outer, inner, nested 或 local 类。

7.  不能 用 synchronized 关键字。

8.  不能 有循环 (for, while, do..while)。

9.  不能 继承任何类 (即父类只能是 java.lang.Object)。

10. 不能 实现接口。

11. 不能 包含 assert 语句.

12. 不能 使用 class literals.

虽然不需要重启就可以使用 BTrace,
但是使用不当仍然会导致应用终止，比如目标程序可能会发生下面这种错误:

    Exception in thread &#34;main&#34; java.lang.NoSuchMethodError: com.foo.btraceDemo.Target.$btrace$com$foo$btraceDemo$Tracer$func(Ljava/lang/Object;IJII)V

一个简单的 `BTrace 程序` 演示
=============================

**Tracer.java — BTrace 要执行的程序（脚本）.**

    // 导入所有 BTrace 注解。
    import com.sun.btrace.annotations.*;
    // 导入 BTraceUtils 的静态方法。
    import static com.sun.btrace.BTraceUtils.*;

    // @BTrace annotation tells that this is a BTrace program
    @BTrace 
    public class HelloWorld {

        // @OnMethod 注解表明 Probe Point(位置)。
        // 在这个例子中，我们对进入 Thread.start() 方法感兴趣。
        @OnMethod(
            clazz=&#34;java.lang.Thread&#34;,
            method=&#34;start&#34;
        )
        public static void func() {
            // println 是 BTraceUtils 的静态方法
            println(&#34;about to start a thread!&#34;);
            // System.out.println(&#34;123&#34;); 
            // new Tracer(); : 
        }
    }

-   错误: Tracer.java:21:method calls are not allowed - only calls to
    BTraceUtils are allowed

-   错误: Tracer.java:22:object creation is not allowed

-   不要忘记 @BTrace 注解

**Target.java — 模拟的线上应用.**

    import java.util.concurrent.TimeUnit;

    public class Target {
        public static void main(String[] args) throws InterruptedException {
            try {
                TimeUnit.SECONDS.sleep(15);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(&#34;start main method...&#34;);
            while(true) {
                System.out.println(&#34;starting new thread...&#34;);
                new Thread(() -&gt; {
                    while (true) {
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }).start();
                TimeUnit.SECONDS.sleep(5);
            }
        }
    }

运行 Target.java 后，使用 jps -v 查看该程序 pid 号。

    &gt; jps -v

    21952 Target -javaagent:D:\program\JetBrains\IntelliJ IDEA017.2.2\lib\idea_rt.jar=55132:D:\program\JetBrains\IntelliJ
    IDEA 2017.2.2\bin -Dfile.encoding=UTF-8

使用 btrace 启用 Tracer.java。

    ## 首先切换到 Tracer.java 所在目录，然后执行
    &gt; btrace -v 21952 Tracer.java 

-   -v 打印出 DEBUG 信息，无 -v 只输出你自己想要输出的信息

可以看到在 Target.java 和 BTrace 的窗口都输出了调试信息。在 Target
中一旦一个新线程被创建，BTrace 便可以打印出相关信息。

继续 — BTrace 注解
==================

假设我们想诊断这样一个 add 方法：

**Target.java.**

    package com.foo.btraceDemo;

    import java.lang.management.ManagementFactory;
    import java.util.concurrent.TimeUnit;

    public class Target {
        private static final String pid = ManagementFactory.getRuntimeMXBean().getName().split(&#34;@&#34;)[0];

        public static void main(String[] args) throws InterruptedException {
            System.out.println(&#34;pid: &#34; &#43; pid);
            while(true) {
                System.out.println(&#34;准备调用 add 方法...&#34;);
                add(3, 5);
                System.out.println(&#34;调用 add 方法结束...&#34;);
            }
        }

        public static int add(int a, int b) throws InterruptedException {
            TimeUnit.SECONDS.sleep(5);
            a = a &#43; a;
            System.out.println(&#34;dododo&#34;);
            a = a / 2;
            return a &#43; b;
        }
    }

Method Annotations 方法注释
===========================

1.  @OnMethod

    **Tracer.java:func.**

        @OnMethod(clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;,      location = @Location(Kind.RETURN))
            public static void func(@Return int result, @Duration long     time, int paramA, int paramB) {
                println(&#34;param:&#34; &#43; paramA &#43; &#34;, &#34; &#43; paramB);
                println(&#34;result:&#34; &#43; result);
                println(&#34;costs(ms):&#34; &#43; time/1000/1000);
            }

    输出结果

        param:3, 5
        result:8
        costs(ms):5020
        param:3, 5
        result:8
        costs(ms):5013

    1.  如何通过 clazz/method 指定要诊断的方法

        1.  全限定名

                clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;

        2.  正则表达式

                clazz = &#34;&#43;java.sql.Statement&#34;, method = &#34;/execute($|Update|Query|Batch)/&#34;

        3.  接口/父类，注解

                clazz = &#34;&#43;java.sql.Statement&#34; // 匹配所有实现该接口或父类的类
                clazz = &#34;@org.springframework.stereotype.Controller&#34; // 匹配所有 @Controller 注解的类

        4.  构造函数

                clazz=&#34;java.lang.Throwable&#34;,method=&#34;&lt;init&gt;&#34;,
                    location=@Location(Kind.RETURN) // 匹配任何异常被构造完成准备抛出

        5.  静态内部类

                clazz=&#34;com.foo.bar$YourInnerClass&#34;, method=&#34;mName&#34;)

        6.  重载方法区别方法见下文

    2.  如何通过 @Location 指定诊断方法的时机

        1.  Kind.Entry Kind.Return

            -   Kind.Entry 方法进入时，为默认值。

            -   Kind.Return 方法完成时, 指定此 Kind 可以使用 @Duration
                获取方法耗时, @Return 获取方法返回结果

                    @OnMethod(clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;,    ocation = @Location(Kind.RETURN))
                    public static void func(@Return int result, @Duration long me,     int paramA, int paramB) {
                              println(&#34;param:&#34; &#43; paramA &#43; &#34;, &#34; &#43; paramB);
                        println(&#34;result:&#34; &#43; result);
                        println(&#34;costs(ms):&#34; &#43; time/1000/1000);
                    }

        2.  Kind.Error, Kind.Throw和 Kind.Catch

            -   Kind.Error: 异常抛出方法之外

            -   Kind.Throw: 异常被 throw 之处

            -   Kind.Catch: 异常被 catch 之处

                    @OnMethod(clazz = &#34;java.net.ServerSocket&#34;, method = &#34;bind&#34;, ocation = @Location(Kind.ERROR))
                    public static void onBind(Throwable exception, @Duration long uration) // 这种写法待验证

        3.  Kind.Call与Kind.Line

                Kind.ENTRY （默认值）只关注目标方法（OnMethod 的clazz/method），Kind.CALL 关注目标方法中，调用的其它哪些类哪些方法（Location的class，method）。注意，一般网上的教程，在描述 Kind.ENTRY 和 Kind.CALL 时，会将 CALL 描述为“还关注目标方法中其它方法的调用”，但是经过验证，此处不是“还”的关系，而是过滤的逻辑。也就是说，指定了这里的 clazz 和 method（例如 MethodB），将忽略目标方法（MethodA）的其它逻辑的时间。示例代码如下：

            **Target.java — add.**

                public int add(int a, int b) throws InterruptedException {
                    TimeUnit.SECONDS.sleep(5);
                    a = a &#43; a;
                    System.out.println(&#34;dododo&#34;);
                    a = a / 2;
                    return a &#43; b;
                }

            **Kind.ENTRY.**

                @OnMethod(clazz = &#34;com.foo.btraceDemo.Target&#34;, method = &#34;add&#34;, location = @Location(value = Kind.RETURN))
                public static void add1(@Duration long time, int a, int b) {
                    print(&#34;add1 costs1(ns):&#34; &#43; time);
                    totalTime &#43;= time;
                    counter &#43;&#43;;
                    println(&#34;, average(ns): &#34; &#43; (totalTime/counter));
                }

                add1 costs1(ns):5053722750, average(ns): 5053722750
                add1 costs1(ns):5050729632, average(ns): 5052226191
                add1 costs1(ns):5036155712, average(ns): 5046869364

            **KIND.CALL 匹配 println.**

                @OnMethod(clazz = &#34;com.foo.btraceDemo.Target&#34;, method = &#34;add&#34;,
                            location = @Location(value = Kind.CALL, clazz = &#34;java.io.PrintStream&#34;, method = &#34;println&#34;, where = Where.AFTER))
                public static void add2(@Duration long time) {
                    print(&#34;add2 costs(ns):&#34; &#43; time);
                    totalTime &#43;= time;
                    counter &#43;&#43;;
                    println(&#34;, average(ns): &#34; &#43; (totalTime/counter));
                }

                add2 costs(ns):148064, average(ns): 148064
                add2 costs(ns):91911, average(ns): 119987
                add2 costs(ns):143314, average(ns): 127763

            **KIND.CALL 匹配 \*.**

                @OnMethod(clazz = &#34;com.foo.btraceDemo.Target&#34;, method = &#34;add&#34;,
                            location = @Location(value = Kind.CALL, clazz = &#34;/.*/&#34;, method = &#34;/.*/&#34;, where = Where.AFTER))
                public static void add3(@Duration long time) {
                    print(&#34;add3 costs(ns):&#34; &#43; time);
                    totalTime &#43;= time;
                    counter &#43;&#43;;
                    println(&#34;, average(ns): &#34; &#43; (totalTime/counter));
                }

                add3 costs(ns):5166920098, average(ns): 5166920098 
                add3 costs(ns):81295, average(ns): 2583500696
                add3 costs(ns):5160914306, average(ns): 3442638566
                add3 costs(ns):68445, average(ns): 2581996036
                add3 costs(ns):5075955184, average(ns): 3080787865
                add3 costs(ns):67048, average(ns): 2567334396
                add3 costs(ns):5060307945, average(ns): 2923473474
                add3 costs(ns):115098, average(ns): 2558053677
                add3 costs(ns):5276136213, average(ns): 2860062848
                add3 costs(ns):93308, average(ns): 2574065894

            -   注意这种写法打印的时间，你看出什么规律了吗?

2.  @OnTimer 定时任务，单位 ms, 没有什么好解释的。

        @OnTimer(5000)
        public static void run(){
            println(counter);
        }

3.  @OnEvent

        @OnEvent
        public static void defalutEvent(){
            println(&#34;default event sent...&#34;);
        }
        @OnEvent(&#34;speak&#34;)
        public static void namedEvent(){
            println(&#34;speak something...&#34;);
        }

    输出结果

        Please enter your option:
                1. exit
                2. send an event
                3. send a named event
                4. flush console output
        &gt;3
        Please enter the event name: &gt;speak
        speak something...
        &gt;1
        default event sent...

4.  @OnError

    Arguments Annotations 参数注释  
    jstat -J-Djstat.showUnsupported=true -name
    btrace.com.sun.btrace.samples.ThreadCounter.count &amp;lt;pid&amp;gt;

更多解释请参考 BTrace 本地安装路径下的帮助文档，或者 wiki

[wiki 页面](https://github.com/btraceio/btrace/wiki/BTrace-Annotations)

[Btrace使用小结](https://blog.csdn.net/lirenzuo/article/details/76576064)

[btrace记忆](http://agapple.iteye.com/blog/962119?spm=a2c4e.11153940.blogcont7569.25.3d5955c8anNsnZ)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/btrace-manual/  

