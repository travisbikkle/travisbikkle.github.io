# Electron 中文搜索


這篇文章快速演示如何使用 js-search, nodejieba（結巴）來在 Electron 中實現中文搜索。

它快速，實時，比你見過的任何一種搜索都快，快到爆漿。

| tech      | version |
|-----------|---------|
| electron  | 30.0.6  |
| nodejieba | 2.6.0   |
| js-search | 2.0.1   |

本文將帶你解決在中國大陸使用 npm 鏡像及 nodejieba 可能遇到的一系列問題：

1. npmmirror 中的 nodejieba 包不存在或無法下載
2. nodejieba 無人維護，不支持在 win11 及 vs studio 2022 版本運行
3. nodejieba 不支持 typescript

## 添加依賴
```bash
npm i js-search
npm i nodejieba@2.6.0 --save-optional --ignore-scripts
```
爲什麼 nodejieba 要採取這種方式？因爲 nodejieba 是用 c&#43;&#43; 編寫，而它的社區已經不活躍了。它的編譯腳本會失敗。我們需要跳過它的腳本，自己編譯。

** 你需要安裝 vs studio 2022，並勾選使用 c&#43;&#43; 桌面開發 **。

或者使用下面的 powershell 命令，僅安裝需要的組件：

```powershell
Invoke-WebRequest -Uri &#39;https://aka.ms/vs/17/release/vs_BuildTools.exe&#39; -OutFile &#34;$env:TEMP\vs_BuildTools.exe&#34;

&amp; &#34;$env:TEMP\vs_BuildTools.exe&#34; --passive --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended
```

### 修復 nodejieba
nodejieba 不支持 c&#43;&#43; 17 標準，而修改方法很簡單。

你只需要在它編譯之前，將 github.com/yanyiwu/limonp 中的 StringUtil_latest.hpp 替換到 nodejieba 即可。

這是一個樣例。

