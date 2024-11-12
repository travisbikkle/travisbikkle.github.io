# 获取用户真实IP


这是一个非常简单的问题，但是在信息传播过程中，发生了一些错误。

比如一些人会说，你配置 X-Real-Ip 啊，这样就行了。

或者有些人会问，X-Real-Ip 和 X-Forward-For 有什么区别，原理是什么？

俨然是把 X-Real-Ip 给误解了。

本文带你看看，到底什么是 X-Real-Ip。

## 快速回答
X-Real-Ip 什么也不是。你可以使用 My-Real-Ip，His-Real-Ip，随便什么字符串。

```lua
server {
        ...
        location / {
                ...
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $host;
                proxy_set_header His-Real-Ip2 $remote_addr;
        }
}
```
以上配置设置了三个请求头，分别是 X-Forwarded-For、Host、His-Real-Ip2，后台 java 服务端可以根据 His-Real-Ip2 获取真实的 IP。

听起来很伤人，可是 X-Real-Ip 真的只是看起来是官方的、某个隐藏的字段，实际上它并不是。

不要再问什么 `X-Real-Ip 的原理是什么？` 或者 `特殊的请求头 X-Real-Ip` 这样的问题了，实际上真正的值，是 nginx 的内置变量 $remote_addr。

## Docker 中的 nginx

docker 中的 nginx 可以获取到的 $remote_addr，可能是 172.*.0.1。这些地址，是容器网桥的地址。

那么如何获取到用户的真实 ip 呢？你的 nginx 容器必须使用主机网络。

以下是一个 docker compose 示例：

```yaml
  my-nginx:
    restart: always
    container_name: my-nginx
    image: nginx:latest
    network_mode: host  # note this mode is host
  my-server:
    restart: always
    container_name: my-server
    networks:
      - spring_cloud_default
  networks:
    spring_cloud_default:
      driver: bridge
```

在你的后端代码中，考虑生产和开发环境，你应该做如下判断（以 java 为例，假设我们使用 His-Real-Ip2）：
```java
RequestAttributes ra = RequestContextHolder.getRequestAttributes();
ServletRequestAttributes sra = (ServletRequestAttributes) ra;
if (null != sra) {
	HttpServletRequest request = sra.getRequest();
	realIp = request.getHeader(&#34;His-Real-Ip&#34;); // 假设你有两个 nginx，做了两层转发，第一层 nginx 设置了 His-Real-Ip
	if (notValidIp(realIp)) { // notValidIp 可以自己实现，比如不以 172 开头，或者 192.168 等等开头
		realIp = request.getHeader(&#34;His-Real-Ip2&#34;); // 第二层 nginx 设置了 His-Real-Ip2。通过这样的设置，就能区分哪些流量是走网关进来，哪些是直连进来。没什么用，只是作为一个说明
		if (notValidIp(realIp)) {
			realIp = request.getRemoteAddr();
		}
	}
}
```

## 结束语
好了，在技术工作过程中，我们不要以讹传讹，虽然这是一个简单的问题，但是要说清它，并不简单。



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/07/get-user-real-ip/  

