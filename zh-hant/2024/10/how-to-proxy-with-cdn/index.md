# 使用 Cdn 低成本加速網站，或解決443和80端口被封問題


CDN是個好東西，我們一般需要使用CDN，來讓我們的網站能夠在全球都能快速訪問，並且降低服務器壓力，防止DDOS攻擊等。

簡單來說，你把你的域名的CNAME(如`a.example.com`) 指向CDN廠商的域名(如`abc.cloudfront.net`)。

用戶訪問`a.example.com`的時候，實際上訪問的是`abc.cloudfront.net`。`abc.cloudfront.net`上面當然沒有你的站點資源，所以會回頭訪問你配置的源站（這一步叫做`回源`）拉取資源，並存儲到本地（這一步叫做`緩存`）。

你也可以在站點發布後，第一時間讓CDN拉取站點資源，並下發到它位於全球的站點，防止用戶在高峯期一窩蜂地訪問過來，導致源站壓力過大。這一步叫做`預熱`。

用戶訪問CDN廠商的域名，CDN會分配個給用戶最近的站點，這樣用戶訪問的速度也快了。

CDN還支持指定回源的協議和端口，本文後面將會提及，這對使用家庭寬帶部署服務並且443和80端口被封鎖的用戶非常有幫助。

雖然我測試的域名在騰訊雲，但是我並沒有使用騰訊雲的cdn服務，因爲在選擇試用的時候，騰訊雲提示我沒有資格。

於是我只測試了cloudflare，aws，阿里雲的cdn服務，並在此記錄下來過程和配置，希望能夠幫助有需要的人。

## Cloudflare
1. 右上角點 `Add site`，增加你的域名，按照推薦的配置即可

   ![add site](/images/posts/cdn-tutorial/add-site-cloudflare.png)
2. 將Cloudflare分配的DNS服務器地址增加到域名註冊商

   ![add cloudflare dns to your domain](/images/posts/cdn-tutorial/add-cloudflare-dns-to-your-domain.png)
   無需刪除原來的DNS服務器，你的域名可以擁有多個DNS服務器。
3. Cloudflare會自動檢測你的域名是否添加到了指定的DNS服務器，成功後會發送郵件。如果你要手動檢測，一個小時最多隻能檢測一次。

4. 添加成功後，在此處添加DNS記錄

   ![where to add dns](/images/posts/cdn-tutorial/add-dns-record-1.png)
5. 如果你的靜態頁面部署在 Cloudflare，可以直接使用如下配置，將 `a.example.com` 指向 `mypage.pages.dev`

   ![add static page](/images/posts/cdn-tutorial/add-static-page.png)
6. 如果希望將 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一個自建的服務器
   
   增加這樣一條DNS記錄：

   ![add static page](/images/posts/cdn-tutorial/add-static-page.png)

7. 如果源站的端口不是80和443，比如是1080和1443

   增加一條Origin Rule，將80轉發到源站的1080

   ![add origin rule](/images/posts/cdn-tutorial/add-origin-rule.png)

   增加一條Origin Rule，將443轉發到源站的1443

   ![add origin rule2](/images/posts/cdn-tutorial/add-origin-rule443.png)
8. HTTPS 配置

   此處的HTTPS，主要是指CDN和源站之間的通訊。由Cloudflare生成證書，你的源站以該證書啓動服務。

   如果你的網站只是靜態頁面，比如個人博客，那麼不用配置HTTPS。

   在`SSL/TLS-Origin Server`中，生成源站的證書，並點擊下載，將該證書配置到你的服務器（略）。
   ![alt text](/images/posts/cdn-tutorial/create-origin-server-cert-cf.png)

   在`SSL/TLS-Overview`中，點擊`Configure`，選擇`Full(Strict)`即可。

   ![alt text](/images/posts/cdn-tutorial/configure-cf-ssl.png)

9. 好了，現在可以使用如下命令，測試 `a.example.com` 是否被 Cloudflare 代理了
   ```
   dig a.example.com
   ```

## 阿里雲
阿里雲的 CDN 服務，現在叫做`邊緣安全加速ESA`（202410）。並且如果你的源站端口不是默認的80和443端口，其中一個必要的步驟，是要提工單才能完成。

假設我們仍然希望將 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一個自建的服務器 `gateway.myserver.com`:

1. 點擊全站分發服務，域名管理，添加域名

   ![add-domain-aliyun](/images/posts/cdn-tutorial/add-domain-aliyun.png)

   這一步驟需要驗證 example.com 的域名所有權，按提示操作即可。

2. 在下面新增源站信息中，添加源站，選擇你的源站端口80或者443

   ![add-origin-aliyun](/images/posts/cdn-tutorial/add-origin-aliyun.png)

   注意，如果你的源站端口不是80或者443，此處先隨便選擇一個端口（阿里的界面操作邏輯是比較凌亂的）。

