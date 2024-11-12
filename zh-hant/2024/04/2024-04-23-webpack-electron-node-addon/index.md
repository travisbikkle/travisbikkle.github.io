# 在 Webapck&#43;electron&#43;typescript中使用go開發的node插件



# node 插件，electron 和 webpack 那些事

## 首先要明確在哪裏引入 node 的插件， main，preload還是 renderer？

我們開發了一個 node 的插件，需要在 electron 中引入。我們一開始當然是希望在 renderer 中引入，畢竟最接近業務邏輯，省事。

不過會遇到報錯 &#39;require is not defined&#39;，也就是沒有 require 函數。

這個時候網上可能有些回答會讓你在 main.ts 中打開 nodeIntegration：

```typescript
mainWindow = new BrowserWindow({
  height: 800,
  width: 1280,
  maxHeight: 2160,
  webPreferences: {
    nodeIntegration: true,
    devTools: nodeEnv.dev,
    preload: path.join(__dirname, &#39;./preload.bundle.js&#39;),
  },
});
```

實際上這是不推薦的，爲什麼要在 renderer 中允許執行本地的命令，如 fs 等等？如果是一個惡意的網站，他就能訪問你本機所有的文件。當然如果你確保自己的應用不訪問外部網站，也可以。

我們可以瞭解下比較安全的做法。

&gt; 爲了解決這個問題，我花了整整一天的時間，我這個項目的技術棧是 TypeScript, Webpack 5, 並且需要引入一個 go 寫的 node 插件，現代 javascript 的buff 疊滿了屬於是，我這個後端開發感受到了前端滿滿的惡意了。 
&gt; 開發插件並在 node 中跑通不到兩小時，可是把這個插件放到 webpack &#43; electron 中花了我整整 7 個小時。

閒話少說，方法有兩種：
1. 在 main.ts 加載插件然後和 renderer 使用 ipc 通信（麻煩）
   這樣的話需要寫大量這樣的 ipc 接口，這當然不是我們想要的。
   ```typescript
   // preload
   import { ipcRenderer } from &#39;electron&#39;;
   
   function showFolderPicker() {
       return ipcRenderer.invoke(&#39;dialog:openDirectory&#39;);
   }
   
   export default { showFolderPicker };
   // main
   ipcMain.handle(&#39;dialog:openDirectory&#39;, async () =&gt; {
       const { canceled, filePaths } = await dialog.showOpenDialog(mainWindow!, {
           properties: [&#39;openDirectory&#39;],
       });
       if (canceled) {
           return &#34;&#34;;
       }
       return filePaths[0];
   });
   ```
2. 現在（electron 30.0） preload 仍然保留有訪問 node api 的權限，將插件在 preload 暴露給 renderer。
   ```typescript
   // preload
   import { contextBridge } from &#39;electron&#39;;
   import ipcAPI from &#39;_preload/ipc-api&#39;;
   import myplugin from &#39;myplugin&#39;; // node 插件

   contextBridge.exposeInMainWorld(&#39;ipcAPI&#39;, ipcAPI);
   contextBridge.exposeInMainWorld(&#39;myplugin&#39;, myplugin);
   
   // global.d.ts 別忘了定義一個全局的類型文件
   declare global {
       interface Window {
           /** APIs for Electron IPC */
           ipcAPI?: typeof import(&#39;_preload/ipc-api&#39;).default;
           myplugin?: typeof import(&#39;myplugin&#39;);
       }
   }
   // Makes TS sees this as an external modules so we can extend the global scope.
   export { };

   // renderer 中就可以調用了
   windows.myplugin.hello()
   ```
   在生產環境的配置方法見下文。


### 爲什麼會有這個問題

這要看 electron 在安全方面做了哪些變動。

1. Electron 1 nodeIntegration 默認是 true
   Renderer 可以訪問全部 node 接口。

2. Electron 5 nodeIntegration 默認是 false
   此時可用 preload 來暴露接口，無論 nodeIntegration 怎麼設置，preload 都是能訪問 node 接口的。 
   ```javascript
   //preload.js
   window.api = {
       deleteFile: f =&gt; require(&#39;fs&#39;).unlink(f)
   }
   ```
   
3. Electron 5 contextIsolation 默認是 true
   這會導致 preload 在一個隔離的環境中運行，這樣你就沒法 windows.api = xxx 了，你需要 exposeInMainWorld
   ```
   //preload.js
   const { contextBridge } = require(&#39;electron&#39;)
   contextBridge.exposeInMainWorld(&#39;api&#39;, {
       deleteFile: f =&gt; require(&#39;fs&#39;).unlink(f)
   })
   ```
