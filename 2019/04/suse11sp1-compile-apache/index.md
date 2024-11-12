# Susesp 编译 Apache ..


背景：suse 11 sp1 机器编译安装带有指定模块的 apache 2.4.34.
暂时没有bicp代码访问权限，先放这里 == 第一步 安装openssl

    cd /data/bicpinstall/
    tar zxvf openssl-1.1.0i.tar.gz
    cd openssl-1.1.0i
    export LDFLAGS=-ldl
    export LIBPATH=&#34;/data/bicpinstall/ssl&#34;
    export LIBS=&#34;-L/data/bicpinstall/ssl&#34;
    export SSL_LIBS=&#34;-L/data/bicpinstall/ssl&#34;
    export CPPFLAGS=&#34;-I/data/bicpinstall/ssl/include/openssl&#34;
    ./config --prefix=/data/bicpinstall/ssl shared
    make &amp;&amp; make install
    rm -rf /data/bicpinstall/ssl/ssl/man

第二步 生成证书
===============

    cd /data/bicpinstall/ssl/bin
    openssl genrsa -passout pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm -des3 -out server.key 1024
    openssl req -new -out server.csr -key server.key -passin pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm -passout pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm -subj /C=CN/O=huawei/CN=10.139.200.36 -config ../ssl/openssl.cnf
    openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt -passin pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm
    openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt -passin pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm -passout pass:cHpxHUm&#43;v5teANoYurlMvA2&#43;Gdg&#43;ifm

第三步 安装 httpd 2.4.34 依赖的 apr 和 apr-util
===============================================

该部分不再捆绑发布，需要自行安装，版本需要 1.5 以上，部分功能需要 1.6
以上，版本要求见https://www.apache.org/dist/httpd/Announcement2.4.html

    cd /data/bicpinstall/
    tar zxvf apr-1.6.3.tar.gz
    tar zxvf apr-util-1.6.1.tar.gz
    apr
    cd /data/bicpinstall/apr-1.6.3/
    ./configure --prefix=/data/bicpinstall/httpd-2.4.34/srclib/apr
    make &amp; make install
    apr-utils
    expat （apr-utils依赖 libexpat，https://www.apache.org/dist/apr/CHANGES-APR-UTIL-1.6）
    cd /data/bicpinstall/
    tar xvjf expat-2.2.5.tar.bz2
    cd expat-2.2.5
    ./configure --prefix=/data/bicpinstall/httpd-2.4.34/srclib/expat
    make &amp; make install

    cd /data/bicpinstall/apr-util-1.6.1
    ./configure --prefix=/data/bicpinstall/httpd-2.4.34/srclib/apr-util --with-apr=/data/bicpinstall/httpd-2.4.34/srclib/apr --with-expat=/data/bicpinstall/httpd-2.4.34/srclib/expat
    make &amp; make install

