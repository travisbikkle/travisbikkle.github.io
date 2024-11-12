# Centos 安裝 Apache Ambari


版本說明
========

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;部件&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;版本號&lt;/p&gt;&lt;/td&gt;
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
&lt;td&gt;&lt;p&gt;時間&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;20180814&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

背景
====

對於 Ambari 能做什麼，對於搜索到此文的同學來說應該毋庸贅述。目前 Ambari
安裝的官方手冊主要是
[Apache](https://cwiki.apache.org/confluence/display/AMBARI/Installation&#43;Guide&#43;for&#43;Ambari&#43;2.6.2)
和
[Hortonworks](https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.2.2/bk_ambari-installation/content/install-ambari-server.html)，我首先是參考
Apache 的說明，通過 maven 編譯源碼的方式，在 安裝 linux-mint
的機器上嘗試安裝 `Ambari 2.7.0`，遇到過以下問題：

1.  由於系統不符合 Ambari 的要求，因此通過更改其中的
    `ambari-commons/OSCheck.py:is_ubuntu_family()` 函數強制安裝 server
    和 agent.

2.  由於採用的國內 maven 倉庫，ambari web legacy
    始終編譯不過，通過更改其依賴編譯通過.

3.  maven compiler plugin 報錯 json-simple 的相關依賴問題，最後刪除該
    legacy 模塊.

4.  其它 node, yarn, npm 的代理設置問題. 最終在安裝 agent 的時候遇到 ssl
    連接錯誤，時間已晚，選擇放棄這種安裝方式。轉而使用文檔支持較好的
    CentOS 和 yum 倉庫安裝的方式。雖然如此，Hortonworks
    的文檔邏輯也稍顯混亂，過於簡單，本文對安裝過程做詳細記錄，以備查詢。`本文所有操作均使用 root 用戶完成。`

準備
====

我使用的是 VMware WorkStation，CentOS 7 下載路徑爲
[點我](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-1804.iso)，安裝步驟略過，建議安裝
4 臺機器(網絡方式選擇NAT模式)，其中安裝 Ambari
服務器的機器硬盤大小不得小於 30
GB，如果不小心硬盤大小分配過小，參見另一篇博文link:/2018/10/13/如何對
centos 7 分區進行擴容/\[如何對 centos 7 分區進行擴容\]。其餘 3
臺機器作爲集羣機器以備後續使用。

====== 靜態 IP ，hostname 設置，關閉防火牆，設置NTP時間同步服務

參見另一篇博文 link:/2018/10/13/CentOS 7 如何…\[CentOS 7 如何…\]

====== 設置 hosts 文件以便識別自定義的 hostname

    # vi /etc/hosts
    x.x.x.x linux-1
    x.x.x.x linux-2
    x.x.x.x linux-3
    x.x.x.x linux-4

然後 `scp /etc/hosts 其它機器主機名:/etc/hosts` 到其它機器。

====== 設置無密碼訪問 ssh

參見另一篇博文 link:/2018/10/13/免密碼 ssh 到其它機器\[免密碼 ssh
到其它機器\]

====== umask 文件默認權限設置

    ## 默認權限更改爲 755
    umask 0022

====== 關閉 ssl 檢查（注意，後面安裝 HDP 還會有一次 openssl 相關的報錯）

    vi /etc/python/cert-verification.cfg
    # verify=platformxxx 改爲
    verify=disable

====== 安裝 httpd 服務器作爲後續離線安裝包服務器

    yum install httpd
    chkconfig httpd on
    service httpd start

====== 關於 JDK

如果環境中沒有 JDK，ambari 在安裝設置階段可以自動在線安裝
JDK。但是如果已經安裝了 JDK（要求1.8 版本），設置階段指定 JDK\_HOME
所在的路徑即可，參見後續 ambari-server setup 階段。

====== 關於 python 和其它

如果你使用的是我上面提供的官方 CentOS 7 鏡像，python 的版本應該爲
2.7，不需要任何修改。 後續安裝需要使用 wget 工具，請使用
`yum install wget` 安裝。 後續安裝需要使用其它軟件源管理工具，請使用
`yum install yum-utils createrepo yum-plugin-priorities -y`
安裝。執行以下修改，以關閉 gpg 校驗（否則後面安裝會報錯）。

    vi /etc/yum/pluginconf.d/priorities.conf
    ## 添加或更改爲以下內容
    gpgcheck = 0

安裝包獲取
==========

CentOS 7 採用 yum 安裝
ambari-server，該軟件可以通過\`\`在線\`\`和\`\`離線\`\`兩種方式下載到本地，出於國情，很明顯我們應該選擇離線安裝的方式。

====== 步驟1 下載離線包到本地

&lt;table&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 100%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;由於我們使用的是 Ambari 2.6.2.2 ，配套的 HDP 版本爲 2.[4&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;5&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;6]，本文選用 HDP 2.6，附上需要下載的所有包路徑(&lt;code&gt;以下 tar 包都需要下載&lt;/code&gt;)：&lt;/p&gt;&lt;/td&gt;
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
&lt;td&gt;&lt;p&gt;路徑&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;ambari&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.2/ambari-2.6.2.2-centos7.tar.gz&#34;&gt;點我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;HDP&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/HDP-2.6.5.0-centos7-rpm.tar.gz&#34;&gt;點我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;HDP-UTILS&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;a href=&#34;http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz&#34;&gt;點我&lt;/a&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

====== 步驟2 上傳到需要安裝 Ambari 的機器上，解壓到 httpd 的服務器目錄中

    cd /var/www/html
    mkdir hdp ambari
    tar zxvf &lt;你上傳的路徑&gt;/ambari-2.6.2.2-centos7.tar.gz -C /var/www/html/ambari
    tar zxvf &lt;你上傳的路徑&gt;/HDP-2.6.5.0-centos7-rpm.tar.gz -C /var/www/html/hdp
    tar zxvf &lt;你上傳的路徑&gt;/HDP-UTILS-1.1.0.22-centos7.tar.gz -C /var/www/html/hdp

====== 步驟3 使用 createrepo 工具配置生成源描述文件(可省略，會影響後續
repo 文件的路徑配置)

    cd /var/www/html/ambari
    createrepo ./
    cd /var/www/html/hdp
    createrepo ./

====== 步驟4 下載 HDP 和 Ambari 的 yum repo 文件

    cd /etc/yum.repos.d
    wget -nv http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/hdp.repo
    wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.2/ambari.repo

====== 步驟5 配置 yum repo 文件指向本地的軟件源

    vi /etc/yum.repos.d/ambari.repo
    ## 更改 baseurl 和 gpgcheck 兩項
    baseurl=http:///&lt;你的主機名&gt;/ambari
    gpgcheck=0

    vi /etc/yum.repos.d/hdp.repo ## 根據步驟3，HDP 和 HDP-UTILS 可以使用同一個 baseurl
    ## 更改 baseurl 和 gpgcheck 兩項
    baseurl=http:///&lt;你的主機名&gt;/hdp
    gpgcheck=0

====== 步驟6 刷新軟件源

    yum clean all
    yum makecache
    ## 如果此過程出現 404 錯誤，檢查 httpd 服務是否正常，或者步驟3

安裝、配置、啓動與登錄
======================

此過程較爲簡單。

====== 安裝

    yum install ambari-server

====== 配置

    ambari-server setup
    其中可以配置是否創建用戶、JDK、Ambari 自用元數據庫（默認 Postgre）等，可以選擇一路回車。

====== 啓動

    service ambari-server start

====== 登錄

訪問 `http://&lt;你的主機 IP 地址&gt;:8080/`，使用默認的 `admin/admin`
賬戶登錄即可。

使用 Ambari 安裝 hadoop 集羣可以參考另外一篇博文link:/2018/10/13/使用
Ambari 安裝 HDP 集羣\[使用 Ambari 安裝 hadoop 集羣\]。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/centos7-ambari/  

