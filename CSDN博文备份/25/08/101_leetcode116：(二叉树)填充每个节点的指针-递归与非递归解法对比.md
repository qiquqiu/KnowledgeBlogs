# leetcode116：(二叉树)填充每个节点的指针-递归与非递归解法对比

> 原创 于 2025-08-02 07:30:00 发布 · 公开 · 1.9k 阅读 · 47 · 32 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149843742

**文章目录**

[TOC]


[LeetCode 116 填充每个节点的下一个右侧节点指针](https://leetcode.cn/problems/populating-next-right-pointers-in-each-node/description/) ，【难度：中等；通过率：74.6%】，这道题要求我们为一种特殊的 `Node` 结构填充其 `next` 指针，使其指向同一层的下一个节点

## 一、 题目描述

给定一个 **完美二叉树** ，其所有叶子节点都在同一层，每个父节点都有两个子节点。二叉树定义如下：

```java
class Node {
    public int val;
    public Node left;
    public Node right;
    public Node next;
    // ... 构造函数 ...
};
```

填充它的每个 `next` 指针，让这个指针指向其下一个右侧节点。如果找不到下一个右侧节点，则将 `next` 指针设置为 `NULL` 

初始状态下，所有 `next` 指针都被设置为 `NULL` 

**示例:** 
 ![示例 1](./assets/101_1.png)

```
输入: root = [1,2,3,4,5,6,7]
输出: [1,#,2,3,#,4,5,6,7,#]
解释：给定二叉树如图 A 所示，你的函数应该填充它的每个 next 指针，以指向其下一个右侧节点，如图 B 所示。序列化的输出按层序遍历排列，同一层节点由 next 指针连接，'#' 标志着每一层的结束
```

---

## 解法一：层序遍历 (BFS) - 直观易懂的通用解

这是解决“按层处理”问题最自然的方法。我们使用队列进行广度优先搜索（BFS），逐层处理节点

### 核心思路

1. 使用一个队列来进行层序遍历

2. 在遍历每一层时，记录下当前层的节点数量 `size` 

3. 通过一个 `for` 循环，精确地处理当前层的所有 `size` 个节点

4. 对于当前层的每一个节点，如果它不是该层的最后一个节点 ( `i < size - 1` )，那么它的 `next` 指针就应该指向队列中紧邻的下一个节点（即队头 `queue.peek()` ）

5. 在处理当前节点时，将其左右孩子加入队列，为下一层的遍历做准备

### 代码实现

```java
class Solution {
    // 该方法也适用于非完美二叉树 (LeetCode 117)
    public Node connect(Node root) {
        if (root == null) {
            return null;
        }

        Queue<Node> queue = new LinkedList<>();
        queue.offer(root);

        while (!queue.isEmpty()) {
            int size = queue.size(); // 锁定当前层的节点数量

            // 遍历当前层的所有节点
            for (int i = 0; i < size; i++) {
                Node node = queue.poll(); // 取出当前节点

                // 如果不是当前层的最后一个节点，则连接 next 指针
                if (i < size - 1) {
                    // queue.peek() 正是当前节点右侧的下一个节点
                    node.next = queue.peek();
                }

                // 将子节点加入队列，为下一层做准备
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
        }
        return root;
    }
}
```

### 优缺点分析

-  **优点** ：思路直观，代码逻辑清晰，易于理解。并且这个解法非常通用，也适用于非完美二叉树（LeetCode 117）

-  **缺点** ：需要使用队列，空间复杂度为 O(W)，其中 W 是树的最大宽度。对于一个完美的二叉树，最后一层有大约 N/2 个节点，所以空间复杂度最坏为 O(N)

---

## 解法二：递归 (DFS) - 分而治之

这种方法更具技巧性，它将连接问题分解为三个子问题，然后递归地解决

### 核心思路

这个思路与LeetCode101（对称二叉树） **等一系列题目** ，都有相似之处。对于任意一对作为参数传入的 `left` 和 `right` 节点，我们需要完成三件事：

1.  **连接当前层** ：将 `left` 的 `next` 指针指向 `right` 

2.  **递归处理子树内部** ：

   - 连接 `left` 自己的左右孩子： `connect(left.left, left.right)` 

   - 连接 `right` 自己的左右孩子： `connect(right.left, right.right)` 

3.  **递归处理跨子树连接 (最关键)** ：连接 `left` 的右孩子和 `right` 的左孩子： `connect(left.right, right.left)` 

通过这三步递归，我们确保了所有可能的连接都被建立起来

### 代码实现

```java
class Solution {
    public Node connect(Node root) {
        if (root == null) {
            return null;
        }
        // 从根节点的左右孩子开始启动递归
        dfs(root.left, root.right);
        return root;
    }

    // 辅助递归函数，负责连接两个节点以及它们的子孙节点
    private void dfs(Node left, Node right) {
        // 基准情况：如果传入的节点为空，说明到达了叶子节点的下一层
        if (left == null || right == null) {
            return;
        }
      
        // 1. 连接当前传入的两个节点
        left.next = right;

        // 2. 递归处理子树内部的连接
        // 连接 left 节点自己的左右孩子
        dfs(left.left, left.right);
        // 连接 right 节点自己的左右孩子
        dfs(right.left, right.right);

        // 3. 递归处理跨子树的连接
        // 连接 left 的右孩子和 right 的左孩子
        dfs(left.right, right.left);
    }
}
```

### 优缺点分析

-  **优点** ：代码优雅，不使用额外的队列。对于一个完美的二叉树，其递归深度为 O(log N)，因此空间复杂度为 O(log N)，优于 BFS 的 O(N)

-  **缺点** ：逻辑相对抽象，特别是对跨子树的连接 `dfs(left.right, right.left)` 的理解需要花些时间。并且，这个简单的递归版本高度依赖于“完美二叉树”的结构

---

## 四、 总结与对比

|  | 解法一 (层序遍历 / BFS) | 解法二 (递归 / DFS) |
|:---|:---|:---|
|  **核心思路**  | 使用队列，逐层处理，模拟连接过程 | 分而治之，将连接问题分解为 3 个子问题递归解决 |
|  **时间复杂度**  | O(N) | O(N) |
|  **空间复杂度**  | O(N) (最坏情况下，队列存储最后一层节点) | O(log N) (完美二叉树的递归栈深度) |
|  **代码可读性**  | 高，直观易懂 | 中等，递归逻辑稍显抽象 |
|  **通用性**  |  **高** ，同样适用于非完美二叉树 (LC117) |  **低** ，此简化版递归依赖于完美二叉树的结构 |