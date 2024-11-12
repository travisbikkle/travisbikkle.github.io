# Electron Linux 風格標題欄


本文將演示如何在 Electron 中快速實現一個標題欄，可以捲起，拖動，並帶你進一步瞭解 Electron 的 bug。

## 隱藏默認標題欄

在你創建 BrowserWindow 的方法中，指定以下參數：

```typescript jsx
function createWindow() {
  if (prod) Menu.setApplicationMenu(null);
  mainWindow = new BrowserWindow({
    titleBarStyle: dev ? &#39;default&#39; : &#39;hiddenInset&#39;,
    titleBarOverlay: true,
    frame: false,
    // ...
```

## 編寫自己的標題欄
現在默認的標題欄已經消失了，你應該編寫一個 div，作爲自己的標題欄。這個 div 和其它 div 沒什麼兩樣，除了它要支持以下三種東西：

1. 可以拖動
2. 可以捲起
3. 有一個紅綠燈組件（捲起/放下，最大化，最小化，關閉）

```typescript jsx
 &lt;div className=&#34;title-bar&#34;&gt;
  &lt;div className=&#34;logo-and-name&#34;&gt;&lt;img src=&#34;public/assets/icon.ico&#34; alt=&#34;logo&#34; /&gt;My Application&lt;/div&gt;
  &lt;div className=&#34;traffic-light&#34;&gt;
    &lt;MinMaxClose /&gt; &lt;!-- MinMaxClose 你可以自己實現，四個按鈕，捲起/放下，最大化，最小化，關閉 --&gt; 
  &lt;/div&gt;
&lt;/div&gt;
```

## 拖動

### 編寫一個 Drag 組件，它的所有 children 都可以拖動。

```typescript jsx
import * as React from &#34;react&#34;;
import { HTMLAttributes } from &#34;react&#34;;
import nconsole from &#34;_rutils/nconsole&#34;;

interface DragProps extends HTMLAttributes&lt;HTMLDivElement&gt; {
  children: React.ReactNode;
}

function Drag(props: DragProps) {
  const { children, ...rest } = props;
  const stopMove = () =&gt; {
    window.ipcAPI?.initMoveWindow(false);
  };

  const startMove = (e: React.SyntheticEvent&lt;HTMLDivElement&gt;) =&gt; {
    let elementDraggable = true;

    if (e.target instanceof HTMLInputElement // 輸入框，按鈕等不能拖動，可以自由添加不希望拖動的組件
      || e.target instanceof HTMLButtonElement
      || e.target instanceof HTMLTextAreaElement
    ) {
      elementDraggable = false;
    }

    if (elementDraggable) {
      window.ipcAPI?.initMoveWindow(true);
      window.ipcAPI?.moveWindow();
    }
  };

  return (
    &lt;div
      {...rest}
      onMouseDown={(e) =&gt; startMove(e)}
      onMouseUp={(e) =&gt; stopMove()}
    &gt;
      { children }
    &lt;/div&gt;
  );
}

export default Drag;
```

這個`window.ipcAPI?.initMoveWindow(true);` `window.ipcAPI?.moveWindow();` 是什麼呢？

```typescript jsx
function initMoveWindow(moveState: boolean) {
  ipcRenderer.send(&#39;window-move-init&#39;, moveState);
}

function moveWindow() {
  ipcRenderer.send(&#39;window-move&#39;);
}
```

### 使用 ipcMain 來控制窗口移動

來看看 `window-move-init` and `window-move` 做了什麼。

1. `window-move-init` 會在你點擊標題欄的瞬間調用，它告訴 electron：“準備移動！”
2. `window-move` 會在你鼠標移動的每一幀調用

把以下代碼放到你的 main.ts 或者其它能夠調用 ipcMain.on 的位置。

```typescript jsx

let winStartPosition = { x: 0, y: 0 };
let mouseStartPosition = { x: 0, y: 0 };
let size = [0, 0];
let ready2move = false;
let movingInterval: string | number | NodeJS.Timeout | null | undefined;

ipcMain.on(&#34;window-move-init&#34;, (e, moveState: boolean) =&gt; {
  if (moveState) {
    const winPosition = win.getPosition();
    winStartPosition = { x: winPosition[0], y: winPosition[1] };
    mouseStartPosition = screen.getCursorScreenPoint();
    size = win.getSize();
  } else {
    if (movingInterval) clearInterval(movingInterval);
    movingInterval = null;
  }
  ready2move = moveState;
});

ipcMain.on(&#34;window-move&#34;, (e) =&gt; {
  if (ready2move) {
    if (movingInterval) {
      clearInterval(movingInterval);
    }
    // 使用 setInterval 是爲了解決鼠標移動太快離開窗口，導致 mouseMove 事件丟失的問題
    movingInterval = setInterval(() =&gt; {
      // 實時更新位置
      const cursorPosition = screen.getCursorScreenPoint();
      const x = winStartPosition.x &#43; cursorPosition.x - mouseStartPosition.x;
      const y = winStartPosition.y &#43; cursorPosition.y - mouseStartPosition.y;
      // 你必須用 setBounds，而不能用 setPosition，否則窗口會慢慢變大，這就是我說的 electron 的 bug，至今不修復
      win.setBounds({ // win 就是你的 mainWindow，本示例中沒有體現
        x,
        y,
        width: size[0],
        height: size[1],
      });
    }, 1); // 1ms 並不會導致你的 app 性能變慢
  } else {
    if (movingInterval) clearInterval(movingInterval);
    movingInterval = null;
  }
});
```

