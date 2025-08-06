# leetcode102：二叉树的层序遍历（队列实现）

> 原创 于 2025-07-25 17:37:05 发布 · 公开 · 565 阅读 · 15 · 6 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149644738

**文章目录**

[TOC]


二叉树遍历中的另一种重要遍历方式—— **层序遍历** ，对应的就是 LeetCode [102. 二叉树的层序遍历 - 力扣（LeetCode）](https://leetcode.cn/problems/binary-tree-level-order-traversal/description/) 【难度：中；通过率：70.0%】

层序遍历与我们之前接触的前序、中序、后序遍历（深度优先搜索 DFS）不同，它属于 **广度优先搜索 (BFS)** 的范畴，代码并 **不涉及“递归”** ，所以理论上是更好且更容易理解

## 一、题目描述

给你二叉树的根节点 `root` ，返回其节点值的层序遍历。（即逐层地，从左到右访问所有节点）

**示例:** 

```
输入: root = [3,9,20,null,null,15,7]
输出: [[3],[9,20],[15,7]]
```

 ![示例 1](./assets/083_1.jpeg)

---

## 二、 核心思路：队列 (Queue) 实现 BFS

层序遍历的核心思想是： **一层一层地访问节点** 。这与 **队列（Queue）的“先进先出”特性** 完美契合

具体步骤如下：

1.  **初始化** ：

   - 创建一个 `List<List<Integer>>` 来存储最终结果，其中每个 `List<Integer>` 代表一层

   - 创建一个 `Queue<TreeNode>` ，用于存储待访问的节点

2.  **起始** ：将根节点 `root` 加入队列

3.  **循环遍历** ：

   - 当队列不为空时，循环继续

   -  **获取当前层的节点数量** ：在每次循环开始时，记录下当前队列中元素的数量 `len = queue.size()` 。这个 `len` 就是当前层的所有节点数

   -  **遍历当前层** ：使用一个 `for` 循环，迭代 `len` 次：

     - 从队列中取出一个节点 `node` 

     - 将 `node.val` 加入到当前层的结果列表中

     - 如果 `node` 有左孩子，将 `node.left` 加入队列

     - 如果 `node` 有右孩子，将 `node.right` 加入队列

   - 当前层的所有节点处理完毕后，将当前层的结果列表加入到最终结果列表中

---

## 三、 代码实现与深度解析

【一种参考代码】：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    // 二叉树的层序遍历
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> res = new ArrayList<>(); // 存储最终结果，每个子列表代表一层

        // 如果根节点为空，直接返回空列表，因为没有节点可以遍历
        if (root == null) {
            return res;
        }

        // 主要用增删操作，我们选择 LinkedList 作为 Queue 的实现
        // Queue 是一个接口，LinkedList 实现了它，提供了队列的特性（先进先出，FIFO）
        Queue<TreeNode> queue = new LinkedList<>();
      
        // 将根节点加入队列，作为层序遍历的起点
        queue.add(root);

        // 当队列不为空时，循环继续，表示还有节点待访问
        while (!queue.isEmpty()) {
            // 创建一个列表来存储当前层的所有节点值
            List<Integer> level = new ArrayList<>();
          
            // 记录当前队列的大小，这个大小就是当前层的节点数量
            // 关键点：在开始遍历当前层之前固定住这一层的节点数量
            int currentLevelSize = queue.size(); 
          
            // 遍历当前层的所有节点
            for (int i = 0; i < currentLevelSize; i++) {
                // 从队列头部取出节点（先进先出）
                TreeNode node = queue.poll();
              
                // 将当前节点的值加入到当前层的列表中
                // 由于我们已经处理了 root == null 的情况，并且只在 node 不为 null 时才加入其子节点，
                // 所以这里的 node 理论上不会是 null
                level.add(node.val);

                // 如果当前节点有左孩子，将其加入队列尾部，等待下一轮（下一层）处理
                if (node.left != null) {
                    queue.add(node.left);
                }
                // 如果当前节点有右孩子，将其加入队列尾部，等待下一轮（下一层）处理
                if (node.right != null) {
                    queue.add(node.right);
                }
            }
            // 当前层的所有节点都已处理完毕并加入 level 列表，将其添加到最终结果中
            res.add(level);
        }
      
        // 返回包含所有层节点值的列表
        return res;
    }
}
```

---

## 四、 关键点与复杂度分析

-  **广度优先搜索 (BFS)** ：层序遍历是 BFS 在二叉树上的应用。BFS 的核心是使用队列

-  **队列的作用** ：队列用于存储“下一批要访问的节点”。它确保我们总是先访问同一层的节点，然后才进入下一层

-  **分层处理的技巧** ：在 `while` 循环内部，通过 `int currentLevelSize = queue.size();` 来锁定当前层的节点数量，然后通过一个 `for` 循环精确地处理当前层的所有节点，这是实现按层输出的关键

-  **时间复杂度** ： **O(N)** 其中 N 是二叉树的节点数。每个节点都会被访问一次，并被加入和移出队列一次

-  **空间复杂度** ： **O(W)** 其中 W 是二叉树的最大宽度。在最坏情况下（例如，满二叉树的最后一层），队列中可能会存储大约 N/2 个节点，所以空间复杂度可以认为是 O(N)

