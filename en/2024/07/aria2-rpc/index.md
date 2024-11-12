# Aria2 Rpc


This article will demonstrate how to develop a download module using aria2 rpc, 
note that code provided in this article is not a complete application, just to give you some inspiration.

| tech                         | version  |
|------------------------------|----------|
| electron                     | 30.0.6   |
| webpack                      | 5.91.0   |
| nodejs                       | v20.14.0 |
| aria2                        | 1.37.0   |
| React                        | 18.2.0   |
| react-use-websocket          | 4.8.1    |
| @mui/x-charts/SparkLineChart | 7.3.2    |

&gt; [aria2 docs](https://aria2.github.io/manual/en/html/aria2c.html)
&gt; [react-use-websocket docs](https://www.npmjs.com/package/react-use-websocket)

### Demonstration of the finished project

![](/images/posts/20240729-download-demo.jpg)

## Package and start aria2

### How to integrate aria2 into your project

You can either ask your users to install aria2c.exe by themselves, or package aria2c.exe directly into your project.

If you choose the latter one, here are some examples.

Suppose your project directory is:

```text
src
build
  |-- aria2c.exe
package.json
```

#### Packaging

The following is an example of using Electron Builder, which copies build/aria2c.exe into the root directory of the installation.

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

Put this code snippet in package.json.

#### Calling

We want this aria2c.exe to be ready for both development and production.

```typescript
// prod
let downloadBin = path.join(path.dirname(process.execPath), &#39;aria2c.exe&#39;);
if (dev) {
  // dev
  downloadBin = path.join(process.cwd(), &#39;build&#39;, &#39;aria2c.exe&#39;);
}
```

### How to start it
Just give it some necessary args, and start it.

```typescript
function buildAargs(pid: number) {
  const mustOptions = [`--enable-rpc`, `--stop-with-process=${pid}`];
  // ... you can write some code to enable configurations from the program startup parameters
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

aria2c listens to ws://127.0.0.1:6800/jsonrpc by default, 
which actually doesn&#39;t conflict with the http port, 
so you don&#39;t have to do anything about port conflicts for now.

## Use react-use-websocket

The following is a simple example, you should use this small example first to make sure you can read the message from the rpc server.

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

Here is another module that uses this hook:

```typescript jsx
const { sendJsonMessage, readyState } = useMyAria({
  onMessage(e: MessageEvent&lt;any&gt;) {
    if (!e.data) {
      return;
    }
    console.log(e.data);
  }
});

// Show current active download tasks
const onButtonClick = (e) =&gt; {
  sendJsonMessage([{
    jsonrpc: &#34;2.0&#34;,
    method: &#34;aria2.tellActive&#34;,
    id: &#34;rpc_timer_tell_active&#34;
  }]);
};

// Download two files at once that require cookie authentication
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

The format of json rpc is very odd, you have to get used to it. In general, here is what it looks like:

```typescript
interface JsonRpcMessage {
  id: string,
  method: string,
  jsonrpc: string,
  params: [string[], { [key: string]: any }],
}
```

The first element of params, which is params[0], is a list of download addresses, and params[1] is an object with a lot of configuration items in it. 

For example, you can configure which folder to download to, how many downloads to split, and headers and cookies.

You **must** assemble a json string like the one above in code and send it to aria2c.

&gt; [aria2 rpc](https://aria2.github.io/manual/en/html/aria2c.html#rpc-interface)
&gt; [react-use-websocket 文档](https://www.npmjs.com/package/react-use-websocket)

## Status Monitoring

A lot of people might have given up by this point.

If you want to show real-time download progress, you can initiate a rpc call to aria and receive its results, again, this code won&#39;t run directly, just to give you some ideas.

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

And aria supports some rpc event notifications:

```typescript
type ariaEvent =
  &#34;aria2.onDownloadStart&#34;
  | &#34;aria2.onDownloadPause&#34;
  | &#34;aria2.onDownloadStop&#34;
  | &#34;aria2.onDownloadComplete&#34;
  | &#34;aria2.onDownloadError&#34;
  | &#34;aria2.onBtDownloadComplete&#34;;
```

These events won&#39;t tell you the progress of the downloads, 

but they will notify you when the download reaches an important status. 

You can use them for scenarios like failed downloads, re-downloads, etc.

### Magnet, direct links and torrent download

Magnet, direct links and torrent downloads are handled differently.

Magnet and torrent tasks, after the downloads are complete, will do the seeding. 

The result of your query, although the download progress has reached 100%, the status is still not complete.

So you should judge whether a task is completed based on the size of the download, whether it is active, etc. 

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

Of course, you can also rely on onDownloadComplete events, but they are not stable because they send notifications only once.

### File validation after download

| option            | description                                                                     |
|-------------------|---------------------------------------------------------------------------------|
| checksum          | enable file validation, like checksum=sha-1=123123...                           |
| --check-integrity | whether re-download if failed to checksum, won&#39;t work if checksum not specified |

When both parameters are turned on, files are verified when the download completes and the task will have the verifyIntegrityPending=true attribute in the results of the query status.

This can be used to determine the progress of the download task.

## Done

By this point you will have sent json rpc messages via electron, interacted with aria2c, created new downloads, and queried the status of downloads.

I&#39;m sorry I can&#39;t post the full code, but as I&#39;ve emphasized again and again, everyone&#39;s project is different and there are many ways to achieve these goals. This article is meant to inspire you.

You can leave your comments if you would like help.

---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/aria2-rpc/  