4. Electron 6 如果你在 mainWindow 設置了 sandbox: true,
   ```typescript
   mainWindow = new BrowserWindow({
       height: 800,
       width: 1280,
       maxHeight: 2160,
       webPreferences: {
           devTools: nodeEnv.dev,
           preload: path.join(__dirname, &#39;./preload.bundle.js&#39;),
           sandbox: true, // 就是這
   ```
   那你的 preload 得這麼寫：
   ```typescript
   //preload.js
   const { contextBridge, remote } = require(&#39;electron&#39;)
   
   contextBridge.exposeInMainWorld(&#39;api&#39;, {
      deleteFile: f =&gt; remote.require(&#39;fs&#39;).unlink(f)
   })
   ```
5. Electron 10 enableRemoteModule 默認是 false (remote module 在 Electron 12 中就廢棄了)

   remote 模塊大家都很熟悉了，如果你需要訪問 main 進程的 api，你就得用它。沒有它你就要寫大量的 ipc，就像上面說的方法1.


### 推薦做法
設置
```javascript
{nodeIntegration: false, contextIsolation: true, enableRemoteModule: false}
```
如果覺得不夠安全，就開 sandbox，這樣你就可以愉快的寫大量的 ipc 代碼了。

sandbox 關閉的時候，preload 可以直接訪問 node api，比如 `require(&#39;fs&#39;).readFile`，只要你別這麼玩，你就是安全的：
```typescript
//bad
contextBridge.exposeInMainWorld(&#39;api&#39;, {
    readFile: require(&#39;fs&#39;).readFile
})
```

## 具體代碼
具體怎麼在 webpack 和 electron 裏面跑起來，我相信很多人都會。但是奈何我就是找不到一篇能落地的文章。我這裏拋磚引玉，希望大家都說說自己怎麼實現的，也希望能節省未來某一個少年的時間吧。

1. 將 node-gyp 生成的包，直接本地 npm install
   假設你的工程目錄如下，插件在 `src/plugin/build` 下面。
   ```
   publid
   package.json
   src
     |-plugin
       |- build // 這一層有 package.json 的就是你的插件的包描述文件
          |- build
            |- Release
              |- myplugin.node
          |- package.json
          |- index.js
          |- index.d.ts
          |- myplugin.dll
   ```
   執行 `npm i src/plugin/build`

