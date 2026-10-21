# N 皇后

[LeetCode 51 N皇后](https://leetcode-cn.com/problems/n-queens/)   要求输出所有解法

[LeetCode 52 N皇后II](https://leetcode-cn.com/problems/n-queens-ii/) 要求输出所有解的数量



## 代码框架

```python
def backtrack(row):
    if(row == n):
        # 说明n行都已经正确摆放了，输出结果
    else:
        # 循环摆放第1列至第N列
        for i in range(n):
            # 只有当前摆放结果合法时，才进行操作，否则进入下一循环
            if(isValid()):
                # 递归进行下一行的摆放
                backtrack(row + 1)
```



## 摆放合法性的判断`isValid()`

方法一：使用三个集合`set`：`columns`表示当前列，`diagonal1`表示左上至右下斜线，`diagonal2`表示右上至左下斜线。

方法二：使用位运算。



## 完整代码

```python
def solveNQueens(n):
    def generateBoard():
        board = list()
        for i in range(n):
            row[queens[i]] = "Q"
            board.append("".join(row))
            row[queens[i]] = "."
        return board

    def backtrack(row):
        if row == n:
            board = generateBoard()
            solutions.append(board)
        else:
            for i in range(n):
                if i in columns or row - i in diagonal1 or row + i in diagonal2:
                    continue
                queens[row] = i
                columns.add(i)
                diagonal1.add(row - i)
                diagonal2.add(row + i)
                backtrack(row + 1)
                columns.remove(i)
                diagonal1.remove(row - i)
                diagonal2.remove(row + i)

    solutions = list()
    queens = [-1] * n
    columns = set()
    diagonal1 = set()
    diagonal2 = set()
    row = ["."] * n
    backtrack(0)
    return solutions
```