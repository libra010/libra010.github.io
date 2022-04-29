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
def postOrder(root):
    if not root:
        return
    postOrder(root.left)
    postOrder(root.right)
    # doSomething(root.val)
```

可见三种遍历方式写起来是差不多的，区别仅在于对节点值的访问顺序不同。

前序遍历：先访问根节点，再递归访问左子树节点，再递归访问右子树节点。

中序遍历：先访问左子树节点，再递归访问根节点，再递归访问右子树节点。

后序遍历：先访问左子树节点，再递归访问右子树节点，再递归访问根节点。





## 迭代

```python
# 前序遍历
def preOrder(root):
    stack = []
    while stack or root:
    	while root:
        	# doSomething(root.val)
        	stack.append(root)
        	root = root.left
        root = stack.pop()
        root = root.right
        
# 中序遍历
def inOrder(root):
    stack = []
    while stack or root:
    	while root:
        	stack.append(root)
        	root = root.left
        root = stack.pop()
        # doSomething(root.val)
        root = root.right
        
# 后序遍历
def postOrder(root):
    stack = []
    prev = None
    while root or stack:
        while root:
            stack.append(root)
            root = root.left
        root = stack.pop()
        if not root.right or root.right == prev:
            # doSomething(root.val)
            prev = root
            root = None
        else:
            stack.append(root)
            root = root.right
```

迭代版本的入栈顺序就是前序遍历顺序（先根节点入栈，然后左子节点入栈，然后直到根节点出栈后，右子节点入栈），出栈顺序就是中序遍历顺序（先左子节点出栈，然后根节点出栈，然后直到根节点入栈后，右子节点出栈）

前序遍历是访问节点之后入栈，所以入栈顺序就是前序遍历顺序。

中序遍历是出栈之后访问节点，所以出栈顺序就是中序遍历顺序。



注：`while root:` 代码块，循环将左子树节点入栈一直到左子树节点为空。之后就是`stack.pop()`出栈。

出栈规则：出栈的节点其左子树都已经访问完毕。

之后前序遍历和中序遍历都是直接访问右子树。

在后序遍历中由于我们需要先访问右子树再访问根节点，但是现在无法判断右子树是否已经被访问过，所以需要引入额外的变量`prev`来记录历史访问，当访问完一棵子树的时候，我们用prev指向该节点；这样，在回溯到父节点的时候，我们可以依据prev是指向左子节点，还是右子节点，来判断父节点的访问情况。

`not root.right or root.right == prev`：如果没有右子树，或者右子树访问完了，也就是上一个访问的节点是右子节点时，说明可以访问当前节点。否则访问右子树。





## Morris 遍历

规则：记录当前节点为`cur`。若`cur`无左子树，将`cur`右移。若`cur`有左子树，找到左子树的最右节点，记录为`mostright`；若`mostright`的右指针为空，将其指向`cur`，`cur`向左移动；若`mostright`的右指针指向`cur`，将其指向空，`cur`向右移动。

实质：利用二叉树叶节点的空余位置，实现常数级别的空间复杂度。

```python
# 前序遍历
def preOrder(self, root):
    if not root:
        return []
    cur = root
    while cur:
        mostright = cur.left
        if mostright:
            # 找到左子树的最右节点（非cur节点）
            while mostright.right and mostright.right != cur:
                mostright = mostright.right
            if not mostright.right:
                # doSomething(root.val)
                mostright.right = cur
                cur = cur.left
                continue
            else:
                mostright.right = None
        else:
            # doSomething(root.val)
        cur = cur.right
    
    return res
```





## 层序遍历

前序遍历，中序遍历，后序遍历都是二叉树的深度优先遍历。

二叉树的广度优先遍历又称层序遍历。

详细内容参考 [LeetCode 102 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/ ':ignore') 