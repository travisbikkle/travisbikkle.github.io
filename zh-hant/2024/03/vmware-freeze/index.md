# Vmware 死機但是鼠標可以動


## 問題現象

| 宿主機  | Win11|
| -- | -- |
| 客戶機 | Kubuntu 22.04   |
|vmware| vmware workstation pro 17.5 |

&lt;br /&gt;

在使用 VSCode 的時候，我的客戶機總是沒有規律的死機，這個時候鼠標可以移動，但是無法點擊。能看出來客戶機並沒有真正的死機。我曾經監控過 /var/log 下面的日誌，以及用 top 監控過系統資源情況，但是沒能發現任何異常。後來在宿主機的虛擬機目錄 vmware.log 中，看到如下一行日誌，才明白髮生了什麼：

```txt
2024-03-06T16:33:01.365Z In(05) mks VMMouse: Dropping move received while input queue was full
```
這是最後一行日誌，非常好發現。這說明客戶機不再接收任何輸入，難怪表現爲好像死機了！

## 解決方案
我搜索了這個報錯，在 vmware 社區很多年前的一篇文章中找到了如下方法。

{{&lt; notice tip &gt;}}
1. config.ini 的位置可能因操作系統不同而不同，請自行搜索位置。
2. config.ini 修改後保存時可能沒有權限，可以先另存到桌面，然後拖動到原位置。
3. vmx文件在修改後可能發生錯誤，請先備份該文件。
{{&lt; /notice &gt;}}

1. 在 C:\ProgramData\VMware\VMware Workstation\config.ini 添加以下兩行
   ```ini
   prefvmx.useRecommendedLockedMemSize = &#34;TRUE&#34;
   prefvmx.minVmMemPct = &#34;100&#34;
   ```
   一些遇到鼠標或者鍵盤一直輸入問題，例如 characterrrrrrrrr，可能還需要添加如下兩行（未經驗證）
   ```ini
   mks.disableTypematic = &#34;TRUE&#34;
   mks.disableRemoteClientTypematic = &#34;TRUE&#34;
   ```
2. 修改 xxxx.vmx（該文件存在於你的客戶機根目錄，請將 xxxx.vmx 替換爲實際名稱）
   ```ini
   sched.mem.pshare.enable = &#34;FALSE&#34;
   mainMem.useNamedFile = &#34;FALSE&#34;
   MemTrimRate = &#34;0&#34;
   MemAllowAutoScaleDown = &#34;FALSE&#34;
   ```
3. 如果問題仍然沒有解決，可以降級到 17.0.2 的版本，你的虛擬機 vmx 中的如下一行應該修改爲：
   ```ini
   virtualHW.version = &#34;20&#34;
   ```

我不清楚這些配置到底有什麼用，但是後來再也沒有出現過這種情況。神奇的 vmware。

&gt; https://communities.vmware.com/t5/VMware-Workstation-Pro/Strange-VM-WS-6-behavior-Random-intermittent-Freeze-of-VM-and/m-p/1977361/highlight/true#M115876

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/03/vmware-freeze/  

