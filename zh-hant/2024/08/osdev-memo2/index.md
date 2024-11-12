# OS 開發備忘2


## 實現堆內存
### 本地變量與棧內存
outer 方法調用 inner 方法的棧，從上至下。
![本地變量](/images/posts/20240822-local-variables.png)

inner 方法執行完成後。
![inner執行完成後](/images/posts/20240822-after-inner-executed.png)

### 靜態變量
靜態變量存儲在獨立於堆棧的固定內存位置。

該內存位置在編譯時由鏈接器分配，並在可執行文件中編碼。

![靜態變量](/images/posts/20240822-static-variables.png)

### 動態內存
本地變量只生存於作用域內，靜態變量全局可用、不能夠回收、所有權不清晰。

兩者都有一個固定的大小，所以無法存儲會動態增長的變量。

爲了解決這些問題，編程語言都會引入第三塊內存區域： heap。

假設使用 allocate 和 deallocate 來分配和釋放內存：

![heap](/images/posts/20240822-dynamic-memory.png)

在調用 deallocate 之後，內存變成如下圖所示：

![釋放之後](/images/posts/20240822-after-deallocate.png)

所以相對於靜態變量來說，z[1] 這塊內存區域，我們可以複用。

但是 z[0] 和 z[2] 永遠不釋放，這就造成了內存泄漏。

#### 其它錯誤
除了內存泄漏，但它並不會使我們的程序更易受到攻擊，還有兩種類型的錯誤會有更嚴重的後果：
1. use-after-free: 在 deallocate 之後嘗試使用這個變量，這種錯誤會導致無法預料的行爲，經常被攻擊者利用來執行任意代碼。
2. double-free: 釋放兩次，可能會釋放掉一個在 deallocate 之後又重新分配的地址，也有可能導致 use-after-free 問題。

即使是最好的程序員，也無法永遠不出錯誤地處理這些分配和釋放流程，而且每年都會有新的錯誤，
比如可以在搜索引擎嘗試搜索 use-after-free 2024，你可以把 2024 換成當年的年份。

爲了解決這些問題，java 和 python 會使用垃圾回收機制，核心思想是，編碼者無需手動寫 allocate 和 deallocate，而是定時暫停、
掃描程序中未使用的堆變量，然後釋放掉這些內存。這樣，以上錯誤就不會再出現，缺點是定時掃描導致的性能開銷，以及長時間的暫停。

rust 使用了不一樣的解決方法：它使用所有權設計，來在編譯時檢查動態內存操作的正確性，於是rust消滅了垃圾回收，當然也就沒有性能開銷了。
另一個好處是，編碼者仍然對內存擁有細粒度的控制，就像 C、C&#43;&#43;，缺點就是你會有很多編譯錯誤需要解決。

### rust 中的分配
Rust 中最重要的類型之一是 Box，它其實是堆變量的封裝，使用 Box::new 接收一個變量，執行 allocate 分配一個變量大小的區域，然後將變量
的值轉移到堆中。要釋放這塊區域，Box 中還有 Drop 特質，它會調用 deallocate。

光有 Box 肯定不夠，下面的代碼塊，z[1]的引用被分配給了x（這被稱爲 borrow），緊接着z會被釋放，但是你可能在代碼塊之外調用x。
```rust
let x = {
    let z = Box::new([1,2,3]);
    &amp;z[1]
}; // z goes out of scope and `deallocate` is called
println!(&#34;{}&#34;, x);
```
於是所有權發揮作用了，你會遇到編譯錯誤：
![img.png](/images/posts/20240822-rust-ownership-demo.png)

borrow 一個變量和現實生活中的`借`一樣，你沒有所有權，不能銷燬它。rust 通過檢查在一個變量銷燬前，所有的 borrow 都結束，來確保不會發生
use-after-free。

rust 的這一切都是在編譯期發生的，因此不會像在 C 中手寫內存管理一樣產生性能開銷。

### 實現一個操作系統的堆內存
實現堆內存，需要如下幾步：
1. 創建一塊內存區域
   1. 確定一塊虛擬內存區域
   2. 將這塊區域映射到物理 frame
2. 創建一個 allocator

#### 1.1 確定內存區域
一個開始地址，一個大小，就可以確定一塊區域。開始地址可以隨便寫，只要不與程序中的其它內存區域衝突。
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

