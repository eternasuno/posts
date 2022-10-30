---
title: "Morris 遍历算法"
date: "2021-04-13"
tags: ["algorithm"]
---

普通的二叉树遍历算法在访问到叶子结点之后再访问后继结点往往需要回溯到之前访问过的结点，这时就需要使用栈来保存后继结点，因此普通二叉树遍历算法的空间复杂度为 $O(n)$。但是如果把叶子结点空闲的左右指针利用起来，分别指向当前结点的前驱和后继结点，那就不再需要栈来保存结点了。Morris 遍历算法就是这样一种不需要额外存储空间来遍历二叉树的算法，空间复杂度为 $O(n)$，空间复杂度为 $O(1)$。

<!-- excerpt -->

二叉树结点的定义如下：

```go
type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}
```

Morris 算法一般指的是二叉树的中序遍历，不过前序遍历也只需要稍加修改，而后续遍历则相对比较复杂。

## 中序遍历

### 步骤：

1. 如果当前结点（记为 curr）的左子树为空，则输出 curr 并访问 curr 的右子树，即 curr = curr.Right。
2. 如果当前结点有左子树，则找到左子树上最右的结点（即左子树中序遍历的最后一个结点，curr 在中序遍历中的前驱），记为 pre。根据 pre 的右子树是否为空，进行如下操作。
    - 如果 pre 的右子树为空，则将其右指针指向 cur，即 pre.Right = cur， 然后访问 cur 的左子树，即 cur = cur.Left。
    - 如果 pre 的右子树不为空，则此时其右指针已经指向 cur，说明 cur 的左子树已经遍历完了，把 pre 的右指针重新置空。输出 cur，然后访问 curr 的右子树，即 curr = curr.Right。
3. 重复上述步骤直至访问完整颗树。

### 代码：

```go
func inOrder(root *TreeNode) {
	for root != nil {
		if root.Left != nil {
			pre := root.Left
			for pre.Right != nil && pre.Right != root {
				// 有右子树且不指向 root，则继续向右
				pre = pre.Right
			}
			if pre.Right == nil {
				pre.Right = root
				root = root.Left
			} else {
				visit(root)
				pre.Right = nil
				root = root.Right
			}
		} else {
			visit(root)
			root = root.Right
		}
	}
}
```

## 前序遍历

前序遍历与中序遍历类似，不同在于输出的顺序。

### 步骤：

1. 如果当前结点（记为 curr）的左子树为空，则输出 curr 并访问 curr 的右子树，即 curr = curr.Right。
2. 如果当前结点有左子树，则找到左子树上最右的结点（即左子树**中序遍历**的最后一个结点，curr 在**中序遍历**中的前驱），记为 pre。根据 pre 的右子树是否为空，进行如下操作。
    - 如果 pre 的右子树为空，则将其右指针指向 cur，即 pre.Right = cur。输出 curr （在这里输出，这里是和中序遍历唯一的不同）， 然后访问 cur 的左子树，即 cur = cur.Left。
    - 如果 pre 的右子树不为空，则此时其右指针已经指向 cur，说明 cur 的左子树已经遍历完了，把 pre 的右指针重新置空。然后访问 curr 的右子树，即 curr = curr.Right。
3. 重复上述步骤直至访问完整颗树。

### 代码：

```go
func preOrder(root *TreeNode) {
	for root != nil {
		if root.Left != nil {
			pre := root.Left
			for pre.Right != nil && pre.Right != root {
				// 有右子树且不指向 root，则继续向右
				pre = pre.Right
			}
			if pre.Right == nil {
				pre.Right = root
        // 这里是和中序遍历唯一的不同
				visit(root)
				root = root.Left
			} else {
				pre.Right = nil
				root = root.Right
			}
		} else {
			visit(root)
			root = root.Right
		}
	}
}
```

## 后序遍历

### 步骤：

1. 如果当前结点（记为 curr）的左子树为空，访问 curr 的右子树，即 curr = curr.Right。
2. 如果当前结点有左子树，则找到左子树上最右的结点（即左子树**中序遍历**的最后一个结点，curr 在**中序遍历**中的前驱），记为 pre。根据 pre 的右子树是否为空，进行如下操作。
    - 如果 pre 的右子树为空，则将其右指针指向 cur，即 pre.Right = cur。 然后访问 cur 的左子树，即 cur = cur.Left。
    - 如果 pre 的右子树不为空，则此时其右指针已经指向 cur，说明 cur 的左子树已经遍历完了，把 pre 的右指针重新置空。倒序输出 curr.Left 到 pre 这条路径上的所有结点。然后访问 curr 的右子树，即 curr = curr.Right。
3. 重复上述步骤直至访问完整颗树。

可以发现上述步骤如果从根节点开始，那么根节点的右子树上最右的结点并没有指向根节点，也就是说上述步骤在遍历完右子树后并不能回到根节点进行最后一次输出。因此创建一个临时结点 temp，让 temp 的左子树指向根节点，即 temp.Left = root，遍历从 temp 开始。

### 代码：

```go
func postOrder(root *TreeNode) {
	temp := &TreeNode{Left: root}
	root = temp
	for root != nil {
		if root.Left != nil {
			pre := root.Left
			for pre.Right != nil && pre.Right != root {
				// 有右子树且不指向 root，则继续向右
				pre = pre.Right
			}
			if pre.Right == nil {
				pre.Right = root
				root = root.Left
			} else {
				pre.Right = nil
				// 逆序输出root.Left 到 pre 路径上的所有结点
				reverseVisit(root.Left, pre)
				root = root.Right
			}
		} else {
			root = root.Right
		}
	}
}

func reverseVisit(from, to *TreeNode) {
	// 反转指针
	reverse(from, to)
	for p := to; p != from; p = p.Right {
		visit(p)
	}
	visit(from)
	reverse(to, from)
}

func reverse(from, to *TreeNode) {
	if from == to {
		return
	}
	p, q := from, from.Right
	for p != to {
		r := q.Right
		q.Right = p
		p = q
		q = r
	}
	from.Right = nil
}
```
