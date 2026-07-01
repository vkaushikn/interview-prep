# Graphs and Trees 

## DFS is your best friend

Use DFS as your workhorse best friend for most of the questions. Memorize the following DFS mega code and optimize as required in the question. 
Strategically, note that we are using recursion because it is easier in a interview setting, we should use iterative in production if required.

```python
def explore(nodes: list[Node], is_directed: bool):
    """
    Assumes Node is defined as:
        class Node:
            id: int | str
            val: Any
            children: list[Node]

    A Tree will exactly have two children (mostly tracked separately as Left / Right).
    Pass in all nodes so we can handle disconnected graphs.
    """

    # State
    paths: dict[int, list[int]] = {}         # path taken to reach each node
    topo_sort: list[int] = []                # populated post-order, reversed at the end
    connected_components: list[list[int]] = []

    visited: set[int] = set()               # globally visited node ids

    def dfs(node: Node, parent: Node | None, path: set[int], component: list[int]) -> bool:
        """
        path   — node ids on the current DFS path (for cycle detection).
                 We pass a COPY on each recursive call so backtracking is implicit.
                 (Backtracking alternative: add before call, remove after — same result, less memory)
        Returns True if a cycle is found.
        """

        paths[node.id] = list(path)
        component.append(node.id)

        for child in node.children:

            # For undirected graphs, skip the edge back to parent — not a cycle
            if not is_directed and child == parent:
                continue

            # Child already on current path → cycle
            if child.id in path:
                return True

            if child.id not in visited:
                visited.add(child.id)
                if dfs(child, node, path | {child.id}, component):  # path | {child.id} is a copy
                    return True

        topo_sort.append(node.id)  # post-order: append after all children processed
        return False

    for node in nodes:
        if node.id not in visited:
            visited.add(node.id)
            component: list[int] = []
            if dfs(node, None, {node.id}, component):
                return None, None, None          # cycle found — topo sort undefined
            connected_components.append(component)

    return connected_components, paths, topo_sort[::-1]
```

Optimizations for specific problems:
- Only need topo sort? Remove `paths` and `connected_components`.
- Only need cycle detection? Return early, drop `topo_sort`.
- Tree (single root, no disconnected nodes)? Drop the outer `for` loop, just call `dfs(root, None, {root.id}, [])`.

When to keep `path` (cycle detection):
- General graph with no cycle guarantee → keep `path`
- Problem explicitly states DAG → drop `path`, trust the input
- Tree → drop `path` entirely, trees have no cycles by definition

---

## Iterative DFS / BFS

Keep this in your back-pocket for problems where you do not need backtracking or cycle detection via path.
Swap `stack` / `stack.pop()` for `deque` / `queue.popleft()` to get BFS.

```python
def explore(root: Node):
    visited = {root.id}
    stack = [root]              # deque([root]) for BFS
    parents = {root.id: None}

    while stack:
        node = stack.pop()      # queue.popleft() for BFS

        # Do work here — e.g., call URL in web crawler, record level in level-order

        for child in node.children:
            if child.id not in visited:
                visited.add(child.id)
                parents[child.id] = node.id
                stack.append(child)     # queue.append(child) for BFS
```

---

## When to use BFS

Use BFS when the problem mentions:
- Level order
- Shortest path / fewest steps
- Fastest route

Otherwise default to recursive DFS.

---

## Tree Traversal Patterns

Trees are graphs with exactly two children (left / right). The key question for any tree DFS:
**what do I return to my parent?** Answer that first, then the recursion writes itself.

### Pre-Order — Node → Left → Right
```python
visit(node)
dfs(node.left)
dfs(node.right)
```
Use for: clone/copy a tree.

### Post-Order — Left → Right → Node
```python
dfs(node.left)
dfs(node.right)
visit(node)   # compute answer using left and right results
```
Use for: anything where you need the answer from children before computing the current node.
E.g., depth, diameter, path sum, LCA.
```python
depth = max(depth(node.left), depth(node.right)) + 1
```

### In-Order — Left → Node → Right
```python
dfs(node.left)
visit(node)
dfs(node.right)
```
Use for: sorted order in a Binary Search Tree.

---

## Tree Pattern Recognition — The Hard Part

The code is not the problem. Deriving the recursion relationship is. Two questions to answer before writing any code:

**Question 1: Can my parent use my full answer, or only part of it?**

- Yes → return the full answer up, no global tracker needed
- No → return only the part the parent can use, track the full answer globally

| Problem | What parent can use | What to track globally |
|---------|-------------------|----------------------|
| Max depth | full depth | nothing |
| Path sum (root to leaf) | True/False | nothing |
| LCA | node or None | nothing |
| Validate BST | True/False + bounds | nothing |
| Diameter | one side only (`1 + max(left, right)`) | `left + right` at each node |
| Max path sum | one side only (`max(left, right) + node.val`) | `left + right + node.val` at each node |
| Balanced tree | height + bool | nothing (short circuit) |

**Question 2: What is the base case (None or leaf)?**
- Usually `None → return 0` (depth problems) or `None → return None` (search problems) or `None → return True` (validation problems)

**The split pattern** (diameter, max path sum):
- These are the hardest because you maintain TWO values simultaneously
- Return to parent: `1 + max(left, right)` — can only go one way
- Track globally: `left + right` — the full path through this node
- Whenever a problem asks for "longest/maximum path between ANY two nodes" → expect this split

