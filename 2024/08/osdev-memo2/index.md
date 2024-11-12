# OS 开发备忘2


## 实现堆内存
### 本地变量与栈内存
outer 方法调用 inner 方法的栈，从上至下。
![本地变量](/images/posts/20240822-local-variables.png)

inner 方法执行完成后。
![inner执行完成后](/images/posts/20240822-after-inner-executed.png)

### 静态变量
静态变量存储在独立于堆栈的固定内存位置。

该内存位置在编译时由链接器分配，并在可执行文件中编码。

![静态变量](/images/posts/20240822-static-variables.png)

### 动态内存
本地变量只生存于作用域内，静态变量全局可用、不能够回收、所有权不清晰。

两者都有一个固定的大小，所以无法存储会动态增长的变量。

为了解决这些问题，编程语言都会引入第三块内存区域： heap。

假设使用 allocate 和 deallocate 来分配和释放内存：

![heap](/images/posts/20240822-dynamic-memory.png)

在调用 deallocate 之后，内存变成如下图所示：

![释放之后](/images/posts/20240822-after-deallocate.png)

所以相对于静态变量来说，z[1] 这块内存区域，我们可以复用。

但是 z[0] 和 z[2] 永远不释放，这就造成了内存泄漏。

#### 其它错误
除了内存泄漏，但它并不会使我们的程序更易受到攻击，还有两种类型的错误会有更严重的后果：
1. use-after-free: 在 deallocate 之后尝试使用这个变量，这种错误会导致无法预料的行为，经常被攻击者利用来执行任意代码。
2. double-free: 释放两次，可能会释放掉一个在 deallocate 之后又重新分配的地址，也有可能导致 use-after-free 问题。

即使是最好的程序员，也无法永远不出错误地处理这些分配和释放流程，而且每年都会有新的错误，
比如可以在搜索引擎尝试搜索 use-after-free 2024，你可以把 2024 换成当年的年份。

为了解决这些问题，java 和 python 会使用垃圾回收机制，核心思想是，编码者无需手动写 allocate 和 deallocate，而是定时暂停、
扫描程序中未使用的堆变量，然后释放掉这些内存。这样，以上错误就不会再出现，缺点是定时扫描导致的性能开销，以及长时间的暂停。

rust 使用了不一样的解决方法：它使用所有权设计，来在编译时检查动态内存操作的正确性，于是rust消灭了垃圾回收，当然也就没有性能开销了。
另一个好处是，编码者仍然对内存拥有细粒度的控制，就像 C、C&#43;&#43;，缺点就是你会有很多编译错误需要解决。

### rust 中的分配
Rust 中最重要的类型之一是 Box，它其实是堆变量的封装，使用 Box::new 接收一个变量，执行 allocate 分配一个变量大小的区域，然后将变量
的值转移到堆中。要释放这块区域，Box 中还有 Drop 特质，它会调用 deallocate。

光有 Box 肯定不够，下面的代码块，z[1]的引用被分配给了x（这被称为 borrow），紧接着z会被释放，但是你可能在代码块之外调用x。
```rust
let x = {
    let z = Box::new([1,2,3]);
    &amp;z[1]
}; // z goes out of scope and `deallocate` is called
println!(&#34;{}&#34;, x);
```
于是所有权发挥作用了，你会遇到编译错误：
![img.png](/images/posts/20240822-rust-ownership-demo.png)

borrow 一个变量和现实生活中的`借`一样，你没有所有权，不能销毁它。rust 通过检查在一个变量销毁前，所有的 borrow 都结束，来确保不会发生
use-after-free。

rust 的这一切都是在编译期发生的，因此不会像在 C 中手写内存管理一样产生性能开销。

### 实现一个操作系统的堆内存
实现堆内存，需要如下几步：
1. 创建一块内存区域
   1. 确定一块虚拟内存区域
   2. 将这块区域映射到物理 frame
2. 创建一个 allocator

#### 1.1 确定内存区域
一个开始地址，一个大小，就可以确定一块区域。开始地址可以随便写，只要不与程序中的其它内存区域冲突。
```rust
pub const HEAP_START: usize = 0x_4444_4444_0000;
pub const HEAP_SIZE: usize = 100 * 1024; // 100 KiB
```
#### 1.2 映射到物理地址
```toml
linked_list_allocator = &#34;0.9.0&#34;
```

