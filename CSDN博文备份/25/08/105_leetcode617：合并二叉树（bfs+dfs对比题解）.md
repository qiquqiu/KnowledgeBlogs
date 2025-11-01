# leetcode617：合并二叉树（bfs+dfs对比题解）

> 原创 于 2025-08-04 08:00:00 发布 · 公开 · 831 阅读 · 12 · 16 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149887497

**文章目录**

[TOC]


[LeetCode 617：合并两个二叉树](https://leetcode.cn/problems/merge-two-binary-trees/) ，【难度：简单；通过率：79.8%】，这道题是理解树的递归和迭代遍历例题，我们将通过两种核心方法——深度优先搜索 (dfs) 和广度优先搜索 (bfs) 来深入剖析它

## 一、 题目描述

给你两棵二叉树： `root1` 和 `root2` 

想象一下，当你将其中一棵覆盖到另一棵之上时，两棵树上的一些节点将会重叠（而另一些不会）。你需要将这两棵树合并成一棵新二叉树。合并的规则是：如果两个节点重叠，那么将这两个节点的值相加作为合并后节点的新值；否则，不为 `null` 的节点将直接作为新二叉树的节点

返回合并后的二叉树

**注意:** 合并过程必须从两个树的根节点开始

**示例:** 
 ![示例 1](./assets/105_1.jpeg)

```
输入: root1 = [1,3,2,5], root2 = [2,1,3,null,4,null,7]
输出: [3,4,5,5,4,null,7]
```

---

## 解法一：深度优先搜索 (dfs)

这是解决树相关问题时最常见、最优雅的思路。我们将合并操作分解为对当前节点的操作，然后递归地对左右子树进行相同的合并操作

### 核心思路

函数的任务是返回合并后的 `(子)树` 

1.  **基准情况** ：

   - 如果 `root1` 为空，说明在当前位置没有可合并的节点，直接返回 `root2` 作为结果

   - 如果 `root2` 为空，同理，直接返回 `root1` 

2.  **合并当前节点** ：如果 `root1` 和 `root2` 都不为空，我们将 `root2` 的值加到 `root1` 上。我们选择直接在 `root1` 上修改，以节省空间

3.  **递归合并子树** ：

   - 递归调用 `mergeTrees(root1.left, root2.left)` ，将合并后的左子树结果赋值给 `root1.left` 

   - 递归调用 `mergeTrees(root1.right, root2.right)` ，将合并后的右子树结果赋值给 `root1.right` 

4.  **返回结果** ：返回修改后的 `root1` 

### 代码实现

```java
class Solution {
    public TreeNode mergeTrees(TreeNode root1, TreeNode root2) {
        // 1. 基准情况：如果一个节点为空，则返回另一个节点
        if (root1 == null) {
            return root2;
        }
        if (root2 == null) {
            return root1;
        }

        // 2. 合并当前节点的值 (在 root1 上修改)
        root1.val += root2.val;

        // 3. 递归地合并左右子树
        root1.left = mergeTrees(root1.left, root2.left);
        root1.right = mergeTrees(root1.right, root2.right);

        // 4. 返回合并后的根节点
        return root1;
    }
}
```

### 优缺点分析

-  **优点** ：代码极其 **简洁** 、优雅，完美地体现了递归和分治思想

-  **缺点** ：当树非常深时，递归调用栈可能会很深，有栈溢出的风险（实际可以忽略，并且是力扣此题的最佳解法）。空间复杂度为 O(H)

---

## 解法二：广度优先搜索 (bfs)

如果我们想避免深度递归，可以使用迭代的方式，通过 **队列** 进行广度优先搜索；这种解法也行，只不过需要使用队列，步骤稍显繁琐，也并非力扣该题的最佳解法

### 核心思路

1.  **初始化** ：创建一个队列，用于存放需要同步处理的节点 **对** `[node1, node2]` 

2.  **处理边界** ：如果任意一个根节点为空，直接返回另一个。如果都不为空，将 `[root1, root2]` 加入队列

3.  **循环合并** ：当队列不为空时，循环执行：
   a. 从队列中取出一对节点 `[node1, node2]` 
   b. 合并它们的值： `node1.val += node2.val` 
   c. **智能处理子节点** ：检查它们的左右孩子
   * 如果 `node1` 和 `node2` 的左孩子都存在，将 `[node1.left, node2.left]` 加入队列，等待后续合并
   * 如果只有 `node2` 的左孩子存在，说明 `node1` 在该位置没有节点，直接将 `node2` 的左子树“嫁接”到 `node1` 上即可 ( `node1.left = node2.left` )
   * 对右孩子执行完全相同的逻辑

### 代码实现

```java
class Solution {
    public TreeNode mergeTrees(TreeNode root1, TreeNode root2) {
        if (root1 == null) return root2;
        if (root2 == null) return root1;

        Queue<TreeNode[]> queue = new LinkedList<>();
        queue.offer(new TreeNode[]{root1, root2});

        while (!queue.isEmpty()) {
            TreeNode[] nodes = queue.poll();
            TreeNode node1 = nodes[0];
            TreeNode node2 = nodes[1];

            // 因为能进入队列的 node2 必不为 null，所以直接合并
            node1.val += node2.val;

            // 处理左子节点
            if (node1.left != null && node2.left != null) {
                queue.offer(new TreeNode[]{node1.left, node2.left});
            } else if (node1.left == null) {
                node1.left = node2.left;
            }

            // 处理右子节点
            if (node1.right != null && node2.right != null) {
                queue.offer(new TreeNode[]{node1.right, node2.right});
            } else if (node1.right == null) {
                node1.right = node2.right;
            }
        }
        return root1;
    }
}
```

### 优缺点分析

-  **优点** ：使用迭代，避免了递归栈溢出的风险。思路稳健

-  **缺点** ：代码相对递归版本要繁琐一些。空间复杂度为 O(W)，W 是树的最大宽度

---

## 四、 总结与对比

|  | 解法一 (dfs / 递归) | 解法二 (bfs/ 迭代) |
|:---|:---|:---|
|  **核心思路**  | 分而治之，自上而下递归合并 | 逐层处理，使用队列同步遍历节点对 |
|  **数据结构**  | 隐式调用栈 | 显式队列 (Queue) |
|  **代码风格**  |  **简洁、优雅**  |  **稳健、繁琐**  |
|  **空间复杂度**  | O(H) (H 是树的最大高度) | O(W) (W 是树的最大宽度) |
|  **适用场景**  | 大多数树问题，当代码简洁性是首要考虑时 | 树的深度极大，或需要按“层”处理问题时 |