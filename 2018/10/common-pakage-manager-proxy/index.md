# Npm 镜像等设置


## 中国加速镜像

1. npm可能未及时同步
2. pip可能未同步官方已经删除的恶意软件包

| manager | useful commands          | command                                                                                                                                                                                                                                  |
| ------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| pnpm    | pnpm get registry        | pnpm config set registry https://registry.npmmirror.com                                                                                                                                                                                  |
| npm     | npm get registry         | npm config set registry https://registry.npmmirror.com                                                                                                                                                                                   |
| yarn    | yarn config get registry | yarn config set registry https://registry.npmmirror.com                                                                                                                                                                                  |
| pip     | pip config list          | # for pip install&lt;br&gt;pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/&lt;br&gt;# for pip search&lt;br&gt;pip config set global.index https://mirrors.aliyun.com/pypi&lt;br&gt;pip config set global.trusted-host mirrors.aliyun.com |

## 代理

注意如果你没有翻墙软件就不用做下面的配置，请将地址修改为你实际的代理地址。

1. npm(~/.npmrc)

   ```INI
   proxy=http://localhost:8118
   https_proxy=https://localhost:8118
   strict-ssl=false
   ```

2. yarn(~/.yarnrc)
   ```txt
   env:
       proxy &#39;http://localhost:8118&#39;
       https_proxy &#39;https://localhost:8118&#39;
       strict-ssl false
   ```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/common-pakage-manager-proxy/  

