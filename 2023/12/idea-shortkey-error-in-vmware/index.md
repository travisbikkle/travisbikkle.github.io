# Vmware 中 Intellij Idea 快捷键错误


在VMware中，&lt;kbd&gt;CTRL ALT&lt;/kbd&gt;是默认的将鼠标焦点从虚拟机中移开的热键，这将导致一切在Vmware中运行的虚拟机系统中以该组合键
位开头的热键不可用。比如Intellij Idea的默认后退快捷键&lt;kbd&gt;CTRL ALT LEFT&lt;/kbd&gt;。

一开始我以为是KDE热键冲突的问题，因为KDE默认有很多的快捷键，但是经过查看KDE的系统设置，并没有发现热键占用的情况：

![Screenshot_20231201_163449.png](/images/other/Screenshot_20231201_163449.png)

直到我意识到screenkey也没有提示，这说明我的KDE根本没有接收到&lt;kbd&gt;CTRL ALT LEFT&lt;/kbd&gt;按键。答案显而易见，是Vmware的问题了。
为了防止冲突，在Vmware的 _编辑-首选项-热键_ 中可以将这四个按键都勾选上。
![20231201_vmware_pref_hotkey.png](/images/other/20231201_vmware_pref_hotkey.png)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/12/idea-shortkey-error-in-vmware/  

