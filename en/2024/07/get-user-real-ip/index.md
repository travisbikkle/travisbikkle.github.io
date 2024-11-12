# Get User Real Ip


It&#39;s a pretty simple question, but in the process of information dissemination, some mistakes happen.

For example, some people will say, just configure X-Real-Ip, and that&#39;s it.

Or some people will ask, what is the difference between X-Real-Ip and X-Forward-For, and what is the principle?

Obviously, they misunderstood `X-Real-Ip`.

This article will show you what X-Real-Ip is.

## Quick Answer

X-Real-Ip is nothing. You can use My-Real-Ip, His-Real-Ip, whatever.

```lua
server {
        ...
        location / {
                ...
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                $proxy_add_x_forwarded_for; proxy_set_header Host $host; proxy_set_header
                proxy_set_header His-Real-Ip2 $remote_addr; }
        }
}
```
The above configuration sets three request headers, X-Forwarded-For, Host, and His-Real-Ip2, and the backend java server can get the real IP based on His-Real-Ip2.

It sounds hurtful, but `X-Real-Ip` really just seems to be an official, hidden field of some sort, which it really isn&#39;t.

Stop asking questions like `What is the principle of X-Real-Ip? ` or `special request header X-Real-Ip`, what really matters is the built-in variable $remote_addr.

## nginx in Docker

The $remote_addr could be 172.*.0.1 in docker nginx, which is the address of the docker bridge interface ip addr.

So how do you get the user&#39;s real ip? 

Your nginx container must use the host network.

Here is a docker compose example:

```yaml
services:
  my-nginx:
      restart: always
      container_name: my-nginx
      image: nginx:latest
      network_mode: host # note this mode is host
  my-server:
    restart: always
    container_name: my-server
    network_mode: host
      - spring_cloud_default
networks:
    spring_cloud_default:
    driver: bridge
```

In your back-end server code, you should write something like (using java as an example, assuming we are using His-Real-Ip2):

```java
RequestAttributes ra = RequestContextHolder.getRequestAttributes();
ServletRequestAttributes sra = (ServletRequestAttributes) ra;
if (null != sra) {
	HttpServletRequest request = sra.getRequest();
	realIp = request.getHeader(&#34;His-Real-Ip&#34;); // Let&#39;s say you have two nginx, a forward to b, b forward to server. the first nginx set His-Real-Ip
	if (notValidIp(realIp)) { // notValidIp can be implemented on your own, e.g. not starting with 172, or 192.168, etc.
		realIp = request.getHeader(&#34;His-Real-Ip2&#34;); // The second nginx set His-Real-Ip2. By doing so, it is possible to distinguish which traffic is coming in via the gateway and which is coming in directly. It&#39;s not useful, just as an illustration
		if (notValidIp(realIp)) {
			realIp = request.getRemoteAddr();
		}
	}
}
```

## Conclusion

Well, let&#39;s not make a falsehood out of a falsehood, 
although it&#39;s a simple question, it&#39;s not simple to realize it.

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/get-user-real-ip/  

