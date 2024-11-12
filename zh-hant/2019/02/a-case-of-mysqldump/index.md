# Mysqldump 使用案例


背景
====

客戶N在使用H部門提供的MySQL遇到部分性能問題後，未得到H部門的及時支撐。機緣巧合，我們的服務化MySQL剛剛發佈第一版，客戶N有意切換我們的MySQL。由於部門策略調整，我們準備由原來的社區MySQL切換爲部門R的商業版MySQL，其間對接問題不提，客戶提出的首要問題是前期嘗試通過mysqldump備份數據，發現有報錯並且很慢，我們的策略是
爲拓展業務先把鍋接下來吧 答應先提供數據遷移方案供客戶評估。

機器、數據、應用情況
====================

1.  源機器cpu核心數16，內存32G；

2.  兩臺機器，一個是master，一個是slave；未配置互爲主備；

3.  開啓了基於GTID的主從複製；

4.  從鏡像庫來看，數據量3800W左右，實際生產環境每天還會增加約不到100w；

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
    &lt;td&gt;&lt;p&gt;0-1w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1w-10w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;10w-50w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;50w-100w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;100w-1000w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;&amp;gt;1000w&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;表數量約&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;2105&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;83&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;28&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;5&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;6&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

5.  MySQL爲社區版5.7.23，所有表均爲INNODB引擎；

6.  據客戶N的業務人員反饋，他們嘗試使用mysqldump可能會報錯。

一些準備工作
============

爲了能夠順滑的開展後期工作，我習慣先整理一些常用的命令，以備隨時複製粘貼…

    -- 查詢所有業務數據庫的表名，數據庫，存儲引擎信息
    select table_name,table_schema,engine from information_schema.tables where engine=&#39;innodb&#39; and table_schema not in(&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查詢所有業務數據庫的表的數量
    select count(*) from information_schema.tables where engine=&#39;innodb&#39; and table_schema not in(&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查詢所有表的數據量
    SELECT CONCAT(TABLE_SCHEMA,&#39;.&#39;,TABLE_NAME) AS table_name, IFNULL(TABLE_ROWS,0) as table_rows FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;) ORDER BY 2;


    -- 查詢所有業務數據庫的視圖數量
    select table_name,table_schema from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);
    select count(*) from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查詢所有routines(存儲過程和函數)的數量
    select * from mysql.proc where db not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查詢所有觸發器的數量
    SELECT * FROM information_schema.triggers where TRIGGER_SCHEMA not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查詢所有事件的數量
    SELECT * FROM information_schema.EVENTS where EVENT_SCHEMA not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查詢所有用戶數量
    select user,host from mysql.user;

    -- 查看磁盤IO信息
    iostat -x -p /dev/mapper/vg02-lv02 1 10 -m
    iostat -x -p 1 10 -m

首先嚐試使用原生mysqldump
=========================

業務誠不欺我，果然有坑，報錯如下(安全需要，隱藏關鍵信息)。

mysqldump: Couldn’t execute *SHOW FIELDS FROM `XX`*: View *XX.XX*
references invalid table(s) or column(s) or function(s) or
definer/invoker of view lack rights to use them (1356)

報錯信息很明顯了，本次實踐中，主要是視圖引用創建語句中子查詢的列不存在，select
都會報錯，這個我們只能讓業務自己去審視，決策是否刪除或者修復。

由於通過mysqldump來發現那些視圖有問題非常不效率，所有寫了一個簡單的腳本：

