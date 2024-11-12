# Termux Useful Command


## useful termux built-in commands
Type `termux` in the console and press tab, a list of commands starts with `termux` will show up.
Here are some most commonly used commands:

```bash
# change apt repository
termux-change-repo
# storage permission
termux-setup-storage
# keep awake
termux-wake-lock
```

## copy file from phone

```bash
# storage permission
termux-setup-storage
# restart termux and type
ln -s /sdcard/Download ~/download
# or below on some phone
ln -s /sdcard/Downloads ~/download
```

## crontab

```bash
# keep awake
termux-wake-lock
# install crond &amp; start it
apt install cronie
crond
# add a new crontab entry
crontab -e
# type your crontab entry or use https://crontab-generator.org/ to generate one, like
1 * * * * bash ~/download/update_domain.sh &gt; ~/download/update.log
# press ctrl o to save, and quit with ctrl x
```

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/01/termux-useful-command/  

