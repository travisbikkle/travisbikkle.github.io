# Webview2 Tauri Bypass CORS


| item              | version  |
|-------------------|----------|
| tauri             | 2.0.0-rc |

## Prerequisites

When you use javascript at foo.com to access bar.com/api, you may encounter an error like the one below (but you are able to access it using curl):

```text
Access to fetch at &#39;bar.com/api&#39; 
from origin &#39;foo.com&#39; has been blocked by CORS policy: 
No &#39;Access-Control-Allow-Origin&#39; header is present on the requested resource. 
If an opaque response serves your needs, set the request&#39;s mode to &#39;no-cors&#39; to fetch the resource with CORS disabled.
```

First you need to understand a few concepts:

1. What is CORS
   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS

2. What are CORS preflighted requests and why using fetch etc. in js triggers CORS checking? And what is an OPTION request?
   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#preflighted_requests

3. What are simple requests and why will they not trigger preflighted requests?
   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests

4. Why does setting `Access-Control-Allow-Origin: *` in your request header not work?
   Access-Control-Allow-Origin is a response header, not request header.

   &gt; https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers

## How to fix this in Tauri

You can call rust in js to send the request. But I recommend you to use the official solution `@tauri-apps/plugin-http`:

1. Install dependencies
   ```bash
   pnpm add @tauri-apps/plugin-http
   ```
   add tauri-plugin-http to Cargo.toml.
   ```toml
   [dependencies]
   tauri-plugin-http = &#34;2.0.0-rc&#34;
   ```
   &gt; https://github.com/tauri-apps/tauri-plugin-http

2. Enable the appropriate permissions in capabilities\default.json

   ```json
   &#34;permissions&#34;: [
     &#34;http:allow-fetch&#34;,
     &#34;http:default&#34;,
     {
       &#34;identifier&#34;: &#34;http:default&#34;,
       &#34;allow&#34;: [{ &#34;url&#34;: &#34;https://**&#34; }]
     }
   ```
3. Register this plugin in lib.rs::run
   ```rust
       tauri::Builder::default()
           .setup(|app| {
              ... // omit 
           })
           .plugin(tauri_plugin_shell::init()) // another plugin
           .plugin(tauri_plugin_http::init())  // the plugin we need
   ```
   &gt; https://v2.tauri.app/plugin/http-client/

4. Called it (in typescript, for example)
   ```typescript
   import {fetch} from &#34;@tauri-apps/plugin-http&#34;;
   /// fetch(...)
   ```

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/08/tauri-bypass-cors/  

