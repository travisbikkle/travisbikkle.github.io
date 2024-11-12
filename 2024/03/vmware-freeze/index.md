# Vmware 死机但是鼠标可以动


## 问题现象

| 宿主机  | Win11|
| -- | -- |
| 客户机 | Kubuntu 22.04   |
|vmware| vmware workstation pro 17.5 |

&lt;br /&gt;

在使用 VSCode 的时候，我的客户机总是没有规律的死机，这个时候鼠标可以移动，但是无法点击。能看出来客户机并没有真正的死机。我曾经监控过 /var/log 下面的日志，以及用 top 监控过系统资源情况，但是没能发现任何异常。后来在宿主机的虚拟机目录 vmware.log 中，看到如下一行日志，才明白发生了什么：

```txt
2024-03-06T16:33:01.365Z In(05) mks VMMouse: Dropping move received while input queue was full
```
这是最后一行日志，非常好发现。这说明客户机不再接收任何输入，难怪表现为好像死机了！

## 解决方案
我搜索了这个报错，在 vmware 社区很多年前的一篇文章中找到了如下方法。

{{&lt; notice tip &gt;}}
1. config.ini 的位置可能因操作系统不同而不同，请自行搜索位置。
2. config.ini 修改后保存时可能没有权限，可以先另存到桌面，然后拖动到原位置。
3. vmx文件在修改后可能发生错误，请先备份该文件。
{{&lt; /notice &gt;}}

1. 在 C:\ProgramData\VMware\VMware Workstation\config.ini 添加以下两行
   ```ini
   prefvmx.useRecommendedLockedMemSize = &#34;TRUE&#34;
   prefvmx.minVmMemPct = &#34;100&#34;
   ```
   一些遇到鼠标或者键盘一直输入问题，例如 characterrrrrrrrr，可能还需要添加如下两行（未经验证）
   ```ini
   mks.disableTypematic = &#34;TRUE&#34;
   mks.disableRemoteClientTypematic = &#34;TRUE&#34;
   ```
2. 修改 xxxx.vmx（该文件存在于你的客户机根目录，请将 xxxx.vmx 替换为实际名称）
   ```ini
   sched.mem.pshare.enable = &#34;FALSE&#34;
   mainMem.useNamedFile = &#34;FALSE&#34;
   MemTrimRate = &#34;0&#34;
   MemAllowAutoScaleDown = &#34;FALSE&#34;
   ```
3. 如果问题仍然没有解决，可以降级到 17.0.2 的版本，你的虚拟机 vmx 中的如下一行应该修改为：
   ```ini
   virtualHW.version = &#34;20&#34;
   ```

我不清楚这些配置到底有什么用，但是后来再也没有出现过这种情况。神奇的 vmware。

&gt; https://communities.vmware.com/t5/VMware-Workstation-Pro/Strange-VM-WS-6-behavior-Random-intermittent-Freeze-of-VM-and/m-p/1977361/highlight/true#M115876

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/03/vmware-freeze/  