```javascript
const fs = require(&#39;fs&#39;);
const path = require(&#39;path&#39;);
const projectDir = path.dirname(path.resolve(__dirname));

const patchFile = path.resolve(projectDir, &#39;SOME_FOLDER&#39;, &#39;StringUtil_latest.hpp&#39;); // 將 StringUtil.hpp 保存到本地的某個位置，如 SOME_FOLDER/StringUtil_latest.hpp

const dest = path.resolve(projectDir, &#39;node_modules&#39;, ...&#39;/nodejieba/deps/limonp/StringUtil.hpp&#39;.split(&#34;/&#34;));
// first install nodejieba with `npm install nodejieba@2.6.0 --ignore-scripts`
// https://github.com/yanyiwu/limonp/issues/34
fs.copyFile(patchFile, dest, (err) =&gt; {
  err &amp;&amp; console.error(err) &amp;&amp; process.exit(1);
})
```
&gt; [limonp-StringUtil.hpp](https://github.com/yanyiwu/limonp/raw/master/include/limonp/StringUtil.hpp)

你也可以選擇提交到 nodejieba 倉庫。我希望中國的開源軟件，都能善始善終，後繼有人。

### 修改 package.json
我們仍然希望打包的時候，nodejieba 可以被 electron-rebuild 識別。

```json
&#34;scripts&#34;: {
    &#34;preinstall&#34;: &#34;npm i nodejieba@2.6.0 --save-optional --ignore-scripts&#34;,
    &#34;build:plugin&#34;: &#34;electron-rebuild -f&#34;,
```

electron-rebuild 幫你完成 node-gyp 需要做的事情。 

&gt; [electron-rebuild](https://github.com/electron/rebuild)


## 爲 nodejieba 寫一個工具類

### 拷貝 nodejieba 的字典文件

假設你使用 Electron Builder，該段代碼將 node_modules/nodejieba/dict/ 拷貝到安裝目錄的根目錄。

```json
&#34;build&#34;: {
    &#34;extraFiles&#34;: [
      {
        &#34;from&#34;: &#34;node_modules/nodejieba/dict/&#34;,
        &#34;to&#34;: &#34;dict/&#34;
      }
    ],
```

不要更改以下代碼中的任意一行。

### 加載本地 node addon 的工具類
```typescript jsx
import fs from &#34;fs&#34;;
import path from &#34;path&#34;;
import * as process from &#34;process&#34;;
import bindings from &#34;bindings&#34;;
// eslint-disable-next-line import/no-extraneous-dependencies
import logger from &#34;_main/logger&#34;;
import nconsole from &#34;_rutils/nconsole&#34;;
import { dev } from &#39;_utils/node-env&#39;;

function loadAddon(pluginName: string) {
  logger.log(&#34;preloading plugin&#34;);
  let moduleRoot = path.dirname(process.execPath);
  let tries = [[&#34;module_root&#34;, &#34;bindings&#34;]];
  if (dev) {
    moduleRoot = process.cwd();
    tries = [[&#34;module_root&#34;, &#34;build&#34;, &#34;bindings&#34;]];
    if (!fs.existsSync(path.join(moduleRoot, &#34;build&#34;, pluginName &#43; &#34;.node&#34;))) {
      tries = [[&#34;module_root&#34;, &#34;bindings&#34;]];
    }
  }
  logger.log(&#34;using tries: &#34; &#43; JSON.stringify(tries));
  let nodeAddon;
  try {
    nodeAddon = bindings({
      bindings: pluginName,
      module_root: moduleRoot,
      try: tries,
    });
  } catch (e) {
    logger.error(e);
  }
  return nodeAddon;
}

export default loadAddon;

```

### 加載 nodejieba 插件

```typescript jsx
import path from &#34;path&#34;;
import loadAddon from &#39;./load_node_addon&#39;;

const jbAddon = loadAddon(&#34;fastx&#34;);

let dictDirRoot = process.cwd();
if (process.env.NODE_ENV === &#39;development&#39;) {
  dictDirRoot = path.resolve(process.cwd(), &#39;node_modules&#39;, &#39;nodejieba&#39;);
}

let isDictLoaded = false;

const defaultDict = {
  dict: `${dictDirRoot}/dict/jieba.dict.utf8`,
  hmmDict: `${dictDirRoot}/dict/hmm_model.utf8`,
  userDict: `${dictDirRoot}/dict/user.dict.utf8`,
  idfDict: `${dictDirRoot}/dict/idf.utf8`,
  stopWordDict: `${dictDirRoot}/dict/stop_words.utf8`,
};

interface LoadOptions {
  dict?: string;
  hmmDict?: string;
  userDict?: string;
  idfDict?: string;
  stopWordDict?: string;
}

export const load = (dictJson?: LoadOptions) =&gt; {
  const finalDictJson = {
    ...defaultDict,
    ...dictJson,
  };
  isDictLoaded = true;
  return jbAddon.load(
    finalDictJson.dict,
    finalDictJson.hmmDict,
    finalDictJson.userDict,
    finalDictJson.idfDict,
    finalDictJson.stopWordDict,
  );
};

export const DEFAULT_DICT = defaultDict.dict;
export const DEFAULT_HMM_DICT = defaultDict.hmmDict;
export const DEFAULT_USER_DICT = defaultDict.userDict;
export const DEFAULT_IDF_DICT = defaultDict.idfDict;
export const DEFAULT_STOP_WORD_DICT = defaultDict.stopWordDict;

export interface TagResult {
  word: string;
  tag: string;
}

export interface ExtractResult {
  word: string;
  weight: number;
}

const mustLoadDict = (f: any, ...args: any[]):any =&gt; {
  if (!isDictLoaded) {
    load();
  }
  return f.apply(undefined, [...args]);
};

export const cut = (content: string, strict: boolean): string[] =&gt; mustLoadDict(jbAddon.cut, content, strict);
export const cutAll = (content: string): string[] =&gt; mustLoadDict(jbAddon.cutAll, content);
export const cutForSearch = (content: string, strict: boolean): string[] =&gt; mustLoadDict(jbAddon.cutForSearch, content, strict);
export const cutHMM = (content: string): string[] =&gt; mustLoadDict(jbAddon.cutHMM, content);
export const cutSmall = (content: string, limit: number): string[] =&gt; mustLoadDict(jbAddon.cutSmall, content, limit);
export const extract = (content: string, threshold: number): ExtractResult[] =&gt; mustLoadDict(jbAddon.extract, content, threshold);
export const textRankExtract = (content: string, threshold: number): ExtractResult[] =&gt; mustLoadDict(jbAddon.textRankExtract, content, threshold);
export const insertWord = (word: string): boolean =&gt; mustLoadDict(jbAddon.insertWord, word);
export const tag = (content: string): TagResult[] =&gt; mustLoadDict(jbAddon.tag, content);

export default {
  load,
  cut,
  cutAll,
  cutForSearch,
  cutHMM,
  cutSmall,
  extract,
  textRankExtract,
  insertWord,
  tag,
  DEFAULT_DICT,
  DEFAULT_HMM_DICT,
  DEFAULT_USER_DICT,
  DEFAULT_IDF_DICT,
  DEFAULT_STOP_WORD_DICT,
};
```
你應該將該工具類，通過 window 暴露給 renderer 進程，然後 renderer 進程就可以調用這些方法，例如 window.myAddons.cutForSearch.

## 將 js-search 和 nodejieba 結合

假設你要搜索這樣一個對象。

```typescript jsx
export interface Product {
  [key: string]: any;

  productCode: string;
  name: string;
  namePinyin: string;
  nameEnglish: string;
}
```

你在搜索的組件中這樣寫：

```typescript jsx
import * as JsSearch from &#39;js-search&#39;;
import { Search } from &#39;js-search&#39;;

const [search, setSearch] = React.useState&lt;string&gt;(&#34;&#34;);
const jsSearchGames = React.useRef&lt;Search&gt;();
const [omnisearch_games, setOmnisearchGames] = React.useState&lt;any[]&gt;([]);
const [omnisearch_loading, setOmnisearchLoading] = React.useState(false);

// ... 

// 在頁面加載的時候，構造搜索控件和數據
useEffect(() =&gt; {
  const buildJsSearch = (uidField: string, documents: any[], ...index: string[]) =&gt; {
    const jsSearch = new JsSearch.Search(uidField);
    jsSearch.tokenizer = {
      tokenize: (text) =&gt; {
        const r = window.myAddons.cutForSearch(text, true); // cutForSearch 就是上面工具類中的方法
        return r;
      },
    };
    index.forEach((i) =&gt; jsSearch.addIndex(i));
    jsSearch.addDocuments(documents);
    return jsSearch;
  };


  jsSearchGames.current = buildJsSearch(&#39;productCode&#39;, p, &#39;productCode&#39;, &#39;name&#39;, &#39;namePinyin&#39;, &#39;nameEnglish&#39;);
}, []);

// 如果在搜索框中輸入了字符，將開始搜索
useEffect((): (() =&gt; void) | void =&gt; {
  if (!search) {
    return;
  }
  const q = search.trim();
  if (!q) {
    return;
  }
  setOmnisearchGames([]);
  setOmnisearchLoading(true);
  // 清空上一次的搜索，如果還沒超過1s的話
  if (currentSearchId.current) {
    clearTimeout(currentSearchId.current);
  }
  const doSearch = async () =&gt; new Promise&lt;searchResult&gt;((resolve, reject) =&gt; {
    // 1s 之後纔開始搜索
    currentSearchId.current = setTimeout(() =&gt; {
      const result = {
        sitemap: match_sitemap(q),
        games: jsSearchGames.current?.search(q) as Product[],
        gamesPrecisely: jsSearchGamesPrecisely.current?.search(q) as Product[],
        orders: jsSearchOrders.current?.search(q) as Order[],
        news: jsSearchNews.current?.search(q) as NotificationType[],
        tags: jsSearchTags.current?.search(q) as Tags[],
      };
      resolve(result);
    }, 200);
  });
  doSearch().then((d) =&gt; {
    setOmnisearchGames(d.games.filter((p) =&gt; p.type !== Constants.API_TYPE_PRODUCT &amp;&amp; p.type !== Constants.API_TYPE_GAMEBOX_APP));
    if (d.games.length === 0 &amp;&amp; q.length &gt;= 2 &amp;&amp; q.indexOf(&#34;&#39;&#34;) &lt; 0) {
      Object.keys(requests_in_flght.current).forEach((k) =&gt; {
        if (q.indexOf(k) === 0) {
          clearTimeout(requests_in_flght.current[k]);
          delete requests_in_flght.current[k];
        }
      });
      // cut q to keep its largest length to 32
      requests_in_flght.current[q] = setTimeout(() =&gt; {
        post(&#34;/saveRecord&#34;, {
          searchString: q.substring(0, 32),
        }).catch(() =&gt; {
        });
      }, 5000);
    }
  })
    .catch(openError)
    .finally(() =&gt; setOmnisearchLoading(false));
}, [search]);


return (
  &lt;div className=&#34;OmniSearch-container&#34;&gt;
    {inputElement()}
    {(search_focus || omniMouseOver || null) &amp;&amp; search &amp;&amp; (
      &lt;aside className=&#34;OmniSearch-results-container&#34;&gt;
        {(omnisearch_loading || null) &amp;&amp; &lt;div className=&#34;loading&#34;&gt;加載中&lt;/div&gt;}
        {((!omnisearch_loading &amp;&amp; omnisearch_result_count === 0) || null) &amp;&amp; (
          &lt;div className=&#34;no-results&#34;&gt;
            未找到
          &lt;/div&gt;
        )}
        {(omnisearch_games.length || null) &amp;&amp; (
          &lt;div className=&#34;results&#34;&gt;
            &lt;h3&gt;遊戲&lt;/h3&gt;
            {omnisearch_games.map((e) =&gt; (
              &lt;div className=&#34;result&#34; key={e.productCode}&gt;
                &lt;Link to={`/productDetail/${e.type}/${e.productCode}`}&gt;{e.name}&lt;/Link&gt;
              &lt;/div&gt;
            ))}
          &lt;/div&gt;
        )}
      &lt;/aside&gt;
    )}
  &lt;/div&gt;
);
```

## 完成
好了，按照這樣的思路，你就可以實現下面這種搜索效果了。

![](/images/posts/20240729-search-demo.gif)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/chinese-search/  

