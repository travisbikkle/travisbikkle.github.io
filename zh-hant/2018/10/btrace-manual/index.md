# Btrace 使用教程

**BTrace** 是 Java
的一個診斷工具，可以在不重啓應用的情況下，對應用進行時間耗費、參數及結果跟蹤、方法調用跟蹤等分析。

BTrace 術語
===========

Probe Point  
位置

Trace Actions or Actions  
追蹤語句

Action Methods  
追蹤語句所在的靜態方法

BTrace 程序結構
===============

一個 **BTrace 程序** 是一個 Java 類，包含數個由 `BTrace 註解` 註釋的
`public static void` 方法。這些註解被用來指定被追蹤程序的 **Probe
Point**. **Tracing Actions**
在這些靜態方法內定義。這些靜態方法也即上文提到的 **Action Methods**。

BTrace 的限制
=============

1.  不能 創建對象，數組。

2.  不能 拋出、捕獲異常。

3.  不能 調用任意實例或靜態方法，只能調用 BTraceUtils 中的方法。

4.  不能 修改目標程序的靜態或實例變量，不過 BTrace 程序自己不做限制。

5.  不能 有實例變量或方法，方法不能有返回值類型，BTrace
    程序的所有方法必須是 public static 1. oid 的，所有的字段都必須是
    static 的。

6.  不能 有 outer, inner, nested 或 local 類。

7.  不能 用 synchronized 關鍵字。

8.  不能 有循環 (for, while, do..while)。

9.  不能 繼承任何類 (即父類只能是 java.lang.Object)。

10. 不能 實現接口。

11. 不能 包含 assert 語句.

12. 不能 使用 class literals.

雖然不需要重啓就可以使用 BTrace,
但是使用不當仍然會導致應用終止，比如目標程序可能會發生下面這種錯誤:

    Exception in thread &#34;main&#34; java.lang.NoSuchMethodError: com.foo.btraceDemo.Target.$btrace$com$foo$btraceDemo$Tracer$func(Ljava/lang/Object;IJII)V

一個簡單的 `BTrace 程序` 演示
=============================

