# Aria2 Rpc 使用


本文將演示如何使用 aria2 rpc 開發一個下載模塊，注意，這不是一個完整的應用，僅僅是爲了給你一些啓發。

| tech                         | version  |
|------------------------------|----------|
| electron                     | 30.0.6   |
| webpack                      | 5.91.0   |
| nodejs                       | v20.14.0 |
| aria2                        | 1.37.0   |
| React                        | 18.2.0   |
| react-use-websocket          | 4.8.1    |
| @mui/x-charts/SparkLineChart | 7.3.2    |

&gt; [aria2 文檔](https://aria2.github.io/manual/en/html/aria2c.html)
&gt; [react-use-websocket 文檔](https://www.npmjs.com/package/react-use-websocket)

### 成品演示

![](/images/posts/20240729-download-demo.jpg)

## 加載並啓動 aria2

### 怎麼將 aria2 集成到你的項目中

你可以要求你的用戶自行安裝 aria2c.exe，或者將 aria2c.exe 直接打包到你的項目中。

如果你採用後者，下面是一些示例。

假設你的工程目錄是：

```text
src
build
  |-- aria2c.exe
package.json
```

#### 打包

下面是一個使用 Electron Builder 的示例，它將 build/aria2c.exe 拷貝到安裝後的根目錄。

```json
&#34;scripts&#34;: {
    &#34;dev&#34;: &#34;xxxxxxx&#34;
},
&#34;build&#34;: {
    &#34;extraFiles&#34;: [
        {
            &#34;from&#34;: &#34;build/aria2c.exe&#34;,
            &#34;to&#34;: &#34;&#34;
        },
}
```

將它放到 package.json 中。

#### 調用

我們希望開發和生產的時候，都能調用到這個 aria2c.exe.

```typescript
// prod
let downloadBin = path.join(path.dirname(process.execPath), &#39;aria2c.exe&#39;);
if (dev) {
  // dev
  downloadBin = path.join(process.cwd(), &#39;build&#39;, &#39;aria2c.exe&#39;);
}
```

### 如何啓動

給它一些必要的參數，啓動！

```typescript
function buildAargs(pid: number) {
  const mustOptions = [`--enable-rpc`, `--stop-with-process=${pid}`];
  // ... 比如支持從程序啓動參數中自由添加配置，可以隨意定製
  return [...mustOptions];
}

const start = async () =&gt; new Promise &lt; number | undefined &gt; ((resolve, reject) =&gt; {
  const mainPid = process.pid;
  const args = buildAargs(mainPid);

  if (!fs.existsSync(downloadBin)) {
    logger.error(downloadBin &#43; &#34; not exists.&#34;);
    reject(something);
    return;
  }

  logger.log(`aria2c.exe started`);
  const downloadClient = spawn(downloadBin, args);

  downloadClient.stdout.on(&#39;data&#39;, (data) =&gt; {
    const str = JSON.stringify(data);
    if (str.indexOf(&#34;RPC: listening on TCP port&#34;) &gt; 0) {
      resolve(downloadClient.pid);
      // ...
    }
  });

  downloadClient.stderr.on(&#39;data&#39;, (data) =&gt; {
    logger.error(`aria2c.exe error: ${data}`);
    // ...
  });

  downloadClient.on(&#39;close&#39;, (code) =&gt; {
    logger.log(`aria2c.exe: child process exited with code ${code || &#34;&#34;}`);
    // ...
  });
});
```

aria2c默認會監聽到 ws://127.0.0.1:6800/jsonrpc，實際上該端口不會和 http 端口衝突，所以你暫且可以不用做一些端口衝突的處理。

## 使用react-use-websocket連接

以下是一個簡單的示例，最終使用的示例在下一小節，你應該先使用這個小示例，確保能夠從 rpc server 中讀取到消息。

```typescript
import useWebSocket from &#34;react-use-websocket&#34;;
import { Options } from &#34;react-use-websocket/src/lib/types&#34;;