```rust
// lib/allocator.rs
use alloc::alloc::GlobalAlloc;
use linked_list_allocator::LockedHeap;
use x86_64::{
    structures::paging::{
        FrameAllocator, Mapper, mapper::MapToError, Page, PageTableFlags, Size4KiB,
    },
    VirtAddr,
};

pub const HEAP_START: usize = 0x_4444_4444_0000;
pub const HEAP_SIZE: usize = 100 * 1024; // 100 KiB

pub fn init_heap(
    mapper: &amp;mut impl Mapper&lt;Size4KiB&gt;,
    frame_allocator: &amp;mut impl FrameAllocator&lt;Size4KiB&gt;,
) -&gt; Result&lt;(), MapToError&lt;Size4KiB&gt;&gt; {
    let page_range = {
        let heap_start = VirtAddr::new(HEAP_START as u64);
        let heap_end = heap_start &#43; HEAP_SIZE - 1u64;
        let heap_start_page = Page::containing_address(heap_start);
        let heap_end_page = Page::containing_address(heap_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };

    for page in page_range {
        let frame = frame_allocator
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        unsafe {
            mapper.map_to(page, frame, flags, frame_allocator)?.flush()
        };
    }

    unsafe {
        ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE);
    }

    Ok(())
}

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();
```

```rust
// src/main.rs
...
extern crate alloc;
entry_point!(kernel_main);

// 非测试情况，在程序崩溃的时候调用的函数
#[cfg(not(test))] // new attribute
#[panic_handler]
fn panic(info: &amp;PanicInfo) -&gt; ! {
   ...
}

#[cfg(test)]
#[panic_handler]
fn panic(info: &amp;PanicInfo) -&gt; ! {
    rust_os::test_panic_handler(info)
}

fn kernel_main(boot_info: &amp;&#39;static BootInfo) -&gt; ! {
    // region use
    ...
    // endregion use

    println!(&#34;Hello World{}&#34;, &#34;!&#34;);

    // region 异常与中断
    rust_os::init();
    // endregion

    // region 内存映射
    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&amp;boot_info.memory_map)
    };
    allocator::init_heap(&amp;mut mapper, &amp;mut frame_allocator)
        .expect(&#34;heap initialization failed&#34;);
    // endregion 内存映射

    // region 测试堆内存使用
    let heap_value = Box::new(41);
    println!(&#34;heap_value at {:p}&#34;, heap_value);

    // create a dynamically sized vector
    let mut vec = Vec::new();
    for i in 0..500 {
        vec.push(i);
    }
    println!(&#34;vec at {:p}&#34;, vec.as_slice());

    // create a reference counted vector -&gt; will be freed when count reaches 0
    let reference_counted = Rc::new(vec![1, 2, 3]);
    let cloned_reference = reference_counted.clone();
    println!(&#34;current reference count is {}&#34;, Rc::strong_count(&amp;cloned_reference));
    core::mem::drop(reference_counted);
    println!(&#34;reference count is {} now&#34;, Rc::strong_count(&amp;cloned_reference));
    // endregion 测试堆内存使用

    // region 单元测试入口
    ...
    // endregion 单元测试入口

    println!(&#34;It did not crash!&#34;);
    rust_os::hlt_loop();
}
```

我们创建了一个Box，一个Vec，一个Rc包裹的变量。

![测试堆内存使用](/images/posts/20240822-test-heap.png)

如此就实现了堆内存。注意vec分配的地址比box便宜了0x800，不是因为box值大小是0x800字节大，而是vec需要增加容量，发生了重分配。

比如vector的容量是32，我们尝试添加一个新元素，vector就分配一个新的64大小的数组，将之前的元素全部拷贝过来，这一步就会释放旧的内存。


## 内存 Allocator 手动实现
### 主要目标
* allocate 返回可用的内存区域；
* deallocate 释放指定的内存区域。

### Bump Allocator
也叫 Stack Allocator。

缺点：有严重的限制，只能一次释放所有内存，释放完成后才能服用内存。
有点：很快

如下图，next永远指向可用内存的开头位置。
![20240823-bump-allocator.png](/images/posts/20240823-bump-allocator.png)

### Linked List Allocator
缺点：性能差，可能要遍历链表才能找到一块区域。

![linked list allocator](/images/posts/20240823-bump-allocator.png)
每个链表中的节点都保存有当前内存区域大小，以及下一块未使用内存的地址。

### Fixed-Size Block Allocator
* 多链表，每个链表固定大小。
* 不足大小向上取整

缺点：容易造成空间浪费。

![20240823-fixed-size-allocator.png](/images/posts/20240823-fixed-size-allocator.png)

