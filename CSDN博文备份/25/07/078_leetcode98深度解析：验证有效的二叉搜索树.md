# leetcode98深度解析：验证有效的二叉搜索树

> 原创 已于 2025-07-24 18:48:45 修改 · 公开 · 903 阅读 · 13 · 9 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149612398

**文章目录**

[TOC]


LeetCode 98： [判断一棵给定的二叉树是否是有效的二叉搜索树（BST）](https://leetcode.cn/problems/validate-binary-search-tree/description/) 【难度：中等；通过率：39.6%】，这个问题不仅考察对 **BST 性质** 的理解，更是对二叉树遍历和 **递归** 技巧的实践

## 一、 题目描述

给定一个二叉树的根节点 `root` ，判断其是否是一个有效的二叉搜索树

**有效的二叉搜索树定义如下：** 

> 

- 节点的左子树只包含 **小于** 当前节点的数

- 节点的右子树只包含 **大于** 当前节点的数

- 所有左子树和右子树自身必须也是二叉搜索树

**示例:** 

```
输入:
    2
   / \
  1   3
输出: true

输入:
    5
   / \
  1   4
     / \
    3   6
输出: false
解释: 根节点的值是 5 ，但是其右子节点值为 4 
```

---

## 二、 核心思路：利用 BST 的中序遍历特性

解决这道题，最直观的思路是利用二叉搜索树的一个重要性质： **如果对二叉搜索树进行中序遍历，得到的结果序列是严格升序的。** 

因此，我们可以：

> 

1. 对给定的二叉树进行 **中序遍历** 

2. 在遍历过程中，实时比较当前节点的值和前一个节点的值

3. 如果发现当前节点的值 **小于或等于** 前一个节点的值，那么这棵树就不是 BST，立即返回 `false` 

4. 如果整个中序遍历完成，所有节点都满足升序关系，则说明这是一棵有效的 BST，返回 `true` 

为了实现“实时比较当前节点和前一个节点”，我们需要一个额外的变量来保存中序遍历的“前一个节点”

---

## 三、 代码实现与深度解析

【参考代码】：

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
    // pre 节点：用于保存中序遍历中当前节点的前一个节点
    // 初始为 null，表示尚未处理任何节点
    TreeNode pre = null;

    /**
     * 验证二叉搜索树的有效性
     * 这是一个递归函数，它模拟了二叉树的中序遍历过程
     * @param root 当前需要判断的子树的根节点
     * @return 如果以 root 为根的子树是有效的 BST，则返回 true；否则返回 false
     */
    public boolean isValidBST(TreeNode root) {
        // 1. 递归终止条件 (Base Case):
        // 如果当前节点为空，说明已经遍历到了叶子节点的下方，或者空树
        // 任何空树都被认为是有效的 BST
        if (root == null) {
            return true;
        }

        // 2. 中序遍历的“左”：递归处理左子树
        // 如果左子树本身不是 BST，那么整个树也不是 BST，直接返回 false
        boolean leftIsValid = isValidBST(root.left);
        // 如果 leftIsValid 已经是 false，就没必要继续了，直接返回
        if (!leftIsValid) {
            return false;
        }

        // 3. 中序遍历的“中”：处理当前节点 (root)
        // 这一步是核心的比较逻辑
        // 只有当 pre 不为 null 时才进行比较
        // pre 为 null 意味着这是中序遍历的第一个节点（最左下角的叶子节点），它没有前驱
        if (pre != null) {
            // 比较当前节点的值与前一个节点的值
            // 如果当前节点的值小于或等于前一个节点的值，则不符合 BST 的严格升序特性
            if (root.val <= pre.val) {
                return false;
            }
        }
        // 更新 pre 节点：将当前节点设为下一次递归的“前一个节点”
        // 无论是否进行了比较，pre 都需要更新，以便后续的节点能与当前节点进行比较
        // 这里的 pre 和 root 构成了一对“双指针”，
        // 它们在中序遍历的序列中一步步向前移动，pre 始终指向 root 的前一个元素
        pre = root;

        // 4. 中序遍历的“右”：递归处理右子树
        // 如果右子树本身不是 BST，那么整个树也不是 BST，直接返回 false
        boolean rightIsValid = isValidBST(root.right);

        // 5. 最终结果：
        // 只有当左子树是 BST 且右子树是 BST 且当前节点与前驱节点比较通过时，
        // 整个以 root 为根的子树才是有效的 BST
        return leftIsValid && rightIsValid;
    }
}
```

---

## 四、 关键点与复杂度分析

-  **中序遍历的重要性** ：该解法的核心在于利用了 BST 的中序遍历特性。理解这一点是解决问题的关键

-  **`pre` 指针的妙用** ： `pre` 变量充当了“全局”状态，记录了中序遍历中的前一个节点。它使得我们能够在递归过程中，将当前节点与它的中序前驱进行比较

-  **`return` 逻辑的精细处理** ：

  - 一旦 `root.val <= pre.val` 成立，立即 `return false` ，因为整个树已经不满足 BST 条件

  -  **不能在 `root.val > pre.val` 时立即 `return true`** ，因为这只是当前节点通过了验证，其右子树（以及更深层的节点）尚未验证。必须等待所有递归调用都返回 `true` ，才能最终确定整个树是 BST

-  **时间复杂度** ： **O(N)** 。其中 N 是二叉树的节点数。每个节点都会被访问一次，进行常数次操作

-  **空间复杂度** ： **O(H)** 。其中 H 是二叉树的高度。这主要是递归调用栈的空间开销。最坏情况下（树退化为链表），H 可以达到 N，所以空间复杂度为 O(N)

---

## 五、 总结与拓展

LeetCode98 是一个考察二叉树、递归的题，包含对二叉搜索树性质的理解以及递归遍历二叉树的能力等

通过中序遍历并维护一个 `pre` 指针来验证升序性，是一种高效且优雅的解法