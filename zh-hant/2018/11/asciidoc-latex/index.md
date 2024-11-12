# 在 Asciidoc 文檔中使用 Latex

本文基於 asciidoctor 1.5.7.13，其通過 mathjax 實現 LaTex
字體的顯示，方法和 markdown 差不多，區別是
markdown（不同差距實現方法不同）使用 `$$` 或者 ``` $``$ ``` 包圍 LaTex
語法，而 asciidoctor 使用 `stem:[]` 包圍 LaTex 語法。

&lt;table&gt;
&lt;caption&gt;單個符號對照表&lt;/caption&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;渲染後&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;源碼&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\cdot]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\cdot]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\times]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\times]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[a^{prime} a]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[a^{prime} a]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[a’’]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[a’’]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[a’’’]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[a’’’]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\pm]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\pm]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\mp]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\mp]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[!]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[!]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\dots]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\dots]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\ldots]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\ldots]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\cdots]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\cdots]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\vdots]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\vdots]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;stem:[\ddots]&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;&lt;code&gt;stem:[\ddots]&lt;/code&gt;&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

&lt;table&gt;
&lt;caption&gt;行列式&lt;/caption&gt;
&lt;colgroup&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;col style=&#34;width: 50%&#34; /&gt;
&lt;/colgroup&gt;
&lt;tbody&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;p&gt;渲染後&lt;/p&gt;&lt;/td&gt;
&lt;td&gt;&lt;p&gt;源碼&lt;/p&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;15\\ 7 \end{array}right)&lt;/td&gt;
&lt;td&gt;&lt;div class=&#34;sourceCode&#34; id=&#34;cb1&#34;&gt;&lt;pre class=&#34;sourceCode latex&#34;&gt;&lt;code class=&#34;sourceCode latex&#34;&gt;&lt;span id=&#34;cb1-1&#34;&gt;&lt;a href=&#34;#cb1-1&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;fu&#34;&gt;\vec&lt;/span&gt;{a} =&lt;/span&gt;
&lt;span id=&#34;cb1-2&#34;&gt;&lt;a href=&#34;#cb1-2&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;fu&#34;&gt;\left&lt;/span&gt;[&lt;span class=&#34;kw&#34;&gt;\begin&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;array&lt;/span&gt;}{rrrr}  &lt;/span&gt;
&lt;span id=&#34;cb1-3&#34;&gt;&lt;a href=&#34;#cb1-3&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;  15&lt;span class=&#34;fu&#34;&gt;\\&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb1-4&#34;&gt;&lt;a href=&#34;#cb1-4&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;  7&lt;/span&gt;
&lt;span id=&#34;cb1-5&#34;&gt;&lt;a href=&#34;#cb1-5&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;kw&#34;&gt;\end&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;array&lt;/span&gt;}&lt;span class=&#34;fu&#34;&gt;\right&lt;/span&gt;)         &lt;/span&gt;&lt;/code&gt;&lt;/pre&gt;&lt;/div&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;odd&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;div class=&#34;sourceCode&#34; id=&#34;cb2&#34;&gt;&lt;pre class=&#34;sourceCode latex&#34;&gt;&lt;code class=&#34;sourceCode latex&#34;&gt;&lt;span id=&#34;cb2-1&#34;&gt;&lt;a href=&#34;#cb2-1&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;[latexmath]&lt;/span&gt;
&lt;span id=&#34;cb2-2&#34;&gt;&lt;a href=&#34;#cb2-2&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&#43;&#43;&#43;&#43;&lt;/span&gt;
&lt;span id=&#34;cb2-3&#34;&gt;&lt;a href=&#34;#cb2-3&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;kw&#34;&gt;\begin&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;cases&lt;/span&gt;}&lt;/span&gt;
&lt;span id=&#34;cb2-4&#34;&gt;&lt;a href=&#34;#cb2-4&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;u_{tt}(x,t)= b(t)&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\triangle&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt; u(x,t-4)&amp;amp;&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\\&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb2-5&#34;&gt;&lt;a href=&#34;#cb2-5&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;sc&#34;&gt;\ \hspace&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;{42pt}- q(x,t)f[u(x,t-3)]&#43;te^{-t}&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\sin&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;^2 x,  &amp;amp;  t &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\neq&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt; t_k; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\\&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb2-6&#34;&gt;&lt;a href=&#34;#cb2-6&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;u(x,t_k^&#43;) - u(x,t_k^-) = c_k u(x,t_k), &amp;amp; k=1,2,3&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ldots&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt; ;&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\\&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb2-7&#34;&gt;&lt;a href=&#34;#cb2-7&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;u_{t}(x,t_k^&#43;) - u_{t}(x,t_k^-) =c_k u_{t}(x,t_k), &amp;amp;&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb2-8&#34;&gt;&lt;a href=&#34;#cb2-8&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt; k=1,2,3&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ldots\ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;.&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb2-9&#34;&gt;&lt;a href=&#34;#cb2-9&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;kw&#34;&gt;\end&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;cases&lt;/span&gt;}&lt;/span&gt;
&lt;span id=&#34;cb2-10&#34;&gt;&lt;a href=&#34;#cb2-10&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;]&lt;/span&gt;
&lt;span id=&#34;cb2-11&#34;&gt;&lt;a href=&#34;#cb2-11&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&#43;&#43;&#43;&#43;&lt;/span&gt;&lt;/code&gt;&lt;/pre&gt;&lt;/div&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;tr class=&#34;even&#34;&gt;
&lt;td&gt;&lt;/td&gt;
&lt;td&gt;&lt;div class=&#34;sourceCode&#34; id=&#34;cb3&#34;&gt;&lt;pre class=&#34;sourceCode latex&#34;&gt;&lt;code class=&#34;sourceCode latex&#34;&gt;&lt;span id=&#34;cb3-1&#34;&gt;&lt;a href=&#34;#cb3-1&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;q(x,t)=&lt;/span&gt;
&lt;span id=&#34;cb3-2&#34;&gt;&lt;a href=&#34;#cb3-2&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;kw&#34;&gt;\begin&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;cases&lt;/span&gt;}&lt;span class=&#34;ss&#34;&gt;(t-k&#43;1)x^2,&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\quad&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ \ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;&amp;amp;&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb3-3&#34;&gt;&lt;a href=&#34;#cb3-3&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt;  t&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\in\big&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;(k-1,k-&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\dfrac&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;{1}{2}&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\big&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;],&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\\&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb3-4&#34;&gt;&lt;a href=&#34;#cb3-4&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;ss&#34;&gt;  (k-t)x^2, &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\quad&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt; &lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\ \ &lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;&amp;amp; t&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\in\big&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;(k-&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\dfrac&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;{1}{2},k&lt;/span&gt;&lt;span class=&#34;sc&#34;&gt;\big&lt;/span&gt;&lt;span class=&#34;ss&#34;&gt;],&lt;/span&gt;&lt;/span&gt;
&lt;span id=&#34;cb3-5&#34;&gt;&lt;a href=&#34;#cb3-5&#34; aria-hidden=&#34;true&#34;&gt;&lt;/a&gt;&lt;span class=&#34;kw&#34;&gt;\end&lt;/span&gt;{&lt;span class=&#34;ex&#34;&gt;cases&lt;/span&gt;}&lt;/span&gt;&lt;/code&gt;&lt;/pre&gt;&lt;/div&gt;&lt;/td&gt;
&lt;/tr&gt;
&lt;/tbody&gt;
&lt;/table&gt;

