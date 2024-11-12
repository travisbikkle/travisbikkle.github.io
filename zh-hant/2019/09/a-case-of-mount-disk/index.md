# Mount磁盤被秒umount的一個問題


問題描述
========

在 ubuntu 18.04 的機器上，自己搭了一個 samba 服務器。有一天要添加一塊磁盤，因爲服務器上還運行了一些其他服務，不想重啓，因此使用 partprobe 動態掃描了磁盤，分區，寫入 /etc/fstab，一切正常。 執行 mount -a，沒有任何報錯，不過磁盤就是沒有掛載上去。

解決思路
========

1. 使用 mount 命令，可以手動掛載
2. 無任何報錯出現，使用 umount 提示並未掛載
3. 查看 journalctl -xe，發現是 systemd 在 umount 磁盤
4. 在修改 /etc/fstab 後應該執行 systemctl daemon-reload 
5. 解決後，重新mount -a 解決

解決方案來自
&lt;https://unix.stackexchange.com/questions/169909/systemd-keeps-unmounting-a-removable-drive&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/09/a-case-of-mount-disk/  

