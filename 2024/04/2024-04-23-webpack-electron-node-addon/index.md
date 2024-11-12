# 在 Webapck&#43;electron&#43;typescript中使用go开发的node插件



# node 插件，electron 和 webpack 那些事

## 首先要明确在哪里引入 node 的插件， main，preload还是 renderer？

我们开发了一个 node 的插件，需要在 electron 中引入。我们一开始当然是希望在 renderer 中引入，毕竟最接近业务逻辑，省事。

不过会遇到报错 &#39;require is not defined&#39;，也就是没有 require 函数。

这个时候网上可能有些回答会让你在 main.ts 中打开 nodeIntegration：

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

实际上这是不推荐的，为什么要在 renderer 中允许执行本地的命令，如 fs 等等？如果是一个恶意的网站，他就能访问你本机所有的文件。当然如果你确保自己的应用不访问外部网站，也可以。

我们可以了解下比较安全的做法。

&gt; 为了解决这个问题，我花了整整一天的时间，我这个项目的技术栈是 TypeScript, Webpack 5, 并且需要引入一个 go 写的 node 插件，现代 javascript 的buff 叠满了属于是，我这个后端开发感受到了前端满满的恶意了。 
&gt; 开发插件并在 node 中跑通不到两小时，可是把这个插件放到 webpack &#43; electron 中花了我整整 7 个小时。

闲话少说，方法有两种：
1. 在 main.ts 加载插件然后和 renderer 使用 ipc 通信（麻烦）
   这样的话需要写大量这样的 ipc 接口，这当然不是我们想要的。
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
2. 现在（electron 30.0） preload 仍然保留有访问 node api 的权限，将插件在 preload 暴露给 renderer。
   ```typescript
   // preload
   import { contextBridge } from &#39;electron&#39;;
   import ipcAPI from &#39;_preload/ipc-api&#39;;
   import myplugin from &#39;myplugin&#39;; // node 插件

   contextBridge.exposeInMainWorld(&#39;ipcAPI&#39;, ipcAPI);
   contextBridge.exposeInMainWorld(&#39;myplugin&#39;, myplugin);
   
   // global.d.ts 别忘了定义一个全局的类型文件
   declare global {
       interface Window {
           /** APIs for Electron IPC */
           ipcAPI?: typeof import(&#39;_preload/ipc-api&#39;).default;
           myplugin?: typeof import(&#39;myplugin&#39;);
       }
   }
   // Makes TS sees this as an external modules so we can extend the global scope.
   export { };

   // renderer 中就可以调用了
   windows.myplugin.hello()
   ```
   在生产环境的配置方法见下文。


### 为什么会有这个问题

这要看 electron 在安全方面做了哪些变动。

1. Electron 1 nodeIntegration 默认是 true
   Renderer 可以访问全部 node 接口。

2. Electron 5 nodeIntegration 默认是 false
   此时可用 preload 来暴露接口，无论 nodeIntegration 怎么设置，preload 都是能访问 node 接口的。 
   ```javascript
   //preload.js
   window.api = {
       deleteFile: f =&gt; require(&#39;fs&#39;).unlink(f)
   }
   ```
   
3. Electron 5 contextIsolation 默认是 true
   这会导致 preload 在一个隔离的环境中运行，这样你就没法 windows.api = xxx 了，你需要 exposeInMainWorld
   ```
   //preload.js
   const { contextBridge } = require(&#39;electron&#39;)
   contextBridge.exposeInMainWorld(&#39;api&#39;, {
       deleteFile: f =&gt; require(&#39;fs&#39;).unlink(f)
   })
   ```
4. Electron 6 如果你在 mainWindow 设置了 sandbox: true,
   ```typescript
   mainWindow = new BrowserWindow({
       height: 800,
       width: 1280,
       maxHeight: 2160,
       webPreferences: {
           devTools: nodeEnv.dev,
           preload: path.join(__dirname, &#39;./preload.bundle.js&#39;),
           sandbox: true, // 就是这
   ```
   那你的 preload 得这么写：
   ```typescript
   //preload.js
   const { contextBridge, remote } = require(&#39;electron&#39;)
   
   contextBridge.exposeInMainWorld(&#39;api&#39;, {
      deleteFile: f =&gt; remote.require(&#39;fs&#39;).unlink(f)
   })
   ```
