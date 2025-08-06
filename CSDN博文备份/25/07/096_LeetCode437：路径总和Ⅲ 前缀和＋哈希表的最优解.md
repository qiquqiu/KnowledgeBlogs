# LeetCode437：路径总和Ⅲ 前缀和＋哈希表的最优解

> 原创 于 2025-07-31 07:00:00 发布 · 公开 · 1k 阅读 · 33 · 19 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149788454

**文章目录**

[TOC]


在上一篇中，我们探讨了 [LeetCode 437 路径总和Ⅲ 的双重递归解法](https://blog.csdn.net/lyh2004_08/article/details/149759440) 。虽然该方法思路直观，但效率并非最优。下面我们使用一个更高效、更巧妙的解法： **前缀和 + 哈希表** ，它能将时间复杂度优化到 O(N)

## 一、 题目回顾

给定一个二叉树的根节点 `root` ，和一个整数 `targetSum` ，求该二叉树里节点值之和等于 `targetSum` 的 **路径** 的数目

**路径** 不需要从根节点开始，也不需要在叶子节点结束，但路径方向必须是向下的（只能从父节点到子节点）

**示例:** 

 ![示例 1](./assets/096_1.jpeg)

```
输入: root = [10,5,-3,3,2,null,11,3,-2,null,1], targetSum = 8
输出: 3
```

---

## 二、 核心思路：前缀和 + 哈希表

这个解法来源于数组中的“子数组和”问题。如果我们要在一个数组中找到和为 `target` 的连续子数组，我们可以使用前缀和 `P[i] = nums[0] + ... + nums[i]` 。那么， `nums[j] + ... + nums[i]` 的和就是 `P[i] - P[j-1]` 。如果 `P[i] - P[j-1] == target` ，则意味着 `P[j-1] == P[i] - target` 。我们只需在遍历时，用哈希表记录之前出现过的前缀和及其次数

现在将这个思想迁移到二叉树上：

1.  **路径的定义** ：题目中的路径是“向下”的，这意味着它仍然是树中的一条从某个祖先节点到某个后代节点的链

2.  **前缀和** ：我们定义 `current_path_sum` 为从 **根节点** 到 **当前节点** 的路径上所有节点的和

3.  **目标转换** ：如果从根节点到当前节点 `curNode` 的路径和是 `current_path_sum` ，我们想找到一条以某个祖先节点 `ancestorNode` 为起点，以 `curNode` 为终点的路径，其和为 `targetSum` 
   这意味着： `(从根到 curNode 的和) - (从根到 ancestorNode 父节点的和) == targetSum` 
   即： `current_path_sum - prefix_sum_of_ancestor_parent == targetSum` 
   所以，我们需要查找是否存在一个 `prefix_sum_of_ancestor_parent` 等于 `current_path_sum - targetSum` 

4.  **哈希表的作用** ：我们使用一个 `HashMap<Long, Integer>` 来存储在当前 DFS 路径上，从根节点到 **任意祖先节点** （包括根节点本身）的前缀和以及它们出现的次数

   - 键 (Key)：某个前缀和 `P` 

   - 值 (Value)：该前缀和 `P` 出现的次数

---

## 三、 算法步骤

我们进行一次深度优先搜索 (DFS)，并维护以下信息：

1.  **`current_path_sum`** ：从根节点到当前节点 `node` 的路径和

2.  **`prefixSumCount`** ：一个 `HashMap` ，记录从根节点到当前节点路径上，所有祖先节点（包括根节点）的前缀和出现的次数

**DFS 过程：** 

1.  **进入节点 `node`** ：

   - 更新 `current_path_sum = current_path_sum + node.val` 

   -  **查找目标前缀和** ：在 `prefixSumCount` 中查找是否存在键为 `current_path_sum - targetSum` 的项。如果存在，其对应的值就是以当前节点 `node` 为终点，且路径和为 `targetSum` 的路径数量。将这个数量累加到总结果 `count` 中

   -  **更新哈希表** ：将 `current_path_sum` 加入 `prefixSumCount` ，或者将其计数加 1

2.  **递归子节点** ：

   - 递归调用 `dfs(node.left, current_path_sum, prefixSumCount)` 

   - 递归调用 `dfs(node.right, current_path_sum, prefixSumCount)` 

3.  **回溯** ：

   - 当从 `node` 的子树返回时，为了不影响其他分支的计算，我们需要将 `current_path_sum` 从 `prefixSumCount` 中移除（或者将其计数减 1）。这是 **显式回溯** 的关键一步，确保了哈希表中只包含当前路径上的前缀和信息

**初始状态：** 

- 在 DFS 开始前，我们需要在哈希表中放入一个 `(0, 1)` 的映射。这代表“从根节点上方到根节点”的路径和为 0，且出现 1 次。这样可以处理从根节点开始的路径。例如，如果 `root.val == targetSum` ，那么 `current_path_sum (root.val) - targetSum == 0` ，就能在哈希表中找到 `0` ，从而正确计数

---

## 四、 代码实现与深度解析

【标准代码（含注释讲解）】：

```java
class Solution {
    int targetSum;
    int pathCount = 0; // 存储最终路径数量

    public int pathSum(TreeNode root, int targetSum) {
        this.targetSum = targetSum;
        // 使用 HashMap 存储前缀和及其出现的次数
        // 初始时，前缀和 0 出现 1 次，用于处理从根节点开始的路径
        Map<Long, Integer> prefixSumCount = new HashMap<>();
        prefixSumCount.put(0L, 1); 
      
        // 启动dfs
        dfs(root, 0L, prefixSumCount);
      
        return pathCount;
    }

    /**
     * 深度优先搜索，利用前缀和和哈希表统计路径数量
     * @param node           当前访问的节点
     * @param currentPathSum 从根节点到当前节点（不含）的路径和
     * @param prefixSumCount 存储从根节点到当前节点路径上所有前缀和的出现次数
     */
    private void dfs(TreeNode node, long currentPathSum, Map<Long, Integer> prefixSumCount) {
        // 基准情况：节点为空，返回
        if (node == null) {
            return;
        }

        // 1. 更新当前路径和：从根节点到当前节点的路径和
        currentPathSum += node.val;

        // 2. 查找目标前缀和：
        // 如果 (当前路径和 - targetSum) 存在于哈希表中，
        // 则说明存在一条以某个祖先节点为起点，以当前节点终点的路径，其和为targetSum
        // 将该前缀和对应的次数累加到结果中
        pathCount += prefixSumCount.getOrDefault(currentPathSum - targetSum, 0);

        // 3. 更新哈希表：将当前路径和加入哈希表，或增加其计数
        // 注意：这里先使用再更新，因为当前节点是路径的终点，它的前缀和应该被考虑在内
        prefixSumCount.put(currentPathSum, prefixSumCount.getOrDefault(currentPathSum, 0) + 1);

        // 4. 递归探索左右子树
        dfs(node.left, currentPathSum, prefixSumCount);
        dfs(node.right, currentPathSum, prefixSumCount);

        // 5. 回溯：
        // 在当前节点的所有子树探索完毕后，需要将当前路径和从哈希表中移除（或减计数）
        // 这样可以确保哈希表中只包含当前dfs路径上的前缀和信息，不影响其他分支的计算
        prefixSumCount.put(currentPathSum, prefixSumCount.get(currentPathSum) - 1);
    }
}
```

---

## 五、 关键点与复杂度分析

-  **前缀和思想** ：将“子路径和”问题转化为“两个前缀和的差值”问题

-  **哈希表优化** ： `HashMap` 提供了 O(1) 的查找和更新，使得我们能够高效地检查是否存在目标前缀和

-  **`long` 类型** ：使用 `long` 类型来存储 `currentPathSum` 和 `targetSum` ，以防止节点值累加后可能出现的整数溢出

-  **`prefixSumCount.put(0L, 1);`** ：这是处理从根节点开始的路径的关键一步。它使得当 `currentPathSum == targetSum` 时， `currentPathSum - targetSum` 等于 `0` ，从而在哈希表中找到对应的计数

-  **回溯时的哈希表操作** ： `prefixSumCount.put(currentPathSum, prefixSumCount.get(currentPathSum) - 1);` 是显式回溯的体现。它确保了当我们从一个节点返回到其父节点时，哈希表的状态是正确的，只包含父节点路径上的前缀和

-  **时间复杂度** ： **O(N)** 每个节点只被访问一次，每次访问在哈希表中的操作（插入、查找、删除）都是平均 O(1) 的

-  **空间复杂度** ： **O(H)** 主要由递归栈的深度决定，以及哈希表的最大大小。在最坏情况下（树退化为链表），哈希表的大小和递归栈的深度都为 O(N)

---

## 六、 路径总和“三部曲”终极对比分析

将 LC437 的最优解法也纳入对比，全面回顾“路径总和”系列问题

|  | LeetCode 112 (路径总和) | LeetCode 113 (路径总和 II) | LeetCode 437 (路径总和 III) |
|:---|:---|:---|:---|
|  **路径定义**  |  **必须从根到叶**  |  **必须从根到叶**  |  **任意节点到其后代节点**  |
|  **问题目标**  |  **判断存在性** (返回 `boolean` ) |  **记录所有满足条件的路径** (返回 `List<List<Integer>>` ) |  **统计所有满足条件的路径数量** (返回 `int` ) |
|  **核心算法**  | 单次 DFS + (隐式)回溯 | 单次 DFS + **显式回溯**  |  **前缀和 + 哈希表** (最优解) |
|  **回溯方式**  | 隐式回溯 (传递 `int` ) | 显式回溯 (传递 `List` ，需 `remove` ) | 显式回溯 (哈希表 `put` / `get` / `remove` ) |
|  **剪枝**  |  **可以** (找到一个即可) |  **不可以** (需找到所有) |  **不可以** (需统计所有) |
|  **核心逻辑**  | 在叶子节点判断 `sum == target`  | 在叶子节点判断后， **深拷贝** 路径 `new ArrayList<>(path)`  | 查找 `current_path_sum - targetSum` 在哈希表中的计数 |
|  **时间复杂度**  | O(N) | O(N*H) |  **O(N)**  |
|  **空间复杂度**  | O(H) | O(H) | O(H) (哈希表和递归栈) |


**总结：** 

-  **LC112** 入门二叉树 DFS 和路径求和

-  **LC113** 掌握显式回溯和如何收集所有路径

-  **LC437** 引入了“任意路径”的概念，迫使我们思考更高级的优化技巧，如 **前缀和** 。前缀和思想在数组、链表、树等数据结构中处理“子序列/子路径和”问题时非常强大，值得深入学习和掌握

