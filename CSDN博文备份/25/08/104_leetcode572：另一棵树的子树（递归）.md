# leetcode572：另一棵树的子树（递归）

> 原创 于 2025-08-04 07:45:00 发布 · 公开 · 573 阅读 · 17 · 12 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149887449

**文章目录**

[TOC]


[LeetCode 572：判断一棵树是否是另一棵树的子树](https://leetcode.cn/problems/subtree-of-another-tree/description/) ，【难度：简答；通过率：49.1%】，这道题将两个不同的递归函数嵌套在一起，形成一种“交错递归”或“套娃递归”的模式

## 一、 题目描述

给你两棵二叉树 `root` 和 `subRoot` 。检验 `root` 中是否包含和 `subRoot` 具有相同结构和节点值的子树。如果存在，返回 `true` ；否则，返回 `false` 

树 `s` 的一个子树包括 `s` 的一个节点 `c` 和这个节点 `c` 的所有后代。树 `s` 也可以看做它自身的一个子树

**示例:** 

**示例 1:** 
 ![示例 1](./assets/104_1.png)

```
输入: root = [3,4,5,1,2], subRoot = [4,1,2]
输出: true
```

**示例 2:** 
 ![示例 2](./assets/104_2.png)

```
输入: root = [3,4,5,1,2,null,null,null,null,0], subRoot = [4,1,2]
输出: false
```

---

## 二、 核心思路：双重递归，职责分离

解决这个问题的关键在于将问题分解。我们要判断“ `subRoot` 是否是 `root` 的子树”，可以分解为以下三个 **子问题** ：

1. 当前 `root` 树本身是否 **和** `subRoot` **完全相同** ？

2. 如果不同，那么 `subRoot` 是否是 `root` 的 **左子树** 的子树？

3. 如果还不同，那么 `subRoot` 是否是 `root` 的 **右子树** 的子树？

只要这三个条件中有一个成立，那么答案就是 `true` 。这个 “OR” 的逻辑关系，天然地指向了递归

这引导我们设计两个不同职责的递归函数：

-  **`isSubtree(root, subRoot)`** ：主函数，它的任务是在 `root` 树中 **寻找** 一个与 `subRoot` 根节点值相同的节点作为“疑似起点”。它负责“遍历” `root` 树

-  **`isSameTree(p, q)`** ：辅助函数（与 [leetcode100.相同的树（递归练习题）](https://blog.csdn.net/lyh2004_08/article/details/149612742) 完全相同），它的任务是 **比较** 两个给定的节点，判断以它们为根的两棵树是否在结构和节点值上完全相同。它负责“匹配”

---

## 三、 代码实现与深度解析

【参考解法】

```java
class Solution {
    /**
     * 判断 subRoot 是否是 root 的子树
     * 这是一个“遍历”递归
     */
    public boolean isSubtree(TreeNode root, TreeNode subRoot) {
        // 基准情况 1: 如果主树为空，子树必然不存在（除非子树也为空）
        if (root == null) {
            return subRoot == null;
        }
        // 基准情况 2: 如果子树为空，根据定义，空树是任何树的子树
        if (subRoot == null) {
            return true;
        }

        // 核心逻辑：分解为三个“或”关系的子问题
        // 1. 当前 root 和 subRoot 是否是相同的树？
        // 2. 或者，subRoot 是否是 root.left 的子树？
        // 3. 或者，subRoot 是否是 root.right 的子树？
        return isSameTree(root, subRoot) || 
               isSubtree(root.left, subRoot) || 
               isSubtree(root.right, subRoot);
    }

    /**
     * 辅助函数：判断两棵树是否完全相同 (同 LeetCode 100)
     * 这是一个“匹配”递归
     */
    public boolean isSameTree(TreeNode r1, TreeNode r2) {
        // 基准情况 1: 如果一个为空，另一个不为空，则不同
        if (r1 == null || r2 == null) {
            // 只有在两者都为空时才返回 true
            return r1 == null && r2 == null;
        }

        // 基准情况 2: 如果两个节点值不同，则不同
        if (r1.val != r2.val) {
            return false;
        }

        // 递归比较左右子树
        return isSameTree(r1.left, r2.left) && isSameTree(r1.right, r2.right);
    }
}
```

---

## 四、 关键点与复杂度分析

-  **递归的嵌套** ： `isSubtree` 函数调用了 `isSameTree` ，同时又递归地调用了自身。这种结构是本题的精髓，清晰地分离了“寻找起点”和“判断相同”两个职责

-  **`isSameTree` 的重要性** ：它是整个算法的基石。必须能准确地判断两棵树是否完全相同

-  **短路效应** ： `||` (逻辑或) 操作符具有短路效应。一旦 `isSameTree(root, subRoot)` 返回 `true` ，后续的递归调用将不会执行，这在一定程度上提高了效率

-  **时间复杂度** ： **O(M * N)** 其中 M 是 `root` 树的节点数，N 是 `subRoot` 树的节点数。在最坏的情况下（例如， `root` 树的每个节点都与 `subRoot` 的根节点值相同），我们需要对 `root` 树的每个节点都进行一次 `isSameTree` 的比较，而每次比较需要遍历 `subRoot` 的所有节点

-  **空间复杂度** ： **O(H)** 其中 H 是 `root` 树的高度。这主要取决于 `isSubtree` 递归调用的栈深度

