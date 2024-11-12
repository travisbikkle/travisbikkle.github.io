# OS 开发备忘


## 显示（VGA TEXT MODE）

* 要在 VGA 文本模式下将字符打印到屏幕上，必须将其写入 VGA 硬件的文本缓冲区。
* VGA 文本缓冲区是一个二维数组，通常有 25 行 80 列，可直接渲染到屏幕上。
* 每个数组条目通过以下格式描述一个屏幕字符：

| Bit(s) | Value         |
|--------|---------------|
| 0-7    | ASCII 字符，8bit |
| 8-11   | 前景色，4bit      |
| 12-14  | 背景色，3bit      |
| 15     | 闪烁，1bit       |

### 如何设计（伪代码）：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    // 蓝绿
    Cyan = 3,
    Red = 4,
    // 洋红
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    ...
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
/// 表示颜色代码
struct ColorCode(u8);
impl ColorCode {
    fn new(foreground: Color, background: Color) -&gt; ColorCode {
        ColorCode((background as u8) &lt;&lt; 4 | (foreground as u8))
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
/// 表示写入 buffer 中的字符，由字符和颜色构成
struct ScreenChar {
    // 第一字节
    ascii_character: u8,
    // 第二字节
    color_code: ColorCode,
}

/// vga text buffer 的数组大小
const BUFFER_HEIGHT: usize = 25;
const BUFFER_WIDTH: usize = 80;

#[repr(transparent)]
struct Buffer {
    chars: [[Volatile&lt;ScreenChar&gt;; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

## CPU 异常

### IDT Interrupt Descriptor Table

* 为了捕捉和处理异常，我们必须建立一个所谓的中断描述符表（IDT）。
* 在该表中，我们可以为每个 CPU 异常指定一个处理函数（handler function）。
* 硬件会直接使用该表，因此我们需要遵循预定义的格式。
* 每个条目必须具有以下 16 字节结构（一个u16需要2字节，下面的表格共有16字节）：

  | 类型  | 名称                       | 描述                      |
              |-----|--------------------------|-------------------------|
  | u16 | Function Pointer [0:15]  | handler function 指针低位.  |
  | u16 | GDT selector             | GDT（见下文） 代码片段的选择器.      |
  | u16 | Options                  | (见下文)                   |
  | u16 | Function Pointer [16:31] | handler function 指针中位.  |
  | u32 | Function Pointer [32:63] | handler function 指针其余位. |
  | u32 | 保留                       |                         |

Options（16bit） 遵从如下格式：

| 位     | 名称                               | 描述                                                                                                              |
|-------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 0-2   | Interrupt Stack Table Index      | 0: Don’t switch stacks, 1-7: Switch to the n-th stack in the Interrupt Stack Table when this handler is called. |
| 3-7   | 保留                               |                                                                                                                 |
| 8     | 0: Interrupt Gate, 1: Trap Gate  | 0 代表不中断                                                                                                         |
| 9-11  | 必须是1                             |                                                                                                                 |
| 12    | 必须是0                             |                                                                                                                 |
| 13‑14 | Descriptor Privilege Level (DPL) | 调用此函数需要的最低权限等级.                                                                                                 |
| 15    | Present                          |                                                                                                                 |

CPU异常详见[此处](https://wiki.osdev.org/Exceptions)。

### CPU 异常处理的大致步骤
1. 将一些寄存器推入堆栈，包括指令指针和 [RFLAGS 寄存器](https://en.wikipedia.org/wiki/FLAGS_register)。(我们稍后将使用这些值）。
2. 从中断描述符表（IDT）中读取相应的条目。例如，当发生页面故障时，CPU 会读取第 14 个条目。
3. 检查条目是否存在，如果不存在，则引发双重故障(double fault)。
4. 如果条目是中断门（未设置第 40 位，也就是 Options 表第 8 位），则禁用硬件中断。
5. 将指定的 GDT 选择器载入 CS（代码段）。
6. 跳转到指定的处理函数。

### 如何设计（直接使用现有的 crate）：
[x86_64 IDT](https://docs.rs/x86_64/0.14.2/x86_64/structures/idt/struct.InterruptDescriptorTable.html)

### 断点异常 (breakpoint exception)
* 当用户设置断点时，调试器会用 int3 指令覆盖相应的指令，这样 CPU 在运行到该行时就会抛出断点异常。
* 当用户想继续运行程序时，调试器会再次用原来的指令替换 int3 指令，然后继续运行程序。

使用 x86-interrupt 实现的一个断点异常处理函数示例： 

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
    // 触发一次断点异常
    x86_64::instructions::interrupts::int3();
}
```

### 双重异常（double fault）
双重故障是 CPU 无法调用异常处理程序时出现的一种特殊异常。

例如，当页面故障被触发，但中断描述符表（IDT）中没有注册页面故障处理程序时，就会出现这种情况。

因此，它有点类似于有异常的编程语言中的 catch-all 块，如 C&#43;&#43; 中的 catch(...) 或 Java 或 C# 中的 catch(Exception e)。

双重故障的行为与普通异常类似。我们可以在 IDT 中为它定义一个普通的处理函数。

提供双重故障处理程序非常重要，因为如果双重故障未得到处理，就会发生致命的三重故障。三重故障无法被捕获，大多数硬件都会做出系统复位的反应。

## 硬件中断(Hardware Interrupts)
中断提供了一种从硬件设备通知 CPU 的方法。

因此，与其让内核定期检查键盘是否键入新字符（这一过程称为轮询），不如让键盘在每次按键时通知内核。

这样做效率更高，因为内核只需在发生事情时采取行动。由于内核可以立即做出反应，而不是在下一次轮询时才做出反应，因此反应时间也更快。

将所有硬件设备直接连接到 CPU 是不可能的。相反，一个独立的中断控制器会汇总来自所有设备的中断，然后通知 CPU：

```text
                                    ____________             _____
               Timer ------------&gt; |            |           |     |
               Keyboard ---------&gt; | Interrupt  |---------&gt; | CPU |
               Other Hardware ---&gt; | Controller |           |_____|
               Etc. -------------&gt; |____________|
```

### 8259 可编程中断控制器 (programmable interrupt controller (PIC))
英特尔 8259 是 1976 年推出的可编程中断控制器 (PIC)。
它早已被较新的 APIC 所取代，但出于向后兼容的考虑，其接口在当前系统中仍受支持。

8259 有八条中断线路和几条与 CPU 通信的线路。以前的典型系统配置有两个 8259 PIC 实例，一个主 PIC，一个辅助 PIC，分别与主 PIC 的一条中断线相连：

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
每个控制器可通过两个 I/O 端口（一个`命令`端口和一个`数据`端口）进行配置。对于主控制器，这些端口分别为 0x20（命令）和 0x21（数据）。

对于辅助控制器，它们分别是 0xa0（命令）和 0xa1（数据）。

## 内存保护，分段、分页(Segmentation, Paging)
操作系统有责任将不同程序的内存隔离开来，以保证安全性。

x86 架构有两种方法实现内存隔离，分段和分页。

### 分段
分段实际上引入了虚拟内存技术。

![虚拟内存](/images/posts/20240821-virtual-memory.png)

### 碎片化
分段容易产生碎片化，造成空间浪费。

![碎片化](/images/posts/20240821-fragmentation.png)

### 分页
分页将虚拟内存和物理内存都划分为更小、固定大小的区域。

这些被划分出的一块块 block，在虚拟内存被称为 page，
在物理内存则称为 frame。

![分页](/images/posts/20240821-paging.png)

#### Page Table
虚拟内存中的 page 和物理内存中的 frame 一一对应信息存储的地方。



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/08/osdev-memo/  

