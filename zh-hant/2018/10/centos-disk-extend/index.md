# 如何對 Centos 分區進行擴容


    fdisk -l

在 Id 一列可以看到分區類型爲\`\`8e\`\`
\`\`83\`\`等十進制數值，\`\`8e\`\`代表該分區由 Linux LVM
管理，適用本文的擴容方法，如果你的分區類型爲\`\`83\`\`，代表其是 Linux
Native Partion，可以參考link:\[另一篇博文（尚未書寫）\]。

我使用的是 VMware Workstation
pro，在編輯虛擬機設置裏可以輕鬆增加磁盤大小（磁盤爲單個文件，而不是分割文件，如果你的硬盤是分割的多個文件，參考另一篇博文
[VMware 分割磁盤如何擴容(尚未編寫)]()

    fdisk -l
    ## 在 fdisk 輸出信息中，可以看到 Disk /dev/sda： 30 GB 類似的信息，證明磁盤增加成功，位置確認。

    ## 以下命令爲交互式命令
    fdisk /dev/sda

    # 輸入 n 以創建新分區
    n
    # 輸入 p 以設置爲主分區
    p
    # 根據 fdisk -l 的信息，決定分區的編號，由於我的機器 fdisk -l 已經有 /dev/sda1 /dev/sda2 兩個，所以此處輸入 3
    3
    # 此處輸入兩次回車，以決定分區的開始和結束位置，默認使用剩餘全部未分配空間
    First cylinder.... 回車
    Last cylinder.... 回車
    # 此處輸入 t，並輸入 3 以選擇我們上面步驟剛剛創建的分區
    t
    3
    # 在 Hex code 的輸入步驟，輸入我們希望使用的 LVM 代碼符號：8e
    8e
    # 最後，輸入 w 以使上述所有更改生效
    w

在我的機器上，不需要重啓已經可以使用 `fdisk -l` 查看到新創建的
/dev/sda3，但是推薦你在此處先重啓一次，然後執行後續操作

關鍵步驟來了。此處使用到了 pv，vg，lv
等名詞，請自行搜索瞭解，如果不瞭解，也不影響操作執行。

    # 在 /dev/sda3 創建 pv
    pvcreate /dev/sda3 # 如果提示 Device /dev/sda3 not found, 請先重啓。
    # 查看 vg 信息，獲取到 vg 的 name，一般是你的機器名稱，我的機器爲 centos
    vgdisplay
    # 添加 pv 到 vg
    vgextend centos /dev/sda3 # 這裏的 centos 是上一步查詢出的 vg name
    # 查看 lv 的 path 信息
    lvdisplay # 此處我的 path 信息爲 /dev/centos/root
    # 將新分區擴容到 lv
    lvextend /dev/centos/root /dev/sda3 # 此處 /dev/centos/root 爲上一步查詢出的 path
    # 最後一步
    xfs_growfs /dev/centos/root # centos 7/RedHat 默認使用 xfs 文件系統，如果是 ext 文件系統，可以使用 resize2fs /dev/Mega/root 命令

使用 df -h 查看擴容的結果吧~


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/centos-disk-extend/  

