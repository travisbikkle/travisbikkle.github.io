# Hadoop .. 学习笔记


HDFS读写流程
============

todo

HDFS文件权限
============

todo

安全模式
========

todo

注意事项
========

todo

JDK 版本应该使用 1.8，JDK 10 遇到启动过程中 warning 并且 datanode
无法启动的问题。

集群安装
========

最小配置文件（hadoop 2.9.1）
----------------------------

**core-site.xml.**

    &lt;configuration&gt;
            &lt;property&gt;
                    &lt;name&gt;fs.defaultFS&lt;/name&gt;
                    &lt;value&gt;hdfs://linux-1:8020/&lt;/value&gt;
                    &lt;description&gt;NameNode URI&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;io.file.buffer.size&lt;/name&gt;
                    &lt;value&gt;131072&lt;/value&gt;
                    &lt;description&gt;Buffer size&lt;/description&gt;
            &lt;/property&gt;
    &lt;/configuration&gt;

**hdfs-site.xml.**

    &lt;configuration&gt;
            &lt;property&gt;
                    &lt;name&gt;dfs.secondary.http.address&lt;/name&gt;
                    &lt;value&gt;linux-2:50090&lt;/value&gt;
            &lt;/property&gt;
            &lt;property&gt;
                    &lt;name&gt;dfs.http.address&lt;/name&gt;
                    &lt;value&gt;linux-1:50070&lt;/value&gt;
            &lt;/property&gt;
            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.name.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/namenode&lt;/value&gt;
                    &lt;description&gt;NameNode directory for namespace and transaction logs storage.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.edits.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/namenode&lt;/value&gt;
                    &lt;description&gt;DFS name node should store the transaction (edits) file.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.datanode.data.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/datanode&lt;/value&gt;
                    &lt;description&gt;DataNode directory&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.checkpoint.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/secondarynamenode&lt;/value&gt;
                    &lt;description&gt;Secondary Namenode directory&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.edits.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/namenode&lt;/value&gt;
                    &lt;description&gt;DFS name node should store the transaction (edits) file.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.datanode.data.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/datanode&lt;/value&gt;
                    &lt;description&gt;DataNode directory&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.checkpoint.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/secondarynamenode&lt;/value&gt;
                    &lt;description&gt;Secondary Namenode directory&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.checkpoint.edits.dir&lt;/name&gt;
                    &lt;value&gt;file:///opt/hdfs/secondarynamenode&lt;/value&gt;
                    &lt;description&gt;DFS secondary name node should store the temporary edits to merge.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.checkpoint.period&lt;/name&gt;
                    &lt;value&gt;7200&lt;/value&gt;
                    &lt;description&gt;The number of seconds between two periodic checkpoints.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.checkpoint.txns&lt;/name&gt;
                    &lt;value&gt;1000000&lt;/value&gt;
                    &lt;description&gt;SecondaryNode or CheckpointNode will create a checkpoint of namespace every 1000000 transactions&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.replication&lt;/name&gt;
                    &lt;value&gt;2&lt;/value&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.permissions&lt;/name&gt;
                    &lt;value&gt;false&lt;/value&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.datanode.use.datanode.hostname&lt;/name&gt;
                    &lt;value&gt;true&lt;/value&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;dfs.namenode.datanode.registration.ip-hostname-check&lt;/name&gt;
                    &lt;value&gt;true&lt;/value&gt;
            &lt;/property&gt;
    &lt;/configuration&gt;

**mapred-site.xml.**

    &lt;configuration&gt;
            &lt;property&gt;
                    &lt;name&gt;mapreduce.framework.name&lt;/name&gt;
                    &lt;value&gt;yarn&lt;/value&gt;
                    &lt;description&gt;MapReduce framework name&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;mapreduce.jobhistory.address&lt;/name&gt;
                    &lt;value&gt;linux-1:10020&lt;/value&gt;
                    &lt;description&gt;Default port is 10020.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;mapreduce.jobhistory.webapp.address&lt;/name&gt;
                    &lt;value&gt;linux-1:19888&lt;/value&gt;
                    &lt;description&gt;Default port is 19888.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;mapreduce.jobhistory.intermediate-done-dir&lt;/name&gt;
                    &lt;value&gt;/mr-history/tmp&lt;/value&gt;
                    &lt;description&gt;Directory where history files are written by MapReduce jobs.&lt;/description&gt;
            &lt;/property&gt;

            &lt;property&gt;
                    &lt;name&gt;mapreduce.jobhistory.done-dir&lt;/name&gt;
                    &lt;value&gt;/mr-history/done&lt;/value&gt;
                    &lt;description&gt;Directory where history files are managed by the MR JobHistory Server.&lt;/description&gt;
            &lt;/property&gt;
    &lt;/configuration&gt;

**yarn-site.xml.**

    &lt;configuration&gt;

    &lt;!-- Site specific YARN configuration properties --&gt;
        &lt;property&gt;
                &lt;name&gt;yarn.nodemanager.aux-services&lt;/name&gt;
                &lt;value&gt;mapreduce_shuffle&lt;/value&gt;
                &lt;description&gt;Yarn Node Manager Aux Service&lt;/description&gt;
        &lt;/property&gt;

        &lt;property&gt;
                &lt;name&gt;yarn.nodemanager.aux-services.mapreduce.shuffle.class&lt;/name&gt;
                &lt;value&gt;org.apache.hadoop.mapred.ShuffleHandler&lt;/value&gt;
        &lt;/property&gt;

        &lt;property&gt;
                &lt;name&gt;yarn.nodemanager.local-dirs&lt;/name&gt;
                &lt;value&gt;file:///opt/yarn/local&lt;/value&gt;
        &lt;/property&gt;

        &lt;property&gt;
                &lt;name&gt;yarn.nodemanager.log-dirs&lt;/name&gt;
                &lt;value&gt;file:///opt/yarn/logs&lt;/value&gt;
        &lt;/property&gt;

    &lt;/configuration&gt;

**hadoop-env.sh.**

    ##update this line
    export JAVA_HOME=/opt/jdk1.8.0_181
    ##add this to last
    export HADOOP_HOME=/opt/hadoop-2.9.1
    export HADOOP_CONF_DIR=/opt/hadoop-2.9.1/etc/hadoop
    export HADOOP_LOG_DIR=${HADOOP_HOME}/logs

**/etc/profile.**

    export HADOOP_INSTALL=/opt/hadoop-2.9.1
    export PATH=$PATH:$HADOOP_INSTALL/bin
    export PATH=$PATH:$HADOOP_INSTALL/sbin
    export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
    export HADOOP_COMMON_HOME=$HADOOP_INSTALL
    export HADOOP_HDFS_HOME=$HADOOP_INSTALL
    export YARN_HOME=$HADOOP_INSTALL
    export HADOOP_CONF_DIR=$HADOOP_INSTALL/etc/hadoop
    export HADOOP_PREFIX=$HADOOP_INSTALL

启动命令
========

start-all.sh (废弃)

NameNode
--------

start-dfs.sh  
&lt;http://192.168.44.128:50070/dfshealth.html#tab-overview&gt;

ResourceManager
---------------

start-yarn.sh  
&lt;http://192.168.44.128:8088/cluster&gt;

JobHistoryServer
----------------

\`mr-jobhistory-daemon.sh --config /opt/hadoop-2.9.1/etc/hadoop start
historyserver \` &lt;http://192.168.44.128:19888/jobhistory&gt;

参考
====

&gt; &lt;http://gaurav3ansal.blogspot.com/2018/06/install-hadoop-291-pseudo-distributed.html&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/hadoop-learning-note/  

