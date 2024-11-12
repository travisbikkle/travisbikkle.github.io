# 使用Nginx代理k8s Cadvisor一例


k8s 自帶 cadvisor 監控，UI 界面監聽在 4194 端口，不過 HW 的 k8s
這裏監聽的地址是 127.0.0.1，因此相當於是一個擺設。使用開源的 nginx
可以代理該 url
並暴露在一個可以訪問的網卡上，不過出於學習的目的，使用我們自己編譯的類似於
nginx 的一個 NSP 來實現這個目的。

着手
====

包地址在內網，無法提供。運行此包有三個限制：

1.  使用名稱爲 lb 的用戶執行，否則會報錯
    getpwnam(&#34;lb&#34;)，因爲他們編譯寫死了執行用戶

2.  LD\_LIBRARY\_PATH要加上包目錄中的 lib, luajit/lib, lualib/ 三個目錄

3.  包最好放在 /usr/local，因爲編譯寫死了這個路徑…

配置
====

配置好在仍然兼容開源 nginx，關鍵配置如下：

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

然後使用瀏覽器，訪問[http://172.200.8.173:4195](http://172.200.8.173:4195)即可出現 cadvisor 的頁面。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/09/nginx-k8s-cadvisor/  

