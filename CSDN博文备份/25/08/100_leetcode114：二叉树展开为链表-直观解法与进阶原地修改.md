# leetcode114：二叉树展开为链表-直观解法与进阶原地修改

> 原创 于 2025-08-02 07:15:00 发布 · 公开 · 637 阅读 · 18 · 16 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/149843390

**文章目录**

[TOC]


[LeetCode 114，将二叉树原地展开为链表](https://leetcode.cn/problems/flatten-binary-tree-to-linked-list/description/) ，【难度：中等；通过率：75.8%】，这道题不仅考察二叉树的遍历，更考验对 **双指针操作** 和空间复杂度的理解

## 一、 题目描述

给你二叉树的根节点 `root` ，请你将它展开为一个单链表：

- 展开后的单链表应该同样使用 `TreeNode` 类，其中 `right` 指针指向链表中下一个节点，而 `left` 指针始终为 `null` 

- 展开后的单链表应该与二叉树的 **[前序遍历](https://baike.baidu.com/item/%E5%85%88%E5%BA%8F%E9%81%8D%E5%8E%86/6442839?fr=aladdin)** 顺序相同

**示例:** 
 ![示例 1](./assets/100_1.jpeg)

```
输入: root = [1,2,5,3,4,null,6]
输出: [1,null,2,null,3,null,4,null,5,null,6]
```

---

## 二、 方法一：前序遍历 + 列表辅助

这是最容易想到的方法。既然要求结果是前序遍历的顺序，那我们就先进行一次前序遍历，把所有节点存起来，然后再把它们串成链表

### 核心思路

1.  **遍历并存储** ：对二叉树进行一次前序遍历，将所有节点按顺序存入一个 `ArrayList` 中

2.  **重新连接** ：遍历这个 `ArrayList` ，对于每个节点，将其 `left` 指针设为 `null` ，并将其 `right` 指针指向列表中的下一个节点

### 参考代码实现

```java
class Solution {
    public void flatten(TreeNode root) {
        // 步骤 1: 使用列表存储前序遍历的结果
        List<TreeNode> list = new ArrayList<>();
        preOrder(root, list);

        // 步骤 2: 遍历列表，重新连接节点的 right 指针
        for (int i = 0; i < list.size() - 1; i++) {
            TreeNode currentNode = list.get(i);
            TreeNode nextNode = list.get(i + 1);
            currentNode.left = null;
            currentNode.right = nextNode;
        }
      
        // 处理最后一个节点
        if (!list.isEmpty()) {
            list.get(list.size() - 1).left = null;
            list.get(list.size() - 1).right = null;
        }
    }

    // 经典的前序遍历递归实现
    private void preOrder(TreeNode node, List<TreeNode> list) {
        if (node == null) {
            return;
        }
        list.add(node);
        preOrder(node.left, list);
        preOrder(node.right, list);
    }
}
```

### 优缺点分析

-  **优点** ：思路清晰，代码易于理解和实现

-  **缺点** ：使用了 O(N) 的额外空间来存储列表，但是没有达到“原地”修改的程度， **空间复杂度较高** 

---

## 三、 方法二：原地修改 - 寻找前驱节点 (最优解)

这是本题的“最佳实践”。我们不使用任何额外的存储列表，直接在原树上进行修改

### 核心思路★

我们遍历树，对于每个节点 `curr` ，我们思考如何将它的左子树插入到它和它的右子树之间

1. 如果 `curr` 没有左子树，说明它已经符合“链表”的一部分结构，我们直接处理它的右孩子，即 `curr = curr.right` 

2. 如果 `curr` 有左子树，我们需要：
   a. 找到左子树中 **最右** 的那个节点，我们称之为 `predecessor` （前驱节点）。这个 `predecessor` 在前序遍历中，恰好是 `curr` 的右子树的根节点的前一个节点
   b. 将 `curr` 的 **右子树** ，嫁接到 `predecessor` 的 `right` 指针上
   c. 将 `curr` 的 **左子树** ，移动到 `curr` 的 `right` 指针上
   d. 将 `curr` 的 `left` 指针设为 `null` 
   e. 处理下一个节点，即新的 `curr.right` 

通过不断重复这个过程，我们就好像在“穿针引线”，把整个树串了起来

### 代码实现 (原地修改)

```java
class Solution {
    public void flatten(TreeNode root) {
        TreeNode curr = root;
        while (curr != null) {
            if (curr.left != null) {
                // 1. 找到左子树的最右节点 (前驱节点)
                TreeNode predecessor = curr.left;
                while (predecessor.right != null) {
                    predecessor = predecessor.right;
                }
              
                // 2. 将原右子树嫁接到前驱节点的右侧
                predecessor.right = curr.right;
              
                // 3. 将左子树移动到右侧
                curr.right = curr.left;
              
                // 4. 将左子树置空
                curr.left = null;
            }
          
            // 5. 继续处理下一个节点
            curr = curr.right;
        }
    }
}
```

### 优缺点分析

-  **优点** ：

  -  **空间复杂度为 O(1)** ，没有使用任何额外的存储空间，实现了真正的原地修改

  - 代码逻辑精妙，是树操作的经典技巧

-  **缺点** ：相对于方法一，理解起来稍有难度，需要仔细思考 **指针的移动** 和 **连接** 过程

---

## 四、 总结与对比

|  | 方法一 (列表辅助) | 方法二 (原地修改，思维进阶) |
|:---|:---|:---|
|  **核心思路**  | 先遍历存储，后连接 | 边遍历边修改，寻找前驱节点 |
|  **时间复杂度**  | O(N) | O(N) |
|  **空间复杂度**  |  **O(N)**  |  **O(1)**  |
|  **代码可读性**  | 高 | 中等 |