export function useMyAria(options?: Options) {
  const rpcServer = &#34;ws://127.0.0.1:6800/jsonrpc&#34;;
  // options was shared
  return useWebSocket&lt;Partial&lt;AriaResponse&gt;&gt;(rpcServer, {
    share: true,
    shouldReconnect: () =&gt; true,
    onOpen: () =&gt; {
    },
    onMessage: (e) =&gt; {
    },
    onError: (e) =&gt; {
      nconsole.log(&#34;rpc listener: error occurred&#34; &#43; JSON.stringify(e));
    },
    onClose: () =&gt; {
    },
    ...options,
  });
}
```

假設你在另一個模塊中，使用這個 hook：

```typescript jsx
const { sendJsonMessage, readyState } = useMyAria({
  onMessage(e: MessageEvent&lt;any&gt;) {
    if (!e.data) {
      return;
    }
    console.log(e.data);
  }
});

// 顯示當前的活躍下載數量
const onButtonClick = (e) =&gt; {
  sendJsonMessage([{
    jsonrpc: &#34;2.0&#34;,
    method: &#34;aria2.tellActive&#34;,
    id: &#34;rpc_timer_tell_active&#34;
  }]);
};

// 一次下載兩個需要 cookie 認證的文件
const onDownloadClick = (e) =&gt; {
  const jsonRpcMsg = [
    {
      &#34;id&#34;: &#34;e46bc4e56b7a6e33&#34;,
      &#34;jsonrpc&#34;: &#34;2.0&#34;,
      &#34;method&#34;: &#34;aria2.addUri&#34;,
      &#34;params&#34;: [
        [&#34;https://zzzz.xxxx.com/tttt/myfile.7z.003?fname=myfile.7z.003&#34;],
        {
          &#34;dir&#34;: &#34;D:\\games&#34;,
          &#34;gid&#34;: &#34;e46bc4e56b7a6e33&#34;,
          &#34;header&#34;: [
            &#34;Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7&#34;,
            &#34;Accept-Encoding: gzip, deflate, br, zstd&#34;,
            &#34;Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6,hu;q=0.5&#34;,
            &#34;Connection: keep-alive&#34;,
            &#34;Cookie: your cookie&#34;,
            &#34;Host: yyyy.xxxx.com&#34;,
            &#34;Referer: https://www.xxxx.com/&#34;,
            &#34;Sec-Ch-Ua: \&#34;Chromium\&#34;;v=\&#34;124\&#34;, \&#34;Microsoft Edge\&#34;;v=\&#34;124\&#34;, \&#34;Not-A.Brand\&#34;;v=\&#34;99\&#34;&#34;,
            &#34;Sec-Ch-Ua-Mobile: ?0&#34;,
            &#34;Sec-Ch-Ua-Platform: \&#34;Windows\&#34;&#34;,
            &#34;User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 Edg/124.0.0.0&#34;
          ],
          &#34;max-connection-per-server&#34;: 16,
          &#34;min-split-size&#34;: &#34;1M&#34;,
          &#34;split&#34;: 16
        }
      ]
    },
    {
      &#34;jsonrpc&#34;: &#34;2.0&#34;,
      &#34;method&#34;: &#34;aria2.addUri&#34;,
      &#34;params&#34;: [
        [
          &#34;https://zzzz.xxxx.com/tttt/myfile.7z.001?fname=myfile.7z.001\u0026from=30111\u0026version=3.3.3.3&#34;
        ],
        {
          &#34;dir&#34;: &#34;D:\\games&#34;,
          &#34;gid&#34;: &#34;fcfc5dd991923b96&#34;,
          &#34;header&#34;: [
            &#34;Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7&#34;,
            &#34;Accept-Encoding: gzip, deflate, br, zstd&#34;,
            &#34;Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6,hu;q=0.5&#34;,
            &#34;Connection: keep-alive&#34;,
            &#34;Cookie: your cookie&#34;,
            &#34;Host: yyyy.xxxx.com&#34;,
            &#34;Referer: https://www.xxxx.com/&#34;,
            &#34;Sec-Ch-Ua: \&#34;Chromium\&#34;;v=\&#34;124\&#34;, \&#34;Microsoft Edge\&#34;;v=\&#34;124\&#34;, \&#34;Not-A.Brand\&#34;;v=\&#34;99\&#34;&#34;,
            &#34;User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 Edg/124.0.0.0&#34;
          ],
          &#34;max-connection-per-server&#34;: 16,
          &#34;min-split-size&#34;: &#34;1M&#34;,
          &#34;split&#34;: 16
        }
      ],
      &#34;id&#34;: &#34;fcfc5dd991923b96&#34;
    }
  ];
  sendJsonMessage(jsonRpcMsg);
};

return (
  &lt;div&gt;
    &lt;button onClick={onButtonClick}&gt;顯示當前活躍的下載信息&lt;/button&gt;
    &lt;button onClick={onDownloadClick}&gt;下載&lt;/button&gt;
  &lt;/div&gt;
);
```

jsonRpc 的格式非常奇怪，你必須適應。一般來說，它是這種格式：

```typescript
interface JsonRpcMessage {
  id: string,
  method: string,
  jsonrpc: string,
  params: [string[], { [key: string]: any }],
}
```

它的 params 數組，params[0] 是一系列下載地址，params[1] 是一個 object，裏面有很多的配置項。例如，你可以配置下載的文件夾，分割多少份下載，還有 cookie。

你 **必須** 在代碼中組裝上面這樣的 json 字符串，然後發送給 aria2c。

&gt; [aria2 rpc](https://aria2.github.io/manual/en/html/aria2c.html#rpc-interface)
&gt; [react-use-websocket 文檔](https://www.npmjs.com/package/react-use-websocket)

## 定時輪詢與監聽

一般人到這已經放棄了。

如果你希望顯示實時的下載進度，你可以主動向 aria 發起 rpc 調用，然後接收它的結果，再次聲明，這個代碼無法直接運行，只是爲了給你一些啓發。

```typescript
interface AriaResponse {
  id: string;
  error: {
    code: number;
    message: string;
  };
  jsonrpc: string;
  method: string,
  params: any[],
  result: any,
}

interface AriaGidReport {
  gid: string;
  status: string;
  downloadSpeed: string;
  errorCode: string;
  errorMessage: string;
  completedLength: string;
  followedBy: string[];
  following: string;
  totalLength: string;
  verifyIntegrityPending: string;
  files: { path: string }[];
}

