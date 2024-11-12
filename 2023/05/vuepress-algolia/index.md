# Vuepress Algolia

Algolia 是一个优秀的静态网站搜索插件，非常适用于没有后台的静态网页博客系统（比如github pages）。

经过一定的申请步骤后（见下文链接），Algolia 通过爬虫定期爬取网站的数据将其存放到自己的数据库中。用户在网站上搜索的时候，就可以调用爬取的数据，显示搜索结果。但是 Algolia 有个缺点：它不是开箱即用的。参考以下两个链接，可以申请 Algolia 并在 VuePress 中做初步的配置。

[VuePress 博客优化之开启 Algolia 全文搜索](https://juejin.cn/post/7070109475419455519)

[VuePress 官网](https://v2.VuePress.vuejs.org/zh/reference/plugin/docsearch.html)

但是每个人的环境、步骤可能都不一样，尤其是如果你经历过从 VuePress 1.x 迁移过 VuePress 2.x，可能你会发现无论如何都无法搜索出结果，并且无法从网络上得到答案。

如果你遇到了这个问题，可以参考以下步骤进行排障：
1. 确定你的 VuePress 版本是 1.x 还是 2.x
可以在你的博客代码中的 package.json 中查看。
2. 进入 https://crawler.Algolia.com/admin/ 编辑器检查以下三处配置
 - 2.1 pathsToMatch 这一行默认带了一级 docs，请确认你的网站是否包含，如果不包含请删掉这里的docs
 - 2.2 确认 lvl0.selectors 这一行，是 `.sidebar-heading.active` 还是 `p.sidebar-heading.open`。前者对应的是 VuePress 2.x 版本，后者适配的是 VuePress 1.x 版本，如果错用，在此界面点击 URL Tester 的测试时将可以爬取到 URL 但是无法爬取到 index。如果确认错配了，请在[该网站](https://docsearch.Algolia.com/docs/templates/)找到正确的示例代码覆盖（注意替换为你的网站信息）
 - 2.3 确认你的 appId，apiKey，indexName 等是否正确。
以下是我的配置，基于 VuePress 2.x，请参考。
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

如果在 URL Tester 的调试界面看的以下信息（注意右下角的红色记录），证明问题解决，返回控制台重新运行一次爬虫（restart crawler）即可。

![](/images/cases/20230524_algolia_works.png)

## 题外话
正确的配置方法，在 [Algolia 的官网](https://docsearch.Algolia.com/docs/templates/) 以及 VuePress 的官网都是有的。那为什么还有这么多人配置错误，并且很难找到解决办法呢？
不能简单的归结于大家没有看文档。我认为有以下几点原因：
1. VuePress 2.x 还很年轻，网络上的配置很多是基于 1.x 的，访问 VuePress 官网默认跳转的也是 1.x 的官网，导致大家查询不到需要的信息
2. Algolia 的官网更是复杂，对非英语读者是不友好的，很少有人能通篇看完。并且 Algolia 只有在新建 crawler 的时候才会让你去选择是适配 VuePress 1.x 还是 2.x，而这个新建步骤，大部分人应该都没有经历过。因为在你申请之后，Algolia 默认会生成 1.x 版本的代码。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/05/vuepress-algolia/  

