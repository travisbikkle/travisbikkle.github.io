# 使用Nginx代理k8s Cadvisor一例


k8s 自带 cadvisor 监控，UI 界面监听在 4194 端口，不过 HW 的 k8s
这里监听的地址是 127.0.0.1，因此相当于是一个摆设。使用开源的 nginx
可以代理该 url
并暴露在一个可以访问的网卡上，不过出于学习的目的，使用我们自己编译的类似于
nginx 的一个 NSP 来实现这个目的。

着手
====

包地址在内网，无法提供。运行此包有三个限制：

1.  使用名称为 lb 的用户执行，否则会报错
    getpwnam(&#34;lb&#34;)，因为他们编译写死了执行用户

2.  LD\_LIBRARY\_PATH要加上包目录中的 lib, luajit/lib, lualib/ 三个目录

3.  包最好放在 /usr/local，因为编译写死了这个路径…

配置
====

配置好在仍然兼容开源 nginx，关键配置如下：

    upstream my_server {
        server 127.0.0.1:4194;
        keepalive 2000;
    }
    server {
        listen 4195;
        server_name 172.200.8.173;
        client_max_body_size 1024M;
        location / {
            proxy_pass http://127.0.0.1:4194;
            index index.html;
        }
    }

然后使用浏览器，访问[http://172.200.8.173:4195](http://172.200.8.173:4195)即可出现 cadvisor 的页面。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/09/nginx-k8s-cadvisor/  

