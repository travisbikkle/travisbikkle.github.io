# Kubuntu Vscode 鼠標列編輯設置


鼠標中鍵是某些操作系統的默認粘貼快捷鍵，這導致在VSCode中無法使用鼠標中鍵進入列編輯模式。

1. 點擊 _文件-首選項-設置_
   ![Screenshot_20231115_134220.png](/images/other/20231115_vscode_setting.png)

2. 點擊右上角，打開json文件
   ![Screenshot_20231115_134322.png](/images/other/20231115_vscode_setting_selection.png)

3. 添加如下行

   ```json
   &#34;editor.selectionClipboard&#34;: false
   ```

4. 關閉 VSCode 並重新打開，測試是否生效。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/kubuntu-vscode-mouse/  

