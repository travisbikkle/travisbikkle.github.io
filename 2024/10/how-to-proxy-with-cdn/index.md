# 使用 Cdn 低成本加速网站，或解决443和80端口被封问题


CDN是个好东西，我们一般需要使用CDN，来让我们的网站能够在全球都能快速访问，并且降低服务器压力，防止DDOS攻击等。

简单来说，你把你的域名的CNAME(如`a.example.com`) 指向CDN厂商的域名(如`abc.cloudfront.net`)。

用户访问`a.example.com`的时候，实际上访问的是`abc.cloudfront.net`。`abc.cloudfront.net`上面当然没有你的站点资源，所以会回头访问你配置的源站（这一步叫做`回源`）拉取资源，并存储到本地（这一步叫做`缓存`）。

你也可以在站点发布后，第一时间让CDN拉取站点资源，并下发到它位于全球的站点，防止用户在高峰期一窝蜂地访问过来，导致源站压力过大。这一步叫做`预热`。

用户访问CDN厂商的域名，CDN会分配个给用户最近的站点，这样用户访问的速度也快了。

CDN还支持指定回源的协议和端口，本文后面将会提及，这对使用家庭宽带部署服务并且443和80端口被封锁的用户非常有帮助。

虽然我测试的域名在腾讯云，但是我并没有使用腾讯云的cdn服务，因为在选择试用的时候，腾讯云提示我没有资格。

于是我只测试了cloudflare，aws，阿里云的cdn服务，并在此记录下来过程和配置，希望能够帮助有需要的人。

## Cloudflare
1. 右上角点 `Add site`，增加你的域名，按照推荐的配置即可

   ![add site](/images/posts/cdn-tutorial/add-site-cloudflare.png)
2. 将Cloudflare分配的DNS服务器地址增加到域名注册商

   ![add cloudflare dns to your domain](/images/posts/cdn-tutorial/add-cloudflare-dns-to-your-domain.png)
   无需删除原来的DNS服务器，你的域名可以拥有多个DNS服务器。
3. Cloudflare会自动检测你的域名是否添加到了指定的DNS服务器，成功后会发送邮件。如果你要手动检测，一个小时最多只能检测一次。

4. 添加成功后，在此处添加DNS记录

   ![where to add dns](/images/posts/cdn-tutorial/add-dns-record-1.png)
5. 如果你的静态页面部署在 Cloudflare，可以直接使用如下配置，将 `a.example.com` 指向 `mypage.pages.dev`

   ![add static page](/images/posts/cdn-tutorial/add-static-page.png)
6. 如果希望将 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一个自建的服务器
   
   增加这样一条DNS记录：

   ![add static page](/images/posts/cdn-tutorial/add-static-page.png)

7. 如果源站的端口不是80和443，比如是1080和1443

   增加一条Origin Rule，将80转发到源站的1080

   ![add origin rule](/images/posts/cdn-tutorial/add-origin-rule.png)

   增加一条Origin Rule，将443转发到源站的1443

   ![add origin rule2](/images/posts/cdn-tutorial/add-origin-rule443.png)
8. HTTPS 配置

   此处的HTTPS，主要是指CDN和源站之间的通讯。由Cloudflare生成证书，你的源站以该证书启动服务。

   如果你的网站只是静态页面，比如个人博客，那么不用配置HTTPS。

   在`SSL/TLS-Origin Server`中，生成源站的证书，并点击下载，将该证书配置到你的服务器（略）。
   ![alt text](/images/posts/cdn-tutorial/create-origin-server-cert-cf.png)

   在`SSL/TLS-Overview`中，点击`Configure`，选择`Full(Strict)`即可。

   ![alt text](/images/posts/cdn-tutorial/configure-cf-ssl.png)

9. 好了，现在可以使用如下命令，测试 `a.example.com` 是否被 Cloudflare 代理了
   ```
   dig a.example.com
   ```

## 阿里云
阿里云的 CDN 服务，现在叫做`边缘安全加速ESA`（202410）。并且如果你的源站端口不是默认的80和443端口，其中一个必要的步骤，是要提工单才能完成。

假设我们仍然希望将 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一个自建的服务器 `gateway.myserver.com`:

1. 点击全站分发服务，域名管理，添加域名

   ![add-domain-aliyun](/images/posts/cdn-tutorial/add-domain-aliyun.png)

   这一步骤需要验证 example.com 的域名所有权，按提示操作即可。

2. 在下面新增源站信息中，添加源站，选择你的源站端口80或者443

   ![add-origin-aliyun](/images/posts/cdn-tutorial/add-origin-aliyun.png)

   注意，如果你的源站端口不是80或者443，此处先随便选择一个端口（阿里的界面操作逻辑是比较凌乱的）。

