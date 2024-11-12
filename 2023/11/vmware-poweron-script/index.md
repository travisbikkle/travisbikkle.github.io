# Vmware Power-on 脚本启动失败


VMware 的开机脚本位于 `/etc/vmware-tools/` 下。如果你的虚拟机报错启动客户机脚本失败，你可以在虚拟机-电源-开机中开机，然后进入该目录调试。

如果你的vmware-tools进行过升级，但是升级过程中出现了问题，可能导致脚本没有正确命名。比如，下面是开机、关机等操作对应的脚本名称
```text
poweroff-vm-default
poweron-vm-default
resume-vm-default
suspend-vm-default
vgauth.conf
```
而你的机器上可能是
```text
poweroff-vm-default.dpkg-dist
poweron-vm-default.dpkg-dist
resume-vm-default.dpkg-dist
suspend-vm-default.dpkg-dist
vgauth.conf.dpkg-dist
```
那么请执行以下命令重命名即可
```bash
find /etc/vmware-tools/ -name &#34;*.dpkg-dist&#34; -exec sh -c &#39;x={}; cp &#34;$x&#34; $(echo $x | sed &#39;s/\.dpkg-dist//g&#39;)&#39; \;
```
另外你还可能需要检查 `/etc/vmware-tools/scripts/vmware` 以及各个脚本是否具有可执行权限。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/vmware-poweron-script/  

