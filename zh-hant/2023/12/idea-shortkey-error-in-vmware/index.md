# Vmware 中 Intellij Idea 快捷鍵錯誤


在VMware中，&lt;kbd&gt;CTRL ALT&lt;/kbd&gt;是默認的將鼠標焦點從虛擬機中移開的熱鍵，這將導致一切在Vmware中運行的虛擬機系統中以該組合鍵
位開頭的熱鍵不可用。比如Intellij Idea的默認後退快捷鍵&lt;kbd&gt;CTRL ALT LEFT&lt;/kbd&gt;。

一開始我以爲是KDE熱鍵衝突的問題，因爲KDE默認有很多的快捷鍵，但是經過查看KDE的系統設置，並沒有發現熱鍵佔用的情況：

![Screenshot_20231201_163449.png](/images/other/Screenshot_20231201_163449.png)

直到我意識到screenkey也沒有提示，這說明我的KDE根本沒有接收到&lt;kbd&gt;CTRL ALT LEFT&lt;/kbd&gt;按鍵。答案顯而易見，是Vmware的問題了。
爲了防止衝突，在Vmware的 _編輯-首選項-熱鍵_ 中可以將這四個按鍵都勾選上。
![20231201_vmware_pref_hotkey.png](/images/other/20231201_vmware_pref_hotkey.png)

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/12/idea-shortkey-error-in-vmware/  

