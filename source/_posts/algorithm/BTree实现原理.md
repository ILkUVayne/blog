---
title: BTree实现原理
categories:
- algorithm
tags:
- algorithm
- btree
---

## 前言

BTree和B-Tree都是指B树，不要把B-Tree理解成了B-树。本文主要分析B树的定义以及增删等概念。

## 定义

> 为了方便理解：孩子即树的分支，关键字即树的结点元素

关于BTree在《计算机程序设计与艺术》和《算法导论》对其描述是不一样的。一种是通过"阶"的概念来定义的，另一种是通过"度"的概念定义的，两者描述的本质是一样的。

- 所有叶结点都在同一层
- 结点内有序，结点任一左孩子都小于该结点，右孩子都大于该结点

对于m阶树结点：

- 最多：m个孩子，m-1个关键字
- 最少
  - 根结点：2个孩子，1个关键字
  - 其他结点：[m/2]个孩子，[m/2]-1个关键字 （m/2向上取整）

对于t(t>=2)度树结点：

- 最多：2t个孩子，2t-1个关键字
- 最少
    - 根结点：2个孩子，1个关键字
    - 其他结点：t个孩子，t-1个关键字

## 插入

插入流程如下：

1. 根据要插入的key的值，在BTree查找key存不存，如果已存在，用新的value覆盖旧的value，操作结束，否则执行步骤2
2. 如果没有找到key，则定位到要插入的叶子节点并插入
3. 判断2中刚插入节点key的个数是否满足BTree的定义，如果满足，插入结束，否则执行步骤4
4. 以插入节点中间的key（m/2向上取整）为中心，分裂成左右两部分，然后将中间的key插入到它的父节点中，这个key的左子树指向分裂后的左半部分，右子树指向分裂后的右半部分。然后继续判断父节点满不满足BTree定义，如果不满足，继续对父节点进行分裂，否则流程执行结束

以 Max.Degree=3 为例：

插入5：

![insert_5](/images/algorithm/btree/insert_5.png)

插入15，满足BTree定义，无需向上分裂：

![insert_15](/images/algorithm/btree/insert_15.png)

插入10:

![insert_10_1](/images/algorithm/btree/insert_10_1.png)

由定义可知，Max.Degree=3的BTree结点最多只能有两个关键字，此时不满足BTree定义，需按照流程4向上分裂：

![insert_10_2](/images/algorithm/btree/insert_10_2.png)

插入9，在左孩子中插入，满足BTree定义，无需向上分裂：

![insert_9](/images/algorithm/btree/insert_9.png)

插入6，在左孩子中插入：

![insert_6_1](/images/algorithm/btree/insert_6_1.png)

此时不满足BTree定义，需按照流程4向上分裂：

![insert_6_2](/images/algorithm/btree/insert_6_2.png)

插入1，在左孩子中插入，满足BTree定义，无需向上分裂：

![insert_1](/images/algorithm/btree/insert_1.png)

插入3，在左孩子中插入：

![insert_3_1](/images/algorithm/btree/insert_3_1.png)

此时不满足BTree定义，需按照流程4向上分裂:

![insert_3_2](/images/algorithm/btree/insert_3_2.png)

此时，父结点也不满足BTree定义，继续向上分裂，满足定义，插入结束：

![insert_3_3](/images/algorithm/btree/insert_3_3.png)

由上述插入流程得出，插入的核心就是判断当前插入的叶子结点是否满足BTree定义（结点关键字最多为m-1个），不满足时，将中间的关键字（m/2向上取整）向上分裂，若分裂后父结点不满足，则父结点继续向上分裂，直至满足定义为止

## 删除

删除操作分为删除非叶子结点关键字和删除叶子结点关键字

### 删除非叶子结点关键字

找到该关键字的左子树叶子结点最大的值（或者后继子树的最小值也可以）并替换，后续的删除逻辑按照叶子结点删除即可（即将非叶子结点删除转换为叶子结点删除）

例如如下4阶B树：

![delete_7](/images/algorithm/btree/delete_7.png)

