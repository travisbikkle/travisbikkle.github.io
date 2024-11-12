# Windows 常用命令


#### 在右鍵菜單中添加管理員獲得所有權
1. 打開記事本，將以下內容粘貼
   ```text
   Windows Registry Editor Version 5.00
   [HKEY_CLASSES_ROOT\*\shell\runas]
   @=&#34;管理員取得所有權&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\*\shell\runas\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   [HKEY_CLASSES_ROOT\exefile\shell\runas2]
   @=&#34;管理員取得所有權&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\exefile\shell\runas2\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   [HKEY_CLASSES_ROOT\Directory\shell\runas]
   @=&#34;管理員取得所有權&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\Directory\shell\runas\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; /r /d y &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F /t&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; /r /d y &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F /t&#34;
   ```
2. 另存爲utf16-le格式，後綴名爲reg文件
3. 雙擊該文件，添加註冊表

#### 暫停 win11 更新
操作步驟同上，內容更換爲

```text
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings]
&#34;FlightSettingsMaxPauseDays&#34;=dword:00001b58
&#34;PauseFeatureUpdatesStartTime&#34;=&#34;2023-07-07T10:00:52Z&#34;
&#34;PauseFeatureUpdatesEndTime&#34;=&#34;2042-09-05T09:59:52Z&#34;
&#34;PauseQualityUpdatesStartTime&#34;=&#34;2023-07-07T10:00:52Z&#34;
&#34;PauseQualityUpdatesEndTime&#34;=&#34;2042-09-05T09:59:52Z&#34;
&#34;PauseUpdatesStartTime&#34;=&#34;2023-07-07T09:59:52Z&#34;
&#34;PauseUpdatesExpiryTime&#34;=&#34;2042-09-05T09:59:52Z&#34;
```

還原

```text
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings]
&#34;FlightSettingsMaxPauseDays&#34;=-
&#34;PauseFeatureUpdatesStartTime&#34;=-
&#34;PauseFeatureUpdatesEndTime&#34;=-
&#34;PauseQualityUpdatesStartTime&#34;=-
&#34;PauseQualityUpdatesEndTime&#34;=-
&#34;PauseUpdatesStartTime&#34;=-
&#34;PauseUpdatesExpiryTime&#34;=-
```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/10/windows-common-commands/  

