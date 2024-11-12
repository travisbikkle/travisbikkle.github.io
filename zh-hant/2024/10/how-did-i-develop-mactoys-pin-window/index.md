# SCWindow座標的一些轉換技巧


我們使用最多的是笛卡爾平面直角座標系，它以屏幕左下角爲原點，向上增加y座標，向右增加x座標。

SCWindow的座標則是以主屏幕左上角爲原點的翻轉座標，以下是一個簡單的對比圖：

![alt text](/images/posts/make-mactoys/flip-coordinate.png)

在實際場景，尤其是多屏，在兩種座標系之間轉換，仍然不是一件容易的事情。

假設有兩個顯示屏，如下圖所示排列：

![alt text](/images/posts/make-mactoys/two-screens-1.png)

黑色區域爲目標窗口，也是SCWindow的區域（如果你使用SCContentSharingPicker，那麼這塊區域就是它返回的SCContentFilter.contentRect），我們將要根據這個區域，計算出在平面直角座標系中的左上角座標。

說明：
1. 主屏（2560x1440），左下角原點(0,0)，副屏（1920x1080），左下角原點(2560,360)。
2. 主屏和副屏的排列方式爲，上邊緣對齊
3. 每個屏幕都會有25px的保留區域，可以認爲是系統頂部的狀態欄

你看，有很多的干擾因素。有狀態欄，有大小不一、排列隨時變化的顯示器，以及它們不一樣的原點。

你只有SCWindow.frame的值，你一開始並不知道SCWindow.frame.origin是翻轉的座標，Apple也不會在任何地方告訴你這一點，這就是蘋果軟件開發的日常。

好在經過不停地調試，最後我們能自己搞明白，SCWindow使用的是翻轉的座標。

所以上圖的答案是(2560,1440-25)。

### 那麼做一些變化呢

將副屏向下移動，使其底部邊緣和主屏對齊，原點變爲(2560,0)：

![alt text](/images/posts/make-mactoys/two-screens-2.png)

這次我們知道了，我們要求的結果，和副屏的大小無關，因此省略了無關的細節。

圖中兩個黑色窗口左上角所在平面直角座標系的點，如圖所示。

可見，這個結果和狀態欄的大小也無關。

所以結果就是(目標窗口的x座標, 主屏高度-目標窗口的y座標)，也就是(x, mainScreen.height-y)嗎？

### 將副屏移動到主屏上方驗證一下

![alt text](/images/posts/make-mactoys/two-screens-3.png)

在翻轉的座標系上方出現的點，y座標都變爲負數，並且越往上越小。

如圖所示，根據我們的公式，(x, mainScreen.height-y)，該點在平面直角座標系中也能正確顯示。


這樣看起來很簡單，但是在你排除掉這些干擾因素之前，這個計算就非常的容易出錯了。

以上是我在開發 MacToys 的屏幕置頂功能的一部分過程。

### MacToys 中的 Always On Top 置頂功能

![alt text](/images/posts/make-mactoys/macos-always-on-top.png)

你可以使用 `⌃ &#43; 左鍵` 單擊窗口置頂，也可以使用快捷鍵選擇。你可以使用同樣的`⌃ &#43; 左鍵`單擊已經置頂的窗口取消置頂，或者，右鍵單擊，在彈出菜單中取消置頂。

你還可以將這些快捷鍵改成你喜歡的選擇。

MacToys 中還有很多類似的功能，如置頂、取色、貼圖、右鍵菜單、狀態欄管理、ntfs等等功能，後面將會補充到十幾項功能。

我目前正打算將它上架。怎麼樣，這是你想要的功能嗎？

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/10/how-did-i-develop-mactoys-pin-window/  

