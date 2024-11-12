# Vmware Power-on 腳本啓動失敗


VMware 的開機腳本位於 `/etc/vmware-tools/` 下。如果你的虛擬機報錯啓動客戶機腳本失敗，你可以在虛擬機-電源-開機中開機，然後進入該目錄調試。

如果你的vmware-tools進行過升級，但是升級過程中出現了問題，可能導致腳本沒有正確命名。比如，下面是開機、關機等操作對應的腳本名稱
```text
poweroff-vm-default
poweron-vm-default
resume-vm-default
suspend-vm-default
vgauth.conf
```
而你的機器上可能是
```text
poweroff-vm-default.dpkg-dist
poweron-vm-default.dpkg-dist
resume-vm-default.dpkg-dist
suspend-vm-default.dpkg-dist
vgauth.conf.dpkg-dist
```
那麼請執行以下命令重命名即可
```bash
find /etc/vmware-tools/ -name &#34;*.dpkg-dist&#34; -exec sh -c &#39;x={}; cp &#34;$x&#34; $(echo $x | sed &#39;s/\.dpkg-dist//g&#39;)&#39; \;
```
另外你還可能需要檢查 `/etc/vmware-tools/scripts/vmware` 以及各個腳本是否具有可執行權限。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/vmware-poweron-script/  

