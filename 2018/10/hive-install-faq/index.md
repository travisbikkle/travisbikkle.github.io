# Hive .. 安装常见问题

remote 模式最小配置
===================

    &lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34; standalone=&#34;no&#34;?&gt;
    &lt;?xml-stylesheet type=&#34;text/xsl&#34; href=&#34;configuration.xsl&#34;?&gt;
    &lt;configuration&gt;
    &lt;property&gt;
    &lt;name&gt;javax.jdo.option.ConnectionURL&lt;/name&gt;
    &lt;value&gt;jdbc:mysql://192.168.47.128:3306/hive?createDatabaseIfNotExist=true&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;javax.jdo.option.ConnectionDriverName&lt;/name&gt;
    &lt;value&gt;com.mysql.jdbc.Driver&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;javax.jdo.option.ConnectionUserName&lt;/name&gt;
    &lt;value&gt;root&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;javax.jdo.option.ConnectionPassword&lt;/name&gt;
    &lt;value&gt;km717070&lt;/value&gt;
    &lt;/property&gt;
    &lt;/configuration&gt;

安装问题
========

1.  remote 模式报错 Java.lang.RuntimeException: Unable to instantiate
    org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
    解决：hive 需要先 `hive --service metastore` 先启动 thrift
    server，才能访问 MySQL 参考：[官方手册：Hive Metastore
    配置](https://cwiki.apache.org/confluence/display/Hive/AdminManual&#43;MetastoreAdmin#AdminManualMetastoreAdmin-RemoteMetastoreDatabase)
    理解：MySQL 为 metastore 的 database， Thrift Server 为 metastore
    的服务器

2.  hive --service metastore 启动报错 Unable to open a test connection
    to the given database 解决：MySQL 的配置有问题  
    场景1：MySQL 只允许本地访问  
    场景2：MySQL 白名单未添加相应机器  
    参考：

    &gt; [如何确定 MySQL 使用的配置文件](/2018/10/13/)  
    &gt; link:/2018/10/13/MySQL 访问常见问题\[MySQL 访问常见问题\]  
    &gt; [Unable to open a test connection to the given
    &gt; database](http://hadooptutorial.info/unable-open-test-connection-given-database/)

3.  报错 Version infomation not found in metastore 原因：hive 0.12
    以后版本会验证 metastore version，metastore 中无该信息，因此无法访问
    解决：schematool -dbType mysql -initSchema 刷库

4.  警告 ssl 连接 MySQL 的信息 jdbc 连接串添加 &amp;useSSL=false
    即可，注意在 xml 中的转义（写成 `&#43;&amp;amp;useSSL=false&#43;`）。

5.  hive on mr is deprecated in hive 2, consider using a different
    execution engine like spark. or using a hive 1.x version  
    [hive spark tez
    对比](https://www.slideshare.net/MichTalebzadeh1/query-engines-for-hive-mr-spark-tez-with-llap-considerations)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/hive-install-faq/  

