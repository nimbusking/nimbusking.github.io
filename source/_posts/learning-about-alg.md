---
title: 计算机算法
abbrlink: 1cd8de98
date: 2024-12-28 20:46:10
updated: 2025-02-26 21:22:00
tags:
  - 算法
categories: 算法
---

## 前言
这些东西，没有什么特别好的办法，只能不断练习与理解

<!-- more -->
测试代码仓库：https://github.com/nimbusking/CoreJavaSample


## 相关知识点


以下是关于 **计算机算法** 的高频面试题总结，涵盖 **排序算法**、**数据结构**、**动态规划**、**搜索算法** 等核心知识点，帮助快速掌握关键题型与解题思路：

---

### **一、排序算法**
1. **快速排序**  
   - **原理**：分治思想，选取基准元素，将数组分为左右两部分（左边≤基准，右边≥基准），递归排序。  
   - **时间复杂度**：平均 O(n log n)，最坏 O(n²)（已排序数组）。  
   - **代码实现**：  
     ```python
     def quick_sort(arr):
         if len(arr) <= 1:
             return arr
         pivot = arr[len(arr)//2]
         left = [x for x in arr if x < pivot]
         middle = [x for x in arr if x == pivot]
         right = [x for x in arr if x > pivot]
         return quick_sort(left) + middle + quick_sort(right)
     ```

2. **归并排序**  
   - **原理**：分治思想，将数组分为两半，分别排序后合并。  
   - **时间复杂度**：稳定 O(n log n)，空间复杂度 O(n)。  
   - **合并代码**：  
     ```python
     def merge(left, right):
         result = []
         i = j = 0
         while i < len(left) and j < len(right):
             if left[i] < right[j]:
                 result.append(left[i])
                 i += 1
             else:
                 result.append(right[j])
                 j += 1
         result.extend(left[i:] or right[j:])
         return result
     ```

3. **堆排序**  
   - **原理**：构建最大堆，交换堆顶与末尾元素，调整堆结构。  
   - **时间复杂度**：O(n log n)，空间复杂度 O(1)。  

---

### **二、数据结构**
4. **链表**  
   - **反转链表**：迭代或递归实现。  
     ```python
     def reverse_list(head):
         prev = None
         curr = head
         while curr:
             next_node = curr.next
             curr.next = prev
             prev = curr
             curr = next_node
         return prev
     ```
   - **检测环**：快慢指针法（Floyd判圈算法）。  

5. **二叉树**  
   - **层次遍历**：使用队列实现 BFS。  
   - **验证二叉搜索树（BST）**：中序遍历递增或递归检查节点范围。  

6. **哈希表**  
   - **两数之和**：用哈希表存储遍历过的值及其索引。  
     ```python
     def two_sum(nums, target):
         seen = {}
         for i, num in enumerate(nums):
             if target - num in seen:
                 return [seen[target - num], i]
             seen[num] = i
         return []
     ```

---

### **三、动态规划（DP）**
7. **爬楼梯**  
   - **问题**：每次爬 1 或 2 阶，到第 n 阶有多少种方法？  
   - **状态转移方程**：`dp[i] = dp[i-1] + dp[i-2]`，空间优化为 O(1)。  

8. **最长递增子序列（LIS）**  
   - **定义**：`dp[i]` 表示以 `nums[i]` 结尾的最长递增子序列长度。  
   - **优化**：二分查找维护单调递增序列，时间复杂度 O(n log n)。  

9. **背包问题**  
   - **0-1 背包**：`dp[i][j] = max(dp[i-1][j], dp[i-1][j-w] + v)`，空间优化为一维数组。  

---

### **四、搜索算法**
10. **二分查找**  
    - **模板**：  
      ```python
      def binary_search(nums, target):
          left, right = 0, len(nums) - 1
          while left <= right:
              mid = (left + right) // 2
              if nums[mid] == target:
                  return mid
              elif nums[mid] < target:
                  left = mid + 1
              else:
                  right = mid - 1
          return -1
      ```
    - **变种**：查找左边界/右边界（如 `bisect_left`）。

11. **DFS 与 BFS**  
    - **DFS 代码框架**（递归）：  
      ```python
      def dfs(node):
          if not node:
              return
          # 处理当前节点
          dfs(node.left)
          dfs(node.right)
      ```
    - **BFS 代码框架**（队列）：  
      ```python
      from collections import deque
      def bfs(root):
          queue = deque([root])
          while queue:
              node = queue.popleft()
              # 处理当前节点
              if node.left:
                  queue.append(node.left)
              if node.right:
                  queue.append(node.right)
      ```

---

### **五、高频进阶问题**
12. **LRU 缓存机制**  
    - **实现**：哈希表 + 双向链表，保证 O(1) 的 get 和 put 操作。  

13. **接雨水**  
    - **双指针法**：左右指针向中间移动，计算每个位置的积水量。  

14. **合并区间**  
    - **排序后合并**：按区间起点排序，依次合并重叠区间。  

15. **字符串解码**  
    - **栈实现**：处理嵌套括号，保存当前字符串和重复次数。  

---

### **六、时间与空间复杂度分析**
16. **常见时间复杂度对比**  
    | 算法           | 平均时间复杂度  |  
    |--------------|-----------|  
    | 冒泡排序         | O(n²)     |  
    | 快速排序         | O(n log n)|  
    | 归并排序         | O(n log n)|  
    | 二分查找         | O(log n)  |  
    | 动态规划（一维DP） | O(n)      |  

17. **空间复杂度优化**  
    - **滚动数组**：将二维 DP 压缩为一维（如背包问题）。  
    - **尾递归优化**：减少递归调用栈深度（需编译器支持）。  

