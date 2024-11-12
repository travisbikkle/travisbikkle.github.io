# Windows 常用命令


#### 在右键菜单中添加管理员获得所有权
1. 打开记事本，将以下内容粘贴
   ```text
   Windows Registry Editor Version 5.00
   [HKEY_CLASSES_ROOT\*\shell\runas]
   @=&#34;管理员取得所有权&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\*\shell\runas\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   [HKEY_CLASSES_ROOT\exefile\shell\runas2]
   @=&#34;管理员取得所有权&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\exefile\shell\runas2\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F&#34;
   [HKEY_CLASSES_ROOT\Directory\shell\runas]
   @=&#34;管理员取得所有权&#34;
   &#34;NoWorkingDirectory&#34;=&#34;&#34;
   [HKEY_CLASSES_ROOT\Directory\shell\runas\command]
   @=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; /r /d y &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F /t&#34;
   &#34;IsolatedCommand&#34;=&#34;cmd.exe /c takeown /f \&#34;%1\&#34; /r /d y &amp;&amp; icacls \&#34;%1\&#34; /grant administrators:F /t&#34;
   ```
2. 另存为utf16-le格式，后缀名为reg文件
3. 双击该文件，添加注册表

#### 暂停 win11 更新
操作步骤同上，内容更换为

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

还原

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
> URL: https://travisbikkle.github.io/2023/10/windows-common-commands/  

