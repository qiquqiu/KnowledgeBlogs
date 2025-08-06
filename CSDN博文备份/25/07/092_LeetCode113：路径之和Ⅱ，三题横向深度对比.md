# LeetCode113：路径之和Ⅱ，三题横向深度对比

> 原创 于 2025-07-29 07:00:00 发布 · 公开 · 1k 阅读 · 22 · 29 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149697934

**文章目录**

[TOC]


在解决了“ [判断路径是否存在](https://leetcode.cn/problems/path-sum/description/) ”(LC112) 和“ [找出所有路径的字符串](https://leetcode.cn/problems/binary-tree-paths/description/) ”(LC257) 之后，还有一道前两者 **思路结合** 的进阶中等题—— [LeetCode 113 (路径总和 II)](https://leetcode.cn/problems/path-sum-ii/description/) ，【难度：中等；通过率：64.0%】

> 

- 112题要求判断路径和为 `targetSum` 的路径 **是否存在** ，而113是要我们直接找到该路径和为 `targetSum` 的 **路径** 
> 
> 

- 这道题要求我们找出所有满足特定和的路径，是练习二叉树深度优先搜索（DFS）和显式回溯的经典题目
> 
> 

## 一、 题目描述

给你二叉树的根节点 `root` 和一个整数目标和 `targetSum` ，找出所有 **从根节点到叶子节点** 路径总和等于 `targetSum` 的路径

**叶子节点** 是指没有子节点的节点

**示例:** 

**示例 1:** 
 ![示例 1](./assets/092_1.jpeg)

```
输入: root = [5,4,8,11,null,13,4,7,2,null,null,5,1], targetSum = 22
输出: [[5,4,11,2],[5,8,4,5]]
```

**示例 2:** 
 ![示例 2](./assets/092_2.jpeg)

```
输入: root = [1,2,3], targetSum = 5
输出: []
```

---

## 二、 核心思路：深度优先搜索+ 显式回溯

这道题的目标是 **找出所有** 满足条件的路径，这决定了我们的基本框架：

1.  **深度优先搜索 (DFS)** ：我们需要遍历从根到叶子的每一条路径

2.  **路径记录** ：在遍历过程中，需要一个列表（如 `ArrayList` ）来记录当前走过的路径

3.  **和的校验** ：当到达一个叶子节点时，检查当前路径上所有节点的和是否等于 `targetSum` 

4.  **显式回溯** ：由于我们需要探索所有可能性，当一个节点的所有子树都探索完毕后，需要将该节点从当前路径中“移除”，以便返回到父节点，去探索其他分支。这个“移除”操作就是回溯，对于节点值的和以及对于路径列表均有体现

---

## 三、 代码实现与深度解析

【一种参考代码】：

```java
class Solution {
    int target; // 目标整数
    List<List<Integer>> res = new ArrayList<>(); // 结果集

    public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
        if (root == null) {
            return res;
        }
        target = targetSum;
        // 还是一样的思路，只不过终止条件多了一个
        dfs(0, new ArrayList(), root);
        return res;
    }

    /**
     * 满足则收集结果，否则递归+回溯
     * @param curSum 遍历到当前节点之前的总和
     * @param path 遍历到当前节点之前的路径
     * @param curNode 即将遍历的“当前节点”
     */
    public void dfs(int curSum, List<Integer> path, TreeNode curNode) {
        // 追加当前路径
        path.add(curNode.val);
        // 终止条件校验：是叶子结点，且路径和满足
        if (curSum + curNode.val == target && curNode.left == null && curNode.right == null) {
            res.add(new ArrayList(path)); // 深拷贝
            // 收集完毕结果，返回前删掉当前层，用于回溯上一层
            path.remove(path.size() - 1);
            return;
        }

        // 搜索当前节点curNode的左子树
        if (curNode.left != null) {
            dfs(curSum + curNode.val, path, curNode.left);
        }

        // 搜索右子树
        if (curNode.right != null) {
            dfs(curSum + curNode.val, path, curNode.right);
        }

        // 回溯搜索子树后的path结果
        path.remove(path.size() - 1);
    }
}
```

---

## 四、 关键点与复杂度分析

-  **显式回溯** ： `path.remove(path.size() - 1);` 是整个算法的精髓。它确保了在探索完一个分支后，状态能够正确地恢复，不影响其他分支的探索

-  **深拷贝路径** ： `res.add(new ArrayList<>(path));` 是另一个核心。由于 `path` 列表在整个递归过程中只有一个实例（引用传递），我们必须在找到有效路径时，将其 **内容拷贝** 一份存入结果集，否则最终结果集里的所有路径都会指向同一个被清空了的 `path` 对象

-  **时间复杂度** ： **O(N*H)** 其中 N 是节点数，H 是树的高度。在最坏情况下（例如一个完整的二叉树），我们需要遍历所有 N 个节点，并且在到达每个叶子节点时，需要 O(H) 的时间来复制路径

-  **空间复杂度** ： **O(H)** 这主要取决于递归调用的深度以及 `path` 列表的大小，它们都与树的高度 H 成正比

---

## 五、 三题对比：LC112，LC257，LC113

这三道题是二叉树路径问题的“三部曲”，层层递进，展示了递归和回溯思想的演变

blog： [LeetCode 112 (路径总和)与LeetCode 257 (二叉树的所有路径)对比讲解参考](https://blog.csdn.net/lyh2004_08/article/details/149669777) 

|  | LeetCode 112 (路径总和) | LeetCode 257 (二叉树的所有路径) | LeetCode 113 (路径总和 II) |
|:---|:---|:---|:---|
|  **问题目标**  |  **判断存在性** (返回 `boolean` ) |  **记录所有路径** (返回 `List<String>` ) |  **记录所有满足条件的路径** (返回 `List<List<Integer>>` ) |
|  **回溯方式**  |  **隐式回溯** (Implicit Backtracking)
- 传递基本类型 `int`  |  **显式回溯** (Explicit Backtracking)
- 传递引用类型 `List` ，必须手动 `remove`  |  **显式回溯** (Explicit Backtracking)
- 传递引用类型 `List` ，必须手动 `remove`  |
|  **剪枝 (Pruning)**  |  **可以剪枝** 
- 找到一个解后即可停止搜索。 |  **不可剪枝** 
- 必须遍历整棵树找到所有解 |  **不可剪枝** 
- 必须遍历整棵树找到所有满足条件的解 |
|  **核心逻辑**  | 在叶子节点判断 `sum == target`  | 在叶子节点 **构建并保存** 路径字符串 | 在叶子节点判断 `sum == target` ，然后 **构建并保存** 路径列表 |
|  **结果处理**  | 设置 `boolean` 标志 | 构建 `String` 并加入结果集 |  **深拷贝** `List` ( `new ArrayList<>(path)` ) 并加入结果集 |


**总结：** 

从 LC112 的“是否存在”，到 LC257 的“记录所有路径”，再到 LC113 的“记录所有满足条件的路径”，我们看到问题变得越来越具体，对回溯的要求也越来越高

-  **LC112** 是入门，让我们理解了 DFS 和隐式回溯

-  **LC257** 是进阶，引入了显式回溯和路径记录的概念

-  **LC113** 是融合，将前两者的逻辑结合起来，并强调了在处理引用类型结果时“深拷贝”的重要性

