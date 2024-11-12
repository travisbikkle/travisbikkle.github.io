# Centos 安装 Apache Ambari


版本说明
========

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;部件&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;版本号&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;Ambari&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.6.2.2&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;CentOS&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;7&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HDP&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;2.6&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;时间&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20180814&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

背景
====

对于 Ambari 能做什么，对于搜索到此文的同学来说应该毋庸赘述。目前 Ambari
安装的官方手册主要是
[Apache](https://cwiki.apache.org/confluence/display/AMBARI/Installation&#43;Guide&#43;for&#43;Ambari&#43;2.6.2)
和
[Hortonworks](https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.2.2/bk_ambari-installation/content/install-ambari-server.html)，我首先是参考
Apache 的说明，通过 maven 编译源码的方式，在 安装 linux-mint
的机器上尝试安装 `Ambari 2.7.0`，遇到过以下问题：

1.  由于系统不符合 Ambari 的要求，因此通过更改其中的
    `ambari-commons/OSCheck.py:is_ubuntu_family()` 函数强制安装 server
    和 agent.

2.  由于采用的国内 maven 仓库，ambari web legacy
    始终编译不过，通过更改其依赖编译通过.

3.  maven compiler plugin 报错 json-simple 的相关依赖问题，最后删除该
    legacy 模块.

4.  其它 node, yarn, npm 的代理设置问题. 最终在安装 agent 的时候遇到 ssl
    连接错误，时间已晚，选择放弃这种安装方式。转而使用文档支持较好的
    CentOS 和 yum 仓库安装的方式。虽然如此，Hortonworks
    的文档逻辑也稍显混乱，过于简单，本文对安装过程做详细记录，以备查询。`本文所有操作均使用 root 用户完成。`

准备
====

我使用的是 VMware WorkStation，CentOS 7 下载路径为
[点我](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-1804.iso)，安装步骤略过，建议安装
4 台机器(网络方式选择NAT模式)，其中安装 Ambari
服务器的机器硬盘大小不得小于 30
GB，如果不小心硬盘大小分配过小，参见另一篇博文link:/2018/10/13/如何对
centos 7 分区进行扩容/\[如何对 centos 7 分区进行扩容\]。其余 3
台机器作为集群机器以备后续使用。

====== 静态 IP ，hostname 设置，关闭防火墙，设置NTP时间同步服务

参见另一篇博文 link:/2018/10/13/CentOS 7 如何…\[CentOS 7 如何…\]

====== 设置 hosts 文件以便识别自定义的 hostname

    # vi /etc/hosts
    x.x.x.x linux-1
    x.x.x.x linux-2
    x.x.x.x linux-3
    x.x.x.x linux-4

然后 `scp /etc/hosts 其它机器主机名:/etc/hosts` 到其它机器。

====== 设置无密码访问 ssh

参见另一篇博文 link:/2018/10/13/免密码 ssh 到其它机器\[免密码 ssh
到其它机器\]

====== umask 文件默认权限设置

    ## 默认权限更改为 755
    umask 0022

====== 关闭 ssl 检查（注意，后面安装 HDP 还会有一次 openssl 相关的报错）

    vi /etc/python/cert-verification.cfg
    # verify=platformxxx 改为
    verify=disable

====== 安装 httpd 服务器作为后续离线安装包服务器

    yum install httpd
    chkconfig httpd on
    service httpd start

====== 关于 JDK

如果环境中没有 JDK，ambari 在安装设置阶段可以自动在线安装
JDK。但是如果已经安装了 JDK（要求1.8 版本），设置阶段指定 JDK\_HOME
所在的路径即可，参见后续 ambari-server setup 阶段。

====== 关于 python 和其它

如果你使用的是我上面提供的官方 CentOS 7 镜像，python 的版本应该为
2.7，不需要任何修改。 后续安装需要使用 wget 工具，请使用
`yum install wget` 安装。 后续安装需要使用其它软件源管理工具，请使用
`yum install yum-utils createrepo yum-plugin-priorities -y`
安装。执行以下修改，以关闭 gpg 校验（否则后面安装会报错）。

    vi /etc/yum/pluginconf.d/priorities.conf
    ## 添加或更改为以下内容
    gpgcheck = 0

安装包获取
==========

CentOS 7 采用 yum 安装
ambari-server，该软件可以通过\`\`在线\`\`和\`\`离线\`\`两种方式下载到本地，出于国情，很明显我们应该选择离线安装的方式。

====== 步骤1 下载离线包到本地

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 100%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;由于我们使用的是 Ambari 2.6.2.2 ，配套的 HDP 版本为 2.[4&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;5&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;6]，本文选用 HDP 2.6，附上需要下载的所有包路径(&lt;code&gt;以下 tar 包都需要下载&lt;/code&gt;)：&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;包名&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;路径&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;ambari&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.2/ambari-2.6.2.2-centos7.tar.gz&#34;&gt;点我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;HDP&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/HDP-2.6.5.0-centos7-rpm.tar.gz&#34;&gt;点我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HDP-UTILS&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz&#34;&gt;点我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

====== 步骤2 上传到需要安装 Ambari 的机器上，解压到 httpd 的服务器目录中

    cd /var/www/html
    mkdir hdp ambari
    tar zxvf &lt;你上传的路径&gt;/ambari-2.6.2.2-centos7.tar.gz -C /var/www/html/ambari
    tar zxvf &lt;你上传的路径&gt;/HDP-2.6.5.0-centos7-rpm.tar.gz -C /var/www/html/hdp
    tar zxvf &lt;你上传的路径&gt;/HDP-UTILS-1.1.0.22-centos7.tar.gz -C /var/www/html/hdp

====== 步骤3 使用 createrepo 工具配置生成源描述文件(可省略，会影响后续
repo 文件的路径配置)

    cd /var/www/html/ambari
    createrepo ./
    cd /var/www/html/hdp
    createrepo ./

====== 步骤4 下载 HDP 和 Ambari 的 yum repo 文件

    cd /etc/yum.repos.d
    wget -nv http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/hdp.repo
    wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.2/ambari.repo

====== 步骤5 配置 yum repo 文件指向本地的软件源

    vi /etc/yum.repos.d/ambari.repo
    ## 更改 baseurl 和 gpgcheck 两项
    baseurl=http:///&lt;你的主机名&gt;/ambari
    gpgcheck=0

    vi /etc/yum.repos.d/hdp.repo ## 根据步骤3，HDP 和 HDP-UTILS 可以使用同一个 baseurl
    ## 更改 baseurl 和 gpgcheck 两项
    baseurl=http:///&lt;你的主机名&gt;/hdp
    gpgcheck=0

====== 步骤6 刷新软件源

    yum clean all
    yum makecache
    ## 如果此过程出现 404 错误，检查 httpd 服务是否正常，或者步骤3

安装、配置、启动与登录
======================

此过程较为简单。

====== 安装

    yum install ambari-server

====== 配置

    ambari-server setup
    其中可以配置是否创建用户、JDK、Ambari 自用元数据库（默认 Postgre）等，可以选择一路回车。

====== 启动

    service ambari-server start

====== 登录

访问 `http://&lt;你的主机 IP 地址&gt;:8080/`，使用默认的 `admin/admin`
账户登录即可。

使用 Ambari 安装 hadoop 集群可以参考另外一篇博文link:/2018/10/13/使用
Ambari 安装 HDP 集群\[使用 Ambari 安装 hadoop 集群\]。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/centos7-ambari/  

