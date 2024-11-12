# Vuepress Algolia

Algolia 是一個優秀的靜態網站搜索插件，非常適用於沒有後臺的靜態網頁博客系統（比如github pages）。

經過一定的申請步驟後（見下文鏈接），Algolia 通過爬蟲定期爬取網站的數據將其存放到自己的數據庫中。用戶在網站上搜索的時候，就可以調用爬取的數據，顯示搜索結果。但是 Algolia 有個缺點：它不是開箱即用的。參考以下兩個鏈接，可以申請 Algolia 並在 VuePress 中做初步的配置。

[VuePress 博客優化之開啓 Algolia 全文搜索](https://juejin.cn/post/7070109475419455519)

[VuePress 官網](https://v2.VuePress.vuejs.org/zh/reference/plugin/docsearch.html)

但是每個人的環境、步驟可能都不一樣，尤其是如果你經歷過從 VuePress 1.x 遷移過 VuePress 2.x，可能你會發現無論如何都無法搜索出結果，並且無法從網絡上得到答案。

如果你遇到了這個問題，可以參考以下步驟進行排障：
1. 確定你的 VuePress 版本是 1.x 還是 2.x
可以在你的博客代碼中的 package.json 中查看。
2. 進入 https://crawler.Algolia.com/admin/ 編輯器檢查以下三處配置
 - 2.1 pathsToMatch 這一行默認帶了一級 docs，請確認你的網站是否包含，如果不包含請刪掉這裏的docs
 - 2.2 確認 lvl0.selectors 這一行，是 `.sidebar-heading.active` 還是 `p.sidebar-heading.open`。前者對應的是 VuePress 2.x 版本，後者適配的是 VuePress 1.x 版本，如果錯用，在此界面點擊 URL Tester 的測試時將可以爬取到 URL 但是無法爬取到 index。如果確認錯配了，請在[該網站](https://docsearch.Algolia.com/docs/templates/)找到正確的示例代碼覆蓋（注意替換爲你的網站信息）
 - 2.3 確認你的 appId，apiKey，indexName 等是否正確。
以下是我的配置，基於 VuePress 2.x，請參考。
   ```javascript
   new Crawler({
     appId: &#34;xxxxxx&#34;,
     apiKey: &#34;xxxxxx&#34;,
     rateLimit: 8,
     maxDepth: 10,
     startUrls: [&#34;https://travisbikkle.github.io/&#34;],
     sitemaps: [&#34;https://travisbikkle.github.io/sitemap.xml&#34;],
     ignoreCanonicalTo: false,
     discoveryPatterns: [&#34;https://travisbikkle.github.io/**&#34;],
     actions: [
       {
         indexName: &#34;snzhaoyuaio&#34;,
         pathsToMatch: [&#34;https://travisbikkle.github.io/**&#34;],
         recordExtractor: ({ $, helpers }) =&gt; {
           return helpers.docsearch({
             recordProps: {
               lvl0: {
                 selectors: &#34;.sidebar-heading.active&#34;,
                 defaultValue: &#34;Documentation&#34;,
               },
               lvl1: &#34;.theme-default-content h1&#34;,
               lvl2: &#34;.theme-default-content h2&#34;,
               lvl3: &#34;.theme-default-content h3&#34;,
               lvl4: &#34;.theme-default-content h4&#34;,
               lvl5: &#34;.theme-default-content h5&#34;,
               content: &#34;.theme-default-content p, .theme-default-content li&#34;,
             },
             indexHeadings: true,
             aggregateContent: true,
   ```

如果在 URL Tester 的調試界面看的以下信息（注意右下角的紅色記錄），證明問題解決，返回控制檯重新運行一次爬蟲（restart crawler）即可。

![](/images/cases/20230524_algolia_works.png)

## 題外話
正確的配置方法，在 [Algolia 的官網](https://docsearch.Algolia.com/docs/templates/) 以及 VuePress 的官網都是有的。那爲什麼還有這麼多人配置錯誤，並且很難找到解決辦法呢？
不能簡單的歸結於大家沒有看文檔。我認爲有以下幾點原因：
1. VuePress 2.x 還很年輕，網絡上的配置很多是基於 1.x 的，訪問 VuePress 官網默認跳轉的也是 1.x 的官網，導致大家查詢不到需要的信息
2. Algolia 的官網更是複雜，對非英語讀者是不友好的，很少有人能通篇看完。並且 Algolia 只有在新建 crawler 的時候纔會讓你去選擇是適配 VuePress 1.x 還是 2.x，而這個新建步驟，大部分人應該都沒有經歷過。因爲在你申請之後，Algolia 默認會生成 1.x 版本的代碼。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/05/vuepress-algolia/  

