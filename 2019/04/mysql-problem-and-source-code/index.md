# Mysql 问题与源码


问题1: 连接数为214， 登录经常报错 too many connections
======================================================

在公司MySQL企业版服务化开发的初期，我们曾经遇到一个问题，即在连接MySQL的时候报错
`too many connections`,
即使是新安装的MySQL。在以前的社区版MySQL也曾经遇到过类似的问题，当时MySQL是用rpm安装并使用systemd启动的方式。
此次企业版的MySQL启动并未托管到systemd，因此解决办法不能照搬。

定位过程
--------

为了能够登录，首先只能重启MySQL，执行

    show variables like &#34;max_conne%&#34;;

发现连接数并非配置文件中定义的 2000，而是一个奇怪的数字 214；

执行
====

ulimit -a 或者 cat /proc/`pidof mysqld`/limits

    发现 open files 为一个较低的默认值 1024；（代码中有改动该值的逻辑，但是最终并未生效，最终发现是公司系统镜
    像/etc/security/limits.d/...的默认值有问题，此处不延伸）

core file size (blocks, -c) 0 data seg size (kbytes, -d) unlimited
scheduling priority (-e) 0 file size (blocks, -f) unlimited pending
signals (-i) 23883 max locked memory (kbytes, -l) 64 max memory size
(kbytes, -m) unlimited open files (-n) 1024 pipe size (512 bytes, -p) 8
POSIX message queues (bytes, -q) 819200 real-time priority (-r) 0 stack
size (kbytes, -s) 8192 cpu time (seconds, -t) unlimited max user
processes (-u) 23883 virtual memory (kbytes, -v) unlimited file locks
(-x) unlimited

    === 为什么是214？
    查看mysqld.cc:adjust_max_connections发现原因:

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

    计算很简单，看一下此处，应该会打印出一行警告日志，可以试试看，日志中是否可以找到这样的信息。

sql\_print\_warning(&#34;Changed limits: max\_connections: %lu (requested
%lu)&#34;, limit, max\_connections);

    值得关注的点是，这个requested_open_files有一个比较复杂的计算过程。

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

    /*这里会有三种计算方案*/

    /* MyISAM requires two file handles per table. */
    limit_1= 10 &#43; max_connections &#43; table_cache_size * 2;

    /*
      We are trying to allocate no less than max_connections*5 file
      handles (i.e. we are trying to set the limit so that they will
      be available).
    */
    limit_2= max_connections * 5;

    /* Try to allocate no less than 5000 by default. */
    //这里可以解释了，为什么很多的系统安装后， /proc/`pidof mysqld`/limits中的值为5000
    //但是这里的代码，是否应该改为 open_files_limit&gt; 5000 ? open_files_limit : 5000;
    limit_3= open_files_limit ? open_files_limit : 5000;

    // 取三种方案的最大值
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

    因此，如果希望max_connections=2000，requested_open_files不能小于stem:[2000&#43;10&#43;400 \times 2=2810].


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/04/mysql-problem-and-source-code/  

