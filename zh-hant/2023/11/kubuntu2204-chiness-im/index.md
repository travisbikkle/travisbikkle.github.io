# Kubuntu 22.04 中文輸入法


### 繼續使用 fcitx5（推薦）

1. 安裝 fcitx5 中文輸入法
   ```shell
   sudo apt install fcitx5-chinese-addons \
   fcitx5-frontend-gtk4 fcitx5-frontend-gtk3 fcitx5-frontend-gtk2 \
   fcitx5-frontend-qt5 $(check-language-support)
   ```
   check-language-support 同時安裝了系統中缺失的中文語言包（在 _區域設置-語言_ 中點擊安裝缺失的語言包沒有反應），這樣才能在第3步中看到 _輸入法_ 的選項。
2. 啓用 fcitx5
   注意，不要相信網上任何要你更改.xinputrc或者在某些文件中添加如下變量的話  
   ~~GTK_IM_MODULE=fcitx~~  
   ~~QT_IM_MODULE=fcitx~~  
   ~~XMODIFIERS=@im=fcitx~~

   請直接使用你登錄的用戶執行以下命令

   ```shell
   im-config
   ```

   在彈出的窗口中，依次選擇確定，是，選擇如下選項即可。
   ![](/images/other/20231108_imconfig.png)

3. 重啓系統
   在 _系統設置-區域設置-輸入法_ 中添加 _拼音_ 或者 _pinyin_
4. 解決 Intellij Idea 中使用 fcitx5 候選框不跟隨的問題 (20240126更新：此時的Idea已經沒有此問題，無需進行此步)
   可以將如下項目的release文件解壓縮到JetBrain產品的jbr目錄中（可能導致markdown無法預覽）
   ```text
   https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64
   ```
5. 程序員推薦在如下配置中關閉簡繁切換快捷鍵  
   _系統設置-區域設置-輸入法-配置附加組件-簡繁切換_ 中刪除&lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SHIFT&lt;/kbd&gt; &lt;kbd&gt;F&lt;/kbd&gt;的簡繁切換快捷鍵。
6. 解決在 Visual Studio Code 中無法使用任何中文輸入法的問題  
   如果你在 Discover 軟件管理工具中下載了 VSCode，請卸載後，在官網下載 deb 包安裝，即可解決無法輸入中文的問題。

### 使用 fcitx4

1. 卸載 fcitx5
   ```shell
   sudo apt-get remove fcitx5
   ```
2. 安裝谷歌拼音輸入法
   ```shell
   sudo apt-get install fcitx-googlepinyin
   ```
3. 參考 [繼續使用 fcitx5](#繼續使用-fcitx5-推薦) 中的第二步，啓用fcitx4  
   你需要重啓以啓用谷歌輸入法，默認 &lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SPACE&lt;/kbd&gt; 切換輸入法，切換到谷歌拼音後可以只使用 &lt;kbd&gt;SHIFT&lt;/kbd&gt;在中文和英文之間來回切換（多麼簡單又困難的事情）。

4. 程序員推薦在如下配置中關閉&lt;kbd&gt;CTRL&lt;/kbd&gt; &lt;kbd&gt;SHIFT&lt;/kbd&gt; &lt;kbd&gt;F&lt;/kbd&gt;的簡繁切換快捷鍵。
   ![](/images/other/20231108_turn_off_ctrl_shift_f.png)

5. 如果希望解決中 Intellij Idea 或者 VSCode 中輸入的問題，請參考[繼續使用 fcitx5](#繼續使用-fcitx5-推薦)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/kubuntu2204-chiness-im/  

