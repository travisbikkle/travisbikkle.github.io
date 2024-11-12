# Kubuntu 22.04 中文输入法


### 继续使用 fcitx5（推荐）

1. 安装 fcitx5 中文输入法
   ```shell
   sudo apt install fcitx5-chinese-addons \
   fcitx5-frontend-gtk4 fcitx5-frontend-gtk3 fcitx5-frontend-gtk2 \
   fcitx5-frontend-qt5 $(check-language-support)
   ```
   check-language-support 同时安装了系统中缺失的中文语言包（在 _区域设置-语言_ 中点击安装缺失的语言包没有反应），这样才能在第3步中看到 _输入法_ 的选项。
2. 启用 fcitx5
   注意，不要相信网上任何要你更改.xinputrc或者在某些文件中添加如下变量的话  
   ~~GTK_IM_MODULE=fcitx~~  
   ~~QT_IM_MODULE=fcitx~~  
   ~~XMODIFIERS=@im=fcitx~~

   请直接使用你登录的用户执行以下命令

   ```shell
   im-config
   ```

   在弹出的窗口中，依次选择确定，是，选择如下选项即可。
   ![](/images/other/20231108_imconfig.png)

3. 重启系统
   在 _系统设置-区域设置-输入法_ 中添加 _拼音_ 或者 _pinyin_
4. 解决 Intellij Idea 中使用 fcitx5 候选框不跟随的问题 (20240126更新：此时的Idea已经没有此问题，无需进行此步)
   可以将如下项目的release文件解压缩到JetBrain产品的jbr目录中（可能导致markdown无法预览）
   ```text
   https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64
   ```
5. 程序员推荐在如下配置中关闭简繁切换快捷键  
   _系统设置-区域设置-输入法-配置附加组件-简繁切换_ 中删除&lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SHIFT&lt;/kbd&gt; &lt;kbd&gt;F&lt;/kbd&gt;的简繁切换快捷键。
6. 解决在 Visual Studio Code 中无法使用任何中文输入法的问题  
   如果你在 Discover 软件管理工具中下载了 VSCode，请卸载后，在官网下载 deb 包安装，即可解决无法输入中文的问题。

### 使用 fcitx4

1. 卸载 fcitx5
   ```shell
   sudo apt-get remove fcitx5
   ```
2. 安装谷歌拼音输入法
   ```shell
   sudo apt-get install fcitx-googlepinyin
   ```
3. 参考 [继续使用 fcitx5](#继续使用-fcitx5-推荐) 中的第二步，启用fcitx4  
   你需要重启以启用谷歌输入法，默认 &lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SPACE&lt;/kbd&gt; 切换输入法，切换到谷歌拼音后可以只使用 &lt;kbd&gt;SHIFT&lt;/kbd&gt;在中文和英文之间来回切换（多么简单又困难的事情）。

4. 程序员推荐在如下配置中关闭&lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SHIFT&lt;/kbd&gt; &lt;kbd&gt;F&lt;/kbd&gt;的简繁切换快捷键。
   ![](/images/other/20231108_turn_off_ctrl_shift_f.png)

5. 如果希望解决中 Intellij Idea 或者 VSCode 中输入的问题，请参考[继续使用 fcitx5](#继续使用-fcitx5-推荐)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/kubuntu2204-chiness-im/  

