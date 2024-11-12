# Electron-Log@5 的使用


我們使用一個開源軟件，一般都是通過搜索示例或教程來學習如何使用。可是如果一個軟件發生過不兼容的變更，而網絡上一般都是些舊的做法，這樣就會給我們帶來非常大的心智負擔。

因此我建議有志之士儘量去看官方文檔，即使它可能是英語，或者沒有太好的示例，仍然要去看，而不是被一些沒頭沒尾或者抄來抄去的博客耽誤你的時間。

Electron 的官方文檔就非常得不詳細，它也在 main，preload，renderer 這一塊變過很多次，詳情可以看看博客中的上一篇文章《在 webapck&#43;electron&#43;typescript中使用go開發的node插件》。

本文說說一個常用的日誌小工具 electron-log 的使用（基於 5.1.4）。

1. 引入
   ```bash
   npm i electron-log
   ```
2. 自己定義一個 logger.ts（也可以在 main.ts 中直接定義）
   ```javascript
   import log from &#39;electron-log/main&#39;;
   log.transports.console.level = false; // 控制檯關閉輸出（只輸出到文件）
   log.transports.file.level = &#39;silly&#39;;
   log.transports.file.maxSize = 1002430; // 文件最大不超過 1M
   log.transports.file.format = &#39;[{y}-{m}-{d} {h}:{i}:{s}.{ms}] [{level}]{scope} {text}&#39;;
   const date = new Date();
   const dateStr = date.getFullYear() &#43; &#39;-&#39; &#43; (date.getMonth() &#43; 1) &#43; &#39;-&#39; &#43; date.getDate();
   log.transports.file.resolvePathFn = () =&gt; &#39;log\\&#39; &#43; dateStr &#43; &#39;.log&#39;; // 在程序的安裝目錄（生產），或代碼根目錄（開發）的 log 文件夾下打印日誌。忘掉 %USERPROFILE%\AppData\... 吧
   export default log;
   ```
3. 在 main.ts 中引入 logger
   ```javascript
   import logger from &#39;_main/logger&#39;; // 路徑自己修改一下到你的 logger.ts，本文已做簡化
   logger.initialize(); // 在任何 logger 使用之前執行
   ```
4. 在 preload 或者 renderer 進程中使用
   ```javascript
   import logger from &#34;_main/logger&#34;; // 路徑自己修改一下到你的 logger.ts，本文已做簡化
   logger.log(&#34;一行日誌&#34;)
   ```

舊的方法比如使用 ipc 等，在老版本是可以用的。但是在 5.1.4 版本，打包後的程序會無法打印日誌。

&gt; https://github.com/megahertz/electron-log/blob/master/docs/migration.md

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/05/2024-05-17-electron-log-v5/  

