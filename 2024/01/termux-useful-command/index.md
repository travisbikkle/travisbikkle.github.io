# Termux 常用命令


## 常用的内置的 termux 命令
输入 `termux` 并按 Tab 提示，控制台会显示一些 `termux` 开头的命令。
下面是一些最常用命令，如更换软件源或者在手机内拷贝文件：

```bash
# 更换软件源
termux-change-repo
# 开启存储权限
termux-setup-storage
# 保持常亮
termux-wake-lock
```
## 从手机复制文件

```bash
# 开启存储权限
termux-setup-storage
# 执行后重新启动 termux，然后输入
ln -s /sdcard/Download ~/download
# 或者有些手机是
ln -s /sdcard/Downloads ~/download
```

## 定时任务

```bash
# 保持常亮
termux-wake-lock
# 安装并启动 crond
apt install cronie
crond
# 增加一个定时任务
crontab -e
# 手动输入或使用 https://crontab-generator.org/ 生成一个，例如
1 * * * * bash ~/download/update_domain.sh &gt; ~/download/update.log
# 按 ctrl o 保存， ctrl x 退出
```

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/01/termux-useful-command/  