删除15，此时找到前驱结点的最大值10，并与15替换，然后执行删除叶子结点15的逻辑即可：

![delete_7_1](/images/algorithm/btree/delete_7_1.png)

后续的删除逻辑同下面删除叶子结点关键字相同

### 删除叶子结点关键字

删除叶子结点后，需要判断当前结点是否满足B树定义，即非根结点关键字数量大于等于[m/2]-1个，假定该情况为“富有”，若是满足，则删除结束

#### 富有

例如如下4阶B树：

![delete_1](/images/algorithm/btree/delete_1.png)

删除15：

![delete_2](/images/algorithm/btree/delete_2.png)

删除后左子树关键字数量为2，大于1 (4/2-1)满足B树定义，所以删除结束

#### 不富有

1. 当删除叶子结点关键字后，当前结点不富有，则尝试向兄弟结点借，若兄弟也不富有，则向父结点借，并合并兄弟结点
2. 此时若父节点也不富有了，则重复步骤1逻辑依次借，直至满足B树定义为止

- 兄弟富有的情况

如下4阶B树：

![delete_3](/images/algorithm/btree/delete_3.png)

删除35，此时它的左右兄弟都富有，这里从左兄弟借（从右兄弟借也可以）将26移到父结点，将父结点30移到原35的位置即可：

![delete_3_1](/images/algorithm/btree/delete_3_1.png)

若继续删除30，此时左兄弟不富有，但右兄弟富有，按照同样的逻辑从右兄弟借：

![delete_3_2](/images/algorithm/btree/delete_3_2.png)

- 兄弟不富有的情况

如下4阶B树：

![delete_3_2](/images/algorithm/btree/delete_3_2.png)

删除37，此时左右兄弟结点均不富有，此时就需要向父结点借，即将父节点借的关键字与兄弟结点合并，完成后父结点任然富有，满足定义，删除结束

![delete_4](/images/algorithm/btree/delete_4.png)

- 兄弟和父亲不富有的情况

如下4阶B树：

![delete_5](/images/algorithm/btree/delete_5.png)

删除40，此时兄弟结点不富有，从父结点借，并与兄弟结点合并：

![delete_5_1](/images/algorithm/btree/delete_5_1.png)

此时，父结点不富有（为空），父结点重复前面的逻辑，发现兄弟也不富有，继续向父结点借，并合并：

![delete_5_2](/images/algorithm/btree/delete_5_2.png)

此时结点富有了，满足定义，根结点为空，即整棵树的高度降了一层，删除结束：

![delete_5_3](/images/algorithm/btree/delete_5_3.png)


- 兄弟和父亲不富有（父亲的兄弟富有）的情况

如下4阶B树(父亲结点的兄弟富有的情况)：

![delete_6](/images/algorithm/btree/delete_6.png)

删除40，逻辑同上述情况类似，只是父结点的兄弟富有，借兄弟的关键字，无需再向上借：

![delete_6_1](/images/algorithm/btree/delete_6_1.png)

总结上述不同的删除例子能得出：

1. 删除非叶子结点关键字，先将需要删除的结点转换为叶子结点关键字，再使用叶子结点的删除逻辑进行删除。
2. 删除叶子结点关键字后，需要判断当前结点是否还满足B树定义，若满足，则删除结束。
3. 若不满足，如果兄弟结点能借，则向兄弟结点借，删除结束。如果兄弟结点不够借（借了之后不满足定义），则向父结点借，并和兄弟结点合并，删除结束。
4. 如果此时父节点也不满足了，则重复步骤3的逻辑借，只至满足定义为止

## 查找

BTree是一种多路平衡树，同时也满足有序性，对于每个节点，它左边子树的所有元素都小于该节点中最小的元素，它右边子树的所有元素都大于该节点中最大的元素。每个节点的内部元素也是有序的。所以采用类似二叉树的中序遍历，得到元素一定是有序的。所以BTree中查找元素的过程很简单，从根节点开始，每次可以定位可能所在的1个子节点，这样一路向下查询，如果在内部节点中没有找到，最后达到叶子节点，如果叶子节点也没有，则说明要查询的元素不在BTree中