# 简单

## 合并二叉树

* 题目

  ```
  给定两个二叉树，想象当你将它们中的一个覆盖到另一个上时，两个二叉树的一些节点便会重叠。
  
  你需要将他们合并为一个新的二叉树。合并的规则是如果两个节点重叠，那么将他们的值相加作为节点合并后的新值，否则不为 NULL 的节点将直接作为新二叉树的节点。
  
  来源：力扣（LeetCode）
  链接：https://leetcode-cn.com/problems/merge-two-binary-trees
  ```

* 题解

  * 我的题解

    **二叉树的遍历操作最常用的就是递归。**递归的基本思想就是一个问题可以划分为很多层，每一层都是相同问题，每一层的解都由下一层的解计算而来；算法设计由上至下，求解由下至上。在使用时，假设其他问题已通过函数调用得到结果，然后最顶层的那个问题；同时要给出递归结束的条件。

    这里，将合并二叉树的问题划分为子树的合并。最顶层的问题就是顶点合并+左子树合并+右子树合并；递归结束的条件就是左或右节点出现NULL。

    ```c++
    /**
     * Definition for a binary tree node.
     * struct TreeNode {
     *     int val;
     *     TreeNode *left;
     *     TreeNode *right;
     *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
     * };
     */
    class Solution {
    public:
        TreeNode* mergeTrees(TreeNode* t1, TreeNode* t2) {
            //结束条件
            if(t1 == NULL){
                return t2;
            }
            if(t2 == NULL){
                return t1;
            }
            //递归结构
            TreeNode* res = new TreeNode(t1->val + t2->val);
            res->left = mergeTrees(t1->left, t2->left);
            res->right = mergeTrees(t1->right, t2->right);
            return res;
        }
    };
    ```

  * 其他题解

    * 用**迭代替换递归**。因为递归太深会有栈溢出的问题，所以有些时候不得不用迭代替换递归。

      将两棵树对应位置的节点组成一个数组，依次放入栈中，然后从栈中依次取出数组构成新的树。

## 计算汉明距离

* 题目

  ```
  两个整数之间的汉明距离指的是这两个数字对应二进制位不同的位置的数目。
  
  给出两个整数 x 和 y，计算它们之间的汉明距离。
  ```

* 题解

  * 两个数之间的汉明距离也就是两个数异或结果中1的个数。**PS:+的优先级高于&**

    ```c++
    class Solution {
    public:
        int hammingDistance(int x, int y) {
            int res = 0;
            int tmp = x ^ y;
            while(tmp)
            {
                res = res + (tmp & 1); //+的优先级高于&
                tmp = tmp >> 1;
            }
            return res;
        }
    };
    ```
