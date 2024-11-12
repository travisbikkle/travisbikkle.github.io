# 一次ssh免密無法登錄的問題


自研數據庫升級的過程中，需要配置一次ssh免密登錄，以便在其中一臺機器，很方便的升級集羣所有服務器。但是在測試過程中，創建免密登錄的腳本失效了。

問題定位
========

通過手動創建公鑰，拷貝到其它機器的authrized\_keys，發現仍然需要輸入密碼登錄；

通過ssh -v &lt;user@ip&gt;，查看詳細信息，發現其提示： Authentication can
continue: publickey, gssapi-key…..password try publickey .ssh/….. try
publickey .ssh/….. using password:

而正常的服務器上，該行爲： Servert accepted key…

1.  看日誌，懷疑ssh客戶端沒有找到正確的公鑰文件，但是該文件確實存在在正確的路徑，且擁有正確的600權限。

2.  嘗試使用其它端口，啓動服務端的sshd，發現可以免密登錄

3.  嘗試在客戶端新建其它用戶，並使用22默認端口，一樣可以免密登錄

4.  執行以下命令，對比新建用戶的目錄，和問題用戶的目錄，發現問題

        ls -laZ

在問題用戶的目錄中，.ssh目錄的label爲unlabel，而正常用戶的.ssh，爲user\_t。通過查詢及測試，發現該目錄爲user\_t或者ssh\_home\_t的標籤，都可以測試通過，但是爲ublabeled不行。
那麼該目錄爲什麼爲unlabel呢？畢竟我們執行的只是ssh-keygen，目錄並非我們生成。
其實這個目錄的標籤，會繼承父目錄的標籤，而父目錄的標籤，由於未知原因，丟失了。因此，selinux的機制不允許ssh使用該目錄作爲公鑰目錄。該問題可以通過以下兩種方法解決：

1.  restorecon -vv -r ~/.ssh

2.  setenforce 0等關閉selinux


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/08/ssh-cant-login/  

