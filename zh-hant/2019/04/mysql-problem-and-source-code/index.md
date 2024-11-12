# Mysql 問題與源碼


問題1: 連接數爲214， 登錄經常報錯 too many connections
======================================================

在公司MySQL企業版服務化開發的初期，我們曾經遇到一個問題，即在連接MySQL的時候報錯
`too many connections`,
即使是新安裝的MySQL。在以前的社區版MySQL也曾經遇到過類似的問題，當時MySQL是用rpm安裝並使用systemd啓動的方式。
此次企業版的MySQL啓動並未託管到systemd，因此解決辦法不能照搬。

定位過程
--------

爲了能夠登錄，首先只能重啓MySQL，執行

    show variables like &#34;max_conne%&#34;;

發現連接數並非配置文件中定義的 2000，而是一個奇怪的數字 214；

執行
====

ulimit -a 或者 cat /proc/`pidof mysqld`/limits

    發現 open files 爲一個較低的默認值 1024；（代碼中有改動該值的邏輯，但是最終並未生效，最終發現是公司系統鏡
    像/etc/security/limits.d/...的默認值有問題，此處不延伸）

core file size (blocks, -c) 0 data seg size (kbytes, -d) unlimited
scheduling priority (-e) 0 file size (blocks, -f) unlimited pending
signals (-i) 23883 max locked memory (kbytes, -l) 64 max memory size
(kbytes, -m) unlimited open files (-n) 1024 pipe size (512 bytes, -p) 8
POSIX message queues (bytes, -q) 819200 real-time priority (-r) 0 stack
size (kbytes, -s) 8192 cpu time (seconds, -t) unlimited max user
processes (-u) 23883 virtual memory (kbytes, -v) unlimited file locks
(-x) unlimited

    === 爲什麼是214？
    查看mysqld.cc:adjust_max_connections發現原因:

    [source, c&#43;&#43;]

void adjust\_max\_connections(ulong requested\_open\_files) { ulong
limit;

    // TABLE_OPEN_CACHE_MIN = 400
    // requested_open_files = 1024
    limit= requested_open_files - 10 - TABLE_OPEN_CACHE_MIN * 2;

    if (limit &lt; max_connections)
    {
      sql_print_warning(&#34;Changed limits: max_connections: %lu (requested %lu)&#34;,
                        limit, max_connections);

        // This can be done unprotected since it is only called on startup.
        max_connections= limit;
      }
    }

    計算很簡單，看一下此處，應該會打印出一行警告日誌，可以試試看，日誌中是否可以找到這樣的信息。

sql\_print\_warning(&#34;Changed limits: max\_connections: %lu (requested
%lu)&#34;, limit, max\_connections);

    值得關注的點是，這個requested_open_files有一個比較複雜的計算過程。

    .mysqld.cc:adjust_related_options
    [source, c&#43;&#43;]

void adjust\_related\_options(ulong **requested\_open\_files) { /** In
bootstrap, disable grant tables (we are about to create them) \*/ if
(opt\_bootstrap) opt\_noacl= 1;

      /* The order is critical here, because of dependencies. */
      adjust_open_files_limit(requested_open_files);
      adjust_max_connections(*requested_open_files);
      adjust_table_cache_size(*requested_open_files);
      adjust_table_def_size();
    }

    .mysqld.cc:adjust_open_files_limit
    [source, c&#43;&#43;]

/\*\* Adjust @c open\_files\_limit. Computation is based on: - @c
max\_connections, - @c table\_cache\_size, - the platform max open file
limit. \*/ void adjust\_open\_files\_limit(ulong
\*requested\_open\_files) { ulong limit\_1; ulong limit\_2; ulong
limit\_3; ulong request\_open\_files; ulong effective\_open\_files;

    /*這裏會有三種計算方案*/

    /* MyISAM requires two file handles per table. */
    limit_1= 10 &#43; max_connections &#43; table_cache_size * 2;

    /*
      We are trying to allocate no less than max_connections*5 file
      handles (i.e. we are trying to set the limit so that they will
      be available).
    */
    limit_2= max_connections * 5;

    /* Try to allocate no less than 5000 by default. */
    //這裏可以解釋了，爲什麼很多的系統安裝後， /proc/`pidof mysqld`/limits中的值爲5000
    //但是這裏的代碼，是否應該改爲 open_files_limit&gt; 5000 ? open_files_limit : 5000;
    limit_3= open_files_limit ? open_files_limit : 5000;

    // 取三種方案的最大值
    request_open_files= max&lt;ulong&gt;(max&lt;ulong&gt;(limit_1, limit_2), limit_3);

    /* Notice: my_set_max_open_files() may return more than requested. */
    effective_open_files= my_set_max_open_files(request_open_files);

    if (effective_open_files &lt; request_open_files)
    {
      if (open_files_limit == 0)
      {
        sql_print_warning(&#34;Changed limits: max_open_files: %lu (requested %lu)&#34;,
                          effective_open_files, request_open_files);
      }
      else
      {
        sql_print_warning(&#34;Could not increase number of max_open_files to &#34;
                          &#34;more than %lu (request: %lu)&#34;,
                          effective_open_files, request_open_files);
      }
    }

      open_files_limit= effective_open_files;
      if (requested_open_files)
        *requested_open_files= min&lt;ulong&gt;(effective_open_files, request_open_files);
    }

    .my_file.c:my_set_max_open_files
    [source, c&#43;&#43;]

uint my\_set\_max\_open\_files(uint files) { struct st\_my\_file\_info
\*tmp; DBUG\_ENTER(&#34;my\_set\_max\_open\_files&#34;);
DBUG\_PRINT(&#34;enter&#34;,(&#34;files: %u my\_file\_limit: %u&#34;, files,
my\_file\_limit));

    files&#43;= MY_FILE_MIN;
    files= set_max_open_files(MY_MIN(files, OS_FILE_LIMIT));
    if (files &lt;= MY_NFILE)
      DBUG_RETURN(files);

    if (!(tmp= (struct st_my_file_info*) my_malloc(key_memory_my_file_info,
                                                   sizeof(*tmp) * files,
                                                   MYF(MY_WME))))
      DBUG_RETURN(MY_NFILE);

      /* Copy any initialized files */
      memcpy((char*) tmp, (char*) my_file_info,
             sizeof(*tmp) * MY_MIN(my_file_limit, files));
      memset((tmp &#43; my_file_limit), 0,
            MY_MAX((int) (files - my_file_limit), 0) * sizeof(*tmp));
      my_free_open_file_info();                     /* Free if already allocated */
      my_file_info= tmp;
      my_file_limit= files;
      DBUG_PRINT(&#34;exit&#34;,(&#34;files: %u&#34;, files));
      DBUG_RETURN(files);
    }

    因此，如果希望max_connections=2000，requested_open_files不能小於stem:[2000&#43;10&#43;400 \times 2=2810].


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/04/mysql-problem-and-source-code/  