最終爲了防止出現不可預料的問題，應該在點擊右鍵或者按 ESC 的時候，取消拖動
```typescript jsx
export function APP() {
  useEffect(() =&gt; {
    // 某些異常場合按 ESC 停止拖動
    const dragFallBack = (e: KeyboardEvent) =&gt; {
      if (e.key === &#34;Escape&#34;) {
        window.ipcAPI?.initMoveWindow(false);
      }
    };
    window.addEventListener(&#34;keydown&#34;, dragFallBack);
    return () =&gt; {
      window.removeEventListener(&#34;keydown&#34;, dragFallBack);
    };
  }, []);
  
  return (
     &lt;section style={{ height: &#34;100%&#34; }} onContextMenu={() =&gt; window.ipcAPI?.initMoveWindow(false)}&gt;
  ) 
}
```

好了，現在試試看拖動效果吧！

## 捲起

捲起比較簡單，和拖動同理，也是利用 ipcMain 來控制窗口。不同的是，捲起放下使用同一個按鈕，因此你應該記錄當前是捲起或者放下。

### renderer
```typescript jsx
&lt;div className=&#34;traffic-light&#34;&gt; &lt;!-- 在標題欄中增加捲起/放下，最大化/最小化，關閉的邏輯 --&gt;
  &lt;MinMaxClose
    scrollButtonStatus={trafficLightScrollButtonStatus}
    onScrollClick={() =&gt; {
      window.ipcAPI?.titleScrollToggle();
    }}
    onMinimize={() =&gt; {
      window.ipcAPI?.mainWindowControl(&#39;minimize&#39;);
    }}
    onMaximize={() =&gt; window.ipcAPI?.mainWindowControl(&#39;maximize&#39;)}
    onClose={() =&gt; alertAndClose()}
  /&gt;
&lt;/div&gt;
```

### ipcMain 的實現
```typescript jsx
let sizeForScroll = [0, 0];
let alreadyScrollUp = false;

ipcMain.handle(&#34;is-window-maximized&#34;, (e) =&gt; {
  return win.isMaximized();
});

ipcMain.handle(&#34;is-window-scrolled-up&#34;, (e) =&gt; {
  return win.getSize()[1] &lt;= 50;
});

const scroll = (method: string) =&gt; {
  if (method === &#34;up&#34;) {
    if (alreadyScrollUp) {
      return;
    }
    sizeForScroll = win.getSize();
    // logger.log(&#34;current size: &#34; &#43; sizeForScroll[0] &#43; &#34;, &#34; &#43; sizeForScroll[1]);
    win.setSize(sizeForScroll[0], titleBarHeight, true);
    alreadyScrollUp = true;
    return;
  }
  alreadyScrollUp = false;
  // 使用當前寬度，和捲起前的高度
  win.setSize(win.getSize()[0], sizeForScroll[1], true);
};

// 滾輪調用，會瞬間多次調用
ipcMain.on(&#34;title-scroll&#34;, (e, method: string) =&gt; {
  scroll(method);
});

// 按鈕調用
ipcMain.on(&#34;title-scroll-toggle&#34;, (e) =&gt; {
  if (alreadyScrollUp) {
    // 放下
    alreadyScrollUp = false;
    // 使用當前寬度，和捲起前的高度
    win.setSize(win.getSize()[0], sizeForScroll[1], true);
    return;
  }
  // 捲起
  sizeForScroll = win.getSize();
  // logger.log(&#34;current size: &#34; &#43; sizeForScroll[0] &#43; &#34;, &#34; &#43; sizeForScroll[1]);
  win.setSize(sizeForScroll[0], titleBarHeight, true);
  alreadyScrollUp = true;
});
```

## 完成
好了。以上並不是完整的代碼，但是它包含了一種解決問題的思路。到此，你應該可以實現封面圖中的效果了。





















---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/linux-like-titlebar/  

