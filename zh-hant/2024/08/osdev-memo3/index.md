# OS 開發備忘3


## 多任務
### Preemptive Multitasking 搶佔式多任務
核心思想是操作系統控制什麼時候切換任務。

![img.png](/images/posts/20240823-multitasks-switching.png)
第一行中，CPU 正在執行程序 A 的任務 A1。在第二行中，CPU 收到一個硬件中斷。如硬件中斷一文所述，
CPU 會立即停止執行任務 A1，並跳轉到中斷描述符表 (IDT) 中定義的中斷處理程序。通過該中斷處理程序，
操作系統現在可以再次控制 CPU，從而切換到任務 B1，而不是繼續執行任務 A1。

#### 保存狀態
由於任務會在任意時間點被中斷，它們可能正在進行某些計算。爲了能在稍後恢復任務，操作系統必須備份任務的整個狀態，
包括其調用堆棧（call stack）和所有 CPU 寄存器（cpu register）的值。這個過程稱爲上下文切換（context switch）。

由於調用堆棧可能非常龐大，操作系統通常會爲每個任務建立單獨的調用堆棧，而不是在每個任務切換時備份調用堆棧內容。
這種擁有自己堆棧的任務稱爲執行線程（thread of execution），簡稱線程（thread）。
通過爲每個任務使用單獨的堆棧，在上下文切換時只需保存寄存器內容（包括程序計數器和堆棧指針）。
這種方法最大限度地減少了上下文切換的性能開銷，這一點非常重要，因爲上下文切換通常每秒會發生 100 次。

#### 優缺點
優點：
1. 操作系統保證 cpu 分配時間公平

缺點：
1. 每個程序需要保存單獨的 stack，浪費內存
2. 操作系統需要爲每次切換保存 cpu register 狀態，即使任務只用了很少一部分 register

### Cooperative Multitasking 協作式多任務
核心思想是程序可以主動交出 cpu 控制權。

#### 保存狀態
由於任務自己定義暫停點，因此它們不需要操作系統來保存狀態。
相反，它們可以在自己暫停之前準確保存繼續運行所需的狀態，這通常會帶來更好的性能。
例如，Rust 的 async/await 實現會將所有仍需使用的局部變量存儲在自動生成的結構體中（見下文）。

通過在暫停前備份調用棧的相關部分，所有任務都可以共享一個調用棧，從而大大降低了每個任務的內存消耗。這樣就可以創建幾乎任意數量的任務，而不會耗盡內存。

#### 優缺點
優點：
1. 性能高

缺點：
1. 一些任務可能佔有全部資源，其它任務獲取不到 cpu 時間

### rust 中的 async/await

#### Future
Future 代表一個現在還不能用的值。

![img.png](/images/posts/20240823-future.png)
##### Future 讀取
```rust
// 一種方法，浪費cpu資源
let future = async_read_file(&#34;foo.txt&#34;);
let file_content = loop {
    match future.poll(…) {
        Poll::Ready(value) =&gt; break value,
        Poll::Pending =&gt; {}, // do nothing
    }
}
// 另一種方法，代碼難以維護
fn example(min_len: usize) -&gt; impl Future&lt;Output = String&gt; {
    async_read_file(&#34;foo.txt&#34;).then(move |content| {
        if content.len() &lt; min_len {
            Either::Left(async_read_file(&#34;bar.txt&#34;).map(|s| content &#43; &amp;s))
        } else {
            Either::Right(future::ready(content))
        }
    })
}
```
#### async/await
async/await 核心思想是可以同步代碼，由編譯器來轉換爲異步代碼。

以下代碼仍然是異步代碼：
```rust
async fn example(min_len: usize) -&gt; String {
    let content = async_read_file(&#34;foo.txt&#34;).await;
    if content.len() &lt; min_len {
        content &#43; &amp;async_read_file(&#34;bar.txt&#34;).await
    } else {
        content
    }
}
```
##### 狀態機
編譯器把 example 函數變成了一個狀態機，每個 await 都是一個狀態。狀態機實現 Future，每次 poll 的時候根據狀態不同，走不同的邏輯：

![img.png](/images/posts/20240823-async-state-machine.png)

##### 狀態保存
由於是狀態機，因此需要保存狀態，編譯器此時就可以派上用場了，它知道哪些變量有用到，哪些沒有。因此它可以生成一些臨時的 struct，如下：
```rust
// example 函數
async fn example(min_len: usize) -&gt; String {
    let content = async_read_file(&#34;foo.txt&#34;).await;
    if content.len() &lt; min_len {
        content &#43; &amp;async_read_file(&#34;bar.txt&#34;).await
    } else {
        content
    }
}

// 編譯器生成的狀態 struct:

struct StartState {
    min_len: usize,
}

struct WaitingOnFooTxtState {
    min_len: usize,
    foo_txt_future: impl Future&lt;Output = String&gt;,
}

struct WaitingOnBarTxtState {
    content: String,
    bar_txt_future: impl Future&lt;Output = String&gt;,
}

struct EndState {}
```

## Future 與 Pin
通過 async/await 創建的 future 實例通常是自引用的。用 Pin 將 Self 包裹起來，並讓編譯器生成狀態機的各種 struct 時，
略過實現 Unpin，這樣就保證了 future 實例不會在 poll 的時候被 move 走，也就是保證了所有的自引用都不會變成懸空引用。

絕大多數類型都不在意是否被移動，也就是自動實現了 UnPin 特徵。

而被結構體 Pin 包裹的值，會實現 !UnPin 特徵，也就是沒有實現 UnPin。

通過下方式可以實現 !UnPin，而一旦一個字段實現了 !UnPin，整個結構體就實現了 !UnPin.

```rust
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned, // 這是一個標記
}
```

```rust
fn poll(self: Pin&lt;&amp;mut Self&gt;, cx: &amp;mut Context) -&gt; Poll&lt;Self::Output&gt;
```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/08/osdev-memo3/  