第四步 安装 pcre，下载地址(&lt;https://ftp.pcre.org/pub/pcre/&gt;)
============================================================

    tar zxvf pcre-8.42.tar.gz
    cd /data/bicpinstall/pcre-8.42
    ./configure --prefix=/data/bicpinstall/httpd-2.4.34/srclib/pcre
    make &amp;&amp; make install

安装 zlib，下载地址(&lt;https://zlib.net/zlib-1.2.11.tar.gz&gt;) 直接
./configure &amp;&amp; make &amp;&amp; make install

第五步 编译安装 httpd
=====================

    cd /data/bicpinstall/httpd-2.4.34
    ./configure --prefix=/data/bicpinstall/apache --with-ssl=/data/bicpinstall/ssl --with-apr=/data/bicpinstall/httpd-2.4.34/srclib/apr --with-apr-util=/data/bicpinstall/httpd-2.4.34/srclib/apr-util --with-pcre=/data/bicpinstall/httpd-2.4.34/srclib/pcre --enable-headers=shared --enable-rewrite=shared --enable-proxy=shared --enable-proxy-connect=shared --enable-proxy-ftp=shared --enable-proxy-http=shared --enable-proxy-scgi=shared --enable-proxy-ajp=shared --enable-proxy-balancer=shared --enable-ssl=shared --enable-deflate=shared
    make &amp;&amp; make install

第六步 压缩包
=============

    cd /data/bicpinstall/
    mkdir apache/lib

    cp -P /data/bicpinstall/httpd-2.4.34/srclib/apr/lib/libapr-1.so* data/bicpinstall/apache/lib/
    cp -P /data/bicpinstall/httpd-2.4.34/srclib/apr-util/lib/libaprutil-1.so* data/bicpinstall/apache/lib/
    cp -P /data/bicpinstall/httpd-2.4.34/srclib/expat/lib/libexpat.so* data/bicpinstall/apache/lib/
    cp -r /data/bicpinstall/ssl /data/bicpinstall/apache/lib

    cd /data/bicpinstall/apache/manual/mod; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/rewrite; rm -rf index.html*
    cd /data/bicpinstall/apache/manual; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/ssl; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/programs; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/developer;rm -rf index.html*
    cd /data/bicpinstall/apache/manual/misc; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/howto; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/platform; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/faq; rm -rf index.html*
    cd /data/bicpinstall/apache/manual/vhosts; rm -rf index.html*
    cd /data/bicpinstall/apache/htdocs; rm -rf index.html*
    cd /data/bicpinstall/apache/cgi-bin;rm -rf *

    zip -r apache.zip apache

附录一 部分软件
---------------

    apr-1.6.3.tar.gz
    apr-util-1.6.1.tar.gz
    expat-2.2.5.tar.bz2.remove.asc
    httpd-2.4.34.tar.gz
    openssl-1.1.0i.tar.gz
    pcre-8.42.tar.gz
    zlib-1.2.11.tar.gz

### 附录二 部分编译工具版本或环境

    cat /etc/SuSE-release
    SUSE Linux Enterprise Server 11 (x86_64)
    VERSION = 11
    PATCHLEVEL = 1

    gcc --version
    gcc (SUSE Linux) 4.3.4 [gcc-4_3-branch revision 152973]
    Copyright (C) 2008 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.


    gunzip --version
    gzip 1.3.12
    Copyright (C) 2007 Free Software Foundation, Inc.
    Copyright (C) 1993 Jean-loup Gailly.
    This is free software.  You may redistribute copies of it under the terms of
    the GNU General Public License &lt;http://www.gnu.org/licenses/gpl.html&gt;.
    There is NO WARRANTY, to the extent permitted by law.

    Written by Jean-loup Gailly.

    make --version
    GNU Make 3.81
    Copyright (C) 2006  Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.
    There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
    PARTICULAR PURPOSE.

    This program built for x86_64-unknown-linux-gnu

附录三 错误解决
---------------

    报错：
    /bin/sh: gcc: command not found
    make[1]: *** [crypto/aes/aes-x86_64.o] Error 127
    make[1]: Leaving directory `/data/bicpinstall/openssl-1.1.0i&#39;
    make: *** [all] Error 2
    解决：安装gcc


    报错：
    checking for APR... no
    configure: error: APR not found.  Please read the documentation.

    报错：
     pcre-config for libpcre not found. PCRE is required and available from http://pcre.org/
     解决：见编译指导.md



    报错：
    checking dirent.h presence... yes
    checking for dirent.h... yes
    checking windows.h usability... no
    checking windows.h presence... no
    checking for windows.h... no
    configure: error: Invalid C&#43;&#43; compiler or C&#43;&#43; compiler flags
     解决：
     zypper in gcc-c&#43;&#43;


    报错：
     checking for zlib location... not found
    checking whether to enable mod_deflate... configure: error: mod_deflate has been requested but can not be built due to prerequisite failures
     解决：编译安装zlib


    报错：
    checking whether to enable mod_deflate... checking dependencies
      adding &#34;-I/lib64/include&#34; to INCLUDES
      setting MOD_INCLUDES to &#34;-I/lib64/include&#34;
      adding &#34;-L/lib64/lib&#34; to LDFLAGS
      setting ap_zlib_ldflags to &#34;-L/lib64/lib&#34;
      adding &#34;-lz&#34; to LIBS
    checking for zlib library... not found
    configure: error: ... Error, zlib was missing or unusable
     解决：安装 zlib


    报错：
    od_proxy_balancer.c &amp;&amp; touch mod_proxy_balancer.slo
    mod_proxy_balancer.c:25:24: error: apr_escape.h: No such file or directory
    mod_proxy_balancer.c: In function &#39;make_server_id&#39;:
    mod_proxy_balancer.c:779: warning: implicit declaration of function &#39;apr_pescape_hex&#39;
    mod_proxy_balancer.c:779: warning: return makes pointer from integer without a cast
    make[4]: *** [mod_proxy_balancer.slo] Error 1
    make[4]: Leaving directory `/data/bicpinstall/httpd-2.4.34/modules/proxy&#39;
    make[3]: *** [shared-build-recursive] Error 1
    make[3]: Leaving directory `/data/bicpinstall/httpd-2.4.34/modules/proxy&#39;
    make[2]: *** [shared-build-recursive] Error 1
    make[2]: Leaving directory `/data/bicpinstall/httpd-2.4.34/modules&#39;
    make[1]: *** [shared-build-recursive] Error 1
    make[1]: Leaving directory `/data/bicpinstall/httpd-2.4.34&#39;
    make: *** [all-recursive] Error 1
     解决：安装 apr-util



    报错：
    xml/apr_xml.c:35:19: error: expat.h: No such file or directory
    xml/apr_xml.c:66: error: expected specifier-qualifier-list before ‘XML_Parser’
    xml/apr_xml.c: In function ‘cleanup_parser’:
    xml/apr_xml.c:364: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:365: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c: At top level:
    xml/apr_xml.c:384: error: expected ‘;’, ‘,’ or ‘)’ before ‘*’ token
    xml/apr_xml.c: In function ‘apr_xml_parser_create’:
    xml/apr_xml.c:401: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:402: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:410: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:411: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:412: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:424: error: ‘apr_xml_parser’ has no member named ‘xp’
    xml/apr_xml.c:424: error: ‘default_handler’ undeclared (first use in this function)
    xml/apr_xml.c:424: error: (Each undeclared identifier is reported only once
     解决：安装 expat，方法见上


    报错：
    aclocal: couldn&#39;t open directory `m4&#39;: No such file or directory
    解决 直接在当前目录 mkdir m4


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/04/suse11sp1-compile-apache/  