5. Electron 10 enableRemoteModule 默认是 false (remote module 在 Electron 12 中就废弃了)

   remote 模块大家都很熟悉了，如果你需要访问 main 进程的 api，你就得用它。没有它你就要写大量的 ipc，就像上面说的方法1.


### 推荐做法
设置
```javascript
{nodeIntegration: false, contextIsolation: true, enableRemoteModule: false}
```
如果觉得不够安全，就开 sandbox，这样你就可以愉快的写大量的 ipc 代码了。

sandbox 关闭的时候，preload 可以直接访问 node api，比如 `require(&#39;fs&#39;).readFile`，只要你别这么玩，你就是安全的：
```typescript
//bad
contextBridge.exposeInMainWorld(&#39;api&#39;, {
    readFile: require(&#39;fs&#39;).readFile
})
```

## 具体代码
具体怎么在 webpack 和 electron 里面跑起来，我相信很多人都会。但是奈何我就是找不到一篇能落地的文章。我这里抛砖引玉，希望大家都说说自己怎么实现的，也希望能节省未来某一个少年的时间吧。

1. 将 node-gyp 生成的包，直接本地 npm install
   假设你的工程目录如下，插件在 `src/plugin/build` 下面。
   ```
   publid
   package.json
   src
     |-plugin
       |- build // 这一层有 package.json 的就是你的插件的包描述文件
          |- build
            |- Release
              |- myplugin.node
          |- package.json
          |- index.js
          |- index.d.ts
          |- myplugin.dll
   ```
   执行 `npm i src/plugin/build`

2. 在开发态，你的代码编译应该就不飘红了。打包运行的时候，因为有 bindings.js，它会去以下位置找你的 myplugin.node 文件。
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
             return opts[p] || p; // 如果 bindings 传入了一个对象 {}，且对象中有 module_root，就用对象中的 module_root 对应的值。否则直接用 try 里面的字符串
           })
   ```
   bindings.js 详解：
   你可以自己在浏览器中测试一下 bindings.js 的逻辑，本文不再贴代码，直接说结论。

   下面是一段 preload.ts 的代码，我写了两种加载插件的方法，请看注释：

   ```typescript
   function getPlugin() {
     // 这是第一种加载方法，一切都是默认，只传一个 myplugin 名称
     const nodeAddon = bindings(&#34;myplugin&#34;); // 这里在 bindings.js 中的 getRoot 和 getFileName 中会做一些运算，根据谁引入的 bindings.js 来计算 module_root，也就是去哪个文件夹中去找。这一段逻辑很繁琐和无趣，可以自行了解一下
     logger.log(&#34;preload.ts1&#34; &#43; JSON.stringify(nodeAddon));

     // 这是第二种加载方法
     let tries = [[&#34;module_root&#34;, &#34;bindings&#34;]]; // 含义：生产环境去加载 process.cwd()/myplugin.node（module_root被替换成了process.cwd(), bindings 被替换成了 myplugin.node）
     if (dev) { // dev = process.env.NODE_ENV === &#39;development&#39;
       tries = [[&#34;module_root&#34;, &#34;build&#34;, &#34;bindings&#34;]]; // 含义：开发环境去加载 process.cwd()/build/myplugin.node（build没有被替换，看下面 bindings 函数的参数，没有传 build）
     }
     const nodeAddon2 = bindings({
       bindings: &#34;myplugin&#34;,
       module_root: process.cwd(), // 含义：binding.js 中将 module_root 替换成 process.cwd()
       try: tries,
     });
   
     logger.log(&#34;preload.ts2&#34; &#43; JSON.stringify(nodeAddon2));
     return nodeAddon2;
   }
   
   contextBridge.exposeInMainWorld(&#39;myplugin&#39;, getPlugin());
   ```


3. 根据以上结论，在 webpack 里面这么设置，将 node 文件和 dll 文件放到工程的根目录下的 build 目录。
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
   而在打包后，比如用 Electron Builder，可以这么配置，直接去安装目录找：
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


我尝试过 webpack 设置 externals 或者 node-loader，都没跑通。你有什么好的方法，欢迎分享，另外说句感想，最近一个月接触的前端，但是感觉前端真的乱。有想知道怎么用 go 写 node 插件的，也可以留言，我单独写一篇。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/04/2024-04-23-webpack-electron-node-addon/  

