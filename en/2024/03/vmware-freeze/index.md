# Vmware Freeze


## Problem

| Host OS  | Win11|
| -- | -- |
| Guest OS | Kubuntu 22.04   |
|vmware| vmware workstation pro 17.5 |

&lt;br /&gt;

Every once in a while, my guest os went into a strange freezed state, where I could move the mouse, but the guest os just didn&#39;t response.
I used to monitor those logs under /var/log, and system resource usage with `top` command, but nothing was found, until I saw the following line in vmware.log.

```txt
2024-03-06T16:33:01.365Z In(05) mks VMMouse: Dropping move received while input queue was full
```
This line was the last in the file, so it&#39;s easy to find. Input was dropped, this explains everything.

## Solve
I searched this error and found a post which was written many years ago in vmware community.
Here is the solution:

{{&lt; notice tip &gt;}}
1. config.ini&#39;s location may vary according to the os, please google it yourself
2. config.ini may could not be saved due to permission issue, please save it to desktop and drag it to destination。
3. vmx file is very important and you should backup it before making any changes to it。
{{&lt; /notice &gt;}}

1. Add following two lines to C:\ProgramData\VMware\VMware Workstation\config.ini
   ```ini
   prefvmx.useRecommendedLockedMemSize = &#34;TRUE&#34;
   prefvmx.minVmMemPct = &#34;100&#34;
   ```
  If you&#39;re experiencing some keyboard issues, like typing &#39;characterrrrrrrrr&#39;, you might also need following two lines(not tested):
   ```ini
   mks.disableTypematic = &#34;TRUE&#34;
   mks.disableRemoteClientTypematic = &#34;TRUE&#34;
   ```
2. Add following two lines to xxxx.vmx（located in the guest os root directory，please replace xxxx.vmx with the right name）
   ```ini
   sched.mem.pshare.enable = &#34;FALSE&#34;
   mainMem.useNamedFile = &#34;FALSE&#34;
   MemTrimRate = &#34;0&#34;
   MemAllowAutoScaleDown = &#34;FALSE&#34;
   ```
3. If the problem still exists, you might need to downgrade to 17.0.2 version of vmware workstation pro, and this line in vmx needs to be changed:
   ```ini
   virtualHW.version = &#34;20&#34;
   ```

I have no idea what those configurations are for but the problem never arises again. Amazing, right?

&gt; https://communities.vmware.com/t5/VMware-Workstation-Pro/Strange-VM-WS-6-behavior-Random-intermittent-Freeze-of-VM-and/m-p/1977361/highlight/true#M115876

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/03/vmware-freeze/  

