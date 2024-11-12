# Android Studio Emulator 启动失败


#### 报错现象

{{&lt;notice warning&gt;}}
The emulator process for AVD Pixel_3a_API_34_extension_level_7_x86_64 has terminated.
{{&lt;/notice&gt;}}

没有任何的报错信息，如果不知道日志在哪里，该如何排查呢？

#### 解决办法
在你的家目录如下位置，执行如下命令，通过命令行查询出虚拟设备并启动，将报错信息打印到前台
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
例如，在上图的最后一行，报错没有足够的磁盘空间，因此扩容后即可解决。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/11/android-studio-emulator-failed/  

