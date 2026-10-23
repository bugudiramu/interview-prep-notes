# Java Level 11 — DSA & Problem Solving

## Part 3 — Trees & Binary Search Trees

### Status

**Level 11: In Progress**

Covered in this part:

- ✅ Binary Tree Fundamentals
- ✅ DFS
- ✅ Inorder Traversal
- ✅ Tree Recursion
- ✅ Maximum Depth
- ✅ Same Tree
- ✅ Invert Binary Tree
- ✅ Binary Search Tree Fundamentals
- ✅ Search in BST
- ✅ Validate BST
- ✅ Lowest Common Ancestor — BST
- ✅ BFS
- ✅ Level Order Traversal

---

# 1. Binary Tree Fundamentals

A Binary Tree is a tree where each node can have at most two children:

```text
left
right
```

Example:

```text
        10
       /  \
      5    15
     / \   / \
    3   7 12  20
```

Java representation:

```java
static class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;

    TreeNode(int val) {
        this.val = val;
    }
}
```

Unlike a Linked List:

```text
1 → 2 → 3 → 4
```

a tree can branch:

```text
        1
       / \
      2   3
     / \
    4   5
```

---

# 2. The Main Tree Question

For most Tree problems, first ask:

> How should I traverse the tree?

The two major approaches are:

```text
DFS
→ Depth First Search

BFS
→ Breadth First Search
```

Mental model:

```text
DFS
→ Go DEEP

BFS
→ Go LEVEL BY LEVEL
```

---

# 3. DFS — Depth First Search

DFS explores one branch deeply before moving to another branch.

The three classic DFS traversals are:

```text
Preorder
Inorder
Postorder
```

The only major difference is:

> When do we process the ROOT?

---

# 4. Preorder Traversal

Order:

```text
ROOT
LEFT
RIGHT
```

Example:

```text
        1
       / \
      2   3
     / \
    4   5
```

Result:

```text
1, 2, 4, 5, 3
```

Implementation:

```java
public static void preorder(TreeNode root) {
    if (root == null) {
        return;
    }

    System.out.println(root.val);

    preorder(root.left);
    preorder(root.right);
}
```

Mental model:

```text
PROCESS
LEFT
RIGHT
```

---

# 5. Inorder Traversal

Order:

```text
LEFT
ROOT
RIGHT
```

For:

```text
        10
       /  \
      5    15
     / \
    3   7
```

result:

```text
3, 5, 7, 10, 15
```

Implementation from practice:

```java
public static List<Integer> inorderTraversal(
        TreeNode root
) {
    List<Integer> visited = new ArrayList<>();

    traverseInOrder(root, visited);

    return visited;
}

private static void traverseInOrder(
        TreeNode node,
        List<Integer> visited
) {
    if (node == null) {
        return;
    }

    traverseInOrder(node.left, visited);

    visited.add(node.val);

    traverseInOrder(node.right, visited);
}
```

Complexity:

```text
Time  → O(n)
Space → O(h)
```

where:

```text
n = number of nodes
h = height of tree
```

Balanced tree:

```text
h ≈ log n
```

Worst-case skewed tree:

```text
h = n
```

---

# 6. Important BST Property of Inorder

For a valid Binary Search Tree:

> Inorder traversal produces values in sorted order.

Example:

```text
        8
       / \
      3   10
     / \    \
    1   6    14
```

Inorder:

```text
1, 3, 6, 8, 10, 14
```

This becomes useful when reasoning about BST validation and ordering.

---

# 7. Postorder Traversal

Order:

```text
LEFT
RIGHT
ROOT
```

Implementation:

```java
public static void postorder(TreeNode root) {
    if (root == null) {
        return;
    }

    postorder(root.left);
    postorder(root.right);

    System.out.println(root.val);
}
```

Mental model:

```text
LEFT
RIGHT
PROCESS
```

---

# 8. DFS Traversal Cheat Sheet

The easiest way to remember the three traversals is to move:

```java
process(root);
```

### Preorder

```text
ROOT
LEFT
RIGHT
```

### Inorder

```text
LEFT
ROOT
RIGHT
```

### Postorder

```text
LEFT
RIGHT
ROOT
```

Remember:

