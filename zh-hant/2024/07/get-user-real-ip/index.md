# 獲取用戶真實IP


這是一個非常簡單的問題，但是在信息傳播過程中，發生了一些錯誤。

比如一些人會說，你配置 X-Real-Ip 啊，這樣就行了。

或者有些人會問，X-Real-Ip 和 X-Forward-For 有什麼區別，原理是什麼？

儼然是把 X-Real-Ip 給誤解了。

本文帶你看看，到底什麼是 X-Real-Ip。

## 快速回答
X-Real-Ip 什麼也不是。你可以使用 My-Real-Ip，His-Real-Ip，隨便什麼字符串。

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
以上配置設置了三個請求頭，分別是 X-Forwarded-For、Host、His-Real-Ip2，後臺 java 服務端可以根據 His-Real-Ip2 獲取真實的 IP。

聽起來很傷人，可是 X-Real-Ip 真的只是看起來是官方的、某個隱藏的字段，實際上它並不是。

不要再問什麼 `X-Real-Ip 的原理是什麼？` 或者 `特殊的請求頭 X-Real-Ip` 這樣的問題了，實際上真正的值，是 nginx 的內置變量 $remote_addr。

## Docker 中的 nginx

docker 中的 nginx 可以獲取到的 $remote_addr，可能是 172.*.0.1。這些地址，是容器網橋的地址。

那麼如何獲取到用戶的真實 ip 呢？你的 nginx 容器必須使用主機網絡。

以下是一個 docker compose 示例：

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

在你的後端代碼中，考慮生產和開發環境，你應該做如下判斷（以 java 爲例，假設我們使用 His-Real-Ip2）：
```java
RequestAttributes ra = RequestContextHolder.getRequestAttributes();
ServletRequestAttributes sra = (ServletRequestAttributes) ra;
if (null != sra) {
	HttpServletRequest request = sra.getRequest();
	realIp = request.getHeader(&#34;His-Real-Ip&#34;); // 假設你有兩個 nginx，做了兩層轉發，第一層 nginx 設置了 His-Real-Ip
	if (notValidIp(realIp)) { // notValidIp 可以自己實現，比如不以 172 開頭，或者 192.168 等等開頭
		realIp = request.getHeader(&#34;His-Real-Ip2&#34;); // 第二層 nginx 設置了 His-Real-Ip2。通過這樣的設置，就能區分哪些流量是走網關進來，哪些是直連進來。沒什麼用，只是作爲一個說明
		if (notValidIp(realIp)) {
			realIp = request.getRemoteAddr();
		}
	}
}
```

## 結束語
好了，在技術工作過程中，我們不要以訛傳訛，雖然這是一個簡單的問題，但是要說清它，並不簡單。



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/get-user-real-ip/  