---

### **七、代码陷阱与技巧**
18. **数组越界**：在处理二分查找或双指针时，注意循环终止条件。  
19. **指针操作**：链表问题中，使用哑节点（Dummy Node）简化头节点处理。  
20. **递归终止条件**：树或动态规划问题中必须明确基线条件。  

---

### **八、扩展练习**
21. **Top K 问题**  
    - **解法**：堆排序（O(n log k)）或快速选择算法（O(n)）。  

22. **最小编辑距离**  
    - **状态定义**：`dp[i][j]` 表示将 word1 前 i 个字符转换为 word2 前 j 个字符的最小操作数。  

23. **岛屿数量**  
    - **DFS/BFS 遍历**：将访问过的陆地标记为水，避免重复计数。  


---

## 分类刷题（LeeCode）
### **一、数组与字符串**
1. **两数之和**（LeetCode 1）  
   - 关键点：哈希表优化查找，时间复杂度 O(n)。
2. **三数之和**（LeetCode 15）  
   - 关键点：排序 + 双指针，去重逻辑。
3. **最长无重复子串**（LeetCode 3）  
   - 关键点：滑动窗口 + 哈希表记录位置。
4. **接雨水**（LeetCode 42）  
   - 关键点：双指针/动态规划，计算每个位置的左右最大高度。
5. **字符串反转/变形**（如反转单词顺序、Z字形变换）  
   - 关键点：原地操作或分块处理。

---

### **二、链表**
1. **反转链表**（LeetCode 206）  
   - 关键点：迭代（三指针）或递归。
2. **链表中环的检测**（LeetCode 141）  
   - 关键点：快慢指针（Floyd判圈法）。
3. **合并两个有序链表**（LeetCode 21）  
   - 关键点：虚拟头节点简化操作。
4. **删除链表的倒数第N个节点**（LeetCode 19）  
   - 关键点：快慢指针定位前驱节点。
5. **链表排序**（LeetCode 148）  
   - 关键点：归并排序（递归或迭代实现）。

---

### **三、栈与队列**
1. **有效的括号**（LeetCode 20）  
   - 关键点：栈的匹配与哈希表映射。
2. **最小栈**（LeetCode 155）  
   - 关键点：辅助栈记录最小值。
3. **用队列实现栈/用栈实现队列**（LeetCode 232/225）  
   - 关键点：双队列或双栈倒序。

---

### **四、树与二叉树**
1. **二叉树遍历**（前序/中序/后序/层序）  
   - 关键点：递归与非递归实现（栈或队列）。
2. **二叉树的最大深度**（LeetCode 104）  
   - 关键点：递归分治或层序遍历。
3. **验证二叉搜索树（BST）**（LeetCode 98）  
   - 关键点：中序遍历递增 或 递归传递上下界。
4. **二叉树的最近公共祖先（LCA）**（LeetCode 236）  
   - 关键点：递归回溯查找左右子树。
5. **平衡二叉树/完全二叉树判断**  
   - 关键点：递归检查高度或层序填充特性。

---

### **五、动态规划（DP）**
1. **爬楼梯**（LeetCode 70）  
   - 关键点：状态转移方程 `dp[i] = dp[i-1] + dp[i-2]`。
2. **最长递增子序列（LIS）**（LeetCode 300）  
   - 关键点：DP O(n²) 或 贪心+二分 O(nlogn)。
3. **背包问题**（01背包、完全背包）  
   - 关键点：状态定义、滚动数组优化。
4. **编辑距离**（LeetCode 72）  
   - 关键点：DP矩阵，三种操作（增删改）。
5. **打家劫舍系列**（LeetCode 198/213）  
   - 关键点：环形数组处理，状态分偷与不偷。

---

### **六、回溯算法**
1. **全排列**（LeetCode 46）  
   - 关键点：路径标记与撤销选择。
2. **子集/组合**（LeetCode 78/77）  
   - 关键点：剪枝优化避免重复。
3. **N皇后问题**（LeetCode 51）  
   - 关键点：对角线冲突检查。
4. **括号生成**（LeetCode 22）  
   - 关键点：左括号数 ≥ 右括号数。

---

### **七、图算法**
1. **岛屿数量**（LeetCode 200）  
   - 关键点：DFS/BFS遍历标记。
2. **课程表**（拓扑排序，LeetCode 207）  
   - 关键点：入度表 + 队列。
3. **最短路径**（Dijkstra、BFS层数）  
   - 关键点：权重处理与优先队列。
4. **克隆图**（LeetCode 133）  
   - 关键点：哈希表记录已克隆节点。

---

### **八、其他高频题**
1. **LRU缓存**（LeetCode 146）  
   - 关键点：哈希表 + 双向链表。
2. **合并区间**（LeetCode 56）  
   - 关键点：排序后贪心合并。
3. **寻找重复数**（LeetCode 287）  
   - 关键点：快慢指针（转化为环形链表问题）。
4. **多数元素**（LeetCode 169）  
   - 关键点：Boyer-Moore投票算法。

---

### **九、复杂度与设计题**
1. **手写快速排序/归并排序**  
   - 关键点：分治思想与边界处理。
2. **设计Twitter/朋友圈（Feed流）**  
   - 关键点：关注列表 + 多路归并。
3. **海量数据问题（Top K、去重）**  
   - 关键点：分桶、堆、位图法。

---

### **准备建议**
1. **刷题策略**：按算法分类刷题，同类题目总结共性。
2. **代码模板化**：如二分查找、DFS/BFS的固定写法。
3. **时间与空间复杂度分析**：每道题需明确计算依据。
4. **边界条件**：空输入、极值（如整数溢出）需特殊处理。


