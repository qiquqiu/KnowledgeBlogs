# leetcode105深度解析：从前序与中序遍历序列构造二叉树

> 原创 已于 2025-07-11 22:30:04 修改 · 公开 · 731 阅读 · 15 · 11 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149284147

**文章目录**

[TOC]



## 一、 题目描述

链接： [105. 从前序与中序遍历序列构造二叉树 - 力扣（LeetCode）](https://leetcode.cn/problems/construct-binary-tree-from-preorder-and-inorder-traversal/description/) 

给定一个二叉树的前序遍历 `preorder` 和中序遍历 `inorder` 的结果，请构建该二叉树并返回其根节点。你可以假设树中没有重复的元素。

**示例:** 

```
preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]

返回的二叉树如下：
    3
   / \
  9  20
    /  \
   15   7
```

---

## 二、 核心思路：前序找根，中序划分

解决这个问题的关键在于深刻理解前序遍历和中序遍历的特性：

1.  **前序遍历 (Preorder):** `[根节点, [左子树的前序遍历], [右子树的前序遍历]]` 

   -  **特点：** 序列的 **第一个元素** 永远是当前子树的 **根节点** 。

2.  **中序遍历 (Inorder):** `[[左子树的中序遍历], 根节点, [右子树的中序遍历]]` 

   -  **特点：** **根节点** 总是位于序列的中间，其 **左边** 是所有左子树的节点， **右边** 是所有右子树的节点。

将这两个特性结合起来，我们的重建思路就豁然开朗了：

1. 从 **前序遍历** 中找到根节点（就是第一个元素）。

2. 在 **中序遍历** 中找到这个根节点的位置。

3. 根据根节点在中序遍历中的位置，将中序遍历序列 **划分** 为左子树和右子树两个部分。

4. 同时，根据左子树的节点数量，我们也能在前序遍历序列中确定左、右子树对应的子序列范围。

5. 现在，我们拥有了左子树的前序和中序序列，以及右子树的前序和中序序列。这变成了两个 **规模更小的相同问题** ，完美契合 **递归** 

---

## 三、 图解算法步骤

我们用示例 `preorder = [3,9,20,15,7]` , `inorder = [9,3,15,20,7]` 来走一遍流程。

**第一轮:** 

1.  **找根节点：** `preorder` 的第一个元素是 `3` ，所以树的根节点是 `3` 。

2.  **中序划分：** 在 `inorder` 中找到 `3` ，它的索引是 `1` 。

   -  `inorder` 中 `3` 的左边是 `[9]` ，这是根节点 `3` 的左子树。

   -  `inorder` 中 `3` 的右边是 `[15, 20, 7]` ，这是根节点 `3` 的右子树。

3.  **前序划分：** 

   - 左子树有 `1` 个节点 ( `9` )。所以在 `preorder` 中，根节点 `3` 后面的 `1` 个元素 `[9]` 对应左子树的前序遍历。

   - 右子树有 `3` 个节点 ( `15, 20, 7` )。所以在 `preorder` 中，接下来的 `3` 个元素 `[20, 15, 7]` 对应右子树的前序遍历。

 ![Tree Construction Step 1](./assets/049_1.jpeg)

**递归构建左子树:** 

- 问题变为：用 `preorder = [9]` 和 `inorder = [9]` 构建树。

-  **找根节点：** `preorder` 第一个元素是 `9` 。

-  **中序划分：** `inorder` 中 `9` 的左边和右边都为空。

-  **结论：** 得到一个值为 `9` 的叶子节点，返回给上一层作为 `3` 的左孩子。

**递归构建右子树:** 

- 问题变为：用 `preorder = [20, 15, 7]` 和 `inorder = [15, 20, 7]` 构建树。

-  **找根节点：** `preorder` 第一个元素是 `20` 。

-  **中序划分：** 在 `inorder` 中找到 `20` ，它的左边是 `[15]` ，右边是 `[7]` 。

-  **递归…** 这个过程会继续下去，直到所有节点都构建完成。

---

## 四、 一种代码实现与解析

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
    // 使用 HashMap 存储中序遍历中 <值, 索引> 的映射，用于快速定位根节点
    Map<Integer, Integer> map;

    public TreeNode buildTree(int[] preorder, int[] inorder) {
        // 初始化 HashMap，空间换时间
        map = new HashMap<>();
        for (int i = 0; i < inorder.length; i++) {
            map.put(inorder[i], i);
        }

        // 调用递归函数，传入初始的完整数组范围
        // 区间定义为左闭右开 [begin, end)
        return build(preorder, 0, preorder.length, inorder, 0, inorder.length);
    }

    /**
     * 递归构建子树的核心函数
     * @param preorder 前序遍历数组
     * @param preBegin 当前处理的前序子数组的起始索引（包含）
     * @param preEnd   当前处理的前序子数组的结束索引（不包含）
     * @param inorder  中序遍历数组
     * @param inBegin  当前处理的中序子数组的起始索引（包含）
     * @param inEnd    当前处理的中序子数组的结束索引（不包含）
     * @return 构建好的子树的根节点
     */
    public TreeNode build(int[] preorder, int preBegin, int preEnd, int[] inorder, int inBegin, int inEnd) {
        // 递归终止条件：如果子数组为空，则没有节点需要构建
        if (preBegin >= preEnd || inBegin >= inEnd) {
            return null;
        }

        // 1. 找到根节点：前序遍历的第一个元素就是根
        int rootVal = preorder[preBegin];
        TreeNode root = new TreeNode(rootVal);

        // 2. 在中序遍历中找到根节点的位置
        int rootIndex = map.get(rootVal);

        // 3. 计算左子树的节点数量
        int leftLength = rootIndex - inBegin;

        // 4. 递归构建左子树
        // 前序遍历中，左子树的范围是 [preBegin + 1, preBegin + 1 + leftLength)
        // 中序遍历中，左子树的范围是 [inBegin, rootIndex)
        root.left = build(preorder, preBegin + 1, preBegin + 1 + leftLength,
                inorder, inBegin, rootIndex);

        // 5. 递归构建右子树
        // 前序遍历中，右子树的范围是 [preBegin + 1 + leftLength, preEnd)
        // 中序遍历中，右子树的范围是 [rootIndex + 1, inEnd)
        root.right = build(preorder, preBegin + 1 + leftLength, preEnd,
                inorder, rootIndex + 1, inEnd);

        // 6. 返回当前构建好的根节点
        return root;
    }
}
```

---

## 五、 关键点与复杂度分析

-  **`HashMap` 优化** ：这是本题的性能关键。如果没有 `HashMap` ，每次在中序遍历中查找根节点都需要 O(N) 时间，导致总时间复杂度为 O(N²)。使用 `HashMap` 后，查找变为 O(1)，总时间复杂度降为 **O(N)** 。

-  **区间定义** ： **统一** 采用左闭右开 `[begin, end)` 的区间表示法，代码会更简洁。例如，区间的长度就是 `end - begin` ，空区间的判断就是 `begin >= end` 。

-  **时间复杂度** ： **O(N)** 。其中 N 是树的节点数。 `HashMap` 的构建需要 O(N)，递归 `build` 函数会对每个节点访问一次，每次访问中的操作（查找、计算）都是 O(1) 的。

-  **空间复杂度** ： **O(N)** 。 `HashMap` 占用了 O(N) 的空间。递归调用栈的深度最坏情况下（树退化成链表）也是 O(N)，因此总空间复杂度为 O(N)。
  shMap `的构建需要 O(N)，递归` build` 函数会对每个节点访问一次，每次访问中的操作（查找、计算）都是 O(1) 的。

-  **空间复杂度** ： **O(N)** 。 `HashMap` 占用了 O(N) 的空间。递归调用栈的深度最坏情况下（树退化成链表）也是 O(N)，因此总空间复杂度为 O(N)。

---

## 六、与力扣106题： `从中序与后序遍历序列` 构造二叉树的对比

参考： [https://blog.csdn.net/lyh2004_08/article/details/149284192](https://blog.csdn.net/lyh2004_08/article/details/149284192) 

| 特性 | LeetCode 105 (前序 + 中序) | LeetCode 106 (后序 + 中序) |
|:---|:---|:---|
|  **找根策略**  |  **前序** 遍历的 **第一个** 元素 `preorder[preBegin]`  |  **后序** 遍历的 **最后一个** 元素 `postorder[postEnd - 1]`  |
|  **划分依据**  | 统一使用 **中序** 遍历 | 统一使用 **中序** 遍历 |
|  **递归构建**  |  `root -> left -> right`  |  `root -> left -> right` (或 `root -> right -> left` ) |
|  **难点**  | 计算前序遍历中右子树的起始位置 `preBegin + 1 + lenOfLeft`  | 计算后序遍历中右子树的区间 `[postBegin + lenOfLeft, postEnd - 1]`  |