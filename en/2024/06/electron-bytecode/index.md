# Electron Source Code Protection


Here is a quick example to show how to enable byte code protection in your Electron project.

## Environment

| tech                               | version  |
|------------------------------------|----------|
| electron                           | 30.0.6   |
| webpack                            | 5.91.0   |
| @herberttn/bytenode-webpack-plugin | 2.3.1    |
| nodejs                             | v20.14.0 |

## Steps
### Webpack Configuration
[Reference](https://www.npmjs.com/package/@herberttn/bytenode-webpack-plugin)

```javascript
// import plugin
const { BytenodeWebpackPlugin } = require(&#39;@herberttn/bytenode-webpack-plugin&#39;);
// enable only in production
const isEnvProduction = process.env.NODE_ENV === &#39;production&#39;;
...
plugins: [
  isEnvProduction &amp;&amp; new BytenodeWebpackPlugin({ compileForElectron: true }),
],
...
// main，preload，renderer entry. I used webpack-merge, if you didn&#39;t, ignore it.
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

### Electron entry main.ts/main.js configuration
```javascript
  mainWindow = new BrowserWindow({
  ...
  webPreferences: {
    nodeIntegration: true, // set to true to enable jsc support
    contextIsolation: false, // set to false to enable jsc support, but it will disable contextBridge, I will show you how to fix it
    preload: path.join(__dirname, &#39;./preload.js&#39;), // bytenode compiled js file, used to load preload.compiled.jsc
    webSecurity: false,
    sandbox: false,
  },
});
```

I know the above configurations don&#39;t meet Electron&#39;s default security requirements, but if they are enabled, basically nothing can be done.

If you wish to use bytecode, you must configure it as above.

### preload contextBridge fix
`contextIsolation: false` causes contextBridge to be unusable, another one of Electron&#39;s interesting decisions.

contextBridge is unavailable, then you can&#39;t be using ipc, but you don&#39;t need it now.

At this point, you can call node&#39;s api directly in the renderer process, such as fs. 

However, I still suggest you to pass it through preload.

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

### Done
With the above three configurations in webpack.js, main.js, and preload.js, you should be able to use bytecode in your project.

Bytecode is still not the final solution because it is easily decompiled and at this stage it just adds some cost to the beginner cracker.

Some people have used the [solution](https://juejin.cn/post/6968291704071782430) of writing native plugins in rust and pairing it with bytecode for obfuscation, 

which you can refer to if you have a strong need to keep your code secret. You need to make a trade-off between maintainability and security.


---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/06/electron-bytecode/  

