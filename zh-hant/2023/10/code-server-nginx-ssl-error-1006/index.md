# Code Server Nginx Ssl Error


```text
An unexpected error occurred that requires a reload of this page.
The workbench failed to connect to the server (Error: WebSocket close with status code 1006)
```

Code server popped up a websocket error when I exposed code-server using a custom domain and a custom port via nginx.

![20231017_code_server_nginx_error_1006.png](/images/other/20231017_code_server_nginx_error_1006.png)

Here is part of my nginx.conf.

```lua
  upstream code{
      server xxxxxxxxx;
  }
  server {
    listen 40032 ssl;
    server_name xxxxxxxx;
    ssl_certificate certs/mycert.pem;
    ssl_certificate_key certs/mykey.pem;
    ssl_ciphers HIGH:!aNULL:!MD5;
    charset utf-8;
    client_max_body_size 500M;
    location / {
        proxy_pass http://code;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection &#34;upgrade&#34;;
        proxy_http_version 1.1;
        proxy_set_header Accept-Encoding gzip;
    }
  }

```
Websocket support has already been enabled by adding `proxy_http_version 1.1` and `proxy_set_header Upgrade $http_upgrade`,
but this error is still being reported.

The key to the problem lies in the `proxy_set_header Host $host;` in the above configuration, after changing it to 
`proxy_set_header Host $http_host;`, problem disappeared.

Here is the explanation:

https://stackoverflow.com/questions/15414810/whats-the-difference-of-host-and-http-host-in-nginx/76875724#76875724

Based on Nginx 1.18, code-server: v4.17.1


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/10/code-server-nginx-ssl-error-1006/  

