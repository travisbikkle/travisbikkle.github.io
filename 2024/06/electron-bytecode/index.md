# Electron 源码保护


这是一个示例，演示如何快速在你的 Electron 项目中启用字节码保护，没有多余的废话。

## 环境

| tech                               | version  |
|------------------------------------|----------|
| electron                           | 30.0.6   |
| webpack                            | 5.91.0   |
| @herberttn/bytenode-webpack-plugin | 2.3.1    |
| nodejs                             | v20.14.0 |

## 步骤
### Webpack 配置
[参考文档](https://www.npmjs.com/package/@herberttn/bytenode-webpack-plugin)

```javascript
// 引入依赖
const { BytenodeWebpackPlugin } = require(&#39;@herberttn/bytenode-webpack-plugin&#39;);
// 在生产环境启用
const isEnvProduction = process.env.NODE_ENV === &#39;production&#39;;
...
plugins: [
  isEnvProduction &amp;&amp; new BytenodeWebpackPlugin({ compileForElectron: true }),
],
...
// main，preload，renderer 需要更改 entry 配置。我使用了 webpack-merge，如果你没有使用，忽略即可。
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
    nodeIntegration: true, // 启用 jsc 支持
    contextIsolation: false, // false 是启用 jsc 支持，但是用不了 preload 的 contextBridge
    preload: path.join(__dirname, &#39;./preload.js&#39;), // bytenode 编译后生成的 js，用于加载 preload.compiled.jsc
    webSecurity: false,
    sandbox: false,
  },
});
```

我知道上面的配置，不符合 Electron 的默认安全配置。但是 Electron 的默认安全配置开启后，你基本上什么也做不了。

如果你希望使用字节码，你必须按照以上配置。

### preload contextBridge 修复
`contextIsolation: false` 会导致 contextBridge 无法使用，这又是 Electron 的一个有趣决定之一。

contextBridge 无法使用，那么你就无法在使用 ipc，当然，你也不需要了。

此时你可以在 renderer 中直接操作 node 的 api，如 fs 等。但是我建议还是通过 preload 中转一下，凡事留一线，日后好相见。
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
通过 webpack.js，main.js，preload.js 中的以上三处配置，你应该可以在你的项目中使用字节码了。

字节码仍然不是最终方案，因为字节码很容易被反编译，目前阶段它只是增加了初级破解者的一些成本。

有些人使用了在 rust 中编写本地插件，和字节码搭配混淆的[方案](https://juejin.cn/post/6968291704071782430)，如果你对代码有强烈的保密需求，可以参考，并在可维护性和安全性上取舍。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/06/electron-bytecode/  

