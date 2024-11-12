# Windows Vmware Amd-v 开启


#### 报错现象
我在 Win 11 中安装了 Vmware，并在Vmware中安装了 Kubuntu 22.04 虚拟机。
当我在虚拟机设置的cpu设置中开启AMD-V的时候，Vmware提示如下错误：
![2023-11-14-exec-gpedit.png](/images/other/20231114_vmware_amdv_error.png)

起初我并没有在意，只是感觉虚拟机的性能有些差，直到我在虚拟机中尝试使用Android Studio附带的安卓模拟器时，遇到了问题。

#### 无法开启 AMD-V 或 Intel VT-x 的影响
1. 性能下降  
   虚拟机的性能将受到限制，因为它不能充分利用硬件虚拟化扩展，指令执行和内存管理开销可能增加。
   这个性能下降是肉眼可见的，你在点击、挪动鼠标的时候会感觉到有一种难以察觉的迟滞感，非常难受。

2. 兼容性问题  
   某些操作系统和应用程序可能无法正常运行或可能出现不稳定的情况。  
   如果你在虚拟机中开发 Android，你可能无法在虚拟机中再开启一个模拟器。

3. 安全性问题  
   虚拟机之间的隔离性可能会下降，从而增加了安全风险。

#### 如何解决
1. 确认你的主板支持 AMD-V 或者 Intel VT-x  
   现在的主板基本都支持，你需要通过搜索引擎，搜索你的主板该如何确认是否已经开启该功能。
2. 关闭组策略中的如下设置  
   在开始菜单中搜索 gpedit，打开组策略编辑器：
   ![2023-11-14-exec-gpedit.png](/images/other/20231114_exec_gpedit.png)
   在 _计算机配置-管理模板-系统-Device Guard_ 中，双击打开 _基于虚拟化的安全_，设置为 _已禁用_：
   ![2023-11-14-gpedit.png](/images/other/20231114_gpedit_change.png)
3. 关闭 _Windows 安全中心_ 中的内存完整性保护  
   ![20231114_windows_security_center.png](/images/other/20231114_windows_security_center.png)
   将以下内存完整性的开关关闭，然后重启系统。
   ![20231114_close_memory_protect.png](/images/other/20231114_close_memory_protect.png)
4. 如果你曾经开启过Hyper-V，还需要在 _控制面板-程序和功能-启用或关闭Windows功能_ 中，去掉勾选Hyper-V  
   ![20231114_disable_hyper_v.png](/images/other/20231114_disable_hyper_v.png)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/win11-vmware-amd-v/  

