# Linux Style TitleBar


This article will show you how to quickly implement a title bar in Electron that can be rolled up, dragged around, 
and take you further through the **Electron bugs**.

## Hide the default title bar

Change properties in the main.ts(or main.js) where you create the BrowserWindow like below:

```typescript jsx
function createWindow() {
  if (prod) Menu.setApplicationMenu(null);
  mainWindow = new BrowserWindow({
    titleBarStyle: dev ? &#39;default&#39; : &#39;hiddenInset&#39;,
    titleBarOverlay: true,
    frame: false,
    // ...
```

## Make a title bar div
Now that the default title bar is gone, you should write a div that will serve as your own title bar. 
This div is no different from any other div, except that it will support three things:

1. the ability to drag
2. can be rolled up
3. have a traffic light component (roll up/down, maximize, minimize, close)

```typescript jsx
 &lt;div className=&#34;title-bar&#34;&gt;
  &lt;div className=&#34;logo-and-name&#34;&gt;&lt;img src=&#34;public/assets/icon.ico&#34; alt=&#34;logo&#34; /&gt;My Application&lt;/div&gt;
  &lt;div className=&#34;traffic-light&#34;&gt;
    &lt;MinMaxClose /&gt; &lt;!-- MinMaxClose You can implement it yourself, four buttons, roll up/down, maximize, minimize, close --&gt; 
  &lt;/div&gt;
&lt;/div&gt;
```

## Make it draggable

### Make a Drag component
A Drag component that has all its children draggable.

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

    if (e.target instanceof HTMLInputElement // input, buttons, etc. can not be dragged, you can freely add components that do not want to be dragged.
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

So what are `window.ipcAPI?.initMoveWindow(true);` `window.ipcAPI?.moveWindow();` ?

```typescript jsx
function initMoveWindow(moveState: boolean) {
  ipcRenderer.send(&#39;window-move-init&#39;, moveState);
}

function moveWindow() {
  ipcRenderer.send(&#39;window-move&#39;);
}
```

See it below.

### Using ipcMain to Control Window Movement

Let&#39;s take a look at the `window-move-init` and `window-move` code.

1. `window-move-init` invokes when you click on the title bar, it tells electron: &#34;Prepare to move!&#34;
2. `window-move` invokes when your mouse moves every frame

Put the following code into your main.ts or any other location where you can call ipcMain.on.

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
    // setInterval is used to solve the problem of mouseMove events being lost when the mouse moves out of the window too quickly.
    movingInterval = setInterval(() =&gt; {
      // real-time location updates
      const cursorPosition = screen.getCursorScreenPoint();
      const x = winStartPosition.x &#43; cursorPosition.x - mouseStartPosition.x;
      const y = winStartPosition.y &#43; cursorPosition.y - mouseStartPosition.y;
      // You have to use setBounds, not setPosition, otherwise the window will slowly get bigger, 
      // which is what I mean by electron&#39;s bug, which is still not fixed!
      win.setBounds({ // win is your mainWindow, which is not reflected in this example.
        x,
        y,
        width: size[0],
        height: size[1],
      });
    }, 1); // 1ms doesn&#39;t slow down your app&#39;s performance.
  } else {
    if (movingInterval) clearInterval(movingInterval);
    movingInterval = null;
  }
});
```

Eventually, to prevent unforeseen problems, you should cancel dragging when you right-click or press ESC

```typescript jsx
export function APP() {
  useEffect(() =&gt; {
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

Okay, now try the drag effect!

## Roll up

Rolling up is simpler and works the same way as dragging, also using ipcMain to control the window. 

The difference is that rollup and down use the same button, so you should keep track of whether you are currently rolled up or down.

### Renderer
```typescript jsx
&lt;div className=&#34;traffic-light&#34;&gt; &lt;!-- Add roll up/down, maximize/minimize, close logic to title bar --&gt;
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

### ipcMain code

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
  // Uses the current width, and height before roll-up
  win.setSize(win.getSize()[0], sizeForScroll[1], true);
};

// when mouse scrolls, which will be called multiple times instantly
ipcMain.on(&#34;title-scroll&#34;, (e, method: string) =&gt; {
  scroll(method);
});

// when button clicked
ipcMain.on(&#34;title-scroll-toggle&#34;, (e) =&gt; {
  if (alreadyScrollUp) {
    // scroll down
    alreadyScrollUp = false;
    // Uses the current width, and height before roll-up
    win.setSize(win.getSize()[0], sizeForScroll[1], true);
    return;
  }
  // scroll up
  sizeForScroll = win.getSize();
  // logger.log(&#34;current size: &#34; &#43; sizeForScroll[0] &#43; &#34;, &#34; &#43; sizeForScroll[1]);
  win.setSize(sizeForScroll[0], titleBarHeight, true);
  alreadyScrollUp = true;
});
```

## Done

OK. The above is not the complete code, but it contains an idea of how to solve the problem. 

At this point, you should be able to achieve the effect in the cover image.


---

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/linux-like-titlebar/  

