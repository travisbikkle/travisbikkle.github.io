# Use Vue-Devtools in Tauri


Tauri claims itself as an un-opinionated, agnostic desktop application framework that does not depend on a specific front-end framework.

Vue is very popular among developers for its simplicity and ease of use.

Vue Devtools is a small tool that displays the details of a vue program in the browser console.

Today we combine them and demonstrate how to configure Vue Devtools in a new tauri project.

## Prerequisites

Creating a tauri project, remember to choose vue as the front end framework.
。
```bash
cargo install create-tauri-app
cargo create-tauri-app --rc
```
or
```bash
pnpm create tauri-app --rc
```

&gt; https://v2.tauri.app/start/create-project/

## Install Vue Devtools

windows
```powershell
$env:ELECTRON_CUSTOM_DIR=&#34;&#34;; npm install -g @vue/devtools
```

linux or macos
```bash
export ELECTRON_CUSTOM_DIR=&#34;&#34;
npm install -g @vue/devtools
```

## Use Vue Devtools

### Start Vue Devtools
```bash
vue-devtools
```

or add it to package.json:
```json
  &#34;scripts&#34;: {
    ...
    &#34;vue:dev-tools&#34;: &#34;vue-devtools&#34;,
    ...
  }
```

### Introduce Vue Devtools in your index.html

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
    &lt;script src=&#34;http://localhost:8098&#34;&gt;&lt;/script&gt; &lt;!-- add this line --&gt;
  &lt;/body&gt;
&lt;/html&gt;

```

## Check
At this point, you might be able to use vue-devtools in tauri.

![check_vue_dev_tools.png](/images/posts/20240805-check_vue_dev_tools.png)



---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/08/2024-08-05-auri/  

