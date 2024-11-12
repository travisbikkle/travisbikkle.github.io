# Electron Chinese Search


This is a quick demo of how to use js-search, nodejieba to implement Chinese search in Electron.

It&#39;s fast, real-time, faster than any other chinese search solutions, fast like never before. 

| tech      | version |
|-----------|---------|
| electron  | 30.0.6  |
| nodejieba | 2.6.0   |
| js-search | 2.0.1   |

This article will walk you through a number of issues you may encounter when using npm mirrors and nodejieba in mainland China:

1. nodejieba package in npmmirror.com does not exist or cannot be downloaded.
2. nodejieba is unmaintained and is not supported on win11 and vs studio 2022 versions.
3. nodejieba does not support typescript.

## Add dependencies
```bash
npm i js-search
npm i nodejieba@2.6.0 --save-optional --ignore-scripts
```
Why does nodejieba take this approach? 

Because nodejieba is written in c&#43;&#43; and its community is no longer active. 

Its installation scripts will fail. We need to skip its scripts and compile it ourselves.

** You need to install vs studio 2022 and check Use c&#43;&#43; desktop development **.

Or use the following powershell command to install only the needed components:

```powershell
Invoke-WebRequest -Uri &#39;https://aka.ms/vs/17/release/vs_BuildTools.exe&#39; -OutFile &#34;$env:TEMP\vs_BuildTools.exe&#34;

&amp; &#34;$env:TEMP\vs_BuildTools.exe&#34; --passive --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended
```

### Fix nodejieba
nodejieba does not support the c&#43;&#43; 17 standard, and the way to fix it is simple.

You just need to replace StringUtil_latest.hpp in github.com/yanyiwu/limonp with nodejieba before it compiles.

Here&#39;s a sample.

```javascript
const fs = require(&#39;fs&#39;);
const path = require(&#39;path&#39;);
const projectDir = path.dirname(path.resolve(__dirname));

const patchFile = path.resolve(projectDir, &#39;SOME_FOLDER&#39;, &#39;StringUtil_latest.hpp&#39;); // Save StringUtil.hpp to a local location such as SOME_FOLDER/StringUtil_latest.hpp

const dest = path.resolve(projectDir, &#39;node_modules&#39;, ...&#39;/nodejieba/deps/limonp/StringUtil.hpp&#39;.split(&#34;/&#34;));
// first install nodejieba with `npm install nodejieba@2.6.0 --ignore-scripts`
// https://github.com/yanyiwu/limonp/issues/34
fs.copyFile(patchFile, dest, (err) =&gt; {
  err &amp;&amp; console.error(err) &amp;&amp; process.exit(1);
})
```
&gt; [limonp-StringUtil.hpp](https://github.com/yanyiwu/limonp/raw/master/include/limonp/StringUtil.hpp)

You can also choose to create a pr to the [nodejieba repository](https://github.com/yanyiwu/nodejieba). 

I hope that all China&#39;s open source software will have a good start and also a good finish.

### Modify package.json

We still want nodejieba to be recognized by electron-rebuild when it is packaged.

```json
&#34;scripts&#34;: {
    &#34;preinstall&#34;: &#34;npm i nodejieba@2.6.0 --save-optional --ignore-scripts&#34;,
    &#34;build:plugin&#34;: &#34;electron-rebuild -f&#34;,
```

electron-rebuild helps you do what node-gyp needs to do.

&gt; [electron-rebuild](https://github.com/electron/rebuild)


## Write a tool to load nodejieba.

### Copying nodejieba&#39;s dictionary file

Assuming you are using Electron Builder, this code copies node_modules/nodejieba/dict/ to the root of the installation directory.

```json
&#34;build&#34;: {
    &#34;extraFiles&#34;: [
      {
        &#34;from&#34;: &#34;node_modules/nodejieba/dict/&#34;,
        &#34;to&#34;: &#34;dict/&#34;
      }
    ],
```

Do not change any of the following lines of code.

### The tool to load a local node addon
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

### Load nodejieba

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
You should expose the tool, through the global object like window, to the renderer process, which can then call methods such as window.myAddons.cutForSearch.

## Combine js-search and nodejieba

Assuming you want to search an object like this:

```typescript jsx
export interface Product {
  [key: string]: any;

  productCode: string;
  name: string;
  namePinyin: string;
  nameEnglish: string;
}
```

Write the code in your search component like this:

```typescript jsx
import * as JsSearch from &#39;js-search&#39;;
import { Search } from &#39;js-search&#39;;

const [search, setSearch] = React.useState&lt;string&gt;(&#34;&#34;);
const jsSearchGames = React.useRef&lt;Search&gt;();
const [omnisearch_games, setOmnisearchGames] = React.useState&lt;any[]&gt;([]);
const [omnisearch_loading, setOmnisearchLoading] = React.useState(false);

// ... 

// construct search component and data on load
useEffect(() =&gt; {
  const buildJsSearch = (uidField: string, documents: any[], ...index: string[]) =&gt; {
    const jsSearch = new JsSearch.Search(uidField);
    jsSearch.tokenizer = {
      tokenize: (text) =&gt; {
        const r = window.myAddons.cutForSearch(text, true); // cutForSearch is the method in the tool
        return r;
      },
    };
    index.forEach((i) =&gt; jsSearch.addIndex(i));
    jsSearch.addDocuments(documents);
    return jsSearch;
  };


  jsSearchGames.current = buildJsSearch(&#39;productCode&#39;, p, &#39;productCode&#39;, &#39;name&#39;, &#39;namePinyin&#39;, &#39;nameEnglish&#39;);
}, []);

// start to search if use type something in the search input
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
  // cancel last search if duration between this search and last search is less than 1 second
  // this is something you need to consider while user using chinese input method 
  if (currentSearchId.current) {
    clearTimeout(currentSearchId.current);
  }
  const doSearch = async () =&gt; new Promise&lt;searchResult&gt;((resolve, reject) =&gt; {
    // start search after 1 second
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
      // save those record that returns empty results to server so we can improve
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
        {(omnisearch_loading || null) &amp;&amp; &lt;div className=&#34;loading&#34;&gt;Loading&lt;/div&gt;}
        {((!omnisearch_loading &amp;&amp; omnisearch_result_count === 0) || null) &amp;&amp; (
          &lt;div className=&#34;no-results&#34;&gt;
            Not Found
          &lt;/div&gt;
        )}
        {(omnisearch_games.length || null) &amp;&amp; (
          &lt;div className=&#34;results&#34;&gt;
            &lt;h3&gt;Games&lt;/h3&gt;
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

## Done
Well, now you can achieve such a search effect as below.

![](/images/posts/20240729-search-demo.gif)

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/chinese-search/  

