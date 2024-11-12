# OS 開發備忘


## 顯示（VGA TEXT MODE）

* 要在 VGA 文本模式下將字符打印到屏幕上，必須將其寫入 VGA 硬件的文本緩衝區。
* VGA 文本緩衝區是一個二維數組，通常有 25 行 80 列，可直接渲染到屏幕上。
* 每個數組條目通過以下格式描述一個屏幕字符：

| Bit(s) | Value         |
|--------|---------------|
| 0-7    | ASCII 字符，8bit |
| 8-11   | 前景色，4bit      |
| 12-14  | 背景色，3bit      |
| 15     | 閃爍，1bit       |

### 如何設計（僞代碼）：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    // 藍綠
    Cyan = 3,
    Red = 4,
    // 洋紅
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    ...
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
/// 表示顏色代碼
struct ColorCode(u8);
impl ColorCode {
    fn new(foreground: Color, background: Color) -&gt; ColorCode {
        ColorCode((background as u8) &lt;&lt; 4 | (foreground as u8))
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
/// 表示寫入 buffer 中的字符，由字符和顏色構成
struct ScreenChar {
    // 第一字節
    ascii_character: u8,
    // 第二字節
    color_code: ColorCode,
}

/// vga text buffer 的數組大小
const BUFFER_HEIGHT: usize = 25;
const BUFFER_WIDTH: usize = 80;

#[repr(transparent)]
struct Buffer {
    chars: [[Volatile&lt;ScreenChar&gt;; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

## CPU 異常

### IDT Interrupt Descriptor Table

* 爲了捕捉和處理異常，我們必須建立一個所謂的中斷描述符表（IDT）。
* 在該表中，我們可以爲每個 CPU 異常指定一個處理函數（handler function）。
* 硬件會直接使用該表，因此我們需要遵循預定義的格式。
* 每個條目必須具有以下 16 字節結構（一個u16需要2字節，下面的表格共有16字節）：

  | 類型  | 名稱                       | 描述                      |
              |-----|--------------------------|-------------------------|
  | u16 | Function Pointer [0:15]  | handler function 指針低位.  |
  | u16 | GDT selector             | GDT（見下文） 代碼片段的選擇器.      |
  | u16 | Options                  | (見下文)                   |
  | u16 | Function Pointer [16:31] | handler function 指針中位.  |
  | u32 | Function Pointer [32:63] | handler function 指針其餘位. |
  | u32 | 保留                       |                         |

Options（16bit） 遵從如下格式：

| 位     | 名稱                               | 描述                                                                                                              |
|-------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 0-2   | Interrupt Stack Table Index      | 0: Don’t switch stacks, 1-7: Switch to the n-th stack in the Interrupt Stack Table when this handler is called. |
| 3-7   | 保留                               |                                                                                                                 |
| 8     | 0: Interrupt Gate, 1: Trap Gate  | 0 代表不中斷                                                                                                         |
| 9-11  | 必須是1                             |                                                                                                                 |
| 12    | 必須是0                             |                                                                                                                 |
| 13‑14 | Descriptor Privilege Level (DPL) | 調用此函數需要的最低權限等級.                                                                                                 |
| 15    | Present                          |                                                                                                                 |

CPU異常詳見[此處](https://wiki.osdev.org/Exceptions)。

### CPU 異常處理的大致步驟
1. 將一些寄存器推入堆棧，包括指令指針和 [RFLAGS 寄存器](https://en.wikipedia.org/wiki/FLAGS_register)。(我們稍後將使用這些值）。
2. 從中斷描述符表（IDT）中讀取相應的條目。例如，當發生頁面故障時，CPU 會讀取第 14 個條目。
3. 檢查條目是否存在，如果不存在，則引發雙重故障(double fault)。
4. 如果條目是中斷門（未設置第 40 位，也就是 Options 表第 8 位），則禁用硬件中斷。
5. 將指定的 GDT 選擇器載入 CS（代碼段）。
6. 跳轉到指定的處理函數。

### 如何設計（直接使用現有的 crate）：
[x86_64 IDT](https://docs.rs/x86_64/0.14.2/x86_64/structures/idt/struct.InterruptDescriptorTable.html)

### 斷點異常 (breakpoint exception)
* 當用戶設置斷點時，調試器會用 int3 指令覆蓋相應的指令，這樣 CPU 在運行到該行時就會拋出斷點異常。
* 當用戶想繼續運行程序時，調試器會再次用原來的指令替換 int3 指令，然後繼續運行程序。

使用 x86-interrupt 實現的一個斷點異常處理函數示例： 

```rust
use crate::println;
use x86_64::structures::idt::{InterruptDescriptorTable, InterruptStackFrame};

use lazy_static::lazy_static;

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        idt
    };
}

pub fn init_idt() {
    IDT.load();
}

extern &#34;x86-interrupt&#34; fn breakpoint_handler(stack_frame: InterruptStackFrame) {
    println!(&#34;EXCEPTION: BREAKPOINT\n{:#?}&#34;, stack_frame);
}

#[test_case]
fn test_breakpoint_exception() {
    // 觸發一次斷點異常
    x86_64::instructions::interrupts::int3();
}
```

### 雙重異常（double fault）
雙重故障是 CPU 無法調用異常處理程序時出現的一種特殊異常。

例如，當頁面故障被觸發，但中斷描述符表（IDT）中沒有註冊頁面故障處理程序時，就會出現這種情況。

因此，它有點類似於有異常的編程語言中的 catch-all 塊，如 C&#43;&#43; 中的 catch(...) 或 Java 或 C# 中的 catch(Exception e)。

雙重故障的行爲與普通異常類似。我們可以在 IDT 中爲它定義一個普通的處理函數。

提供雙重故障處理程序非常重要，因爲如果雙重故障未得到處理，就會發生致命的三重故障。三重故障無法被捕獲，大多數硬件都會做出系統復位的反應。

## 硬件中斷(Hardware Interrupts)
中斷提供了一種從硬件設備通知 CPU 的方法。

因此，與其讓內核定期檢查鍵盤是否鍵入新字符（這一過程稱爲輪詢），不如讓鍵盤在每次按鍵時通知內核。

這樣做效率更高，因爲內核只需在發生事情時採取行動。由於內核可以立即做出反應，而不是在下一次輪詢時才做出反應，因此反應時間也更快。

將所有硬件設備直接連接到 CPU 是不可能的。相反，一個獨立的中斷控制器會彙總來自所有設備的中斷，然後通知 CPU：

```text
                                    ____________             _____
               Timer ------------&gt; |            |           |     |
               Keyboard ---------&gt; | Interrupt  |---------&gt; | CPU |
               Other Hardware ---&gt; | Controller |           |_____|
               Etc. -------------&gt; |____________|
```

### 8259 可編程中斷控制器 (programmable interrupt controller (PIC))
英特爾 8259 是 1976 年推出的可編程中斷控制器 (PIC)。
它早已被較新的 APIC 所取代，但出於向後兼容的考慮，其接口在當前系統中仍受支持。

8259 有八條中斷線路和幾條與 CPU 通信的線路。以前的典型系統配置有兩個 8259 PIC 實例，一個主 PIC，一個輔助 PIC，分別與主 PIC 的一條中斷線相連：

```text
                       ____________                              ____________
1. Real Time Clock --&gt; |            |   1. Timer -------------&gt; |            |
2. ACPI -------------&gt; |            |   2. Keyboard-----------&gt; |            |      _____
3. Available --------&gt; | Secondary  |   ----------------------&gt; | Primary    |     |     |
4. Available --------&gt; | Interrupt  |   4. Serial Port 2 -----&gt; | Interrupt  |---&gt; | CPU |
5. Mouse ------------&gt; | Controller |   5. Serial Port 1 -----&gt; | Controller |     |_____|
6. Co-Processor -----&gt; |            |   6. Parallel Port 2/3 -&gt; |            |
7. Primary ATA ------&gt; |            |   7. Floppy disk -------&gt; |            |
8. Secondary ATA ----&gt; |____________|   8. Parallel Port 1----&gt; |____________|
```
每個控制器可通過兩個 I/O 端口（一個`命令`端口和一個`數據`端口）進行配置。對於主控制器，這些端口分別爲 0x20（命令）和 0x21（數據）。

對於輔助控制器，它們分別是 0xa0（命令）和 0xa1（數據）。

## 內存保護，分段、分頁(Segmentation, Paging)
操作系統有責任將不同程序的內存隔離開來，以保證安全性。

x86 架構有兩種方法實現內存隔離，分段和分頁。

### 分段
分段實際上引入了虛擬內存技術。

![虛擬內存](/images/posts/20240821-virtual-memory.png)

### 碎片化
分段容易產生碎片化，造成空間浪費。

![碎片化](/images/posts/20240821-fragmentation.png)

### 分頁
分頁將虛擬內存和物理內存都劃分爲更小、固定大小的區域。

這些被劃分出的一塊塊 block，在虛擬內存被稱爲 page，
在物理內存則稱爲 frame。

![分頁](/images/posts/20240821-paging.png)

#### Page Table
虛擬內存中的 page 和物理內存中的 frame 一一對應信息存儲的地方。



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/08/osdev-memo/  

