# 使用 Ambari 安装 Hdp 集群


HDP 并不是 hadoop 的辅音简称，而是 Hortonworks 的产品 [Hortonworks Data
Platform](https://community.hortonworks.com/questions/89821/difference-between-apache-hadoop-and-hdp.html)
的简称，是包含 Hadoop 在内的一揽子解决方案。

前置要求：
==========

3-4台 CentOS 7 机器，其中一台机器必须安装 Ambari
服务。教程参考link:/2018/10/13/centos 7 安装 apache-ambari/\[centos 7
安装 apache-ambari\]。安装 master 和 slave 的节点机器，内存最好不要小于
5G。

安装部件：
==========

如前所述，此次安装包含如下服务（请按需安装）：

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;服务&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;版本&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;说明&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HDFS&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 分布式文件系统&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;YARN &#43; MapReduce2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 下一代 MapReduce(YARN)&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Tez&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.7.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Tez 是运行在 YARN 之上的下一代 Hadoop 查询处理框架&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Hive&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.2.1000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;支持即席查询与大数据量分析和存储管理服务的数据仓库系统&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HBase&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.1.2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;非关系型分布式数据库，包括 Phoenix，一个为低延迟应用开发的高性能 sql 扩展&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Pig&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.16.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;分析大数据量的脚本平台&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Sqoop&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.4.6&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;在 Apache Hadoop 和 其它结构化的数据存储位置例如关系数据库 之间批量传递数据的工具&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Oozie&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;4.2.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 的工作引擎之一，另一个是 Azkaban。负责工作流的协调和执行。会按照一个可选的 Oozie Web 客户端，依赖此也会安装 ExtJS 库&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Zookeeper&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3.4.6&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高可用的分布式协调服务&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Falcon&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.10.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;数据管理和处理平台&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Storm&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 流处理框架https://www.cnblogs.com/Jack47/p/storm_intro-1.html[Storm 介绍]&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Flume&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.5.2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;收集，聚合和移动大量流式数据到 HDFS 的分布式服务&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Accumulo&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.7.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高可靠，性能和伸缩性的 Key/Value 存储[各种KV工具对比]https://kkovacs.eu/cassandra-vs-mongodb-vs-couchdb-vs-redis)&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Ambari Infra&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Ambari 管理的部件所使用的核心共享服务&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Ambari Metrics&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Ambari 集群性能监控工具&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Atlas&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.8.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;元数据管理平台&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Kafka&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.0.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高吞吐量的分布式消息系统&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Knox&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.12.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;一个 rest 类型的认证系统，可提供单点登录认证&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Log Search(未安装)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.5.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;日志聚合，分析，可视化&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;SmartSense&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.4.5.2.6.2.2-1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;一款不得不装的 Hortonworks 增值服务，集群诊断功能&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Spark&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.6.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;快速的大规模数据处理引擎&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Spark2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.3.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;https://stackoverflow.com/questions/40168779/apache-spark-vs-apache-spark-2&#34;&gt;spark spark2 对比&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Zeppelin NoteBook&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Web 界面的数据分析系统，可以使用 sql 和 scala 等&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Druid&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.10.1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;快速的列存储分布式系统&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Mahout&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.9.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache 开源机器学习算法库，提供协作筛选（CF，推荐算法），聚类（clustering），分类(classification)实现&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Slider&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.92.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;部署，管理与监控 YARN 上的应用程序&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Superset&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.15.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Airbnb 的开源可视化的数据平台&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

====== 在 *确认主机 Confirm Hosts* 阶段，即使你的 openssl
是最新的，还是可能会报如下错误：

    NetUtil.py:96 EOF occured in violation of protocol (_ssl.c:579)
    和
    SSLError: Failed to connect.Please check openssl library version.

此时需要在每一台节点上加入以下配置：

    vi /etc/ambari-agent/conf/ambari-agent.ini

    [security] ## 在此部分加入以下一行
    force_https_protocol=PROTOCOL_TLSv1_2

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/ambari-hdp-demo/  

