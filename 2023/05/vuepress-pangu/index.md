# Vuepress .X 使用 Pangu.js 实现中英文自动空格


## 偏执狂的自述
&gt; 如果你跟我一樣，每次看到網頁上的中文字和英文、數字、符號擠在一塊，就會坐立難安，忍不住想在它們之間加個空格。這個外掛（支援 Chrome 和 Firefox）正是你在網路世界走跳所需要的東西，它會自動替你在網頁中所有的中文字和半形的英文、數字、符號之間插入空白。

&gt; 漢學家稱這個空白字元為「盤古之白」，因為它劈開了全形字和半形字之間的混沌。另有研究顯示，打字的時候不喜歡在中文和英文之間加空格的人，感情路都走得很辛苦，有七成的比例會在 34 歲的時候跟自己不愛的人結婚，而其餘三成的人最後只能把遺產留給自己的貓。畢竟愛情跟書寫都需要適時地留白。

&gt; 與大家共勉之。

以上这段恶毒的独白来自 `https://github.com/vinta/pangu.js/` 的偏执狂作者 vinta，但依然不能充分的表达有追求的杰青们看到中英文和符号挤在一块的不适。

好在现在可以使用 pangu.js 来帮你规避辛苦的感情路或者孤独终老的风险，赶快来看看如何在 VuePress 中使用吧！

## VuePress 1.x
VuePress 1.x 版本可以参考 &lt;https://shigma.github.io/markdown-it-pangu/vuepress.html&gt; 配置。

## VuePress 2.x
以上插件并没有适配 VuePress 2.x 的版本，但是好在 VuePress 2.x 预留了 [hook](https://v2.vuepress.vuejs.org/zh/reference/plugin-api.html#extendsmarkdown)，查看 `markdown-it-pangu` 的代码之后，我发现可以使用如下方法，略过插件，达成在 VuePress 2.x 中调用 pangu.js 的能力。

```javascript
// 以下代码在 config.ts 或者 config.js 中
import panguPlugin from &#39;markdown-it-pangu&#39;
// ... 省略
export default defineUserConfig({
  extendsMarkdown: (md) =&gt; {
      md.use(panguPlugin)
  },
// ... 省略
```

可以在任一 markdown 文件中测试，如：
```
测试是否可以在中英文之间自动添加空格 你好aa世界
```
将会如下显示：

![](/images/cases/20230525_vuepress2_pangu.png)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/05/vuepress-pangu/  

