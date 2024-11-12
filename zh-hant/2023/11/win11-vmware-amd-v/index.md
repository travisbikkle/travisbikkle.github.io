# Windows Vmware Amd-v 開啓


#### 報錯現象
我在 Win 11 中安裝了 Vmware，並在Vmware中安裝了 Kubuntu 22.04 虛擬機。
當我在虛擬機設置的cpu設置中開啓AMD-V的時候，Vmware提示如下錯誤：
![2023-11-14-exec-gpedit.png](/images/other/20231114_vmware_amdv_error.png)

起初我並沒有在意，只是感覺虛擬機的性能有些差，直到我在虛擬機中嘗試使用Android Studio附帶的安卓模擬器時，遇到了問題。

#### 無法開啓 AMD-V 或 Intel VT-x 的影響
1. 性能下降  
   虛擬機的性能將受到限制，因爲它不能充分利用硬件虛擬化擴展，指令執行和內存管理開銷可能增加。
   這個性能下降是肉眼可見的，你在點擊、挪動鼠標的時候會感覺到有一種難以察覺的遲滯感，非常難受。

2. 兼容性問題  
   某些操作系統和應用程序可能無法正常運行或可能出現不穩定的情況。  
   如果你在虛擬機中開發 Android，你可能無法在虛擬機中再開啓一個模擬器。

3. 安全性問題  
   虛擬機之間的隔離性可能會下降，從而增加了安全風險。

#### 如何解決
1. 確認你的主板支持 AMD-V 或者 Intel VT-x  
   現在的主板基本都支持，你需要通過搜索引擎，搜索你的主板該如何確認是否已經開啓該功能。
2. 關閉組策略中的如下設置  
   在開始菜單中搜索 gpedit，打開組策略編輯器：
   ![2023-11-14-exec-gpedit.png](/images/other/20231114_exec_gpedit.png)
   在 _計算機配置-管理模板-系統-Device Guard_ 中，雙擊打開 _基於虛擬化的安全_，設置爲 _已禁用_：
   ![2023-11-14-gpedit.png](/images/other/20231114_gpedit_change.png)
3. 關閉 _Windows 安全中心_ 中的內存完整性保護  
   ![20231114_windows_security_center.png](/images/other/20231114_windows_security_center.png)
   將以下內存完整性的開關關閉，然後重啓系統。
   ![20231114_close_memory_protect.png](/images/other/20231114_close_memory_protect.png)
4. 如果你曾經開啓過Hyper-V，還需要在 _控制面板-程序和功能-啓用或關閉Windows功能_ 中，去掉勾選Hyper-V  
   ![20231114_disable_hyper_v.png](/images/other/20231114_disable_hyper_v.png)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/win11-vmware-amd-v/  