#### Linux 中使用的 Fixed-Size Block Allocator 变体
[Slab Allocator 和 Buddy Allocator](https://os.phil-opp.com/allocator-designs/#variations)

## Pin 与自引用 struct
通过 allocate 分配在堆内存上的变量，有一个固定的地址，并且同时有一个指针类型例如 Box&lt;T&gt; 指向它。

但是在自引用结构中（上述提到的状态机，会用一种自引用结构实现），这种
```rust
fn main() {
    // 创建一个名为 SelfReferential 的 struct，其中包含一个指针字段 self_ptr。
    let mut heap_value = Box::new(SelfReferential {
       // 首先，使用空指针初始化， 然后使用 Box::new 在堆上分配该 struct。
       self_ptr: 0 as *const _,
    });
    // 然后，我们获取到堆分配结构的内存地址，并将其存储在一个 ptr 变量中。
    let ptr = &amp;*heap_value as *const SelfReferential;
    // 最后，我们将 ptr 变量赋值给 self_ptr 字段，使 struct 成为自引用 struct。
    heap_value.self_ptr = ptr;
    println!(&#34;heap value at: {:p}&#34;, heap_value);
    println!(&#34;internal reference: {:p}&#34;, heap_value.self_ptr);

   // 破坏自引用结构，将 heap_value 转移到了 stack 上，上面的 self_ptr 此时变成了悬空引用 
   let stack_value = mem::replace(&amp;mut *heap_value, SelfReferential {
      self_ptr: 0 as *const _,
   });
   
   // 下面的两个值不一样，结构被破坏
   println!(&#34;value at: {:p}&#34;, &amp;stack_value);
   println!(&#34;internal reference: {:p}&#34;, stack_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
}
// heap value at: 0x55eaf7ad09b0
// internal reference: 0x55eaf7ad09b0
// value at: 0x7ffccbf74a58
// internal reference: 0x55eaf7ad09b0
```
原因在于 Box&lt;T&gt; 返回了一个 &amp;mut T 的引用指向堆变量，因此就可以用 mem::replace 或者 mem::swap 等方法作废堆变量。

### Pin&lt;Box&lt;T&gt;&gt; and Unpin
```rust
use core::marker::PhantomPinned; // 标记类型，唯一作用是不实现 UnPin 特质

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned // 只要有一个字段 UnPin，整个 struct 就是 Unpin 的
}
```
如下代码将会按预料报错。
```rust
use std::mem;
use std::marker::PhantomPinned;

fn main() {
    let mut heap_value = Box::pin(SelfReferential {
        self_ptr: 0 as *const _,
        _pin: PhantomPinned,
    });
    let ptr = &amp;*heap_value as *const SelfReferential;
    heap_value.self_ptr = ptr; // 报错 trait `DerefMut` is required to modify through a dereference
    println!(&#34;heap value at: {:p}&#34;, heap_value);
    println!(&#34;internal reference: {:p}&#34;, heap_value.self_ptr);
    
    // break it
    // 报错 trait `DerefMut` is required to modify through a dereference
    let stack_value = mem::replace(&amp;mut *heap_value, SelfReferential {
        self_ptr: 0 as *const _,
        _pin: PhantomPinned,
    });
    println!(&#34;value at: {:p}&#34;, &amp;stack_value);
    println!(&#34;internal reference: {:p}&#34;, stack_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}
```
现在编译器虽然修复了 self_ptr 的 move 问题，但是我们现在也无法初始化了。要想修复此问题，需要：
```rust
use std::mem;
use std::marker::PhantomPinned;
use std::pin::Pin;

fn main() {
    let mut heap_value = Box::pin(SelfReferential {
        self_ptr: 0 as *const _,
        _pin: PhantomPinned,
    });
    let ptr = &amp;*heap_value as *const SelfReferential;
    
    // safe because modifying a field doesn&#39;t move the whole struct
    unsafe {
        let mut_ref = Pin::as_mut(&amp;mut heap_value);
        Pin::get_unchecked_mut(mut_ref).self_ptr = ptr;
    }

    println!(&#34;heap value at: {:p}&#34;, heap_value);
    println!(&#34;internal reference: {:p}&#34;, heap_value.self_ptr);
    
    // break it
    // 现在这段代码，只有这里报错了。这里是我们需要它报错，我们不希望将 heap 上的变量 move 到 stack 上。
    let stack_value = mem::replace(&amp;mut *heap_value, SelfReferential {
        self_ptr: 0 as *const _,
        _pin: PhantomPinned,
    });
    println!(&#34;value at: {:p}&#34;, &amp;stack_value);
    println!(&#34;internal reference: {:p}&#34;, stack_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}
```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/08/osdev-memo2/  

