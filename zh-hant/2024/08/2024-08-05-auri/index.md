# Tauri 使用 Vue-Devtools


Tauri 號稱不強迫開發者（un-opinionated）、不依賴具體前端框架的（agnostic）桌面應用框架，Vue又以其簡單易上手的特點在開發者中非常流行。

Vue Devtools 是一款可以在瀏覽器控制檯中顯示 vue 程序詳情的小工具。

今天我們將它們結合起來，演示如何在一個新項目中配置 Vue Devtools.


## 前提

使用腳手架命令創建應用。
```bash
cargo install create-tauri-app
cargo create-tauri-app --rc
```
或
```bash
pnpm create tauri-app --rc
```

&gt; https://v2.tauri.app/start/create-project/

## 安裝 Vue Devtools

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

### 啓動 Vue Devtools
```bash
vue-devtools
```

或將該命令加入到 package.json 中
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
    &lt;script src=&#34;http://localhost:8098&#34;&gt;&lt;/script&gt; &lt;!-- 增加這一行 --&gt;
  &lt;/body&gt;
&lt;/html&gt;

```

## 查看
此時，就可以在 tauri 中使用 vue-devtools 了。

![check_vue_dev_tools.png](/images/posts/20240805-check_vue_dev_tools.png)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/08/2024-08-05-auri/  

