# Android Studio Emulator 啓動失敗


#### 報錯現象

{{&lt;notice warning&gt;}}
The emulator process for AVD Pixel_3a_API_34_extension_level_7_x86_64 has terminated.
{{&lt;/notice&gt;}}

沒有任何的報錯信息，如果不知道日誌在哪裏，該如何排查呢？

#### 解決辦法
在你的家目錄如下位置，執行如下命令，通過命令行查詢出虛擬設備並啓動，將報錯信息打印到前臺
   ```text
   yourname@v:~/Android/Sdk/emulator$ ./emulator -list-avds
   Pixel_3a_API_34_extension_level_7_x86_64
   yourname@v:~/Android/Sdk/emulator$ ./emulator @Pixel_3a_API_34_extension_level_7_x86_64
   INFO    | Android emulator version 32.1.15.0 (build_id 10696886) (CL:N/A)
   INFO    | Found systemPath /home/yourname/Android/Sdk/system-images/android-34/google_apis/x86_64/
   INFO    | Storing crashdata in: /tmp/android-yourname/emu-crash.db, detection is enabled
   INFO    | Duplicate loglines will be removed, if you wish to see each indiviudal line launch with the -log-nofilter flag.
   WARNING | Please update the emulator to one that supports the feature(s): SupportPixelFold
   ERROR   | Not enough space to create userdata partition. Available: 2066.355469 MB at /home/yourname/.android/avd/Pixel_3a_API_34_extension_level_7_x86_64.avd, need 7372.800000 MB.
   ```
例如，在上圖的最後一行，報錯沒有足夠的磁盤空間，因此擴容後即可解決。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/android-studio-emulator-failed/  

