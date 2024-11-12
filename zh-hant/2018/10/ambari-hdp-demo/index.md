# 使用 Ambari 安裝 Hdp 集羣


HDP 並不是 hadoop 的輔音簡稱，而是 Hortonworks 的產品 [Hortonworks Data
Platform](https://community.hortonworks.com/questions/89821/difference-between-apache-hadoop-and-hdp.html)
的簡稱，是包含 Hadoop 在內的一攬子解決方案。

前置要求：
==========

3-4臺 CentOS 7 機器，其中一臺機器必須安裝 Ambari
服務。教程參考link:/2018/10/13/centos 7 安裝 apache-ambari/\[centos 7
安裝 apache-ambari\]。安裝 master 和 slave 的節點機器，內存最好不要小於
5G。

安裝部件：
==========

如前所述，此次安裝包含如下服務（請按需安裝）：

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;col style=&#34;width: 33%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;服務&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;版本&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;說明&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HDFS&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 分佈式文件系統&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;YARN &#43; MapReduce2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 下一代 MapReduce(YARN)&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Tez&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.7.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Tez 是運行在 YARN 之上的下一代 Hadoop 查詢處理框架&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Hive&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.2.1000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;支持即席查詢與大數據量分析和存儲管理服務的數據倉庫系統&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HBase&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.1.2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;非關係型分佈式數據庫，包括 Phoenix，一個爲低延遲應用開發的高性能 sql 擴展&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Pig&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.16.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;分析大數據量的腳本平臺&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Sqoop&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.4.6&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;在 Apache Hadoop 和 其它結構化的數據存儲位置例如關係數據庫 之間批量傳遞數據的工具&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Oozie&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;4.2.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 的工作引擎之一，另一個是 Azkaban。負責工作流的協調和執行。會按照一個可選的 Oozie Web 客戶端，依賴此也會安裝 ExtJS 庫&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Zookeeper&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3.4.6&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高可用的分佈式協調服務&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Falcon&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.10.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;數據管理和處理平臺&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Storm&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache Hadoop 流處理框架https://www.cnblogs.com/Jack47/p/storm_intro-1.html[Storm 介紹]&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Flume&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.5.2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;收集，聚合和移動大量流式數據到 HDFS 的分佈式服務&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Accumulo&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.7.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高可靠，性能和伸縮性的 Key/Value 存儲[各種KV工具對比]https://kkovacs.eu/cassandra-vs-mongodb-vs-couchdb-vs-redis)&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Ambari Infra&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Ambari 管理的部件所使用的核心共享服務&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Ambari Metrics&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.1.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Ambari 集羣性能監控工具&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Atlas&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.8.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;元數據管理平臺&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Kafka&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.0.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;高吞吐量的分佈式消息系統&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Knox&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.12.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;一個 rest 類型的認證系統，可提供單點登錄認證&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Log Search(未安裝)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.5.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;日誌聚合，分析，可視化&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;SmartSense&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.4.5.2.6.2.2-1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;一款不得不裝的 Hortonworks 增值服務，集羣診斷功能&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Spark&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.6.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;快速的大規模數據處理引擎&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Spark2&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.3.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;https://stackoverflow.com/questions/40168779/apache-spark-vs-apache-spark-2&#34;&gt;spark spark2 對比&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Zeppelin NoteBook&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.7.3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Web 界面的數據分析系統，可以使用 sql 和 scala 等&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Druid&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.10.1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;快速的列存儲分佈式系統&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Mahout&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.9.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Apache 開源機器學習算法庫，提供協作篩選（CF，推薦算法），聚類（clustering），分類(classification)實現&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;Slider&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.92.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;部署，管理與監控 YARN 上的應用程序&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Superset&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.15.0&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;Airbnb 的開源可視化的數據平臺&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

====== 在 *確認主機 Confirm Hosts* 階段，即使你的 openssl
是最新的，還是可能會報如下錯誤：

    NetUtil.py:96 EOF occured in violation of protocol (_ssl.c:579)
    和
    SSLError: Failed to connect.Please check openssl library version.

此時需要在每一臺節點上加入以下配置：

    vi /etc/ambari-agent/conf/ambari-agent.ini

    [security] ## 在此部分加入以下一行
    force_https_protocol=PROTOCOL_TLSv1_2

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/ambari-hdp-demo/  