3. 開啓自定義端口

   在域名管理處，點擊`配置`，在如下兩處開啓兩個配置（自己選擇是HTTP還是HTTPS，不要直接看圖照搬）

   ![alt text](/images/posts/cdn-tutorial/customize-port-ali2.png)
   
   ![alt text](/images/posts/cdn-tutorial/customize-port-ali1.png)

   注意，阿里的HTTPS是按次收費的。

4. 此時在`基本配置`中，編輯源站信息，可以看到可以填寫自定義端口了
   ![alt text](/images/posts/cdn-tutorial/port-customized-aliyun.png)

   但是，經過測試，這沒什麼用，你還要點擊右上角的`工單`，給阿里雲的工程師提工單，等待工程師給你開通自定義端口的配置。

   我看到阿里雲的文檔說明，這是一個必要的步驟。我不明白爲什麼要有提工單才能完成的操作。

   所幸工程師們的響應比較快，如果你描述的清楚，半天也能搞定。

5. HTTPS 配置
   
   阿里雲支持lets encrypt的證書，當然看到下面的黃色警告，估計很快將不再支持了。

   ![alt text](/images/posts/cdn-tutorial/https-aliyun.png)

   在此處上傳即可。

6. 將你的域名 `a.example.com` 解析到阿里雲提供給你的CNAME地址比如 `all.example.com.w.cdngslb.com`

   該項配置可以在`基本配置`中查詢到。

7. 好了，現在可以使用如下命令，測試 `a.example.com` 是否被阿里雲代理了
   ```
   dig a.example.com
   ```

8. 如果你在訪問的時候，遇到503錯誤，請提工單。

## AWS
AWS的CDN服務叫做Cloudfront，Cloudfront也支持回源到自定義的端口，但是它[不支持訪問使用lets encrypt證書的源站](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-custom-origin.html#using-https-cloudfront-to-origin-certificate)。

假設我們仍然希望將 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一個自建的服務器 `gateway.myserver.com`:

1. 搜索`CloudFront`，在控制檯點擊 `Create Distribution`

   這個頁面的邏輯氛圍 `Origin` `Default cache behavior` `Viewer` `WAF` `Settings` 等區域，雖然多，但是比較清晰，你需要的所有配置都可以在這裏配置好。

   Origin domain填寫你的源站信息。

   Protocol 選擇的是Cloudfront和你的源站之間的通訊協議，分爲HTTP，HTTPS，和按照用戶的輸入URL來決定。

   如果你的源站不是80和443端口，在此處就可以直接填寫自定義端口（阿里雲學習一下）。

   ![alt text](/images/posts/cdn-tutorial/create-distribution-aws.png)

2. 一些配置

   使用默認配置即可。其中，WAF選擇關閉，否則是600美元/月的費用。

   在 `Alternate domain name` 中，填寫 `*.example.com`

   ![alt text](/images/posts/cdn-tutorial/add-domain-aws.png)

3. 用戶端 HTTPS 配置

   `Custom SSL certificate` 處點擊下拉框，選擇你的證書。

   可以在 ACM 服務上傳證書，或使用命令行上傳都行。但是切記，不要使用 lets encrypt 的證書。

   ![alt text](/images/posts/cdn-tutorial/upload-cert-aws.png)  
   
   或者點擊`Request certificate`，爲你的域名生成一個公共證書，證書一定要包含你要加速的域名，如 `example.com` `*.example.com`。

   ![alt text](/images/posts/cdn-tutorial/request-certificate-aws.png)

   如何生成證書（注意，這一步也要驗證域名的所有權，按提示操作即可）：

   ![alt text](/images/posts/cdn-tutorial/create-public-cert-aws.png)

4. Cloudfront和源站之間的HTTPS配置

   源站如果使用 lets encrpyt 證書，Cloudfront將會報錯。

   參考這個網站來配置Cloudfront和源站直接的HTTPS連接：

   https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-custom-origin.html#using-https-cloudfront-to-origin-certificate

5. 將你的域名 `a.example.com` 解析到 Cloudfront 分配給你的 `Distribution domain name` （如 aabbcc.cloudfront.net）即可

6. 好了，現在可以使用如下命令，測試 `a.example.com` 是否被阿里雲代理了
   ```
   dig a.example.com
   ```

## 搭配多個CDN以便全球加速
由於中國網絡技術的發達（😄），大部分網站如果想要在中國和全球都流暢訪問，需要在中國境內和中國之外分別配置CDN。

國內的域名服務商一般都支持在解析DNS的時候，配置境內和境外線路。

![alt text](/images/posts/cdn-tutorial/tencent-dns-china.png)

![alt text](/images/posts/cdn-tutorial/tencent-dns-abroad.png)

這樣，在國內用戶訪問的時候，將會走國內的CDN，在國外用戶訪問的時候，將會走國外的CDN。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/10/how-to-proxy-with-cdn/  

