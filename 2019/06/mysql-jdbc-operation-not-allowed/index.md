# Mysql Jdbc 报错 Operation Not Allowed After Statement Closed


背景
====

业务上来说，连你们的mysql连不上，连别人的（其它部门的mysql）都能连上。查看其日志，报了一行错误&#34;No
operations allowed after statement closed&#34;。
这句话的意思，说的很清楚，不应该用关闭后的statement执行查询。但是因为我们是服务化的mysql，公司在遇到问题甩锅给其它部门的习惯由来已久，所以还是要帮业务解决。

大致代码
========

    class Main {
        private static connection;
        private static statement;

        static {
            connection = getConnection();
            statement = connection.createStatement();
        }
        public static query(){
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }
    }

定位
====

首先，业务反馈的是“连接不上”，但是报错位置其实是在executeUpdate一行，在此鄙视这些人，日志都不看清楚就开始推脱责任。在其代码中加入debug日志

        public static query(){
            log(statement.isClosed());
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }

发现日志打了两次，第一次为false，第二次为true并报错。
查看该类引用位置，发现为一个定义了init-method的bean，类似于

    &lt;bean id=&#34;xxxx&#34; class=&#34;xxxx&#34; init-method=&#34;query&#34; /&gt;

最终建议：

1.  将bean的scope更改为singleton

2.  重构Main类类似如下

&lt;!-- --&gt;

    class Main {
        private static connection;
        private static statement;

        private static void getConnection  {
            DriverManager.xxxx
            statement = connection.createStatement();
        }
        public static query(){
            getConnection();
            statement.executeUpdate(&#34;xxxx&#34;);
            statement.close();
        }
    }

其实有更好的写法，但是每个人的时间都很宝贵，没有义务和必要为你解决你自己的问题。而且从结果来看，其所谓的别人的数据库都能连上其实是不可能的。在帮其定位过程中，看git发现这些有问题的代码都是新增代码，所以，做数据库真的累。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/06/mysql-jdbc-operation-not-allowed/  

