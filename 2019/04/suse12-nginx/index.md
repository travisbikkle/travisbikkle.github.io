# Suse 安装 Nginx


&lt;http://nginx.org/packages/mainline/&gt;
&lt;http://nginx.org/packages/mainline/sles/12/x86_64/&gt;
&lt;http://nginx.org/packages/mainline/sles/12/x86_64/RPMS/nginx-1.15.12-1.sles12.ngx.x86_64.rpm&gt;

rpm -ivh nginx-1.15.12-1.sles12.ngx.x86\_64.rpm

autoindex vi /etc/nginx/conf.d/default.conf

        location / {
            root   /var/www/html;
            index  index.html index.htm;
            autoindex on;
            autoindex_exact_size off;
            autoindex_localtime on;
        }

chmod -R 777 /var/www /usr/sbin/nginx -c /etc/nginx/nginx.conf


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/04/suse12-nginx/  

