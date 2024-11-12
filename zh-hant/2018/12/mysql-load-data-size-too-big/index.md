# Mysql Load Data 數據膨脹


## 發現問題

100w 100字段數據 後臺膨脹係數較大。 用膨脹係數表示load data後MySQL後臺
表名.ibd 文件的大小與所 load 的 data.xdr 文件的比值。
膨脹係數(50f100w)代表使用了50個字段100w行的數據進行測試。

## 分解問題

### 是否是數據量較大，導致膨脹係數較大？

構造 10f10w 和 10f100w 進行對比，排除單純因數據量導致膨脹的推測。

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
&lt;td&gt;&lt;p&gt;數據模型（字段數）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;數據模型（行數）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;數據文件大小（MB）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;load 時長(s)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;表文件大小(MB)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;單次導入增加&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;字段類型&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;10&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;10w&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;58.9&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3.02&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;76&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;76&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&#34;3 int,&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;3 double(20,2),&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;4 VARCHAR(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;&#34;&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;10&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;100w&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;592&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;33.96&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;688&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;688&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&#34;3 int,&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;3 double(20,2),&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;4 VARCHAR(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;&#34;&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

### 是否是因字段數不同，導致膨脹係數較大？


#### 數據模型

    create table loadtest10f(
        record_001 VARCHAR(256),
        record_002 VARCHAR(256),
        record_003 VARCHAR(256),
        record_004 VARCHAR(256),
        record_005 VARCHAR(256),
        record_006 VARCHAR(256),
        record_007 VARCHAR(256),
        record_008 VARCHAR(256),
        record_009 VARCHAR(256),
        record_010 VARCHAR(256),
        ....
    )

因構造數據工具內存限制，100字段最多構造出2w行數據，爲了方便對比，以下所有數據都構造2w行；
因MySQL 默認row
size爲65535，構造的數據模型爲varchar(256)，且服務器採用utf8(每個字符3個字節)，所以最多構造到65535/256/3個字段；

構造同樣是2w行數據的 10f,20f,50f,60f,70f,80f,85f
等數據進行測試，結果如下：

&lt;table style=&#34;width:100%;&#34;&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;col style=&#34;width: 11%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;數據模型（字段數）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;數據模型（行數）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;數據文件大小（MB）&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;load 時長(s)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;表文件大小(MB)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;字段類型&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;最大行大小&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;B&#43;樹高度&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;膨脹係數&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;10&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;15&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;0.63&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;26&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;7680&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.733333333&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;20&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;29&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.04&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;42&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;15360&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.448275862&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;30&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;44&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.63&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;63&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;23040&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.431818182&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;50&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;72&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3.07&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;110&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;38400&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1.527777778&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;60&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;87&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;12.88&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;680&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;46080&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;7.816091954&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;70&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;101&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;35.61&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;1921&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;53760&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;19.01980198&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;80&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;115&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;61.87&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3280&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;61440&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;28.52173913&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;85&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;123&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;70.04&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3985&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;65280&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;3&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;32.39837398&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;100&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20000&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;144&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;varchar(256)&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

#### 說明

數據顯示，字段在50f左右開始，膨脹係數曲線較之前更爲陡峭，該變化記爲 d1；
在50f之後曲線再次平緩，增長速度小於 d1.

#### 分析

幾點說明：

1.  innodb 默認 page size 爲 16834.

        mysql&gt; show variables like &#39;innodb_page_size&#39;;
        &#43;------------------&#43;-------&#43;
        | Variable_name    | Value |
        &#43;------------------&#43;-------&#43;
        | innodb_page_size | 16384 |
        &#43;------------------&#43;-------&#43;
        1 row in set (0.00 sec)

2.  innodb
    採用B&#43;Tree數據結構，查詢這幾個構造的數據表，其根節點頁起始頁碼爲3：

        mysql&gt; SELECT
         -&gt; b.name, a.name, index_id, type, a.space, a.PAGE_NO
         -&gt; FROM
         -&gt; information_schema.INNODB_SYS_INDEXES a,
         -&gt; information_schema.INNODB_SYS_TABLES b
         -&gt; WHERE
         -&gt; a.table_id = b.table_id AND a.space &lt;&gt; 0 AND b.name like &#39;%loadtest%&#39;;
        &#43;-----------------------&#43;-----------------&#43;----------&#43;------&#43;-------&#43;---------&#43;
        | name                  | name            | index_id | type | space | PAGE_NO |
        &#43;-----------------------&#43;-----------------&#43;----------&#43;------&#43;-------&#43;---------&#43;
        | test/loadtest100f100w | GEN_CLUST_INDEX |    30333 |    1 | 16650 |       3 |
        | test/loadtest10f      | GEN_CLUST_INDEX |    30334 |    1 | 16651 |       3 |
        | test/loadtest10f100w  | GEN_CLUST_INDEX |    30329 |    1 | 16646 |       3 |
        | test/loadtest10f10w   | GEN_CLUST_INDEX |    30328 |    1 | 16645 |       3 |
        | test/loadtest20f      | GEN_CLUST_INDEX |    30335 |    1 | 16652 |       3 |
        | test/loadtest20f100w  | GEN_CLUST_INDEX |    30330 |    1 | 16647 |       3 |
        | test/loadtest30f      | GEN_CLUST_INDEX |    30336 |    1 | 16653 |       3 |
        | test/loadtest50f      | GEN_CLUST_INDEX |    30337 |    1 | 16654 |       3 |
        | test/loadtest50f100w  | GEN_CLUST_INDEX |    30331 |    1 | 16648 |       3 |
        | test/loadtest60f      | GEN_CLUST_INDEX |    30340 |    1 | 16657 |       3 |
        | test/loadtest70f      | GEN_CLUST_INDEX |    30341 |    1 | 16658 |       3 |
        | test/loadtest80f      | GEN_CLUST_INDEX |    30338 |    1 | 16655 |       3 |
        | test/loadtest85f      | GEN_CLUST_INDEX |    30342 |    1 | 16659 |       3 |
        &#43;-----------------------&#43;-----------------&#43;----------&#43;------&#43;-------&#43;---------&#43;
        13 rows in set (0.00 sec)

