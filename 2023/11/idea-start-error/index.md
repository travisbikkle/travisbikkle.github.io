# Intellij Idea 启动报错


Intellij Idea 2023.2 启动报错：

{{&lt;notice warning&gt;}}
Error while opening intellij &#34;Cannot connect to already running IDE instance. Exception: Process 2,837 is still running&#34;
{{&lt;/notice&gt;}}


#### youtrack 跟踪
https://youtrack.jetbrains.com/issue/IDEA-330531

#### 解决办法
在菜单中编辑启动项，在idea.sh前加入一个前置的脚本，在其中删除不该存在的.lock文件
```shell
/opt/idea-IU-232.10203.10/bin/prestart.sh; /opt/idea-IU-232.10203.10/bin/idea.sh
```

![](/images/other/20231114_idea_start_error_workground.png)

脚本的内容为
```shell
#!/bin/bash
rm ~/.config/JetBrains/IntelliJIdea2023.2/.lock
```



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/idea-start-error/  

