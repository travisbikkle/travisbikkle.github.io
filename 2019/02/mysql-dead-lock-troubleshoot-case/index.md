# 一次 Mysql 死锁问题解决


一个业务反应，环境多次出现大量服务不可使用，如app导入不响应，用户更新超时，bpm创单超时等等。查看数据库的processlist，发现有大量的处于Waiting
for table metadata
lock状态的查询，其中包含T\_APP\_INFO、TBL\_UM\_USER、T\_TICKET\_BASICINFO等表，跟故障服务一致，确定故障原因是数据库锁表引起；
业务自行导出所有的阻塞task，并按照阻塞时间排序，发现第一条引起阻塞的是一条来来自于localhost的
由root用户发起的批量锁表语句，疑似是问题根因。

上面这段是业务说的，已经排查的比较深入了，给个赞。

我之前通过直接kill掉这个query线程，他们的业务就正常走下去了，因为忙其他事情，所以就没有再关注。后面他们又出现了这个问题，这次必须要解决了。所以记录一下定位过程。

定位思路
========

1.  **\[WHAT\]** &lt;root@localhost&gt; 的进程在做什么？

    **MySQL 所有“卡住”问题，先看进程列表：.**

        show processlist;

        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;
        | Id      | User | Host      | db   | Command | Time | State    | Info             |
        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;
        | 3467133 | root | localhost | NULL | Query   |    320400 | Waiting for table metadata lock | LOCK TABLES `....|
        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;

    看到 &lt;root@localhost&gt; 的用户，有一条状态为 Waiting for table
    metadata lock 的查询。查询语句为“LOCK TABLES……”。

    猜测：是后台备份进程在锁表，由于也有可能业务自己登陆后台锁表，所以需要证明这个确实是备份工具发起的语句。

    证明：当前时间是2月12日下午，Time
    时间显示此语句已经等待320400s（约89小时），往前推算约为2月9日凌晨0点。后台备份文件夹有一个0点的文件夹，里面备份文件为0字节。

2.  **\[QUESTION\]** 为什么会导致这个问题出现

    在
    &lt;https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_databases&gt;
    第一句就有，mysqldump 如果不加
    --single-transaction，执行mysqldump的用户就必须有LOCK
    TABLES的权限，由此推出，这种情况（不加
    --single-transaction，当前MySQL方案）下，就会锁表。那么这种锁表行为分不分存储引擎呢？答案是myisam都会锁表，而innodb会在—single-transaction的时候不锁表，详见官网。

    由于业务在show processlist;的时候，还看到了有另外一条语句B

    select \* from activemq\_lock for update;

    那么即使有这条语句在执行，如果其正常提交了事务，也不会阻塞备份工具锁表。业务由于直接用的开源activemq，所以也说不清楚这个表的作用，那么问题出在哪里？

3.  **\[控制变量\]** 和正常的环境来对比

    顶着业务的各种不满（业务：我也学会kill进程了，你要解决根本问题啊，不能再kill了），直接再次kill掉这个进程，查看正常环境的状态。发现语句B始终存在，并且show
    processlist发现该语句的Time一直在变化。说明其一直在频繁执行。

    猜测：语句B一直在执行获取锁，mysqldump在备份前先 LOCK TABLES
    所有表，其它表都正常锁住，唯有这个表获取不到锁，就一直等待。而其它被锁住的表，此时是无法更新的。

    证明：当前环境再次手工执行一把备份，发现备份脚本卡住，查看processlist，发现与问题描述表现一致。此时业务果然发生了上面的问题。

解决方法
========

1.  疑问点

    早有耳闻mysqldump有—skip-lock-tables、--single-transation、--ignore-table的选项，但是由于不熟悉，所以还要自己验证一番，看看各个参数是不是如自己所想：

    -   --skip-lock-tables
        是跳过获取不到锁的表，还是备份前不加锁，还是备份的语句里面不加锁（曾被误导，以为是备份后的语句）

    -   --single-transaction
        看起来和锁没什么关系，能不能达到我们的目的呢？

    -   --ignore-table
        是忽略表和视图的意思，如果忽略这个表，那还会不会锁住这个表呢？

2.  验证

    带着上面的疑问去自己的测试数据库验证，首先了解本例涉及到的几种锁以及如何构造它们：

    **我们常使用的锁，语句一般就几种：.**

        flush tables with read lock;
        lock tables tablename read;
        select * from table name for update;
        -- 其它 share mode 等暂不谈，也不在此老生常谈排他锁、互斥锁、只读锁等的概念。

    研究如下：

    &lt;table style=&#34;width:100%;&#34;&gt;
    &lt;colgroup&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;/colgroup&gt;
    &lt;tbody&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;说明&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;影响&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;如何定位这种锁&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;如何释放&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;进阶&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;flush tables with read lock;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;全部的表都刷上read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;执行此语句的session，在修改数据会收到报错：ERROR1223 cant execute query because you hanve a conflicting lock.&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改数据会卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. 无法通过show open tables查看&lt;br /&gt;
    2. 无法通过information_schema.innodb_locks等表查看&lt;br /&gt;
    3. 无法通过show engine innodb status\G查看&lt;br /&gt;
    4. 其它被锁住的session，可以通过show processlist;查看到状态:Waiting for global read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;session结束或者unlock tables;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;在手工备份的时候很好用&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;lock tables t_test read;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;只对某一个表刷上read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. 执行此语句的session，在修改数据会收到报错：ERROR1099 table … was locked with read lock and cant be updated.&lt;br /&gt;
    2. 只能查询锁住的表，如果查询其它的表，也会失败&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改数据会卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. show open tables 可以看到表的 in_use &#43; 1&lt;br /&gt;
    2. 无法无法通过information_schema.innodb_locks等表查看&lt;br /&gt;
    3. show engine innodb status\G其中的 transactions 一列显示 mysql tables in use 1,locked 1&lt;br /&gt;
    4. 其它被锁住的session，可以通过show processlist;查看到状态: Wating for table metadata lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;session结束或者unlock tables;还有其它一些场景会释放锁，比如alter table，详见官网文档；&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;释放锁会默认提交事务，具体详见官网文档&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;select * from t_test for update;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;对某一个表刷上排他锁。 只能在一个事务中使用，不在事务中无效 ， 使用见附录&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;执行此语句的session，就是为了更改数据。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改数据会卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. show open tables 可以看到表的 in_use &#43; 1&lt;br /&gt;
    2. 无法无法通过information_schema.innodb_locks等表查看&lt;br /&gt;
    3. show engine innodb status\G其中的 transactions 一列显示 2 lock stucts, 2 row lock(表数据行&#43;1数量的锁)&lt;br /&gt;
    4. 其它被锁住的session，可以通过show processlist;查看到状态: Wating for table metadata lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;commit; 其它未commit的异常状态，锁也会随着session关闭释放掉，具体见官网文档&#34;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;行锁需要有主键或者索引 。本例无，所以是表锁的效果。&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

    经过组合 `select * from t_test for update` 和
    `lock tables t_test read;`
    重现了业务的问题。后续经过验证，上面提到的 mysqldump
    的三个参数，都可以达到目的：

    &lt;table&gt;
    &lt;colgroup&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;/colgroup&gt;
    &lt;tbody&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;方案&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;说明&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;缺点&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;1. mysqldump –skip-lock-tables&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;备份前，不加锁&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;无法保证数据一致性&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;2. mysqldump –single-transaction&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;备份在一个事务中进行&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;备份期间表定义变化等可能导致备份失败（重新执行一次备份即可）&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;3. mysqldump –ignore-table=activemq_lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;略过该表，不会获取锁&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;不备份该表&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

附录
====

**select…for update 的使用方法.**

    begin;
    select * from t_test for update;
    commit;

    begin;
    select * from t_test where id=1111 for update;
    commit;
    https://dev.mysql.com/doc/refman/5.7/en/select.html

**lock tables 的使用方法.**

    lock tables t_test read;

参考：
======

&lt;https://dev.mysql.com/doc/refman/5.7/en/select.html&gt;

&lt;https://dev.mysql.com/doc/refman/5.7/en/lock-tables.html&gt;

&lt;https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_databases&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/02/mysql-dead-lock-troubleshoot-case/  

