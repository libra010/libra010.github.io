# 二叉树的遍历



二叉树每个节点有对左右子节点的引用，每个节点都看作是一个二叉树的根节点，遍历方式很自然有三种：先访问根节点，然后左，然后右（前序遍历）；先访问左，然后根，然后右（中序遍历）；先访问左，然后右，最后根（后序遍历）。

因为每一个左右节点都可以看作一个独立的二叉树，所以这个问题适合用递归方式实现。不过，递归有一个通用原则，就是递归要有一个结束的时候，当一个二叉树的根节点为空时，这个二叉树无需遍历。



## 递归

```python
# 前序遍历
def preOrder(root):
    if not root:
        return
    # doSomething(root.val)
    preOrder(root.left)
    preOrder(root.right)

# 中序遍历
def inOrder(root):
    if not root:
        return
    inOrder(root.left)
    # doSomething(root.val)
    inOrder(root.right)
    
# 后序遍历
def postOrder(root: TreeNode):
    if not root:
        return
    postOrder(root.left)
    postOrder(root.right)
    # doSomething(root.val)
```



## 迭代





## Morris 遍历





## 层序遍历

前序遍历，中序遍历，后序遍历都是二叉树的深度优先遍历。

二叉树的广度优先遍历又称层序遍历。

详细内容参考 [LeetCode 102 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/ ':ignore') 