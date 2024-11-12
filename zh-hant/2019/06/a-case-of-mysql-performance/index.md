# Mysql 性能測試報告慘不忍睹的一次原因


背景
====

新版本發佈在即，新增與kafka，zk，redis等其它服務合設的場景，因此需要性能測試探個底。但是性能測試人員反饋，分設與合設性能差距數萬倍。

問題
====

1.  安裝兩套某版本的MySQL，分別是與redis等服務合設的一套（32U64G），MySQL單獨安裝的一套（8U16G）。

2.  使用jmeter並且採用同樣的測試模型，對服務進行長穩測試，其中合設的一套資源，其它服務也會跑長穩。

3.  觀察jmeter報告，發現合設的MySQL中，平均時間average爲20s以上，throughput在10個/秒以下。而獨立的MySQL，average爲ms級別，throughput爲數萬個。

此時應該隨報告提供環境cpu，磁盤，內存，網絡帶寬，測試模型等情況。但是由於某些原因，測試同學無法提供。

測試模型和環境
==============

該測試模型很簡單，1個int主鍵，20個varchar（256）字段共100w的數據。
使用select \* from xxx where id in random(1, 100w); 進行1000併發測試。

因爲某些原因，測試同學無法提供定位問題需要的幫助，因此，我們只收集了以下幾種信息。

-   cpu及內存（top）

-   iostat -x 1 以及其它 /dev/mysqllv(mysql 數據掛載磁盤)

-   查看jmeter併發服務器的jmap histro

-   查看mysql show processlist，show status like &#34;%Thread%&#34;等

定位過程
========

1.  cpu，內存等資源佔用不高。

2.  iostat 磁盤讀寫速度不高，但是%util佔比居高不下

3.  單獨執行一條sql語句，偶爾很快，偶爾很慢

4.  show processlist反饋，大量連接停留在freeing
    items的狀態，說明連接在等候io操作，與上述iostat結論不謀而合

5.  hdparm -Tt 合設機器只有60m/s的磁盤速度（未停服務，不準確）

6.  查詢審計日誌，發現審計日誌大量積壓，5M/2分鐘的速度在持續增加

7.  走管理手段，要求測試人員提供獨立 MySQL
    機器信息，其反饋非其本人測試並且已卸載。我們強烈要求重新安裝獨立mysql

8.  獨立mysql表現也慢，與合設無異（說明測試沒有控制好變量）

9.  最終發現，其獨立mysql測試版本爲較老版本

結論
====

新版本mysql開啓audit日誌，並且安全部門參照公司《mysql
安全加固規範》中的必須修改項——審計日誌的策略 audit\_log\_strategy 必須爲
`SYNCHRONOUS` ,
利用自動化工具掃描並發現我們的mysql沒有開啓此項，提了單。修改人員在修改時因爲經驗不足，未多想，便直接修改。

該項的另一個可選值爲ASYNCHRONOUS，即異步。與主從複製的同步，半同步及異步類似，作用不言自明。改爲該值後，問題消失。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/06/a-case-of-mysql-performance/  