```text
PRE
→ root BEFORE children

IN
→ root IN BETWEEN children

POST
→ root AFTER children
```

---

# 9. Tree Recursion

Tree recursion follows an extremely important pattern:

```text
Solve LEFT subtree

Solve RIGHT subtree

Combine their answers

Return answer to parent
```

Typical recursive structure:

```java
if (root == null) {
    return baseValue;
}

var left = solve(root.left);
var right = solve(root.right);

return combine(root, left, right);
```

This mental model is more important than memorizing individual Tree solutions.

---

# 10. Maximum Depth of Binary Tree

Given:

```text
        3
       / \
      9   20
         / \
        15  7
```

Maximum depth:

```text
3
```

Recursive relationship:

```text
depth(node)

=

1 + max(
    depth(left),
    depth(right)
)
```

Base case:

```text
node == null
→ depth = 0
```

Implementation:

```java
public static int maxDepth(TreeNode root) {
    if (root == null) {
        return 0;
    }

    return 1 + Math.max(
            maxDepth(root.left),
            maxDepth(root.right)
    );
}
```

Example reasoning:

```text
depth(15) = 1
depth(7)  = 1

depth(20)
= 1 + max(1,1)
= 2

depth(9)
= 1

depth(3)
= 1 + max(1,2)
= 3
```

Complexity:

```text
Time  → O(n)
Space → O(h)
```

---

# 11. Important Tree Recursion Mental Model

Maximum Depth demonstrates a very common pattern:

```text
           NODE
          /    \
       LEFT    RIGHT
        ↓        ↓
      answer   answer
          \    /
          COMBINE
             ↓
         return up
```

Think:

> What information should each subtree return to its parent?

This question solves many recursive Tree problems.

---

# 12. Same Tree

Determine whether two trees have:

```text
Same structure
AND
Same values
```

Example:

```text
Tree P          Tree Q

    1               1
   / \             / \
  2   3           2   3

→ true
```

But:

```text
    1               1
   /                 \
  2                   2

→ false
```

---

# 13. Same Tree Base Cases

There are three important checks.

### Both null

```text
p == null
q == null

→ true
```

### Only one null

```text
p == null
OR
q == null

→ false
```

### Different values

```text
p.val != q.val

→ false
```

Otherwise recursively compare:

```text
LEFT
AND
RIGHT
```

---

# 14. Same Tree Implementation

```java
public static boolean isSameTree(
        TreeNode p,
        TreeNode q
) {
    if (p == null && q == null) {
        return true;
    }

    if (p == null || q == null) {
        return false;
    }

    if (p.val != q.val) {
        return false;
    }

    return isSameTree(p.left, q.left)
            && isSameTree(p.right, q.right);
}
```

Mental model:

```text
Current values same
        AND
Left subtree same
        AND
Right subtree same
```

Complexity:

```text
Time  → O(n)
Space → O(h)
```

---

# 15. Important Same Tree Pitfall

A mistake from practice was using:

```java
isSameTree(p.left, q.left)
==
isSameTree(p.right, q.right)
```

This is incorrect.

For example:

```text
leftSame  = false
rightSame = false

false == false
→ true
```

But both subtrees being wrong does NOT make the trees equal.

We need:

```java
leftSame && rightSame
```

Also remember to compare:

```java
p.val == q.val
```

Structure alone is not enough.

---

# 16. Invert Binary Tree

Given:

```text
        4
       / \
      2   7
     / \ / \
    1  3 6  9
```

Invert:

```text
        4
       / \
      7   2
     / \ / \
    9  6 3  1
```

At every node:

```text
swap left ↔ right
```

Then recursively invert both children.

---

# 17. Invert Tree Implementation

```java
public static TreeNode invertTree(TreeNode root) {
    if (root == null) {
        return null;
    }

    TreeNode temp = root.left;

    root.left = root.right;
    root.right = temp;

    invertTree(root.left);
    invertTree(root.right);

    return root;
}
```

Mental model:

```text
Current node

SWAP
left ↔ right

↓

Invert left

↓

Invert right
```

Complexity:

```text
Time  → O(n)
Space → O(h)
```

---

# 18. Binary Search Tree — BST

A Binary Search Tree adds ordering rules to a Binary Tree.

