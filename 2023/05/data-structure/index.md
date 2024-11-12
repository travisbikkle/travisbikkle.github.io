# 数据结构


### Doubly Linked List
#### 什么是双向链表
双向链表是一种特殊的链表，其中的每个节点都包含前一个和后一个节点的引用。
![](https://media.geeksforgeeks.org/wp-content/cdn-uploads/gq/2014/03/DLL1.png)
下面是一个双向链表的简单示例：


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

#### 对比普通链表的优势
* 可以向前或者向后遍历
* 给定一个节点，如果需要删除它，双向链表比单向链表更快一些，因为你可以执行以下逻辑即可删除，而不需要再次查找一次节点的 prev
  ```java
  node.prev.next = node.next
  node.next.prev = node.prev
  ```
* 同理，给定一个节点，在其前面插入一个新的节点更快

#### 对比普通链表的劣势
* 空间浪费：每个节点都要保存上一个节点的引用。使用 C&#43;&#43; 的一个示例可以解决空间浪费的问题，见此 &lt;https://www.geeksforgeeks.org/xor-linked-list-a-memory-efficient-doubly-linked-list-set-1/&gt;
* 因为多了一个引用，每次操作都需要更多的动作

#### 应用场景
* 浏览器的前进和后退
* 很多程序的 undo 和 redo 功能
* LRU ( Least Recently Used ) / MRU ( Most Recently Used ) Cache

### 2-3 Search Tree
建议看 Coursera 上的视频 &lt;https://www.coursera.org/learn/algorithms-part1/lecture/wIUNW/2-3-search-trees&gt;，本章节的动画文件均由该视频截屏后转换。

* 概念和图
   ![](/images/interview/20230531_23tree_23node.png)
   每个 node 允许 1-2 个元素
    * 2-node: 1 元素，2个子 node
    * 3-node: 2 元素，3个子 node
    * 从 root 到 null link 的每条路径都等长
* 2-3 tree demo 搜索的过程  
   ![](/images/interview/20230531_23tree_find_h.gif)
* 2-3 tree demo 插入  
   ![](/images/interview/20230531_23tree_insert_k.gif)

   插入到一个 3-node 会发生什么？  
   ![](/images/interview/20230531_23tree_insert_z.gif)

* 2-3 tree 构造过程  
   ![](/images/interview/20230531_23tree_construct.png)

* 属性  
始终维护对称和平衡。

* 性能
  1. 完美的平衡：每条从根节点到 null links 的路径都有相同的长度  
  2. 树的高度：
     1. 最差情况：log&lt;sub&gt;2&lt;/sub&gt;N，全部是 2-node
     2. 最好情况：log&lt;sub&gt;3&lt;/sub&gt;N，全部是 3-node
     3. 12-20（100万node）
     4. 18-30（10亿node）
  3. 查询和插入操作对数级性能

### Red Black BSTs
BST means binary search tree.

#### Left-leaning read-black BSTs
左倾红黑树：通过红色胶水（红线）将 3-node 中的两个元素粘起来  
![](/images/interview/20230531_llrb_vs_23tree.png)
&gt; 一个等价的定义

如果一个 BST 满足以下几个条件，那么它就是一个左倾红黑树（LLRB）：
* 没有节点同时连接了两条红线
* 每条从 root 到 null link 的路径有相同数量的黑线（2-3树的特性）
* 红线向左倾斜

实际上，从任意一个节点到其子树的叶子节点经过的黑线都是相同的。

&gt; 搜索忽略颜色，和普通的 BST 一致，但是因为其保持较好的平衡，通常更快。

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
&gt; 因为每个节点和父节点相连的只有一条线，因此可以将颜色编码到代码里面

```java
 private static final boolean RED = true;
 private static final boolean BLACK = false;
 private class Node {
    Key key;
    Value val;
    Node left, right;
    boolean color; // 注意这里 color 是指连接父节点的线的颜色
 }
 private boolean isRed(Node x) {
    if (x == null) return false; // null links are black
    return x.color == RED;
 }
```
上面的代码用图来解释，就是：
![](/images/interview/20230531_llrb_color.png)

&gt; 左旋（右旋同理，不变的是要维持树的平衡和对称）

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

&gt; 请回答，如果左旋以下 BST 中包含E的节点，那么左旋后的 BST 按照 level order 遍历是什么呢？（答案见评论。）

![](/images/interview/20230601_rbtree_left_rotate_test.jpg)

&gt; 颜色反转（color flip）

![](/images/interview/20230601_llrb_color_flip.png)

&gt; 插入：下图插入C，作为 2-3 tree 来考虑，插入到右侧，然后左旋

![img.png](/images/interview/20230601_llrb_insert.png)

&gt; 插入：直观展示

插入 255 个元素，按照最坏情况，从小到大插入

![img.png](/images/interview/20230601_llrb_insertion_visualization.gif)

&gt; 简单代码实现

1. 左黑右红向左旋
2. 左黑到底向右旋
3. 两边都红颜色变

![img.png](/images/interview/20230601_llrb_code_representation.png)

### B Tree

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/05/data-structure/  

