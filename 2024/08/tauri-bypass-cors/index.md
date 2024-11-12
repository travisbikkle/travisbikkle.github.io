# Webview2 Tauri 绕过 CORS


| item              | version  |
|-------------------|----------|
| tauri             | 2.0.0-rc |

## 前提

当你在 foo.com 使用 js 来访问 bar.com/api 的时候，你可能会遇到下面这样的错误（但你能够使用 curl 访问它）：

```text
Access to fetch at &#39;bar.com/api&#39; 
from origin &#39;foo.com&#39; has been blocked by CORS policy: 
No &#39;Access-Control-Allow-Origin&#39; header is present on the requested resource. 
If an opaque response serves your needs, set the request&#39;s mode to &#39;no-cors&#39; to fetch the resource with CORS disabled.
```

首先你需要了解几个概念：

1. 什么是 CORS
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS

2. 什么是 CORS 预检请求，为什么在 js 中使用 fetch 等会会触发CORS检查？以及什么是 OPTION 操作？
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#%E9%A2%84%E6%A3%80%E8%AF%B7%E6%B1%82

3. 什么是简单请求，为什么简单请求不会触发CORS检查？
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#%E7%AE%80%E5%8D%95%E8%AF%B7%E6%B1%82

4. 为什么你在请求头中设置 `Access-Control-Allow-Origin: *` 无效？
   Access-Control-Allow-Origin 是一个响应头，在服务端设置。

   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers

## 如何在 Tauri 中解决这个问题

你可以在 js 中调用 rust 发送请求。但是我推荐你用官方的方案 `@tauri-apps/plugin-http`：

1. 安装依赖
   ```bash
   pnpm add @tauri-apps/plugin-http
   ```
   并在 src-tauri 工程的 Cargo.toml 中加入 tauri-plugin-http.
   ```toml
   [dependencies]
   tauri-plugin-http = &#34;2.0.0-rc&#34;
   ```
   &gt; https://github.com/tauri-apps/tauri-plugin-http

2. 在 capabilities\default.json 中开启相应的权限

   ```json
   &#34;permissions&#34;: [
     &#34;http:allow-fetch&#34;,
     &#34;http:default&#34;,
     {
       &#34;identifier&#34;: &#34;http:default&#34;,
       &#34;allow&#34;: [{ &#34;url&#34;: &#34;https://**&#34; }]
     }
   ```
3. 在 lib.rs 中注册该插件
   ```rust
       tauri::Builder::default()
           .setup(|app| {
              ... // 省略 
           })
           .plugin(tauri_plugin_shell::init()) // 其它插件
           .plugin(tauri_plugin_http::init())  // 本插件
   ```
   &gt; https://v2.tauri.app/plugin/http-client/

4. 在前端调用（以 typescript 为例）
   ```typescript
   import {fetch} from &#34;@tauri-apps/plugin-http&#34;;
   /// fetch(...)
   ```

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/08/tauri-bypass-cors/  