hexo中集成asciidoctor後渲染latex的bug
=====================================

hexo, asciidoctor, latex 三者在一起組成了一個很小衆的東西。實際上使用
ruby 安裝的 asciidoctor 在使用起來完全沒有問題，但是 hexo 中因爲使用的是
nodejs 中的一個
[hexo-renderer-asciidoc](https://github.com/hcoona/hexo-renderer-asciidoc/)
插件對 hexo 增強了 asciidoctor 的功能，並且該插件會對 `{` `}` 進行
[轉義](https://github.com/hcoona/hexo-renderer-asciidoc/blob/fc64b0e493ed81267c9573ef78b27523f2291018/lib/renderer.js#L33)，因此會導致莫名其妙的
[問題](https://github.com/hcoona/hexo-renderer-asciidoc/issues)出現。

另外 hexo 的 theme\\next 主題中有 mathjax 的配置，將其設置爲 true
後所有的頁面都會引用 mathjax 的 js。

**themes/next/\_config.yml.**

    mathjax:
      enable: true
      per_page: false
      cdn: //cdn.bootcss.com/mathjax/2.7.1/latest.js?config=TeX-AMS-MML_HTMLorMML

因此開啓此項配置後，結合插件，頁面上的 LaTex 公式就可以正常顯示了。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/11/asciidoc-latex/  