// 非測試情況，在程序崩潰的時候調用的函數
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

    // region 異常與中斷
    rust_os::init();
    // endregion

    // region 內存映射
    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&amp;boot_info.memory_map)
    };
    allocator::init_heap(&amp;mut mapper, &amp;mut frame_allocator)
        .expect(&#34;heap initialization failed&#34;);
    // endregion 內存映射

    // region 測試堆內存使用
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
    // endregion 測試堆內存使用

    // region 單元測試入口
    ...
    // endregion 單元測試入口

    println!(&#34;It did not crash!&#34;);
    rust_os::hlt_loop();
}
```

我們創建了一個Box，一個Vec，一個Rc包裹的變量。

![測試堆內存使用](/images/posts/20240822-test-heap.png)

如此就實現了堆內存。注意vec分配的地址比box便宜了0x800，不是因爲box值大小是0x800字節大，而是vec需要增加容量，發生了重分配。

比如vector的容量是32，我們嘗試添加一個新元素，vector就分配一個新的64大小的數組，將之前的元素全部拷貝過來，這一步就會釋放舊的內存。


## 內存 Allocator 手動實現
### 主要目標
* allocate 返回可用的內存區域；
* deallocate 釋放指定的內存區域。

### Bump Allocator
也叫 Stack Allocator。

缺點：有嚴重的限制，只能一次釋放所有內存，釋放完成後才能服用內存。
有點：很快

如下圖，next永遠指向可用內存的開頭位置。
![20240823-bump-allocator.png](/images/posts/20240823-bump-allocator.png)

### Linked List Allocator
缺點：性能差，可能要遍歷鏈表才能找到一塊區域。

![linked list allocator](/images/posts/20240823-bump-allocator.png)
每個鏈表中的節點都保存有當前內存區域大小，以及下一塊未使用內存的地址。

### Fixed-Size Block Allocator
* 多鏈表，每個鏈表固定大小。
* 不足大小向上取整

缺點：容易造成空間浪費。

![20240823-fixed-size-allocator.png](/images/posts/20240823-fixed-size-allocator.png)

#### Linux 中使用的 Fixed-Size Block Allocator 變體
[Slab Allocator 和 Buddy Allocator](https://os.phil-opp.com/allocator-designs/#variations)

## Pin 與自引用 struct
通過 allocate 分配在堆內存上的變量，有一個固定的地址，並且同時有一個指針類型例如 Box&lt;T&gt; 指向它。

但是在自引用結構中（上述提到的狀態機，會用一種自引用結構實現），這種
```rust
fn main() {
    // 創建一個名爲 SelfReferential 的 struct，其中包含一個指針字段 self_ptr。
    let mut heap_value = Box::new(SelfReferential {
       // 首先，使用空指針初始化， 然後使用 Box::new 在堆上分配該 struct。
       self_ptr: 0 as *const _,
    });
    // 然後，我們獲取到堆分配結構的內存地址，並將其存儲在一個 ptr 變量中。
    let ptr = &amp;*heap_value as *const SelfReferential;
    // 最後，我們將 ptr 變量賦值給 self_ptr 字段，使 struct 成爲自引用 struct。
    heap_value.self_ptr = ptr;
    println!(&#34;heap value at: {:p}&#34;, heap_value);
    println!(&#34;internal reference: {:p}&#34;, heap_value.self_ptr);

   // 破壞自引用結構，將 heap_value 轉移到了 stack 上，上面的 self_ptr 此時變成了懸空引用 
   let stack_value = mem::replace(&amp;mut *heap_value, SelfReferential {
      self_ptr: 0 as *const _,
   });
   
   // 下面的兩個值不一樣，結構被破壞
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
原因在於 Box&lt;T&gt; 返回了一個 &amp;mut T 的引用指向堆變量，因此就可以用 mem::replace 或者 mem::swap 等方法作廢堆變量。

### Pin&lt;Box&lt;T&gt;&gt; and Unpin
```rust
use core::marker::PhantomPinned; // 標記類型，唯一作用是不實現 UnPin 特質

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned // 只要有一個字段 UnPin，整個 struct 就是 Unpin 的
}
```
如下代碼將會按預料報錯。
```rust
use std::mem;
use std::marker::PhantomPinned;

fn main() {
    let mut heap_value = Box::pin(SelfReferential {
        self_ptr: 0 as *const _,
        _pin: PhantomPinned,
    });
    let ptr = &amp;*heap_value as *const SelfReferential;
    heap_value.self_ptr = ptr; // 報錯 trait `DerefMut` is required to modify through a dereference
    println!(&#34;heap value at: {:p}&#34;, heap_value);
    println!(&#34;internal reference: {:p}&#34;, heap_value.self_ptr);
    
    // break it
    // 報錯 trait `DerefMut` is required to modify through a dereference
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
現在編譯器雖然修復了 self_ptr 的 move 問題，但是我們現在也無法初始化了。要想修復此問題，需要：
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
    // 現在這段代碼，只有這裏報錯了。這裏是我們需要它報錯，我們不希望將 heap 上的變量 move 到 stack 上。
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
> URL: https://travisbikkle.github.io/zh-hant/2024/08/osdev-memo2/  

