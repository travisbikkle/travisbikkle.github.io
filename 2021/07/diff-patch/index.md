# Diff Patch 应用


假设有三个文件，你需要将修复的bug的代码，合入到file_1.1中，生成一个名为file_1.1.fix的文件。

| 文件名         | 说明                            |
|----------------|-------------------------------|
| file_1.0.orgin | 旧版本的原始文件                |
| file_1.0.fix   | 旧版本的修正文件，修改了一些 bug |
| file_1.1       | 新版本的原始文件                |

## 参考以下命令

1. diff 用于对比生成补丁
   ```bash
   diff -urN old new &gt; patch
   diff -urN file_1.0.origin file_1.0.fix &gt; bug_patch
   ```

2. patch 用于将补丁应用到新文件
   ```bash
   patch new -i patch -o target
   patch file_1.1 -i bug_patch -o file_1.1.fix
   ```

&gt; diff path 是 linux 标准命令，windows 可以下载 git-bash 后使用。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2021/07/diff-patch/  

