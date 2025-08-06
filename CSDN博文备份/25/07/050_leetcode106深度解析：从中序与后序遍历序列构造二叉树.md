# leetcode106深度解析：从中序与后序遍历序列构造二叉树

> 原创 于 2025-07-11 22:27:51 发布 · 公开 · 1k 阅读 · 26 · 25 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149284192

**文章目录**

[TOC]



## 一、 题目回顾

链接： [106. 从中序与后序遍历序列构造二叉树 - 力扣（LeetCode）](https://leetcode.cn/problems/construct-binary-tree-from-inorder-and-postorder-traversal/description/) 

给定一个二叉树的中序遍历 `inorder` 和后序遍历 `postorder` 的结果，请构建该二叉树并返回其根节点。你可以假设树中没有重复的元素。

**示例:** 

```
inorder  = [9, 3, 15, 20, 7]
postorder = [9, 15, 7, 20, 3]

返回的二叉树如下：
    3
   / \
  9  20
    /  \
   15   7
```

---

## 二、 核心思路：后序找根，中序划分

如果你已经理解了 [LC105](https://blog.csdn.net/lyh2004_08/article/details/149284147) 的“前序找根，中序划分”，那么解决这道题的思路转换就非常自然了。我们只需要把目光从前序遍历转移到后序遍历。

1.  **后序遍历 (Postorder):** `[[左子树的后序遍历], [右子树的后序遍历], 根节点]` 

   -  **特点：** 序列的 **最后一个元素** 永远是当前子树的 **根节点** 。

2.  **中序遍历 (Inorder):** `[[左子树的中序遍历], 根节点, [右子树的中序遍历]]` 

   -  **特点：** (保持不变) **根节点** 划分了左、右子树。

我们的新策略是：

> 

1. 从 **后序遍历** 的末尾找到根节点。

2. 在 **中序遍历** 中找到这个根节点的位置，从而划分出左、右子树。

3. 根据左子树的节点数量，在 **后序遍历** 序列中确定左、右子树对应的子序列范围。

4. 这又形成了两个规模更小的相同问题，继续 **递归** 解决！

---

## 三、 图解算法步骤

我们用示例 `inorder = [9,3,15,20,7]` , `postorder = [9,15,7,20,3]` 来走一遍流程。

**第一轮:** 

1.  **找根节点：** `postorder` 的最后一个元素是 `3` ，所以树的根节点是 `3` 。

2.  **中序划分：** 在 `inorder` 中找到 `3` 。

   -  `inorder` 中 `3` 的左边是 `[9]` (左子树)。

   -  `inorder` 中 `3` 的右边是 `[15, 20, 7]` (右子树)。

3.  **后序划分 (关键步骤):** 

   - 左子树有 `1` 个节点 ( `9` )。后序遍历的顺序是“左-右-根”，所以 `postorder` 的 **前 `1` 个元素** `[9]` 对应左子树的后序遍历。

   - 右子树有 `3` 个节点 ( `15, 20, 7` )。所以在 `postorder` 中， **接在左子树后面的 `3` 个元素** `[15, 7, 20]` 对应右子树的后序遍历。

 ![Tree Construction Step 2](./assets/050_1.jpeg)

**递归构建左子树:** 

- 问题变为：用 `inorder = [9]` 和 `postorder = [9]` 构建树。

-  **结论：** 得到一个值为 `9` 的叶子节点，返回。

**递归构建右子树:** 

- 问题变为：用 `inorder = [15, 20, 7]` 和 `postorder = [15, 7, 20]` 构建树。

-  **找根节点：** `postorder` 的最后一个元素是 `20` 。

-  **中序划分：** 在 `inorder` 中找到 `20` ，左边是 `[15]` ，右边是 `[7]` 。

-  **后序划分：** 左子树有1个节点，所以 `postorder` 的 `[15]` 是左子树；右子树有1个节点，所以 `[7]` 是右子树。

-  **继续递归…** 直到所有节点构建完成。

---

## 四、 一种代码实现与深度解析

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    // 依然是引入 HashMap 优化，用于 O(1) 查找根节点在中序遍历中的位置
    Map<Integer, Integer> map;

    public TreeNode buildTree(int[] inorder, int[] postorder) {
        // 预处理中序遍历，将 <值, 索引> 存入 HashMap
        map = new HashMap<>();
        for (int i = 0; i < inorder.length; i++) {
            map.put(inorder[i], i);
        }

        // 调用递归函数，初始范围为整个数组
        // 区间标准统一为左闭右开 [begin, end)
        return findNode(inorder, 0, inorder.length, postorder, 0, postorder.length);
    }

    /**
     * 递归构建子树的核心函数
     * @param inorder  中序遍历数组
     * @param inBegin  当前处理的中序子数组的起始索引（包含）
     * @param inEnd    当前处理的中序子数组的结束索引（不包含）
     * @param postorder 后序遍历数组
     * @param postBegin 当前处理的后序子数组的起始索引（包含）
     * @param postEnd   当前处理的后序子数组的结束索引（不包含）
     * @return 构建好的子树的根节点
     */
    public TreeNode findNode(int[] inorder, int inBegin, int inEnd, int[] postorder, int postBegin, int postEnd) {
        // 基准情况: 当区间为空，无法形成节点，返回 null
        // 这个条件覆盖了所有递归终止的场景，无需为 size=1 的情况单独处理。
        if (inBegin >= inEnd || postBegin >= postEnd) {
            return null;
        }

        // 1. 找到根节点：后序遍历区间的最后一个元素
        int rootVal = postorder[postEnd - 1];
        TreeNode root = new TreeNode(rootVal);

        // 2. 在中序遍历中找到根节点的位置
        int rootIndex = map.get(rootVal);

        // 3. 计算左子树的节点数量，这是划分后序数组的关键
        int lenOfLeft = rootIndex - inBegin;

        // 4. 分治思想：递归构建左子树
        // 中序遍历的左子树区间: [inBegin, rootIndex)
        // 后序遍历的左子树区间: 从 postBegin 开始，长度为 lenOfLeft
        // 即 [postBegin, postBegin + lenOfLeft)
        root.left = findNode(inorder, inBegin, rootIndex,
                postorder, postBegin, postBegin + lenOfLeft);

        // 5. 分治思想：递归构建右子树
        // 中序遍历的右子树区间: [rootIndex + 1, inEnd)
        // 后序遍历的右子树区间: 紧跟在左子树之后，且在根节点之前
        // 即 [postBegin + lenOfLeft, postEnd - 1)
        root.right = findNode(inorder, rootIndex + 1, inEnd,
                postorder, postBegin + lenOfLeft, postEnd - 1);

        // 6. 返回当前构建好的（子）树的根节点
        return root;
    }
}
```

---

## 五、 关键点与复杂度分析

-  **后序数组的切分** ：这是本题相较于 LC105 最需要注意的地方。务必记住后序遍历是 `[左, 右, 根]` 。

  - 左子树的后序部分从 `postBegin` 开始，长度为 `lenOfLeft` 。

  - 右子树的后序部分从 `postBegin + lenOfLeft` 开始，结束于根节点之前，即 `postEnd - 1` 。

-  **递归调用顺序** ：在代码中，我们先递归构建 `root.left` ，再构建 `root.right` 。其实这个顺序可以颠倒，只要你传递给递归函数的区间参数是正确的，最终都能构建出同一棵树。这正是分治思想的魅力所在—— **子问题之间相互独立** 。

-  **时间复杂度** ： **O(N)** 。N 是节点数。构建 `HashMap` 为 O(N)，之后每个节点只被访问和创建一次，单次操作为 O(1)。

-  **空间复杂度** ： **O(N)** 。 `HashMap` 占 O(N)，递归栈深度最坏情况下（链表）也为 O(N)。

---

## 六、 总结与对比 (LC105 vs LC106)

| 特性 | LeetCode 105 (前序 + 中序) | LeetCode 106 (后序 + 中序) |
|:---|:---|:---|
|  **找根策略**  |  **前序** 遍历的 **第一个** 元素 `preorder[preBegin]`  |  **后序** 遍历的 **最后一个** 元素 `postorder[postEnd - 1]`  |
|  **划分依据**  | 统一使用 **中序** 遍历 | 统一使用 **中序** 遍历 |
|  **递归构建**  |  `root -> left -> right`  |  `root -> left -> right` (或 `root -> right -> left` ) |
|  **难点**  | 计算前序遍历中右子树的起始位置 `preBegin + 1 + lenOfLeft`  | 计算后序遍历中右子树的区间 `[postBegin + lenOfLeft, postEnd - 1]`  |


ft -> right `(或` root -> right -> left `) | | **难点** | 计算前序遍历中右子树的起始位置 ` preBegin + 1 + lenOfLeft `| 计算后序遍历中右子树的区间` [postBegin + lenOfLeft, postEnd - 1]` |

> 这两道题关键不仅是一种递归思路和对二叉树的熟悉，更多的是一种**“分治”**思想