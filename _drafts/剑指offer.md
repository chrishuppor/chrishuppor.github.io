# 基本数据结构

## 数组

* 优点
  * 可以直接用下标定位数据
    * 原因：数组存储在连续的一块内存中
    * 应用：哈希表/计划搜索

* 缺点

  * 空间效率不高
    * 原因：即使只存储一个元素，也要为整个数组分配内存空间
    * 改进——动态数组，eg: vector。先申请个小空间的，不够了再重新申请个大的。（vector每次扩容，新容量都是原来的2倍。）
      * 因为要申请新空间、复制、释放原空间，所以会带来时间开销
  * 不利于增删
    * 原因：内存连续，无论插入元素还是删除元素都需要移动该元素后面所有的元素
    * 可以从后向前插入元素。

* 当数组作为函数参数进行传递时，数组自动退化为同类型的指针

  ```c++
  int GetSize(int data[])//相当于int GetSize(int* data)
  {
      return sizeof(data);
  }
  
  void main()
  {
      int data1[] = {1,2,3,4,5};
      int* data2 = data1;
      cout << GetSize(data1);//输出的是4，因为在GetSize中data被认为是指向int型数据的指针
      cout << sizeof(data1);//输出的是5*4，因为data1表示数组，这里求数组的size
      cout << sizeof(data2);//输出的是4，因为data2是指针
  }
  
  //ps.二维数组传参 eg int data[][N] 或 int** data;但两者意义不同，第一个表示一个二维数组，列数为N，第二个表示一个指针的指针。
  ```

## 字符串

字符串是元素为字符以\0结尾的数组。

* 常量字符串

  * 通常放在一个单独的内存区域，**当把相同的常量字符串赋值给不同指针时，这些指针指向同一个地址**。

    ```c++
    char* p1 = "hello";
    char* p2 = "hello";
    char str1[] = "hello";
    char str2[] = "hello";
    //p1==p2：因为都指向“hello”存储的位置
    //str1 != str2：因为这是两个数组，程序会为数组分配空间并将“hello”复制进去。
    ```

## 树

### 二叉树

* 类型
  * 二叉搜索树——左节点<=根节点<=右节点，搜索时间O(logn)
  * 堆
    * 最大堆——根节点最大，可以快速找到最大值
    * 最小堆——根节点最小，可以快速找到最小值
  * 红黑树

* 题目

  * 根据先序遍历和中序遍历恢复二叉树

    * 先序遍历中，一组节点最先出现的那个是根节点

    * 中序遍历中，出现在根节点左侧的是左子树节点，出现在根节点右侧的是右子树

      * leetcode代码

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
            TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
                if(preorder.size() == 0)
                {
                    return NULL;
                }
        
                TreeNode* root = new TreeNode(preorder[0]);
                root->left = root->right = NULL;
                
                //std::find可以在容器中查找数据并返回一个iterator
                int leftNodeCount = find(inorder.begin(), inorder.end(), root->val) - inorder.begin();
                
                //vector初始化：可以用已有容器的iterator初始化，第二个参数是end()的位置
                vector<int> left_pre(preorder.begin() + 1, preorder.begin() + 1 + leftNodeCount);
                vector<int> right_pre(preorder.begin() + 1 + leftNodeCount, preorder.end());
                vector<int> left_in(inorder.begin(), inorder.begin() + leftNodeCount); 
                vector<int> right_in(inorder.begin() + leftNodeCount + 1, inorder.end());
                
                root->left = buildTree(left_pre, left_in);
                root->right = buildTree(right_pre, right_in);
                return root;
            }
        };
        ```

  * 找某个节点中序遍历的下一个节点

    * 画图、举例找出问题规律——分三种情况
      1. 有右子树——则下一个节点是右子树的中序遍历的第一个节点，即最左节点
      2. 无右子树
         1. 是父节点的左节点——则下一个节点就是父节点
         2. 是父节点的右节点——则需要继续回溯，直到无父节点或父节点是“祖”节点的左节点。

## 栈和队列

* 特点

  * 栈——先进后出
  * 队列——先进先出

* 题目

  * 栈实现队列——两个栈模拟一个队列（感觉纯粹是为了练习）

  * 队列实现栈

    * 两个队列模拟一个栈

    * 一个队列模拟一个栈

      * 将队列的头作为栈顶，则pop、top与queue.pop、queue.top一致。push可以使用循环队列的思想将queue.push进来的数据移动到front

        ```c++
        void push(int x)
        {
            queue.push(x);
            for(int i = 0; i < queue.size() - 1; i++) //循环size次则与原来一样，循环size-1次则原来的back移动到现在的front，模拟了栈的push
            {
                queue.push(queue.front);
                queue.pop();
            }
        }
        ```


# 基本算法

## 查找

* 顺序查找
* 二分查找
* 哈希查找
* 二叉树查找

## 排序

* 插入排序
* 冒泡排序
* 归并排序
* 快速排序

## 递归和循环

* 应用场景
  * 重复的多次计算**相同**的问题

## 回溯

* 应用场景
  * 有许多步骤组成，且每个步骤有多个选项
* 所有选项可以组成树状结构

## 动态规划

* 应用场景
  * 常用于求最优解
  * 从上往下分析问题：能把大问题分解为小问题，且小问题存在最优解，且**小问题的最优解能够组成大问题的最优解**（往往将先将问题二分，变为f(n) = f(k)f(n-k)）
  * 从下往上求解问题：避免重复求解子问题，所以先求解子问题并将结果存储下来。

 ## 贪婪算法

* 应用场景

  * 求最优解
  * 每次都做当前最好的选择
    * 需要证明贪恋选择能够得到最优解
      * 归纳法(对算法步数归纳、对问题归纳)
      * 交换论证法（从最优解出发，不变坏地替换，得到贪心策略的解）

* 贪婪算法其实是特殊版的动态规划。动态规划中因为不知道哪个子问题的最优解能够组合成最终的解而需要将子问题的最优解都记录下来，贪婪算法是**直接选择**能够组成最优解的子问题最优解。