For every node:

```text
LEFT subtree values
<
node value

RIGHT subtree values
>
node value
```

Example:

```text
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

For root `8`:

```text
everything left  < 8
everything right > 8
```

---

# 19. BST vs Binary Tree

A Binary Tree only guarantees:

```text
At most two children
```

A BST additionally guarantees ordering:

```text
left < node < right
```

That ordering allows us to eliminate entire subtrees while searching.

This is conceptually similar to:

```text
Binary Search
```

---

# 20. Search in BST

Suppose we search for:

```text
13
```

in:

```text
        8
         \
         10
           \
           14
          /
         13
```

Reasoning:

```text
13 > 8
→ RIGHT

13 > 10
→ RIGHT

13 < 14
→ LEFT

13 == 13
→ FOUND
```

---

# 21. Recursive BST Search

```java
public static TreeNode searchBST(
        TreeNode root,
        int target
) {
    if (root == null) {
        return null;
    }

    if (root.val == target) {
        return root;
    }

    if (target < root.val) {
        return searchBST(
                root.left,
                target
        );
    }

    return searchBST(
            root.right,
            target
    );
}
```

Mental model:

```text
Found?
→ return

Smaller?
→ LEFT

Greater?
→ RIGHT
```

---

# 22. Important BST Search Mistake

This only moves one level:

```java
if (target < root.val) {
    root = root.left;
} else {
    root = root.right;
}

return root;
```

Example:

```text
        8
       /
      3
     /
    1
```

Searching `1`:

```text
8
→ move left
→ return 3 ❌
```

Recursive search must continue:

```java
return searchBST(root.left, target);
```

or:

```java
return searchBST(root.right, target);
```

---

# 23. Iterative BST Search

BST search can also be written without recursion:

```java
public static TreeNode searchBST(
        TreeNode root,
        int target
) {
    while (root != null) {

        if (root.val == target) {
            return root;
        }

        if (target < root.val) {
            root = root.left;
        } else {
            root = root.right;
        }
    }

    return null;
}
```

Complexity:

```text
Time  → O(h)
Space → O(1)
```

Balanced BST:

```text
O(log n)
```

Worst-case skewed BST:

```text
O(n)
```

---

# 24. Validate Binary Search Tree

This is an important BST interview problem.

Consider:

```text
        5
       / \
      3   8
         /
        2
```

At node `8`:

```text
2 < 8
```

looks locally correct.

But the tree is NOT a valid BST.

Why?

Because `2` belongs to the:

```text
RIGHT subtree of 5
```

Therefore:

```text
2 must be > 5
```

but:

```text
2 < 5
```

So BST validation cannot check only parent-child relationships.

---

# 25. Validate BST — Range Pattern

Each node receives a valid range:

```text
(min, max)
```

The node must satisfy:

```text
min < node.val < max
```

For the root:

```text
(-∞, +∞)
```

For the left child:

```text
(min, node.val)
```

For the right child:

```text
(node.val, max)
```

---

# 26. Validate BST Example

For:

```text
        5
       / \
      3   8
         /
        6
```

ranges become:

```text
5
→ (-∞, +∞)

3
→ (-∞, 5)

8
→ (5, +∞)

6
→ (5, 8)
```

Every node must stay within the constraints created by **all of its ancestors**.

---

# 27. Validate BST Implementation

```java
public static boolean isValidBST(TreeNode root) {
    return valid(
            root,
            Long.MIN_VALUE,
            Long.MAX_VALUE
    );
}

private static boolean valid(
        TreeNode node,
        long min,
        long max
) {
    if (node == null) {
        return true;
    }

    if (node.val <= min || node.val >= max) {
        return false;
    }

    return valid(
            node.left,
            min,
            node.val
    ) && valid(
            node.right,
            node.val,
            max
    );
}
```

Complexity:

```text
Time  → O(n)
Space → O(h)
```

---

# 28. Why Use `long` Bounds?

Avoid:

```java
valid(
    root,
    Integer.MIN_VALUE,
    Integer.MAX_VALUE
);
```

Suppose:

```text
root.val = Integer.MIN_VALUE
```

Then:

```java
node.val <= min
```

becomes true even though the value itself may be valid.

Using:

```java
Long.MIN_VALUE
Long.MAX_VALUE
```

creates bounds outside the entire `int` range.

---

# 29. Validate BST Mental Model

Don't ask:

```text
Is left child smaller?

