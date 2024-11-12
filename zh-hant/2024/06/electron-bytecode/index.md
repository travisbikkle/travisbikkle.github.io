# Electron 源碼保護


這是一個示例，演示如何快速在你的 Electron 項目中啓用字節碼保護，沒有多餘的廢話。

## 環境

| tech                               | version  |
|------------------------------------|----------|
| electron                           | 30.0.6   |
| webpack                            | 5.91.0   |
| @herberttn/bytenode-webpack-plugin | 2.3.1    |
| nodejs                             | v20.14.0 |

## 步驟
### Webpack 配置
[參考文檔](https://www.npmjs.com/package/@herberttn/bytenode-webpack-plugin)

```javascript
// 引入依賴
const { BytenodeWebpackPlugin } = require(&#39;@herberttn/bytenode-webpack-plugin&#39;);
// 在生產環境啓用
const isEnvProduction = process.env.NODE_ENV === &#39;production&#39;;
...
plugins: [
  isEnvProduction &amp;&amp; new BytenodeWebpackPlugin({ compileForElectron: true }),
],
...
// main，preload，renderer 需要更改 entry 配置。我使用了 webpack-merge，如果你沒有使用，忽略即可。
// main
const mainConfig = merge(commonConfig, {
  // entry: &#39;./src/main/main.ts&#39;,
  entry: {
    main: &#39;./src/main/main.ts&#39;,
  },
  target: &#39;electron-main&#39;,
  output: {
    filename: &#39;[name].js&#39;,
    devtoolModuleFilenameTemplate: &#39;[absolute-resource-path]&#39;,
  },
  ...
})
// preload
const preloadConfig = merge(commonConfig, {
    // entry: &#39;./src/preload/preload.ts&#39;,
    entry: {
      preload: &#39;./src/preload/preload.ts&#39;,
    },
    target: &#39;electron-preload&#39;,
    output: {
      filename: &#39;[name].js&#39;,
      devtoolModuleFilenameTemplate: &#39;[absolute-resource-path]&#39;,
    },
});
// renderer
const rendererConfig = merge(commonConfig, {
  entry: {
    renderer: &#39;./src/renderer/renderer.tsx&#39;,
  },
  target: &#39;electron-renderer&#39;,
  output: { devtoolModuleFilenameTemplate: &#39;[absolute-resource-path]&#39; },
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, &#39;./public/index.html&#39;),
    }),
  ],
});
```

### Electron 入口 main.ts/main.js 配置
```javascript
  mainWindow = new BrowserWindow({
  ...
  webPreferences: {
    nodeIntegration: true, // 啓用 jsc 支持
    contextIsolation: false, // false 是啓用 jsc 支持，但是用不了 preload 的 contextBridge
    preload: path.join(__dirname, &#39;./preload.js&#39;), // bytenode 編譯後生成的 js，用於加載 preload.compiled.jsc
    webSecurity: false,
    sandbox: false,
  },
});
```

我知道上面的配置，不符合 Electron 的默認安全配置。但是 Electron 的默認安全配置開啓後，你基本上什麼也做不了。

如果你希望使用字節碼，你必須按照以上配置。

### preload contextBridge 修復
`contextIsolation: false` 會導致 contextBridge 無法使用，這又是 Electron 的一個有趣決定之一。

contextBridge 無法使用，那麼你就無法在使用 ipc，當然，你也不需要了。

此時你可以在 renderer 中直接操作 node 的 api，如 fs 等。但是我建議還是通過 preload 中轉一下，凡事留一線，日後好相見。
```text
- import { contextBridge } from &#39;electron&#39;;
- import ipcAPI from &#39;_preload/ipc-api&#39;;
import loadAddon from &#34;_preload/load_node_addon&#34;;
import jb from &#34;_preload/nodejieba&#34;;

- contextBridge.exposeInMainWorld(&#39;ipcAPI&#39;, ipcAPI);
- contextBridge.exposeInMainWorld(&#39;myplugin&#39;, loadAddon(&#39;myplugin&#39;));
- contextBridge.exposeInMainWorld(&#39;jb&#39;, jb);

&#43; window.ipcAPI = ipcAPI;
&#43; window.myplugin = loadAddon(&#39;myplugin&#39;);
&#43; window.jb = jb;
```

### 完成
通過 webpack.js，main.js，preload.js 中的以上三處配置，你應該可以在你的項目中使用字節碼了。

字節碼仍然不是最終方案，因爲字節碼很容易被反編譯，目前階段它只是增加了初級破解者的一些成本。

有些人使用了在 rust 中編寫本地插件，和字節碼搭配混淆的[方案](https://juejin.cn/post/6968291704071782430)，如果你對代碼有強烈的保密需求，可以參考，並在可維護性和安全性上取捨。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/06/electron-bytecode/  

