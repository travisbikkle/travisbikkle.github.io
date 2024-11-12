# 數據結構


### Doubly Linked List
#### 什麼是雙向鏈表
雙向鏈表是一種特殊的鏈表，其中的每個節點都包含前一個和後一個節點的引用。
![](https://media.geeksforgeeks.org/wp-content/cdn-uploads/gq/2014/03/DLL1.png)
下面是一個雙向鏈表的簡單示例：


&lt;CodeGroup&gt;
  &lt;CodeGroupItem title=&#34;Java&#34; active&gt;

```java
// Class for Doubly Linked List
public class DLL {
    // Head of list
    Node head;
    // Doubly Linked list Node
    class Node {
        int data;
        Node prev;
        Node next;
        // Constructor to create a new node
        // next and prev is by default initialized as null
        Node(int d) { data = d; }
    }
}
```
  &lt;/CodeGroupItem&gt;
  &lt;CodeGroupItem title=&#34;Python&#34;&gt;

```python
# Node of a doubly linked list
class Node:
    def __init__(self, next=None, prev=None, data=None):
        # reference to next node in DLL
        self.next = next
         
        # reference to previous node in DLL
        self.prev = prev
        self.data = data
```
  &lt;/CodeGroupItem&gt;
  &lt;CodeGroupItem title=&#34;Go&#34;&gt;

```go
type Node struct {
    Value    int
    Previous *Node
    Next     *Node
}

type LinkedList struct {
    Head  *Node
    Length int
}
```
  &lt;/CodeGroupItem&gt;
&lt;/CodeGroup&gt;

#### 對比普通鏈表的優勢
* 可以向前或者向後遍歷
* 給定一個節點，如果需要刪除它，雙向鏈表比單向鏈表更快一些，因爲你可以執行以下邏輯即可刪除，而不需要再次查找一次節點的 prev
  ```java
  node.prev.next = node.next
  node.next.prev = node.prev
  ```
* 同理，給定一個節點，在其前面插入一個新的節點更快

#### 對比普通鏈表的劣勢
* 空間浪費：每個節點都要保存上一個節點的引用。使用 C&#43;&#43; 的一個示例可以解決空間浪費的問題，見此 &lt;https://www.geeksforgeeks.org/xor-linked-list-a-memory-efficient-doubly-linked-list-set-1/&gt;
* 因爲多了一個引用，每次操作都需要更多的動作

#### 應用場景
* 瀏覽器的前進和後退
* 很多程序的 undo 和 redo 功能
* LRU ( Least Recently Used ) / MRU ( Most Recently Used ) Cache

### 2-3 Search Tree
建議看 Coursera 上的視頻 &lt;https://www.coursera.org/learn/algorithms-part1/lecture/wIUNW/2-3-search-trees&gt;，本章節的動畫文件均由該視頻截屏後轉換。

* 概念和圖
   ![](/images/interview/20230531_23tree_23node.png)
   每個 node 允許 1-2 個元素
    * 2-node: 1 元素，2個子 node
    * 3-node: 2 元素，3個子 node
    * 從 root 到 null link 的每條路徑都等長
* 2-3 tree demo 搜索的過程  
   ![](/images/interview/20230531_23tree_find_h.gif)
* 2-3 tree demo 插入  
   ![](/images/interview/20230531_23tree_insert_k.gif)

   插入到一個 3-node 會發生什麼？  
   ![](/images/interview/20230531_23tree_insert_z.gif)

* 2-3 tree 構造過程  
   ![](/images/interview/20230531_23tree_construct.png)

* 屬性  
始終維護對稱和平衡。

* 性能
  1. 完美的平衡：每條從根節點到 null links 的路徑都有相同的長度  
  2. 樹的高度：
     1. 最差情況：log&lt;sub&gt;2&lt;/sub&gt;N，全部是 2-node
     2. 最好情況：log&lt;sub&gt;3&lt;/sub&gt;N，全部是 3-node
     3. 12-20（100萬node）
     4. 18-30（10億node）
  3. 查詢和插入操作對數級性能

### Red Black BSTs
BST means binary search tree.

#### Left-leaning read-black BSTs
左傾紅黑樹：通過紅色膠水（紅線）將 3-node 中的兩個元素粘起來  
![](/images/interview/20230531_llrb_vs_23tree.png)
&gt; 一個等價的定義

如果一個 BST 滿足以下幾個條件，那麼它就是一個左傾紅黑樹（LLRB）：
* 沒有節點同時連接了兩條紅線
* 每條從 root 到 null link 的路徑有相同數量的黑線（2-3樹的特性）
* 紅線向左傾斜

實際上，從任意一個節點到其子樹的葉子節點經過的黑線都是相同的。

&gt; 搜索忽略顏色，和普通的 BST 一致，但是因爲其保持較好的平衡，通常更快。

```java
public Val get(Key key){
     Node x = root;
     while (x != null){
         int cmp = key.compareTo(x.key);
         if (cmp &lt; 0) x = x.left;
         else if (cmp &gt; 0) x = x.right;
         else if (cmp == 0) return x.val;
     }
     return null;
}
```
&gt; 因爲每個節點和父節點相連的只有一條線，因此可以將顏色編碼到代碼裏面

```java
 private static final boolean RED = true;
 private static final boolean BLACK = false;
 private class Node {
    Key key;
    Value val;
    Node left, right;
    boolean color; // 注意這裏 color 是指連接父節點的線的顏色
 }
 private boolean isRed(Node x) {
    if (x == null) return false; // null links are black
    return x.color == RED;
 }
```
上面的代碼用圖來解釋，就是：
![](/images/interview/20230531_llrb_color.png)

&gt; 左旋（右旋同理，不變的是要維持樹的平衡和對稱）

![](/images/interview/20230531_llrb_left_rotate.gif)

```java
private Node rotateLeft(Node h) {
    assert isRed(h.right);
    Node x = h.right;
    h.right = x.left;
    x.left = h;
    x.color = h.color;
    h.color = RED;
    return x;
}
```

&gt; 請回答，如果左旋以下 BST 中包含E的節點，那麼左旋後的 BST 按照 level order 遍歷是什麼呢？（答案見評論。）

![](/images/interview/20230601_rbtree_left_rotate_test.jpg)

&gt; 顏色反轉（color flip）

![](/images/interview/20230601_llrb_color_flip.png)

&gt; 插入：下圖插入C，作爲 2-3 tree 來考慮，插入到右側，然後左旋

![img.png](/images/interview/20230601_llrb_insert.png)

&gt; 插入：直觀展示

插入 255 個元素，按照最壞情況，從小到大插入

![img.png](/images/interview/20230601_llrb_insertion_visualization.gif)

&gt; 簡單代碼實現

1. 左黑右紅向左旋
2. 左黑到底向右旋
3. 兩邊都紅顏色變

![img.png](/images/interview/20230601_llrb_code_representation.png)

### B Tree

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/05/data-structure/  