Is right child larger?
```

Ask:

```text
What RANGE is this node allowed to be in?
```

Then propagate the range:

```text
LEFT
→ upper bound becomes current value

RIGHT
→ lower bound becomes current value
```

---

# 30. Lowest Common Ancestor — BST

Given:

```text
        6
       / \
      2   8
     / \ / \
    0  4 7  9
      / \
     3   5
```

For:

```text
p = 2
q = 8
```

LCA:

```text
6
```

because this is the first point where their paths split.

---

# 31. BST LCA Observation

At current root:

```text
Both p and q < root
→ LCA must be LEFT
```

```text
Both p and q > root
→ LCA must be RIGHT
```

Otherwise:

```text
One is on each side

OR

root is one of p/q
```

Therefore:

```text
current root = LCA
```

---

# 32. Lowest Common Ancestor Implementation

```java
public static TreeNode lowestCommonAncestor(
        TreeNode root,
        TreeNode p,
        TreeNode q
) {
    if (root == null) {
        return null;
    }

    if (p.val < root.val
            && q.val < root.val) {

        return lowestCommonAncestor(
                root.left,
                p,
                q
        );
    }

    if (p.val > root.val
            && q.val > root.val) {

        return lowestCommonAncestor(
                root.right,
                p,
                q
        );
    }

    return root;
}
```

Example:

```text
p = 3
q = 5
```

Trace:

```text
root = 6

3 < 6
5 < 6

→ LEFT
```

Then:

```text
root = 2

3 > 2
5 > 2

→ RIGHT
```

Then:

```text
root = 4

3 < 4
5 > 4

→ SPLIT
```

Therefore:

```text
LCA = 4
```

---

# 33. LCA Complexity

```text
Time  → O(h)
Space → O(h)
```

with recursion.

Balanced BST:

```text
O(log n)
```

Skewed BST:

```text
O(n)
```

---

# 34. BFS — Breadth First Search

DFS:

```text
go deep
```

BFS:

```text
go level by level
```

Example:

```text
        3
       / \
      9   20
         / \
        15  7
```

BFS order:

```text
Level 0 → 3

Level 1 → 9, 20

Level 2 → 15, 7
```

BFS uses a:

```text
QUEUE
```

because Queue provides:

```text
FIFO
```

---

# 35. Level Order Traversal

Expected result:

```text
[
    [3],
    [9, 20],
    [15, 7]
]
```

Initialize:

```java
Queue<TreeNode> queue =
        new ArrayDeque<>();

queue.offer(root);
```

Then process the tree one level at a time.

---

# 36. The Most Important BFS Trick

At the beginning of each level:

```java
int levelSize = queue.size();
```

Then process exactly:

```text
levelSize
```

nodes.

Why?

Suppose:

```text
queue = [9,20]
```

Then:

```text
levelSize = 2
```

While processing them, we might add:

```text
15,7
```

to the queue.

But:

```text
15,7
```

belong to the NEXT level.

Capturing `queue.size()` beforehand prevents us from mixing levels.

---

# 37. Level Order Implementation

```java
public static List<List<Integer>> levelOrder(
        TreeNode root
) {
    List<List<Integer>> result =
            new ArrayList<>();

    if (root == null) {
        return result;
    }

    Queue<TreeNode> queue =
            new ArrayDeque<>();

    queue.offer(root);

    while (!queue.isEmpty()) {

        int levelSize = queue.size();

        List<Integer> currentLevel =
                new ArrayList<>();

        for (int i = 0; i < levelSize; i++) {

            TreeNode node = queue.poll();

            currentLevel.add(node.val);

            if (node.left != null) {
                queue.offer(node.left);
            }

            if (node.right != null) {
                queue.offer(node.right);
            }
        }

        result.add(currentLevel);
    }

    return result;
}
```

Complexity:

```text
Time  → O(n)
Space → O(w)
```

where:

```text
w = maximum width of tree
```

---

# 38. Level Order Trace

Tree:

```text
        3
       / \
      9   20
         / \
        15  7