**Tracer.java — BTrace 要執行的程序（腳本）.**

    // 導入所有 BTrace 註解。
    import com.sun.btrace.annotations.*;
    // 導入 BTraceUtils 的靜態方法。
    import static com.sun.btrace.BTraceUtils.*;

    // @BTrace annotation tells that this is a BTrace program
    @BTrace 
    public class HelloWorld {

        // @OnMethod 註解表明 Probe Point(位置)。
        // 在這個例子中，我們對進入 Thread.start() 方法感興趣。
        @OnMethod(
            clazz=&#34;java.lang.Thread&#34;,
            method=&#34;start&#34;
        )
        public static void func() {
            // println 是 BTraceUtils 的靜態方法
            println(&#34;about to start a thread!&#34;);
            // System.out.println(&#34;123&#34;); 
            // new Tracer(); : 
        }
    }

-   錯誤: Tracer.java:21:method calls are not allowed - only calls to
    BTraceUtils are allowed

-   錯誤: Tracer.java:22:object creation is not allowed

-   不要忘記 @BTrace 註解

**Target.java — 模擬的線上應用.**

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

運行 Target.java 後，使用 jps -v 查看該程序 pid 號。

    &gt; jps -v

    21952 Target -javaagent:D:\program\JetBrains\IntelliJ IDEA017.2.2\lib\idea_rt.jar=55132:D:\program\JetBrains\IntelliJ
    IDEA 2017.2.2\bin -Dfile.encoding=UTF-8

使用 btrace 啓用 Tracer.java。

    ## 首先切換到 Tracer.java 所在目錄，然後執行
    &gt; btrace -v 21952 Tracer.java 

-   -v 打印出 DEBUG 信息，無 -v 只輸出你自己想要輸出的信息

可以看到在 Target.java 和 BTrace 的窗口都輸出了調試信息。在 Target
中一旦一個新線程被創建，BTrace 便可以打印出相關信息。

繼續 — BTrace 註解
==================

假設我們想診斷這樣一個 add 方法：

**Target.java.**

    package com.foo.btraceDemo;

    import java.lang.management.ManagementFactory;
    import java.util.concurrent.TimeUnit;

    public class Target {
        private static final String pid = ManagementFactory.getRuntimeMXBean().getName().split(&#34;@&#34;)[0];

        public static void main(String[] args) throws InterruptedException {
            System.out.println(&#34;pid: &#34; &#43; pid);
            while(true) {
                System.out.println(&#34;準備調用 add 方法...&#34;);
                add(3, 5);
                System.out.println(&#34;調用 add 方法結束...&#34;);
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

Method Annotations 方法註釋
===========================

1.  @OnMethod

    **Tracer.java:func.**

        @OnMethod(clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;,      location = @Location(Kind.RETURN))
            public static void func(@Return int result, @Duration long     time, int paramA, int paramB) {
                println(&#34;param:&#34; &#43; paramA &#43; &#34;, &#34; &#43; paramB);
                println(&#34;result:&#34; &#43; result);
                println(&#34;costs(ms):&#34; &#43; time/1000/1000);
            }

    輸出結果

        param:3, 5
        result:8
        costs(ms):5020
        param:3, 5
        result:8
        costs(ms):5013

    1.  如何通過 clazz/method 指定要診斷的方法

        1.  全限定名

                clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;

        2.  正則表達式

                clazz = &#34;&#43;java.sql.Statement&#34;, method = &#34;/execute($|Update|Query|Batch)/&#34;

        3.  接口/父類，註解

                clazz = &#34;&#43;java.sql.Statement&#34; // 匹配所有實現該接口或父類的類
                clazz = &#34;@org.springframework.stereotype.Controller&#34; // 匹配所有 @Controller 註解的類

        4.  構造函數

                clazz=&#34;java.lang.Throwable&#34;,method=&#34;&lt;init&gt;&#34;,
                    location=@Location(Kind.RETURN) // 匹配任何異常被構造完成準備拋出

        5.  靜態內部類

                clazz=&#34;com.foo.bar$YourInnerClass&#34;, method=&#34;mName&#34;)

        6.  重載方法區別方法見下文

    2.  如何通過 @Location 指定診斷方法的時機

        1.  Kind.Entry Kind.Return

            -   Kind.Entry 方法進入時，爲默認值。

            -   Kind.Return 方法完成時, 指定此 Kind 可以使用 @Duration
                獲取方法耗時, @Return 獲取方法返回結果

                    @OnMethod(clazz=&#34;com.foo.btraceDemo.Target&#34;, method=&#34;add&#34;,    ocation = @Location(Kind.RETURN))
                    public static void func(@Return int result, @Duration long me,     int paramA, int paramB) {
                              println(&#34;param:&#34; &#43; paramA &#43; &#34;, &#34; &#43; paramB);
                        println(&#34;result:&#34; &#43; result);
                        println(&#34;costs(ms):&#34; &#43; time/1000/1000);
                    }

        2.  Kind.Error, Kind.Throw和 Kind.Catch

            -   Kind.Error: 異常拋出方法之外

            -   Kind.Throw: 異常被 throw 之處

            -   Kind.Catch: 異常被 catch 之處

                    @OnMethod(clazz = &#34;java.net.ServerSocket&#34;, method = &#34;bind&#34;, ocation = @Location(Kind.ERROR))
                    public static void onBind(Throwable exception, @Duration long uration) // 這種寫法待驗證

        3.  Kind.Call與Kind.Line

                Kind.ENTRY （默認值）只關注目標方法（OnMethod 的clazz/method），Kind.CALL 關注目標方法中，調用的其它哪些類哪些方法（Location的class，method）。注意，一般網上的教程，在描述 Kind.ENTRY 和 Kind.CALL 時，會將 CALL 描述爲“還關注目標方法中其它方法的調用”，但是經過驗證，此處不是“還”的關係，而是過濾的邏輯。也就是說，指定了這裏的 clazz 和 method（例如 MethodB），將忽略目標方法（MethodA）的其它邏輯的時間。示例代碼如下：

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

            -   注意這種寫法打印的時間，你看出什麼規律了嗎?

2.  @OnTimer 定時任務，單位 ms, 沒有什麼好解釋的。

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

    輸出結果

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

    Arguments Annotations 參數註釋  
    jstat -J-Djstat.showUnsupported=true -name
    btrace.com.sun.btrace.samples.ThreadCounter.count &amp;lt;pid&amp;gt;

更多解釋請參考 BTrace 本地安裝路徑下的幫助文檔，或者 wiki

[wiki 頁面](https://github.com/btraceio/btrace/wiki/BTrace-Annotations)

[Btrace使用小結](https://blog.csdn.net/lirenzuo/article/details/76576064)

[btrace記憶](http://agapple.iteye.com/blog/962119?spm=a2c4e.11153940.blogcont7569.25.3d5955c8anNsnZ)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/btrace-manual/  

