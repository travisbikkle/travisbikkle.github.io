# 使用 Shell 腳本計算 Ip 網段


背景
====

當無法使用 ipcalc 時，可以考慮使用 shell 實現一個 ipcalc.sh。

**ipcalc.sh.**

    #!/bin/bash
    net=&#34;$1&#34;
    ip=(${net%/*})
    cdr=(${net##*/})

    cdr2mask(){
        #set -- $((5-(&#34;$1&#34;/8))) 255 255 255 255 $((2**8-2**(8-&#34;$1&#34;%8))) 0 0 0
        set -- $(( 5-(&#34;$1&#34;/8) )) 255 255 255 255 $(( (255 &lt;&lt; (8-(&#34;$1&#34;%8))) &amp; 255 )) 0 0 0
        [[ $1 -gt 1 ]] &amp;&amp; shift $1 || shift
        #echo $#:$@
        #255 255 255 255 253 0 0 0 shift
        #^_____________^           255.255.255.255 shift
        #    ^_____________^       255.255.255.253 shift 2
        #        ^___________^     255.255.253.0   shift 3
        #default 0, just in case
        echo ${1-0}.${2-0}.${3-0}.${4-0}
    }

    msk=$(cdr2mask $cdr)
    IFS=. read -r i1 i2 i3 i4 &lt;&lt;&lt; &#34;${ip}&#34;
    IFS=. read -r m1 m2 m3 m4 &lt;&lt;&lt; &#34;${msk}&#34;

    printf &#34;%d.%d.%d.%d&#34; &#34;$((i1 &amp; m1))&#34; &#34;$((i2 &amp; m2))&#34; &#34;$((i3 &amp; m3))&#34; &#34;$((i4 &amp; m5))&#34;

    ## usage
    # route=$(sh ipcalc.sh 192.168.31.123/24)
    # echo $route

**基於上述腳本實現的小程序(僞代碼).**

    get_matched_interface(){
        xxxx
        for eth in eths;do
            ip=get_ip_eth $eth
            if ipcalc.sh $ip == ipcalc.sh $float_ip;then
                return eth.name
            fi
        done
    }


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/06/shell-calculate-ip/  