const { sendJsonMessage, readyState } = useMyAria({
  onMessage(e: MessageEvent&lt;any&gt;) {
    if (!e.data) {
      return;
    }
    const informs = JSON.parse(e.data) as AriaResponse[];
    if (!informs || informs.length === 0) {
      return;
    }
    let totalResults: AriaGidReport[] = [];
    for (let i = 0; i &lt; informs.length; i&#43;&#43;) {
      const msg = informs[i];
      if (msg.id === &#34;rpc_timer_tell_active&#34; || msg.id === &#34;rpc_timer_tell_stop&#34; || msg.id === &#34;rpc_timer_tell_wait&#34;) {
        totalResults = totalResults.concat(msg.result as AriaGidReport[]);
      }
    }
    if (totalResults.length === 0) {
      return;
    }
    memStore.addAllAriaStats(totalResults);// 這裏就是所有的結果，你可以在另外一個模塊中消費它
  },
});

useEffect(() =&gt; {
  const queryStatus = () =&gt; {
    // nconsole.log(&#34;rpc timer query status&#34;);
    sendJsonMessage([{
      jsonrpc: &#34;2.0&#34;,
      method: &#34;aria2.tellActive&#34;,
      id: &#39;rpc_timer_tell_active&#39;,
    },
      {
        jsonrpc: &#34;2.0&#34;,
        method: &#34;aria2.tellWaiting&#34;,
        id: &#39;rpc_timer_tell_wait&#39;,
        params: [0, 1000],
      },
      {
        jsonrpc: &#34;2.0&#34;,
        method: &#34;aria2.tellStopped&#34;,
        id: &#39;rpc_timer_tell_stop&#39;,
        params: [0, 1000],
      },
    ]);
  };

  const queryId = setInterval(queryStatus, queryAndHandleInterval);
  return () =&gt; {
    nconsole.debug(&#34;rpc timer query clearing interval id queryStatus&#34;);
    clearInterval(queryId);
  };
}, [readyState]);
```

而 aria 支持一些 rpc 的通知：

```typescript
type ariaEvent =
  &#34;aria2.onDownloadStart&#34;
  | &#34;aria2.onDownloadPause&#34;
  | &#34;aria2.onDownloadStop&#34;
  | &#34;aria2.onDownloadComplete&#34;
  | &#34;aria2.onDownloadError&#34;
  | &#34;aria2.onBtDownloadComplete&#34;;
```

這些通知不能告訴你下載的進度，但是當下載達到重要節點的時候，會通知你。你可以利用它們做一些事情，例如下載失敗，重新下載等。

### 磁力鏈，直鏈與種子文件

磁力鏈，種子和直鏈下載的處理方式是不一樣的。

磁力鏈和種子任務，下載完成後，會做種。你查詢回的結果，雖然下載進度到了 100%，但是 status 仍然不是 complete。

所以你應該根據下載的大小，是否 active 等結合起來判斷一個任務是否已經完成。

```typescript
const isTaskPending = (task: Task) =&gt; {
  // const gids = ... || [];
  if (gids.length === 1 &amp;&amp; task.type === &#34;magnet&#34;) return true;
  if (gids.length === 0) return true;
  const isPending = gids.some((x) =&gt; {
    const stat = memStore.status.taskBlink[x];
    if (stat) {
      nconsole.log(stat.files[0].path, stat.status, stat.verifyIntegrityPending ? &#34;verifying&#34; : &#34;&#34;, stat.completedLength, stat.totalLength);
    }
    return !stat
      || (Number(stat.totalLength) === 0)
      || (Number(stat.completedLength) &lt; Number(stat.totalLength))
      || stat.status === &#34;waiting&#34;
      || stat.status === &#34;active&#34;
      || stat.verifyIntegrityPending === &#34;true&#34;; // 如果開啓了文件校驗，應該等待文件校驗完畢再判定下載成功
  });

  if (!isPending) {
    // 爲了 debug 用
    nconsole.log(&#34;task finished&#34;);
  }

  return isPending;
};
```

當然你也可以依賴 onDownloadComplete 這些事件，但是它們只發送一次，因此並不可靠。

### 下載完成校驗文件

| option            | description                       |
|-------------------|-----------------------------------|
| checksum          | 開啓文件校驗，如 checksum=sha-1=123123... |
| --check-integrity | 校驗失敗是否重新下載，checksum不開啓沒有作用        |

當開啓這些參數，下載完成後，會進行文件校驗，並且查詢狀態的結果中，該任務會有 verifyIntegrityPending=true 屬性。

可以藉此來判斷下載任務的進度。

## 完成

到此，你已經會通過 electron 發送 json rpc 消息，和 aria2c 交互，創建新的下載，查詢下載狀態，並監控下載完成。

抱歉我不能將完整的代碼發出來，但是如我再三強調，每個人的項目都不一樣，而且實現這些目標的方法有很多種。本文旨在給你啓發。

如果你希望獲得幫助，可以留下你的評論。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/aria2-rpc/  