3.  查詢其 pagelevel （根頁偏移64字節的前2位，即16834\*3&#43;64=49216）

        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest10f.ibd
        000c040 0000 0000 0000 0000 7e76
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest20f.ibd
        000c040 0000 0000 0000 0000 7f76
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest30f.ibd
        000c040 0000 0000 0000 0000 8076
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest50f.ibd
        000c040 0000 0000 0000 0000 8176
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest60f.ibd
        000c040 0200 0000 0000 0000 8476
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest70f.ibd
        000c040 0200 0000 0000 0000 8576
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest80f.ibd
        000c040 0200 0000 0000 0000 8276
        000c04a
        SHA1000130993:/usr/local/mysql/data/test # hexdump -s 49216 -n 10 loadtest85f.ibd
        000c040 0200 0000 0000 0000 8676
        000c04a

4.  獲取 page level 和 B&#43;Tree 高度
    由於本人測試機器字節序爲小端，所以000c040
    0200十六進制字節實際值爲000c040 0002，即2.
    從上一步驟得出50f以後的表pagelevel爲2,50f之前pagelevel爲0.
    所以50f以後的表B&#43;Tree高度爲page level&#43;1=3.
    B&#43;Tree高度一般爲1-3，很少有4。3
    屬於較高的高度，懷疑數據全爲索引所佔。

5.  獲取index所佔page的粗略信息。由於本文測試數據未建索引，所以默認索引爲GEN\_CLUST\_INDEX。主鍵、聚簇索引，本身即是數據，可以看到磁盤基本都是索引佔據。

&lt;!-- --&gt;

    mysql&gt; SELECT
        -&gt; table_name,
        -&gt;        sum(stat_value) pages,
        -&gt;        index_name,
        -&gt;        sum(stat_value) * @@innodb_page_size size
        -&gt; FROM
        -&gt;        mysql.innodb_index_stats
        -&gt; WHERE
        -&gt;            table_name like &#39;%load%&#39;
        -&gt;        AND database_name = &#39;test&#39;
        -&gt;        AND stat_description = &#39;Number of pages in the index&#39;
        -&gt; GROUP BY
        -&gt;        table_name,index_name;
    &#43;------------------&#43;--------&#43;-----------------&#43;-------------&#43;
    | table_name       | pages  | index_name      | size        |
    &#43;------------------&#43;--------&#43;-----------------&#43;-------------&#43;
    | loadtest100f100w | 785472 | GEN_CLUST_INDEX | 12869173248 |
    | loadtest10f      |   1059 | GEN_CLUST_INDEX |    17350656 |
    | loadtest10f100w  |  42112 | GEN_CLUST_INDEX |   689963008 |
    | loadtest10f10w   |   4327 | GEN_CLUST_INDEX |    70893568 |
    | loadtest20f      |   2084 | GEN_CLUST_INDEX |    34144256 |
    | loadtest20f100w  |  85568 | GEN_CLUST_INDEX |  1401946112 |
    | loadtest30f      |   3366 | GEN_CLUST_INDEX |    55148544 |
    | loadtest50f      |   6121 | GEN_CLUST_INDEX |   100286464 |
    | loadtest50f100w  |  99456 | GEN_CLUST_INDEX |  1629487104 |
    | loadtest60f      |  40425 | GEN_CLUST_INDEX |   662323200 |
    | loadtest70f      | 115114 | GEN_CLUST_INDEX |  1886027776 |
    | loadtest80f      | 196778 | GEN_CLUST_INDEX |  3224010752 |
    | loadtest85f      | 239466 | GEN_CLUST_INDEX |  3923410944 |
    &#43;------------------&#43;--------&#43;-----------------&#43;-------------&#43;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/12/mysql-load-data-size-too-big/  