**Pre-order vs Post-order:**
- Passing information DOWN (bounds, remaining sum, flags) → Pre-order
- Combining information FROM children → Post-order (most tree problems)

**Examples by category:**
- Pass bounds down: Validate BST (`min_val, max_val`), Path sum (`remaining`)
- Combine from children: Max depth, Diameter, LCA, Max path sum, Symmetric tree
- Both: Balanced tree (pass nothing down, combine height + bool up)

---

## Tree Problem Code Reference

### Max Depth
```python
def maxDepth(root):
    if root is None: return 0
    return 1 + max(maxDepth(root.left), maxDepth(root.right))
```

### Diameter
```python
def diameterOfBinaryTree(root):
    res = 0
    def dfs(node):
        nonlocal res
        if node is None: return 0
        left = dfs(node.left)
        right = dfs(node.right)
        res = max(res, left + right)   # full path through this node
        return 1 + max(left, right)    # one side only to parent
    dfs(root)
    return res
```

### Max Path Sum
```python
def maxPathSum(root):
    res = float('-inf')
    def dfs(node):
        nonlocal res
        if node is None: return 0
        left = max(dfs(node.left), 0)   # ignore negative subtrees
        right = max(dfs(node.right), 0)
        res = max(res, node.val + left + right)
        return node.val + max(left, right)
    dfs(root)
    return res
```

### LCA
```python
def lowestCommonAncestor(root, p, q):
    def dfs(node):
        if node is None: return None
        if node == p or node == q: return node
        left = dfs(node.left)
        right = dfs(node.right)
        if left and right: return node
        return left or right
    return dfs(root)
```

### Validate BST
```python
def isValidBST(root):
    def dfs(node, min_val, max_val):
        if node is None: return True
        if not (min_val < node.val < max_val): return False
        return dfs(node.left, min_val, node.val) and dfs(node.right, node.val, max_val)
    return dfs(root, float('-inf'), float('inf'))
```

### Balanced Tree
```python
def isBalanced(root):
    def dfs(node):
        if node is None: return True, 0
        left_bal, left_h = dfs(node.left)
        right_bal, right_h = dfs(node.right)
        balanced = left_bal and right_bal and abs(left_h - right_h) <= 1
        return balanced, 1 + max(left_h, right_h)
    return dfs(root)[0]
```

### Symmetric Tree
```python
def isSymmetric(root):
    def mirror(left, right):
        match (left, right):
            case (None, None): return True
            case (_, None) | (None, _): return False
            case (_, _): return left.val == right.val and mirror(left.left, right.right) and mirror(left.right, right.left)
    return mirror(root.left, root.right)
```

### Path Sum (root to leaf)
```python
def hasPathSum(root, target):
    def dfs(node, remaining):
        if node is None: return False
        if not node.left and not node.right:
            return remaining == node.val
        return dfs(node.left, remaining - node.val) or dfs(node.right, remaining - node.val)
    return dfs(root, target)
```

### All Root-to-Leaf Paths
```python
def binaryTreePaths(root):
    def dfs(node):
        if node is None: return []
        if not node.left and not node.right: return [[node.val]]
        left = dfs(node.left)
        right = dfs(node.right)
        return [[node.val] + p for p in left + right]
    return ["->".join(str(v) for v in p) for p in dfs(root)]
```

### Right Side View
```python
def rightSideView(root):
    out = defaultdict(list)
    def dfs(node, level):
        if node is None: return
        out[level].append(node.val)
        dfs(node.left, level + 1)
        dfs(node.right, level + 1)
    dfs(root, 0)
    return [out[l][-1] for l in range(len(out))]
```

### Level Order / Level Averages
```python
def levelOrder(root):                          # swap last line for averages
    result = []
    q = deque([root])
    while q:
        level_size = len(q)
        level = []
        for _ in range(level_size):
            node = q.popleft()
            level.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        result.append(level)                   # or: sum(level)/len(level)
    return result
```

### Sum of Left Leaves
```python
def sumOfLeftLeaves(root):
    def dfs(node, is_left):
        if node is None: return 0
        if not node.left and not node.right:
            return node.val if is_left else 0
        return dfs(node.left, True) + dfs(node.right, False)
    return dfs(root, False)
```

### Path Sum III (any downward path)
```python
def pathSum(root, target):
    def dfs(node):
        if node is None: return []
        left = dfs(node.left)
        right = dfs(node.right)
        return [node.val] + [node.val + s for s in left + right]
    
    def count(node):
        if node is None: return 0
        return dfs(node).count(target) + count(node.left) + count(node.right)
    
    # deduplicate: dfs already includes subtree sums, so just call once from root
    all_sums = []
    def collect(node):
        if node is None: return
        all_sums.extend(dfs(node))  # wrong — overcounts
        collect(node.left)
        collect(node.right)
    # Cleaner O(N^2):
    def count_from(node, remaining):
        if node is None: return 0
        found = 1 if node.val == remaining else 0
        return found + count_from(node.left, remaining - node.val) + count_from(node.right, remaining - node.val)
    def dfs2(node):
        if node is None: return 0
        return count_from(node, target) + dfs2(node.left) + dfs2(node.right)
    return dfs2(root)
```