**蒐集所有有問題的視圖.**

    #!/usr/bin/env bash

    function usage {
      echo &#34;Usage: $0 [-u USER_NAME] [-p PASSWORD] [-d WORKDIR] [-D:DROP ERROR VIEWS]&#34;
      echo &#34;Do not support -uroot, using -u root please.&#34;
      # too 2
      exit 2
    }

    function set_variable {
      local varname=$1
      shift
      if [[ -z &#34;${!varname}&#34; ]]; then
        eval &#34;$varname=\&#34;$@\&#34;&#34;
      else
        echo &#34;Error: $varname already set&#34;
        usage
      fi
    }

    function execMysqlCommand {
        mysql -u${USER_NAME} -p${PASSWORD} --skip-column-names -e &#34;$1&#34;
    }

    function checkView {
        viewName=&#34;$1&#34;
        # too slow
        # mysql -u${USER_NAME} -p${PASSWORD} -e &#34;select 1 from ${viewName} limit 1&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        # not good either
        # mysql -u${USER_NAME} -p${PASSWORD} -e &#34;update ${viewName} set thisIsANotExistCol=123;&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        #
        execMysqlCommand &#34;show fields from ${viewName};&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        return $?
    }

    function checkAllViewsAndGetErrorViews {
        echo &#34;&#34;&gt;/home/mysql/temp/view_error
        i=1
        for view in ${views[@]};do
            echo -n &#34;checking $view ...$i/${#views[@]}&#34; &#34;...&#34;
            checkView ${view}
            result=$?
            [[ ${result} -ne 0 ]] &amp;&amp; echo &#34;bad&#34;
            [[ ${result} -ne 0 ]] &amp;&amp; echo &#34;pass&#34;
            ((i&#43;&#43;))
        done;
        cat /home/mysql/temp/view_error|grep &#34;1356&#34;|awk -F&#34;&#39;&#34; &#39;{print $2}&#39;&gt;/home/mysql/temp/error_list
        rm /home/mysql/temp/view_error -rf
        error_views=(`cat /home/mysql/temp/error_list`)
    }

    function printIgnoreMsg {
        [[ ${#error_views[@]} -gt 0 ]] &amp;&amp; echo &#34;You can add these statements to mysqldump to ignore those error views:&#34;
        for view in ${error_views[@]};do
            echo -n &#34; --ignore-table=${view}&#34;
        done
        echo &#34;&#34;
    }

    function backupErrorViewsSql {
        echo &#34;Backing up create statement of error views to ${WORKDIR}...&#34;
        echo &#34;&#34; &gt; /home/mysql/temp/backup_create_view_sql -rf
        for view in ${error_views[@]};do
            execMysqlCommand &#34;show create view $view;&#34; &gt;&gt;/home/mysql/temp/backup_create_view_sql 2&gt;/dev/null
        done
        cat /home/mysql/temp/backup_create_view_sql|awk -F&#39;\t&#39; &#39;{print $2&#34;;&#34;}&#39;|grep -v &#39;Create View;&#39;&gt;&gt;/home/mysql/temp/backup_create_view
        rm -rf /home/mysql/temp/backup_create_view_sql
        echo &#34;Done backing up create statement of error views.&#34;

    }

    function deleteErrorViews {
        echo &#34;Dropping error views...&#34;
        for view in ${error_views[@]};do
            while [[ &#34;X&#34; == &#34;X${confirm}&#34; ]];do
                read -p &#34;please confirm to delete ${view}:(y/n)&#34; confirm
            done
            if [[ &#34;Xy&#34; == &#34;X${confirm}&#34; ]];then
                execMysqlCommand &#34;drop view $view;&#34; 2&gt;/dev/null
            fi
        done
        echo &#34;Done dropping error views.&#34;
    }

    init() {
        unset DELETE_VIEWS USER_NAME PASSWORD WORKDIR

        while getopts &#39;u:p:d:D?h&#39; option
        do
          case ${option} in
            d) set_variable WORKDIR $OPTARG ;;
            D) set_variable DELETE_VIEWS true ;;
            u) set_variable USER_NAME $OPTARG ;;
            p) set_variable PASSWORD $OPTARG ;;
            h|?) usage ;; esac
        done

        [[ -z &#34;${USER_NAME}&#34; ]] &amp;&amp; usage
        [[ -z &#34;${PASSWORD}&#34; ]] &amp;&amp; usage
        [[ -z &#34;${WORKDIR}&#34; ]] &amp;&amp; set_variable WORKDIR &#34;/home/mysql/temp&#34; &amp;&amp; mkdir -p ${WORKDIR}

        echo &#34;Using directory ${WORKDIR} as temp dir.&#34;
    }

    getAllViews() {
        echo &#34;Getting all views from schema...&#34;
        views=(`execMysqlCommand &#34;select concat(table_schema,&#39;.&#39;,table_name) from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);&#34; 2&gt;/dev/null`)
    }

    init $@
    getAllViews
    checkAllViewsAndGetErrorViews
    printIgnoreMsg

    [[ X&#34;true&#34; == X&#34;${DELETE_VIEWS}&#34; &amp;&amp;  ${#error_views[@]} -gt 0 ]] &amp;&amp; backupErrorViewsSql &amp;&amp; deleteErrorViews

命令優化x
=========

具體方案之前，先加上一些基本的備份對象

    --hex-blob --single-transaction --quick --routines --triggers

方案一 160分鐘
--------------

單線程直接執行mysqldump，大概160分鐘

    &gt; /data01/chroot/usr/local/mysql5.7.23/bin/mysqldump -udbXXXX -pXXXX --all-databases --hex-blob --ignore-table=netcxx.xxxxx --ignore-table=netxxx.rxxx(此處很多忽略的視圖) | gzip &gt; /temp/back0129.sql.gz

方案二 90分鐘
-------------

考慮一個表一個文件，10個線程，大概90分鐘；TODO 測試增加線程

**multidump.sh\[lines=25..55\].**

    multidump() {
        rm -rf ${WORKDIR}/backup
        mkdir -p ${WORKDIR}/backup

        COMMIT_COUNT=0
        COMMIT_LIMIT=10
        error_views_file=&#34;${WORKDIR}/error_list&#34;
        DBTBS=(`cat ${WORKDIR}/listOfTables`)
        i=1
        for DBTB in ${DBTBS[@]};do
            echo &#34;processing $i/${#DBTBS[@]}&#34;
            ((i&#43;&#43;))
            DB=`echo ${DBTB} | sed &#39;s/\./ /g&#39; | awk &#39;{print $1}&#39;`
            TB=`echo ${DBTB} | sed &#39;s/\./ /g&#39; | awk &#39;{print $2}&#39;`
            if [[ &#34;X&#34;`grep -w ${DBTB} ${error_views_file}` != X&#34;&#34; ]];then
                echo skip &#34;${DBTB}&#34;
                continue
            fi
            dumpIt ${DB} ${TB}
            (( COMMIT_COUNT&#43;&#43; ))
            if [[ ${COMMIT_COUNT} -eq ${COMMIT_LIMIT} ]]
            then
                COMMIT_COUNT=0
                wait
            fi
        done
        if [[ ${COMMIT_COUNT} -gt 0 ]]
        then
            wait
        fi
    }

方案三 15-22分鐘
----------------

mysqlpump 是 mysql 提供的工具，文檔和網上教程一大堆，這裏只談使用。
可以很直觀的看到執行到哪個表，剩餘多少行；注意：mysqlpump遇到錯誤會停止繼續，比如命令不正確、數據結構有問題。而且這個數據庫開啓GTID，所以如果你的數據庫沒有此選項，要把命令中的—set-gtid-purged=ON去掉。

兩種壓縮格式的時間差距還是很明顯：

mysqlpump -u*username* -p*password* --compress-output=ZLIB
--default-parallelism=100 --set-gtid-purged=ON --hex-blob
--add-drop-database --add-drop-table --add-drop-user --users |gzip &amp;gt;
/temp/test.sql.gz

Dump progress: 0/xx tables, xx/xxxxxxxxx rows Dump completed in xxxxxx
milliseconds

mysqlpump -u*username* -p*password* --compress-output=LZ4
--default-parallelism=100 --set-gtid-purged=ON --hex-blob
--add-drop-database --add-drop-table --add-drop-user --users &amp;gt;
/temp/testlz4.lz4

方案四
------

mysqlpump
可以針對database進行多線程導出，但是有時候數據分佈不均勻，90%的數據可能都在一個表內，這種情況下mysqlpump顯得無能爲力。有沒有可以對單個大表繼續進行分拆的工具呢？
[mydumper](https://github.com/maxbube/mydumper/releases) 可以做這件事。

### 首先統計表的分佈

    totalSql=&#34;SELECT IFNULL(SUM(TABLE_ROWS),0) as t_rows_sum FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);&#34;
    eachTableSql=&#34;SELECT CONCAT(TABLE_SCHEMA,&#39;.&#39;,TABLE_NAME) AS table_name, IFNULL(TABLE_ROWS,0) as table_rows FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;) ORDER BY 2;&#34;

### 驗證mydumper導出database的效率

1.  只導出netcare，17分鐘-20分鐘左右

mydumper -u *username* -p *password* -v 3 -B *databaseName* --triggers
--events --routines --rows=500000 --compress-protocol -c -t
*threadNum.ie.100* --trx-consistency-only --outputdir /temp/mydumper

### 驗證mydumper導出某一個大表的效率

&lt;https://dev.mysql.com/doc/mysql-enterprise-backup/8.0/en/&gt;

    方案五 優化前：21分鐘
    unset tempdir
    tempdir=/data03/backup_`date &#39;&#43;%y%m%d%H%M%S&#39;`
    mkdir ${tempdir}
    ~/mysqlbackup -udbAdmin -pabcd1234 --backup-dir=${tempdir} --compress backup
    echo &#34;Successfully backing up data to ${tempdir}&#34;

    # 有待優化 https://dev.mysql.com/doc/mysql-enterprise-backup/4.1/en/backup-capacity-options.html
    --limit-memory=MB （default 100）
    --read-threads=num_threads （default 1）
    --process-threads=num_threads （default 6）
    --write-threads=num_threads （default 1）
    ~/mysqlbackup -udbAdmin -pabcd1234 --backup-dir=${tempdir} --compress backup

    #優化後： 15分鐘
    # 整庫備份到單個文件
    ~/mysqlbackup -udbAdmin -pabcd1234 --compress --compress-level=5 --limit-memory=1024 --read-threads=10 --process-threads=15 --write-threads=10 --backup-dir=${tempdir} --backup-image=/data03/`basename ${tempdir}`.bin backup-to-image

    #直接備份到目標機器：
    ~/mysqlbackup -udbAdmin -pabcd1234 --compress --compress-level=5 --limit-memory=1024 --read-threads=10 --process-threads=15 --write-threads=10 --backup-dir=${tempdir} --backup-image=- backup-to-image | ssh root@10.15.32.73 &#39;cat &gt; /opt/temp_for_restore/my_backup.bin&#39;

具體備份恢復使用見另一篇文章 mysql gtid 主從複製數據遷移(物理備份)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/a-case-of-mysqldump/  

