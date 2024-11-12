# Electron Linux 风格标题栏


本文将演示如何在 Electron 中快速实现一个标题栏，可以卷起，拖动，并带你进一步了解 Electron 的 bug。

## 隐藏默认标题栏

在你创建 BrowserWindow 的方法中，指定以下参数：

```typescript jsx
function createWindow() {
  if (prod) Menu.setApplicationMenu(null);
  mainWindow = new BrowserWindow({
    titleBarStyle: dev ? &#39;default&#39; : &#39;hiddenInset&#39;,
    titleBarOverlay: true,
    frame: false,
    // ...
```

## 编写自己的标题栏
现在默认的标题栏已经消失了，你应该编写一个 div，作为自己的标题栏。这个 div 和其它 div 没什么两样，除了它要支持以下三种东西：

1. 可以拖动
2. 可以卷起
3. 有一个红绿灯组件（卷起/放下，最大化，最小化，关闭）

```typescript jsx
 &lt;div className=&#34;title-bar&#34;&gt;
  &lt;div className=&#34;logo-and-name&#34;&gt;&lt;img src=&#34;public/assets/icon.ico&#34; alt=&#34;logo&#34; /&gt;My Application&lt;/div&gt;
  &lt;div className=&#34;traffic-light&#34;&gt;
    &lt;MinMaxClose /&gt; &lt;!-- MinMaxClose 你可以自己实现，四个按钮，卷起/放下，最大化，最小化，关闭 --&gt; 
  &lt;/div&gt;
&lt;/div&gt;
```

## 拖动

### 编写一个 Drag 组件，它的所有 children 都可以拖动。

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

    if (e.target instanceof HTMLInputElement // 输入框，按钮等不能拖动，可以自由添加不希望拖动的组件
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

这个`window.ipcAPI?.initMoveWindow(true);` `window.ipcAPI?.moveWindow();` 是什么呢？

```typescript jsx
function initMoveWindow(moveState: boolean) {
  ipcRenderer.send(&#39;window-move-init&#39;, moveState);
}

function moveWindow() {
  ipcRenderer.send(&#39;window-move&#39;);
}
```

### 使用 ipcMain 来控制窗口移动

来看看 `window-move-init` and `window-move` 做了什么。

1. `window-move-init` 会在你点击标题栏的瞬间调用，它告诉 electron：“准备移动！”
2. `window-move` 会在你鼠标移动的每一帧调用

把以下代码放到你的 main.ts 或者其它能够调用 ipcMain.on 的位置。

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
    // 使用 setInterval 是为了解决鼠标移动太快离开窗口，导致 mouseMove 事件丢失的问题
    movingInterval = setInterval(() =&gt; {
      // 实时更新位置
      const cursorPosition = screen.getCursorScreenPoint();
      const x = winStartPosition.x &#43; cursorPosition.x - mouseStartPosition.x;
      const y = winStartPosition.y &#43; cursorPosition.y - mouseStartPosition.y;
      // 你必须用 setBounds，而不能用 setPosition，否则窗口会慢慢变大，这就是我说的 electron 的 bug，至今不修复
      win.setBounds({ // win 就是你的 mainWindow，本示例中没有体现
        x,
        y,
        width: size[0],
        height: size[1],
      });
    }, 1); // 1ms 并不会导致你的 app 性能变慢
  } else {
    if (movingInterval) clearInterval(movingInterval);
    movingInterval = null;
  }
});
```

最终为了防止出现不可预料的问题，应该在点击右键或者按 ESC 的时候，取消拖动
```typescript jsx
export function APP() {
  useEffect(() =&gt; {
    // 某些异常场合按 ESC 停止拖动
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

好了，现在试试看拖动效果吧！

## 卷起

卷起比较简单，和拖动同理，也是利用 ipcMain 来控制窗口。不同的是，卷起放下使用同一个按钮，因此你应该记录当前是卷起或者放下。

### renderer
```typescript jsx
&lt;div className=&#34;traffic-light&#34;&gt; &lt;!-- 在标题栏中增加卷起/放下，最大化/最小化，关闭的逻辑 --&gt;
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

### ipcMain 的实现
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
  // 使用当前宽度，和卷起前的高度
  win.setSize(win.getSize()[0], sizeForScroll[1], true);
};

// 滚轮调用，会瞬间多次调用
ipcMain.on(&#34;title-scroll&#34;, (e, method: string) =&gt; {
  scroll(method);
});

// 按钮调用
ipcMain.on(&#34;title-scroll-toggle&#34;, (e) =&gt; {
  if (alreadyScrollUp) {
    // 放下
    alreadyScrollUp = false;
    // 使用当前宽度，和卷起前的高度
    win.setSize(win.getSize()[0], sizeForScroll[1], true);
    return;
  }
  // 卷起
  sizeForScroll = win.getSize();
  // logger.log(&#34;current size: &#34; &#43; sizeForScroll[0] &#43; &#34;, &#34; &#43; sizeForScroll[1]);
  win.setSize(sizeForScroll[0], titleBarHeight, true);
  alreadyScrollUp = true;
});
```

## 完成
好了。以上并不是完整的代码，但是它包含了一种解决问题的思路。到此，你应该可以实现封面图中的效果了。





















---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/07/linux-like-titlebar/  

