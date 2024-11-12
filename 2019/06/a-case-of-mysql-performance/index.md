# Mysql 性能测试报告惨不忍睹的一次原因


背景
====

新版本发布在即，新增与kafka，zk，redis等其它服务合设的场景，因此需要性能测试探个底。但是性能测试人员反馈，分设与合设性能差距数万倍。

问题
====

1.  安装两套某版本的MySQL，分别是与redis等服务合设的一套（32U64G），MySQL单独安装的一套（8U16G）。

2.  使用jmeter并且采用同样的测试模型，对服务进行长稳测试，其中合设的一套资源，其它服务也会跑长稳。

3.  观察jmeter报告，发现合设的MySQL中，平均时间average为20s以上，throughput在10个/秒以下。而独立的MySQL，average为ms级别，throughput为数万个。

此时应该随报告提供环境cpu，磁盘，内存，网络带宽，测试模型等情况。但是由于某些原因，测试同学无法提供。

测试模型和环境
==============

该测试模型很简单，1个int主键，20个varchar（256）字段共100w的数据。
使用select \* from xxx where id in random(1, 100w); 进行1000并发测试。

因为某些原因，测试同学无法提供定位问题需要的帮助，因此，我们只收集了以下几种信息。

-   cpu及内存（top）

-   iostat -x 1 以及其它 /dev/mysqllv(mysql 数据挂载磁盘)

-   查看jmeter并发服务器的jmap histro

-   查看mysql show processlist，show status like &#34;%Thread%&#34;等

定位过程
========

1.  cpu，内存等资源占用不高。

2.  iostat 磁盘读写速度不高，但是%util占比居高不下

3.  单独执行一条sql语句，偶尔很快，偶尔很慢

4.  show processlist反馈，大量连接停留在freeing
    items的状态，说明连接在等候io操作，与上述iostat结论不谋而合

5.  hdparm -Tt 合设机器只有60m/s的磁盘速度（未停服务，不准确）

6.  查询审计日志，发现审计日志大量积压，5M/2分钟的速度在持续增加

7.  走管理手段，要求测试人员提供独立 MySQL
    机器信息，其反馈非其本人测试并且已卸载。我们强烈要求重新安装独立mysql

8.  独立mysql表现也慢，与合设无异（说明测试没有控制好变量）

9.  最终发现，其独立mysql测试版本为较老版本

结论
====

新版本mysql开启audit日志，并且安全部门参照公司《mysql
安全加固规范》中的必须修改项——审计日志的策略 audit\_log\_strategy 必须为
`SYNCHRONOUS` ,
利用自动化工具扫描并发现我们的mysql没有开启此项，提了单。修改人员在修改时因为经验不足，未多想，便直接修改。

该项的另一个可选值为ASYNCHRONOUS，即异步。与主从复制的同步，半同步及异步类似，作用不言自明。改为该值后，问题消失。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/06/a-case-of-mysql-performance/  