```

Initially:

```text
queue = [3]

levelSize = 1
```

Process:

```text
3
```

Add children:

```text
queue = [9,20]
```

Result:

```text
[[3]]
```

Next:

```text
levelSize = 2
```

Process:

```text
9
20
```

Add:

```text
15
7
```

Queue:

```text
[15,7]
```

Result:

```text
[
 [3],
 [9,20]
]
```

Finally:

```text
[15,7]
```

Result:

```text
[
 [3],
 [9,20],
 [15,7]
]
```

---

# 39. Important BFS Mistakes

## Mistake 1 — One List Per Node

This:

```java
result.add(List.of(node.val));
```

produces:

```text
[[3], [9], [20], [15], [7]]
```

That's BFS order, but it is **not level grouping**.

Instead create:

```java
List<Integer> currentLevel =
        new ArrayList<>();
```

for each level.

---

## Mistake 2 — Adding Null to ArrayDeque

Don't blindly:

```java
queue.offer(node.left);
queue.offer(node.right);
```

`ArrayDeque` does not allow `null`.

Use:

```java
if (node.left != null) {
    queue.offer(node.left);
}

if (node.right != null) {
    queue.offer(node.right);
}
```

---

## Mistake 3 — Not Capturing Level Size

If you keep processing until the queue is empty inside the same level, newly added children will get mixed with the current level.

Capture:

```java
int levelSize = queue.size();
```

before processing the level.

---

# 40. DFS vs BFS

## DFS

Usually:

```text
Recursion
or
Stack
```

Good for:

```text
Subtree calculations
Depth
Path problems
Tree comparison
Tree transformation
BST validation
```

Mental model:

```text
Go deep
```

---

## BFS

Usually:

```text
Queue
```

Good for:

```text
Level-order traversal
Minimum number of edges/steps
Nearest node
Level calculations
Breadth-based processing
```

Mental model:

```text
Go level by level
```

---

# 41. Tree Pattern Recognition Cheat Sheet

## Traversal

If the problem asks:

```text
Visit every node
Process nodes
Collect values
```

think:

```text
DFS or BFS
```

---

## Recursive Subtree Problem

If the answer depends on:

```text
left subtree answer
+
right subtree answer
```

think:

```text
Tree recursion
```

Template:

```java
if (root == null) {
    return base;
}

var left = solve(root.left);
var right = solve(root.right);

return combine(left, right);
```

---

## Middle / Level Processing

If you see:

```text
level
nearest
minimum number of steps
breadth
```

think:

```text
BFS + Queue
```

---

## BST Search

If:

```text
target < node
→ LEFT

target > node
→ RIGHT
```

think:

```text
Use BST ordering
```

Don't unnecessarily traverse both subtrees.

---

## BST Validation

If asked:

```text
Is this a valid BST?
```

think:

```text
Range constraints

(min, max)
```

not just:

```text
left < parent < right
```

---

## BST LCA

Think:

```text
both left
→ go left

both right
→ go right

split
→ current node
```

---

# 42. Common Mistakes From Our Practice

### Inorder Traversal

If the public method is:

```java
static
```

the directly called helper must also be static unless you're calling it through an instance.

---

### Same Tree

Don't forget:

```java
p.val != q.val
```

And combine subtree results using:

```java
&&
```

not:

```java
==
```

---

### BST Search

Moving:

```java
root = root.left;
```

once isn't recursive search.

Either use:

```java
return searchBST(root.left, target);
```

or an iterative `while`.

---

### Validate BST

Don't only compare with the parent.

Ancestor constraints matter.

Use:

```text
(min, max)
```

---

### Validate BST Bounds

Prefer:

```java
Long.MIN_VALUE
Long.MAX_VALUE
```

instead of integer extremes when node values are `int`.

---

### BFS

Remember the combination:

```text
Queue
+
levelSize
+
currentLevel
```

---

# 43. Complexity Cheat Sheet

| Pattern / Problem | Time | Space |
|---|---:|---:|
| DFS Traversal | `O(n)` | `O(h)` |
| Inorder | `O(n)` | `O(h)` |
| Maximum Depth | `O(n)` | `O(h)` |
| Same Tree | `O(n)` | `O(h)` |
| Invert Tree | `O(n)` | `O(h)` |
| BST Search | `O(h)` | `O(h)` recursive |
| Iterative BST Search | `O(h)` | `O(1)` |
| Validate BST | `O(n)` | `O(h)` |
| BST LCA | `O(h)` | `O(h)` recursive |
| Level Order BFS | `O(n)` | `O(w)` |

Where:

```text
n = number of nodes
h = tree height
w = maximum tree width
```

---

# 44. Tree Complexity Mental Model

For DFS recursion:

```text
Space ≈ tree height
```

Balanced:

```text
        ●
      /   \
     ●     ●
    / \   / \

