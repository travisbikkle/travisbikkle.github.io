# Diff Patch 應用


假設有三個文件，你需要將修復的bug的代碼，合入到file_1.1中，生成一個名爲file_1.1.fix的文件。

| 文件名         | 說明                            |
|----------------|-------------------------------|
| file_1.0.orgin | 舊版本的原始文件                |
| file_1.0.fix   | 舊版本的修正文件，修改了一些 bug |
| file_1.1       | 新版本的原始文件                |

## 參考以下命令

1. diff 用於對比生成補丁
   ```bash
   diff -urN old new &gt; patch
   diff -urN file_1.0.origin file_1.0.fix &gt; bug_patch
   ```

2. patch 用於將補丁應用到新文件
   ```bash
   patch new -i patch -o target
   patch file_1.1 -i bug_patch -o file_1.1.fix
   ```

&gt; diff path 是 linux 標準命令，windows 可以下載 git-bash 後使用。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2021/07/diff-patch/  

