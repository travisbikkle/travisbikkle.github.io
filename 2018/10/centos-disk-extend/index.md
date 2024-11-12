# 如何对 Centos 分区进行扩容


    fdisk -l

在 Id 一列可以看到分区类型为\`\`8e\`\`
\`\`83\`\`等十进制数值，\`\`8e\`\`代表该分区由 Linux LVM
管理，适用本文的扩容方法，如果你的分区类型为\`\`83\`\`，代表其是 Linux
Native Partion，可以参考link:\[另一篇博文（尚未书写）\]。

我使用的是 VMware Workstation
pro，在编辑虚拟机设置里可以轻松增加磁盘大小（磁盘为单个文件，而不是分割文件，如果你的硬盘是分割的多个文件，参考另一篇博文
[VMware 分割磁盘如何扩容(尚未编写)]()

    fdisk -l
    ## 在 fdisk 输出信息中，可以看到 Disk /dev/sda： 30 GB 类似的信息，证明磁盘增加成功，位置确认。

    ## 以下命令为交互式命令
    fdisk /dev/sda

    # 输入 n 以创建新分区
    n
    # 输入 p 以设置为主分区
    p
    # 根据 fdisk -l 的信息，决定分区的编号，由于我的机器 fdisk -l 已经有 /dev/sda1 /dev/sda2 两个，所以此处输入 3
    3
    # 此处输入两次回车，以决定分区的开始和结束位置，默认使用剩余全部未分配空间
    First cylinder.... 回车
    Last cylinder.... 回车
    # 此处输入 t，并输入 3 以选择我们上面步骤刚刚创建的分区
    t
    3
    # 在 Hex code 的输入步骤，输入我们希望使用的 LVM 代码符号：8e
    8e
    # 最后，输入 w 以使上述所有更改生效
    w

在我的机器上，不需要重启已经可以使用 `fdisk -l` 查看到新创建的
/dev/sda3，但是推荐你在此处先重启一次，然后执行后续操作

关键步骤来了。此处使用到了 pv，vg，lv
等名词，请自行搜索了解，如果不了解，也不影响操作执行。

    # 在 /dev/sda3 创建 pv
    pvcreate /dev/sda3 # 如果提示 Device /dev/sda3 not found, 请先重启。
    # 查看 vg 信息，获取到 vg 的 name，一般是你的机器名称，我的机器为 centos
    vgdisplay
    # 添加 pv 到 vg
    vgextend centos /dev/sda3 # 这里的 centos 是上一步查询出的 vg name
    # 查看 lv 的 path 信息
    lvdisplay # 此处我的 path 信息为 /dev/centos/root
    # 将新分区扩容到 lv
    lvextend /dev/centos/root /dev/sda3 # 此处 /dev/centos/root 为上一步查询出的 path
    # 最后一步
    xfs_growfs /dev/centos/root # centos 7/RedHat 默认使用 xfs 文件系统，如果是 ext 文件系统，可以使用 resize2fs /dev/Mega/root 命令

使用 df -h 查看扩容的结果吧~


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/centos-disk-extend/  