height ≈ log n
```

So:

```text
O(log n)
```

A skewed tree:

```text
●
 \
  ●
   \
    ●
     \
      ●
```

has:

```text
height ≈ n
```

So recursion stack:

```text
O(n)
```

---

# 45. Revision Checklist

### Binary Tree Fundamentals

- [x] TreeNode structure
- [x] Binary Tree terminology
- [x] DFS
- [x] BFS

### DFS Traversal

- [x] Preorder concept
- [x] Inorder implementation
- [x] Postorder concept

### Tree Recursion

- [x] Base case
- [x] Recursive subtree calls
- [x] Combine subtree answers
- [x] Maximum Depth

### Tree Comparison / Transformation

- [x] Same Tree
- [x] Invert Binary Tree

### BST

- [x] BST ordering property
- [x] Search BST
- [x] Validate BST
- [x] Range validation
- [x] Lowest Common Ancestor

### BFS

- [x] Queue usage
- [x] Level size
- [x] Level Order Traversal

---

# 46. Level 11 Progress

```text
PART 1

✅ Big-O

✅ Arrays & Hashing

✅ Two Pointers

✅ Sliding Window

✅ Prefix Sum

✅ Stack

✅ Monotonic Stack

✅ Queue / Deque Basics

🟡 Monotonic Deque — revisit


PART 2

✅ Binary Search

✅ Binary Search Boundaries

✅ Rotated Binary Search

✅ Binary Search on Answer

✅ Linked List Fundamentals

✅ Reverse Linked List

✅ Slow / Fast Pointers

✅ Cycle Detection

✅ Dummy Node

✅ Merge Sorted Lists

🟡 Remove Nth From End — implementation pending


PART 3

✅ Binary Tree Fundamentals

✅ DFS

✅ Tree Traversal

✅ Tree Recursion

✅ Maximum Depth

✅ Same Tree

✅ Invert Binary Tree

✅ BST Fundamentals

✅ BST Search

✅ Validate BST

✅ Lowest Common Ancestor

✅ BFS / Level Order


NEXT

⬜ Heap / Priority Queue

⬜ Graphs

⬜ Graph BFS / DFS

⬜ Topological Sort

⬜ Union Find

⬜ Recursion

⬜ Backtracking

⬜ Greedy

⬜ Dynamic Programming

⬜ Intervals
```

---

# Final Tree Mental Model

When you see a Tree problem, don't immediately start writing recursion.

First classify it.

```text
TREE PROBLEM
     │
     ├── Need to go deep?
     │
     │      → DFS
     │
     ├── Need level-by-level?
     │
     │      → BFS + Queue
     │
     ├── Need answers from children?
     │
     │      → Recursion
     │        solve left
     │        solve right
     │        combine
     │
     └── Is it a BST?
            │
            └── Use ordering
                to eliminate branches
```

Then remember the core patterns:

```text
Traversal
→ DFS / BFS

Subtree calculation
→ recursion

Depth
→ 1 + max(left, right)

Compare trees
→ current + left + right

Transform tree
→ modify current + recurse

BST search
→ choose one branch

BST validation
→ carry allowed range

BST LCA
→ follow values until paths split

Level order
→ Queue + levelSize
```

The most important shift with Trees is:

> Don't try to understand the entire tree at once.

At each node ask:

```text
What should I do with THIS node?

What should the LEFT subtree return?

What should the RIGHT subtree return?

How do I combine those answers?
```

Once that becomes natural, a large number of Tree interview problems become variations of the same few patterns.