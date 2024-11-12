# Webview2 Tauri 繞過 CORS


| item              | version  |
|-------------------|----------|
| tauri             | 2.0.0-rc |

## 前提

當你在 foo.com 使用 js 來訪問 bar.com/api 的時候，你可能會遇到下面這樣的錯誤（但你能夠使用 curl 訪問它）：

```text
Access to fetch at &#39;bar.com/api&#39; 
from origin &#39;foo.com&#39; has been blocked by CORS policy: 
No &#39;Access-Control-Allow-Origin&#39; header is present on the requested resource. 
If an opaque response serves your needs, set the request&#39;s mode to &#39;no-cors&#39; to fetch the resource with CORS disabled.
```

首先你需要了解幾個概念：

1. 什麼是 CORS
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS

2. 什麼是 CORS 預檢請求，爲什麼在 js 中使用 fetch 等會會觸發CORS檢查？以及什麼是 OPTION 操作？
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#%E9%A2%84%E6%A3%80%E8%AF%B7%E6%B1%82

3. 什麼是簡單請求，爲什麼簡單請求不會觸發CORS檢查？
   &gt; https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#%E7%AE%80%E5%8D%95%E8%AF%B7%E6%B1%82

4. 爲什麼你在請求頭中設置 `Access-Control-Allow-Origin: *` 無效？
   Access-Control-Allow-Origin 是一個響應頭，在服務端設置。

   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers

## 如何在 Tauri 中解決這個問題

你可以在 js 中調用 rust 發送請求。但是我推薦你用官方的方案 `@tauri-apps/plugin-http`：

1. 安裝依賴
   ```bash
   pnpm add @tauri-apps/plugin-http
   ```
   並在 src-tauri 工程的 Cargo.toml 中加入 tauri-plugin-http.
   ```toml
   [dependencies]
   tauri-plugin-http = &#34;2.0.0-rc&#34;
   ```
   &gt; https://github.com/tauri-apps/tauri-plugin-http

2. 在 capabilities\default.json 中開啓相應的權限

   ```json
   &#34;permissions&#34;: [
     &#34;http:allow-fetch&#34;,
     &#34;http:default&#34;,
     {
       &#34;identifier&#34;: &#34;http:default&#34;,
       &#34;allow&#34;: [{ &#34;url&#34;: &#34;https://**&#34; }]
     }
   ```
3. 在 lib.rs 中註冊該插件
   ```rust
       tauri::Builder::default()
           .setup(|app| {
              ... // 省略 
           })
           .plugin(tauri_plugin_shell::init()) // 其它插件
           .plugin(tauri_plugin_http::init())  // 本插件
   ```
   &gt; https://v2.tauri.app/plugin/http-client/

4. 在前端調用（以 typescript 爲例）
   ```typescript
   import {fetch} from &#34;@tauri-apps/plugin-http&#34;;
   /// fetch(...)
   ```

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/08/tauri-bypass-cors/  

