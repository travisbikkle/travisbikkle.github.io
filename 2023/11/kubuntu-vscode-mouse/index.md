# Kubuntu Vscode 鼠标列编辑设置


鼠标中键是某些操作系统的默认粘贴快捷键，这导致在VSCode中无法使用鼠标中键进入列编辑模式。

1. 点击 _文件-首选项-设置_
   ![Screenshot_20231115_134220.png](/images/other/20231115_vscode_setting.png)

2. 点击右上角，打开json文件
   ![Screenshot_20231115_134322.png](/images/other/20231115_vscode_setting_selection.png)

3. 添加如下行

   ```json
   &#34;editor.selectionClipboard&#34;: false
   ```

4. 关闭 VSCode 并重新打开，测试是否生效。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/kubuntu-vscode-mouse/  

