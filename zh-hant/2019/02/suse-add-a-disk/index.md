# Suse 新增磁盤


    fdisk -l
    DISK=vdb
    disk_list=(`cat /proc/partitions | sort | grep -v &#34;name&#34; |grep -v &#34;loop&#34; |awk &#39;{print $4}&#39;| sed /^[[:space:]]*$/d | grep -v &#34;[[:digit:]]&#34; | uniq`)
    parted -s /dev/${DISK} mklabel gpt
    parted -s /dev/${DISK} print |grep softhome |wc -l
    DISKSIZE=`parted -s /dev/${DISK} unit GB print | grep &#39;^Disk&#39; |grep GB | awk &#39;{print $3}&#39;`
    DISK1=`echo ${DISK}1` #
    parted -s /dev/${DISK} mkpart softhome 0G $DISKSIZE
    parted -s /dev/${DISK} set 1 lvm

    vgname=`echo &#34;/opt&#34; | awk -F&#39;/&#39; &#39;{print $NF}&#39;`
    vgname=&#34;${vgname}vg&#34; # optvg
    lvname=`echo &#34;/opt&#34; | awk -F&#39;/&#39; &#39;{print $NF}&#39;`
    lvname=&#34;${lvname}lv&#34; # optlv

    echo y | pvcreate /dev/${DISK1}

    vgcreate &#34;$vgname&#34; /dev/${DISK1}
    free=`vgdisplay &#34;$vgname&#34; |grep &#34;Total PE&#34; |awk &#39;{print $3}&#39;`
    echo y | lvcreate -l &#34;$free&#34; -n &#34;$lvname&#34; &#34;$vgname&#34;

    lvPath=`lvdisplay &#34;$vgname&#34; | grep &#34;LV Path&#34; | awk &#39;{print $3}&#39;`

    #格式化
    mkfs.ext4 &#34;${lvPath}&#34;

    #掛載
    mount -t ext4 ${lvPath} /opt

    cat /etc/fstab | grep -w $lvPath

    #永久
    echo &#34;$lvPath            ${PATHS}                    ext4       defaults        1 0&#34; &gt;&gt; /etc/fstab


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/suse-add-a-disk/  

