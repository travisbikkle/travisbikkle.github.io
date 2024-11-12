# Termux 常用命令


## 常用的內置的 termux 命令
輸入 `termux` 並按 Tab 提示，控制檯會顯示一些 `termux` 開頭的命令。
下面是一些最常用命令，如更換軟件源或者在手機內拷貝文件：

```bash
# 更換軟件源
termux-change-repo
# 開啓存儲權限
termux-setup-storage
# 保持常亮
termux-wake-lock
```
## 從手機複製文件

```bash
# 開啓存儲權限
termux-setup-storage
# 執行後重新啓動 termux，然後輸入
ln -s /sdcard/Download ~/download
# 或者有些手機是
ln -s /sdcard/Downloads ~/download
```

## 定時任務

```bash
# 保持常亮
termux-wake-lock
# 安裝並啓動 crond
apt install cronie
crond
# 增加一個定時任務
crontab -e
# 手動輸入或使用 https://crontab-generator.org/ 生成一個，例如
1 * * * * bash ~/download/update_domain.sh &gt; ~/download/update.log
# 按 ctrl o 保存， ctrl x 退出
```

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/01/termux-useful-command/  