3. 开启自定义端口

   在域名管理处，点击`配置`，在如下两处开启两个配置（自己选择是HTTP还是HTTPS，不要直接看图照搬）

   ![alt text](/images/posts/cdn-tutorial/customize-port-ali2.png)
   
   ![alt text](/images/posts/cdn-tutorial/customize-port-ali1.png)

   注意，阿里的HTTPS是按次收费的。

4. 此时在`基本配置`中，编辑源站信息，可以看到可以填写自定义端口了
   ![alt text](/images/posts/cdn-tutorial/port-customized-aliyun.png)

   但是，经过测试，这没什么用，你还要点击右上角的`工单`，给阿里云的工程师提工单，等待工程师给你开通自定义端口的配置。

   我看到阿里云的文档说明，这是一个必要的步骤。我不明白为什么要有提工单才能完成的操作。

   所幸工程师们的响应比较快，如果你描述的清楚，半天也能搞定。

5. HTTPS 配置
   
   阿里云支持lets encrypt的证书，当然看到下面的黄色警告，估计很快将不再支持了。

   ![alt text](/images/posts/cdn-tutorial/https-aliyun.png)

   在此处上传即可。

6. 将你的域名 `a.example.com` 解析到阿里云提供给你的CNAME地址比如 `all.example.com.w.cdngslb.com`

   该项配置可以在`基本配置`中查询到。

7. 好了，现在可以使用如下命令，测试 `a.example.com` 是否被阿里云代理了
   ```
   dig a.example.com
   ```

8. 如果你在访问的时候，遇到503错误，请提工单。

## AWS
AWS的CDN服务叫做Cloudfront，Cloudfront也支持回源到自定义的端口，但是它[不支持访问使用lets encrypt证书的源站](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-custom-origin.html#using-https-cloudfront-to-origin-certificate)。

假设我们仍然希望将 `*.example.com`，比如 `a.example.com` `b.example.com` 指向一个自建的服务器 `gateway.myserver.com`:

1. 搜索`CloudFront`，在控制台点击 `Create Distribution`

   这个页面的逻辑氛围 `Origin` `Default cache behavior` `Viewer` `WAF` `Settings` 等区域，虽然多，但是比较清晰，你需要的所有配置都可以在这里配置好。

   Origin domain填写你的源站信息。

   Protocol 选择的是Cloudfront和你的源站之间的通讯协议，分为HTTP，HTTPS，和按照用户的输入URL来决定。

   如果你的源站不是80和443端口，在此处就可以直接填写自定义端口（阿里云学习一下）。

   ![alt text](/images/posts/cdn-tutorial/create-distribution-aws.png)

2. 一些配置

   使用默认配置即可。其中，WAF选择关闭，否则是600美元/月的费用。

   在 `Alternate domain name` 中，填写 `*.example.com`

   ![alt text](/images/posts/cdn-tutorial/add-domain-aws.png)

3. 用户端 HTTPS 配置

   `Custom SSL certificate` 处点击下拉框，选择你的证书。

   可以在 ACM 服务上传证书，或使用命令行上传都行。但是切记，不要使用 lets encrypt 的证书。

   ![alt text](/images/posts/cdn-tutorial/upload-cert-aws.png)  
   
   或者点击`Request certificate`，为你的域名生成一个公共证书，证书一定要包含你要加速的域名，如 `example.com` `*.example.com`。

   ![alt text](/images/posts/cdn-tutorial/request-certificate-aws.png)

   如何生成证书（注意，这一步也要验证域名的所有权，按提示操作即可）：

   ![alt text](/images/posts/cdn-tutorial/create-public-cert-aws.png)

4. Cloudfront和源站之间的HTTPS配置

   源站如果使用 lets encrpyt 证书，Cloudfront将会报错。

   参考这个网站来配置Cloudfront和源站直接的HTTPS连接：

   https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-custom-origin.html#using-https-cloudfront-to-origin-certificate

5. 将你的域名 `a.example.com` 解析到 Cloudfront 分配给你的 `Distribution domain name` （如 aabbcc.cloudfront.net）即可

6. 好了，现在可以使用如下命令，测试 `a.example.com` 是否被阿里云代理了
   ```
   dig a.example.com
   ```

## 搭配多个CDN以便全球加速
由于中国网络技术的发达（😄），大部分网站如果想要在中国和全球都流畅访问，需要在中国境内和中国之外分别配置CDN。

国内的域名服务商一般都支持在解析DNS的时候，配置境内和境外线路。

![alt text](/images/posts/cdn-tutorial/tencent-dns-china.png)

![alt text](/images/posts/cdn-tutorial/tencent-dns-abroad.png)

这样，在国内用户访问的时候，将会走国内的CDN，在国外用户访问的时候，将会走国外的CDN。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/10/how-to-proxy-with-cdn/  