2. 在開發態，你的代碼編譯應該就不飄紅了。打包運行的時候，因爲有 bindings.js，它會去以下位置找你的 myplugin.node 文件。
   ```javascript
   // node-gyp&#39;s linked version in the &#34;build&#34; dir
   [&#39;module_root&#39;, &#39;build&#39;, &#39;bindings&#39;],
   // node-waf and gyp_addon (a.k.a node-gyp)
   [&#39;module_root&#39;, &#39;build&#39;, &#39;Debug&#39;, &#39;bindings&#39;],
   [&#39;module_root&#39;, &#39;build&#39;, &#39;Release&#39;, &#39;bindings&#39;],
   // Debug files, for development (legacy behavior, remove for node v0.9)
   [&#39;module_root&#39;, &#39;out&#39;, &#39;Debug&#39;, &#39;bindings&#39;],
   [&#39;module_root&#39;, &#39;Debug&#39;, &#39;bindings&#39;],
   // Release files, but manually compiled (legacy behavior, remove for node v0.9)
   [&#39;module_root&#39;, &#39;out&#39;, &#39;Release&#39;, &#39;bindings&#39;],
   [&#39;module_root&#39;, &#39;Release&#39;, &#39;bindings&#39;],
   // Legacy from node-waf, node &lt;= 0.4.x
   [&#39;module_root&#39;, &#39;build&#39;, &#39;default&#39;, &#39;bindings&#39;],
   // Production &#34;Release&#34; buildtype binary (meh...)
   [&#39;module_root&#39;, &#39;compiled&#39;, &#39;version&#39;, &#39;platform&#39;, &#39;arch&#39;, &#39;bindings&#39;],
   // node-qbs builds
   [&#39;module_root&#39;, &#39;addon-build&#39;, &#39;release&#39;, &#39;install-root&#39;, &#39;bindings&#39;],
   [&#39;module_root&#39;, &#39;addon-build&#39;, &#39;debug&#39;, &#39;install-root&#39;, &#39;bindings&#39;],
   [&#39;module_root&#39;, &#39;addon-build&#39;, &#39;default&#39;, &#39;install-root&#39;, &#39;bindings&#39;],
   // node-pre-gyp path ./lib/binding/{node_abi}-{platform}-{arch}
   [&#39;module_root&#39;, &#39;lib&#39;, &#39;binding&#39;, &#39;nodePreGyp&#39;, &#39;bindings&#39;]
   ...
   function bindings(opts) {
      // Argument surgery
      if (typeof opts == &#39;string&#39;) {
        opts = { bindings: opts };
      } else if (!opts) {
        opts = {};
      }
      if (!opts.module_root) {
         opts.module_root = exports.getRoot(exports.getFileName());
      } 
       ...
        opts.try[i].map(function(p) {
             return opts[p] || p; // 如果 bindings 傳入了一個對象 {}，且對象中有 module_root，就用對象中的 module_root 對應的值。否則直接用 try 裏面的字符串
           })
   ```
   bindings.js 詳解：
   你可以自己在瀏覽器中測試一下 bindings.js 的邏輯，本文不再貼代碼，直接說結論。

   下面是一段 preload.ts 的代碼，我寫了兩種加載插件的方法，請看註釋：

   ```typescript
   function getPlugin() {
     // 這是第一種加載方法，一切都是默認，只傳一個 myplugin 名稱
     const nodeAddon = bindings(&#34;myplugin&#34;); // 這裏在 bindings.js 中的 getRoot 和 getFileName 中會做一些運算，根據誰引入的 bindings.js 來計算 module_root，也就是去哪個文件夾中去找。這一段邏輯很繁瑣和無趣，可以自行了解一下
     logger.log(&#34;preload.ts1&#34; &#43; JSON.stringify(nodeAddon));

     // 這是第二種加載方法
     let tries = [[&#34;module_root&#34;, &#34;bindings&#34;]]; // 含義：生產環境去加載 process.cwd()/myplugin.node（module_root被替換成了process.cwd(), bindings 被替換成了 myplugin.node）
     if (dev) { // dev = process.env.NODE_ENV === &#39;development&#39;
       tries = [[&#34;module_root&#34;, &#34;build&#34;, &#34;bindings&#34;]]; // 含義：開發環境去加載 process.cwd()/build/myplugin.node（build沒有被替換，看下面 bindings 函數的參數，沒有傳 build）
     }
     const nodeAddon2 = bindings({
       bindings: &#34;myplugin&#34;,
       module_root: process.cwd(), // 含義：binding.js 中將 module_root 替換成 process.cwd()
       try: tries,
     });
   
     logger.log(&#34;preload.ts2&#34; &#43; JSON.stringify(nodeAddon2));
     return nodeAddon2;
   }
   
   contextBridge.exposeInMainWorld(&#39;myplugin&#39;, getPlugin());
   ```


3. 根據以上結論，在 webpack 裏面這麼設置，將 node 文件和 dll 文件放到工程的根目錄下的 build 目錄。
   ```javascript
   const mainConfig = merge(commonConfig, {
      entry: &#39;./src/main/main.ts&#39;,
      target: &#39;electron-main&#39;,
      output: { filename: &#39;main.bundle.js&#39; },
      plugins: [
         new CopyPlugin({
            patterns: [
               {
                // 省略
               },
               {
                  from: &#39;node_modules/myplugin/build/Release/myplugin.node&#39;,
                  to: &#39;../build/&#39;,
               },
               {
                  from: &#39;node_modules/myplugin/myplugin.dll&#39;,
                  to: &#39;../build/&#39;,
               },
            ],
         }),
      ],
   });
   ```
   而在打包後，比如用 Electron Builder，可以這麼配置，直接去安裝目錄找：
   ```json
   &#34;build&#34;: {
    &#34;appId&#34;: &#34;&#34;,
    &#34;productName&#34;: &#34;&#34;,
    ...
    &#34;extraFiles&#34;: [
      {
        &#34;from&#34;: &#34;build/&#34;,
        &#34;to&#34;: &#34;&#34;
      },
    ],
   ```


我嘗試過 webpack 設置 externals 或者 node-loader，都沒跑通。你有什麼好的方法，歡迎分享，另外說句感想，最近一個月接觸的前端，但是感覺前端真的亂。有想知道怎麼用 go 寫 node 插件的，也可以留言，我單獨寫一篇。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/04/2024-04-23-webpack-electron-node-addon/  

