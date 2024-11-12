# Aria2 Rpc 使用


本文将演示如何使用 aria2 rpc 开发一个下载模块，注意，这不是一个完整的应用，仅仅是为了给你一些启发。

| tech                         | version  |
|------------------------------|----------|
| electron                     | 30.0.6   |
| webpack                      | 5.91.0   |
| nodejs                       | v20.14.0 |
| aria2                        | 1.37.0   |
| React                        | 18.2.0   |
| react-use-websocket          | 4.8.1    |
| @mui/x-charts/SparkLineChart | 7.3.2    |

&gt; [aria2 文档](https://aria2.github.io/manual/en/html/aria2c.html)
&gt; [react-use-websocket 文档](https://www.npmjs.com/package/react-use-websocket)

### 成品演示

![](/images/posts/20240729-download-demo.jpg)

## 加载并启动 aria2

### 怎么将 aria2 集成到你的项目中

你可以要求你的用户自行安装 aria2c.exe，或者将 aria2c.exe 直接打包到你的项目中。

如果你采用后者，下面是一些示例。

假设你的工程目录是：

```text
src
build
  |-- aria2c.exe
package.json
```

#### 打包

下面是一个使用 Electron Builder 的示例，它将 build/aria2c.exe 拷贝到安装后的根目录。

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

将它放到 package.json 中。

#### 调用

我们希望开发和生产的时候，都能调用到这个 aria2c.exe.

```typescript
// prod
let downloadBin = path.join(path.dirname(process.execPath), &#39;aria2c.exe&#39;);
if (dev) {
  // dev
  downloadBin = path.join(process.cwd(), &#39;build&#39;, &#39;aria2c.exe&#39;);
}
```

### 如何启动

给它一些必要的参数，启动！

```typescript
function buildAargs(pid: number) {
  const mustOptions = [`--enable-rpc`, `--stop-with-process=${pid}`];
  // ... 比如支持从程序启动参数中自由添加配置，可以随意定制
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

aria2c默认会监听到 ws://127.0.0.1:6800/jsonrpc，实际上该端口不会和 http 端口冲突，所以你暂且可以不用做一些端口冲突的处理。

## 使用react-use-websocket连接

以下是一个简单的示例，最终使用的示例在下一小节，你应该先使用这个小示例，确保能够从 rpc server 中读取到消息。

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

假设你在另一个模块中，使用这个 hook：

```typescript jsx
const { sendJsonMessage, readyState } = useMyAria({
  onMessage(e: MessageEvent&lt;any&gt;) {
    if (!e.data) {
      return;
    }
    console.log(e.data);
  }
});

// 显示当前的活跃下载数量
const onButtonClick = (e) =&gt; {
  sendJsonMessage([{
    jsonrpc: &#34;2.0&#34;,
    method: &#34;aria2.tellActive&#34;,
    id: &#34;rpc_timer_tell_active&#34;
  }]);
};

// 一次下载两个需要 cookie 认证的文件
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
    &lt;button onClick={onButtonClick}&gt;显示当前活跃的下载信息&lt;/button&gt;
    &lt;button onClick={onDownloadClick}&gt;下载&lt;/button&gt;
  &lt;/div&gt;
);
```

jsonRpc 的格式非常奇怪，你必须适应。一般来说，它是这种格式：

```typescript
interface JsonRpcMessage {
  id: string,
  method: string,
  jsonrpc: string,
  params: [string[], { [key: string]: any }],
}
```

它的 params 数组，params[0] 是一系列下载地址，params[1] 是一个 object，里面有很多的配置项。例如，你可以配置下载的文件夹，分割多少份下载，还有 cookie。

你 **必须** 在代码中组装上面这样的 json 字符串，然后发送给 aria2c。

&gt; [aria2 rpc](https://aria2.github.io/manual/en/html/aria2c.html#rpc-interface)
&gt; [react-use-websocket 文档](https://www.npmjs.com/package/react-use-websocket)

## 定时轮询与监听

一般人到这已经放弃了。

如果你希望显示实时的下载进度，你可以主动向 aria 发起 rpc 调用，然后接收它的结果，再次声明，这个代码无法直接运行，只是为了给你一些启发。

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
    memStore.addAllAriaStats(totalResults);// 这里就是所有的结果，你可以在另外一个模块中消费它
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

这些通知不能告诉你下载的进度，但是当下载达到重要节点的时候，会通知你。你可以利用它们做一些事情，例如下载失败，重新下载等。

### 磁力链，直链与种子文件

磁力链，种子和直链下载的处理方式是不一样的。

磁力链和种子任务，下载完成后，会做种。你查询回的结果，虽然下载进度到了 100%，但是 status 仍然不是 complete。

所以你应该根据下载的大小，是否 active 等结合起来判断一个任务是否已经完成。

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
      || stat.verifyIntegrityPending === &#34;true&#34;; // 如果开启了文件校验，应该等待文件校验完毕再判定下载成功
  });

  if (!isPending) {
    // 为了 debug 用
    nconsole.log(&#34;task finished&#34;);
  }

  return isPending;
};
```

当然你也可以依赖 onDownloadComplete 这些事件，但是它们只发送一次，因此并不可靠。

### 下载完成校验文件

| option            | description                       |
|-------------------|-----------------------------------|
| checksum          | 开启文件校验，如 checksum=sha-1=123123... |
| --check-integrity | 校验失败是否重新下载，checksum不开启没有作用        |

当开启这些参数，下载完成后，会进行文件校验，并且查询状态的结果中，该任务会有 verifyIntegrityPending=true 属性。

可以借此来判断下载任务的进度。

## 完成

到此，你已经会通过 electron 发送 json rpc 消息，和 aria2c 交互，创建新的下载，查询下载状态，并监控下载完成。

抱歉我不能将完整的代码发出来，但是如我再三强调，每个人的项目都不一样，而且实现这些目标的方法有很多种。本文旨在给你启发。

如果你希望获得帮助，可以留下你的评论。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/07/aria2-rpc/  

