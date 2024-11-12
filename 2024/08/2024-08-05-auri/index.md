# Tauri 使用 Vue-Devtools


Tauri 号称不强迫开发者（un-opinionated）、不依赖具体前端框架的（agnostic）桌面应用框架，Vue又以其简单易上手的特点在开发者中非常流行。

Vue Devtools 是一款可以在浏览器控制台中显示 vue 程序详情的小工具。

今天我们将它们结合起来，演示如何在一个新项目中配置 Vue Devtools.


## 前提

使用脚手架命令创建应用。
```bash
cargo install create-tauri-app
cargo create-tauri-app --rc
```
或
```bash
pnpm create tauri-app --rc
```

&gt; https://v2.tauri.app/start/create-project/

## 安装 Vue Devtools

windows
```powershell
$env:ELECTRON_CUSTOM_DIR=&#34;&#34;; npm install -g @vue/devtools
```

linux or macos
```bash
export ELECTRON_CUSTOM_DIR=&#34;&#34;
npm install -g @vue/devtools
```

## 使用 Vue Devtools

### 启动 Vue Devtools
```bash
vue-devtools
```

或将该命令加入到 package.json 中
```json
  &#34;scripts&#34;: {
    ...
    &#34;vue:dev-tools&#34;: &#34;vue-devtools&#34;,
    ...
  }
```

### 引入 Vue Devtools
在 index.html 中引入。

```html
&lt;!doctype html&gt;
&lt;html lang=&#34;en&#34;&gt;
  &lt;head&gt;
    &lt;meta charset=&#34;UTF-8&#34; /&gt;
    &lt;link rel=&#34;icon&#34; type=&#34;image/svg&#43;xml&#34; href=&#34;/vite.svg&#34; /&gt;
    &lt;meta name=&#34;viewport&#34; content=&#34;width=device-width, initial-scale=1.0&#34; /&gt;
    &lt;title&gt;Tauri &#43; Vue &#43; Typescript App&lt;/title&gt;
  &lt;/head&gt;

  &lt;body&gt;
    &lt;div id=&#34;app&#34;&gt;&lt;/div&gt;
    &lt;script type=&#34;module&#34; src=&#34;/src/main.ts&#34;&gt;&lt;/script&gt;
    &lt;script src=&#34;http://localhost:8098&#34;&gt;&lt;/script&gt; &lt;!-- 增加这一行 --&gt;
  &lt;/body&gt;
&lt;/html&gt;

```

## 查看
此时，就可以在 tauri 中使用 vue-devtools 了。

![check_vue_dev_tools.png](/images/posts/20240805-check_vue_dev_tools.png)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/08/2024-08-05-auri/  

