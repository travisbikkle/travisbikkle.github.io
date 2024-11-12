# Intellij Idea 啓動報錯


Intellij Idea 2023.2 啓動報錯：

{{&lt;notice warning&gt;}}
Error while opening intellij &#34;Cannot connect to already running IDE instance. Exception: Process 2,837 is still running&#34;
{{&lt;/notice&gt;}}


#### youtrack 跟蹤
https://youtrack.jetbrains.com/issue/IDEA-330531

#### 解決辦法
在菜單中編輯啓動項，在idea.sh前加入一個前置的腳本，在其中刪除不該存在的.lock文件
```shell
/opt/idea-IU-232.10203.10/bin/prestart.sh; /opt/idea-IU-232.10203.10/bin/idea.sh
```

![](/images/other/20231114_idea_start_error_workground.png)

腳本的內容爲
```shell
#!/bin/bash
rm ~/.config/JetBrains/IntelliJIdea2023.2/.lock
```



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/idea-start-error/  

