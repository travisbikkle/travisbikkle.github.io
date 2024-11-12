# Hive .. 安裝常見問題

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

安裝問題
========

1.  remote 模式報錯 Java.lang.RuntimeException: Unable to instantiate
    org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
    解決：hive 需要先 `hive --service metastore` 先啓動 thrift
    server，才能訪問 MySQL 參考：[官方手冊：Hive Metastore
    配置](https://cwiki.apache.org/confluence/display/Hive/AdminManual&#43;MetastoreAdmin#AdminManualMetastoreAdmin-RemoteMetastoreDatabase)
    理解：MySQL 爲 metastore 的 database， Thrift Server 爲 metastore
    的服務器

2.  hive --service metastore 啓動報錯 Unable to open a test connection
    to the given database 解決：MySQL 的配置有問題  
    場景1：MySQL 只允許本地訪問  
    場景2：MySQL 白名單未添加相應機器  
    參考：

    &gt; [如何確定 MySQL 使用的配置文件](/2018/10/13/)  
    &gt; link:/2018/10/13/MySQL 訪問常見問題\[MySQL 訪問常見問題\]  
    &gt; [Unable to open a test connection to the given
    &gt; database](http://hadooptutorial.info/unable-open-test-connection-given-database/)

3.  報錯 Version infomation not found in metastore 原因：hive 0.12
    以後版本會驗證 metastore version，metastore 中無該信息，因此無法訪問
    解決：schematool -dbType mysql -initSchema 刷庫

4.  警告 ssl 連接 MySQL 的信息 jdbc 連接串添加 &amp;useSSL=false
    即可，注意在 xml 中的轉義（寫成 `&#43;&amp;amp;useSSL=false&#43;`）。

5.  hive on mr is deprecated in hive 2, consider using a different
    execution engine like spark. or using a hive 1.x version  
    [hive spark tez
    對比](https://www.slideshare.net/MichTalebzadeh1/query-engines-for-hive-mr-spark-tez-with-llap-considerations)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/hive-install-faq/  

