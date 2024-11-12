# Electron-Log@5 的使用


我们使用一个开源软件，一般都是通过搜索示例或教程来学习如何使用。可是如果一个软件发生过不兼容的变更，而网络上一般都是些旧的做法，这样就会给我们带来非常大的心智负担。

因此我建议有志之士尽量去看官方文档，即使它可能是英语，或者没有太好的示例，仍然要去看，而不是被一些没头没尾或者抄来抄去的博客耽误你的时间。

Electron 的官方文档就非常得不详细，它也在 main，preload，renderer 这一块变过很多次，详情可以看看博客中的上一篇文章《在 webapck&#43;electron&#43;typescript中使用go开发的node插件》。

本文说说一个常用的日志小工具 electron-log 的使用（基于 5.1.4）。

1. 引入
   ```bash
   npm i electron-log
   ```
2. 自己定义一个 logger.ts（也可以在 main.ts 中直接定义）
   ```javascript
   import log from &#39;electron-log/main&#39;;
   log.transports.console.level = false; // 控制台关闭输出（只输出到文件）
   log.transports.file.level = &#39;silly&#39;;
   log.transports.file.maxSize = 1002430; // 文件最大不超过 1M
   log.transports.file.format = &#39;[{y}-{m}-{d} {h}:{i}:{s}.{ms}] [{level}]{scope} {text}&#39;;
   const date = new Date();
   const dateStr = date.getFullYear() &#43; &#39;-&#39; &#43; (date.getMonth() &#43; 1) &#43; &#39;-&#39; &#43; date.getDate();
   log.transports.file.resolvePathFn = () =&gt; &#39;log\\&#39; &#43; dateStr &#43; &#39;.log&#39;; // 在程序的安装目录（生产），或代码根目录（开发）的 log 文件夹下打印日志。忘掉 %USERPROFILE%\AppData\... 吧
   export default log;
   ```
3. 在 main.ts 中引入 logger
   ```javascript
   import logger from &#39;_main/logger&#39;; // 路径自己修改一下到你的 logger.ts，本文已做简化
   logger.initialize(); // 在任何 logger 使用之前执行
   ```
4. 在 preload 或者 renderer 进程中使用
   ```javascript
   import logger from &#34;_main/logger&#34;; // 路径自己修改一下到你的 logger.ts，本文已做简化
   logger.log(&#34;一行日志&#34;)
   ```

旧的方法比如使用 ipc 等，在老版本是可以用的。但是在 5.1.4 版本，打包后的程序会无法打印日志。

&gt; https://github.com/megahertz/electron-log/blob/master/docs/migration.md

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/05/2024-05-17-electron-log-v5/  

