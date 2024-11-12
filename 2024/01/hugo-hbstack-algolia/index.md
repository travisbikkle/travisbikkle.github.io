# 在 Hugo HBstack 主题中使用 Algolia


## TL;DR
[Algolia](https://docsearch.algolia.com/docs/templates/) 没有将 hugo 的 crawler 模板放上去，因为 hugo 是基于主题的，而每个主题使用的 css 都不一样。

如果你使用 hbstack，你可以参考我的 crawler 配置

```javascript
new Crawler({
  appId: &#34;xxxxxxxxx&#34;,  // 更改为你的appId
  apiKey: &#34;xxxxxxxxxx&#34;,  // 更改为你的apiKey
  rateLimit: 8,
  maxDepth: 10,
  startUrls: [&#34;https://travisbikkle.github.io/&#34;],
  // 注意这里的多语言链接，和你hugo.yaml中的配置有关系，你可能是单站点，如zh-hans.youdomain.com
  sitemaps: [
    &#34;https://travisbikkle.github.io/zh-hans/sitemap.xml&#34;,
    &#34;https://travisbikkle.github.io/zh-hant/sitemap.xml&#34;,
    &#34;https://travisbikkle.github.io/en/sitemap.xml&#34;,
  ],
  ignoreCanonicalTo: false,
  discoveryPatterns: [&#34;https://travisbikkle.github.io/**&#34;],
  schedule: &#34;every 1 day at 3:00 pm&#34;,
  actions: [
    {
      indexName: &#34;xxxxxxxx&#34;, // 更改为你的索引名称
      pathsToMatch: [&#34;https://travisbikkle.github.io/**&#34;, &#34;!*.xml&#34;],
      recordExtractor: ({ $, helpers }) =&gt; {
        return helpers.docsearch({
          recordProps: {
            lvl0: {
              selectors: &#34;&#34;,
              defaultValue: &#34;Title&#34;,
            },
            lvl1: [
              &#34;h1.hb-blog-post-title&#34;,
              &#34;.hb-blog-post-content h1&#34;,
              &#34;h1.hb-docs-doc-title&#34;,
              &#34;.breadcrumb-item.active a&#34;,
            ],
            lvl2: [&#34;.hb-blog-post-content h2&#34;, &#34;.hb-docs-doc-content h2&#34;],
            lvl3: [&#34;.hb-blog-post-content h3&#34;, &#34;.hb-docs-doc-content h3&#34;],
            lvl4: [&#34;.hb-blog-post-content h4&#34;, &#34;.hb-docs-doc-content h4&#34;],
            lvl5: [&#34;.hb-blog-post-content h5&#34;, &#34;.hb-docs-doc-content h5&#34;],
            content: [
              &#34;.hb-blog-post-content p, .hb-blog-post-content li&#34;,
              &#34;.hb-docs-doc-content p, .hb-docs-doc-content li&#34;,
            ],
          },
          indexHeadings: true,
          aggregateContent: true,
          recordVersion: &#34;v3&#34;,
        });
      },
    },
  ],
  initialIndexSettings: {
    snzhaoyuaio: {
      attributesForFaceting: [&#34;type&#34;, &#34;lang&#34;],
      attributesToRetrieve: [&#34;hierarchy&#34;, &#34;content&#34;, &#34;anchor&#34;, &#34;url&#34;],
      attributesToHighlight: [&#34;hierarchy&#34;, &#34;content&#34;],
      attributesToSnippet: [&#34;content:10&#34;],
      camelCaseAttributes: [&#34;hierarchy&#34;, &#34;content&#34;],
      searchableAttributes: [
        &#34;unordered(hierarchy.lvl0)&#34;,
        &#34;unordered(hierarchy.lvl1)&#34;,
        &#34;unordered(hierarchy.lvl2)&#34;,
        &#34;unordered(hierarchy.lvl3)&#34;,
        &#34;unordered(hierarchy.lvl4)&#34;,
        &#34;unordered(hierarchy.lvl5)&#34;,
        &#34;unordered(hierarchy.lvl6)&#34;,
        &#34;content&#34;,
      ],
      distinct: true,
      attributeForDistinct: &#34;url&#34;,
      customRanking: [
        &#34;desc(weight.pageRank)&#34;,
        &#34;desc(weight.level)&#34;,
        &#34;asc(weight.position)&#34;,
      ],
      ranking: [
        &#34;words&#34;,
        &#34;filters&#34;,
        &#34;typo&#34;,
        &#34;attribute&#34;,
        &#34;proximity&#34;,
        &#34;exact&#34;,
        &#34;custom&#34;,
      ],
      highlightPreTag: &#39;&lt;span class=&#34;algolia-docsearch-suggestion--highlight&#34;&gt;&#39;,
      highlightPostTag: &#34;&lt;/span&gt;&#34;,
      minWordSizefor1Typo: 3,
      minWordSizefor2Typos: 7,
      allowTyposOnNumericTokens: false,
      minProximity: 1,
      ignorePlurals: true,
      advancedSyntax: true,
      attributeCriteriaComputedByMinProximity: true,
      removeWordsIfNoResults: &#34;allOptional&#34;,
    },
  },
});
```

## 配置
本文假设你已经向 Algolia 申请了 appId 和 appKey。

你应该参考 [HBstack 官方网站的 params.yaml](https://github.com/hbstack/site/blob/main/config/_default/params.yml) 进行配置，建议 fork 一份全局搜索 `docsearch`，看懂并不难。


## 禁用 HBStack 中默认的 search 插件
配置好 Algolia 后，按 &lt;kbd&gt;CTRL K&lt;/kbd&gt; 会弹出两个窗口。

可以全局搜索 search 并在代码中注释掉，包括 go.mod 以及 yaml 配置，重新发布网站即可。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/01/hugo-hbstack-algolia/  

