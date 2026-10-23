# Java Level 11 — DSA & Problem Solving
## Part 5: Graphs — Revision Notes

**Status:** In progress  
**Language:** Java  
**Focus:** Interview patterns, implementation, complexity, and problem recognition

---

# 1. Graph Fundamentals

A graph consists of:
- **Vertices (V):** Nodes.
- **Edges (E):** Connections between nodes.

Unlike trees, graphs can contain cycles, disconnected components, and multiple paths between nodes. They don't necessarily have a root.

## Types of graphs

| Type | Meaning |
|---|---|
| Undirected | Edges work in both directions |
| Directed | Edges have a specific direction |
| Unweighted | Edges have no associated weight |
| Weighted | Edges carry costs or distances |
| Connected | Every node is reachable from every other node |
| Disconnected | Some nodes cannot reach others |

### Adjacency list

The most commonly used representation for interview problems.

```java
Map<Integer, List<Integer>> graph = new HashMap<>();
```

Build an undirected graph:

```java
for (int i = 0; i < n; i++) {
    graph.put(i, new ArrayList<>());
}

for (int[] edge : edges) {
    int u = edge[0];
    int v = edge[1];

    graph.get(u).add(v);
    graph.get(v).add(u);
}
```

For a directed graph, add only `u → v`:

```java
graph.get(u).add(v);
```

**Complexity:** Building an adjacency list takes O(V + E) time and space.

---

# 2. DFS — Depth-First Search

DFS explores as far as possible along one path before backtracking.

It uses recursion (the call stack) or an explicit stack.

## Pattern: Does a path exist?

```java
public static boolean dfs(
        Map<Integer, List<Integer>> graph,
        int source,
        int destination,
        Set<Integer> visited
) {
    if (source == destination) {
        return true;
    }

    if (visited.contains(source)) {
        return false;
    }

    visited.add(source);

    for (int neighbor :
            graph.getOrDefault(source, List.of())) {
        if (dfs(graph, neighbor, destination, visited)) {
            return true;
        }
    }

    return false;
}
```

### Mental model

- `source` changes as DFS moves to neighboring nodes.
- `destination` stays fixed.
- `visited` is shared across recursive calls.
- Return the recursive result when the destination is found.

**Common mistake:** Calling DFS recursively without returning its result. This can cause a successful path to be ignored.

**Complexity:** O(V + E) time and O(V) auxiliary space, excluding the graph.

---

# 3. BFS — Breadth-First Search

BFS explores a graph level by level using a FIFO queue.

```java
Queue<Integer> queue = new ArrayDeque<>();
Set<Integer> visited = new HashSet<>();

visited.add(source);
queue.offer(source);

while (!queue.isEmpty()) {
    int node = queue.poll();

    for (int neighbor :
            graph.getOrDefault(node, List.of())) {
        if (!visited.contains(neighbor)) {
            visited.add(neighbor);
            queue.offer(neighbor);
        }
    }
}
```

Mark nodes visited **when enqueuing**, not when dequeuing. This prevents a node from being added multiple times.

### DFS vs BFS

| DFS | BFS |
|---|---|
| Recursion or stack | Queue |
| Goes deep, then backtracks | Explores level by level |
| Useful for components and cycle detection | Useful for shortest paths in unweighted graphs |
| Does not guarantee a shortest path | Finds shortest distance in number of edges |

**Important:** Use `ArrayDeque` for a normal BFS queue, not `PriorityQueue`. A priority queue orders elements by priority rather than arrival time.

---

# 4. Connected Components

**Problem:** Count the number of disconnected groups in an undirected graph.

Example:

```text
0 --- 1 --- 2

3 --- 4

5
```

There are three connected components: `{0,1,2}`, `{3,4}`, and `{5}`.

## Core pattern

> One DFS discovers one component. A loop around DFS discovers all components.

```java
public static int countComponents(
        int n,
        int[][] edges
) {
    Map<Integer, List<Integer>> graph = new HashMap<>();

    for (int i = 0; i < n; i++) {
        graph.put(i, new ArrayList<>());
    }

    for (int[] edge : edges) {
        int u = edge[0];
        int v = edge[1];

        graph.get(u).add(v);
        graph.get(v).add(u);
    }

    Set<Integer> visited = new HashSet<>();
    int components = 0;

    for (int node = 0; node < n; node++) {
        if (!visited.contains(node)) {
            components++;
            dfsVisit(node, graph, visited);
        }
    }

    return components;
}

private static void dfsVisit(
        int node,
        Map<Integer, List<Integer>> graph,
        Set<Integer> visited
) {
    visited.add(node);

    for (int neighbor :
            graph.getOrDefault(node, List.of())) {
        if (!visited.contains(neighbor)) {
            dfsVisit(neighbor, graph, visited);
        }
    }
}
```

**Common mistake:** Forgetting to check `visited` before the recursive call. In an undirected graph, this can cause infinite recursion between two connected nodes.

**Complexity:** O(V + E) time; O(V + E) space including the graph.

---

# 5. Number of Islands

**Problem:** Given a grid of `'1'` (land) and `'0'` (water), count connected land regions.

Only horizontal and vertical neighbors connect. Diagonal cells do not.

## Core pattern

1. Iterate through every cell.
2. If the cell is water, skip it.
3. If it is land, increment the island count.
4. DFS marks every connected land cell as water.
5. Continue scanning.

```java
public static int numIslands(char[][] grid) {
    int count = 0;

    for (int row = 0; row < grid.length; row++) {
        for (int col = 0; col < grid[row].length; col++) {
            if (grid[row][col] == '1') {
                count++;
                dfsIsland(grid, row, col);
            }
        }
    }

    return count;
}

private static void dfsIsland(
        char[][] grid,
        int row,
        int col
) {
    if (row < 0 || row >= grid.length
            || col < 0 || col >= grid[row].length
            || grid[row][col] != '1') {
        return;
    }

    // Mark visited
    grid[row][col] = '0';

    dfsIsland(grid, row + 1, col);
    dfsIsland(grid, row - 1, col);
    dfsIsland(grid, row, col + 1);
    dfsIsland(grid, row, col - 1);
}
```

### Important insight

The grid is already an implicit graph. You don't need to build an adjacency list or create a separate visited set if you can modify the grid.

**Complexity:** O(R × C) time and O(R × C) worst-case recursion stack space.

**Trade-off:** This implementation modifies the input grid.

---

# 6. Flood Fill

**Problem:** Starting from a pixel `(sr, sc)`, recolor every connected pixel having the same original color.

## Core pattern

Flood Fill uses the same four-directional DFS as Number of Islands.

The difference is that instead of turning visited cells into water, we replace their color.

```java
public static int[][] floodFill(
        int[][] image,
        int sr,
        int sc,
        int color
) {
    int originalColor = image[sr][sc];

    // Important edge case
    if (originalColor == color) {
        return image;
    }

    dfsFloodFill(
            image, sr, sc, color, originalColor
    );

    return image;
}

private static void dfsFloodFill(
        int[][] image,
        int row,
        int col,
        int newColor,
        int originalColor
) {
    if (row < 0 || row >= image.length
            || col < 0 || col >= image[row].length
            || image[row][col] != originalColor) {
        return;
    }

    image[row][col] = newColor;

    dfsFloodFill(image, row + 1, col,
            newColor, originalColor);
    dfsFloodFill(image, row - 1, col,
            newColor, originalColor);
    dfsFloodFill(image, row, col + 1,
            newColor, originalColor);
    dfsFloodFill(image, row, col - 1,
            newColor, originalColor);
}
```

### Why check `originalColor == color`?

If the colors are already equal, recoloring won't mark a cell as visited. DFS could repeatedly revisit the same-color cells.

**Complexity:** O(R × C) time and O(R × C) worst-case recursion stack space.

**Trade-off:** The image is modified in place.

---

# 7. Cycle Detection — Undirected Graph

**Problem:** Determine whether an undirected graph contains a cycle.

## Key rule

> If a neighbor is already visited and is not the current node's parent, a cycle exists.

Why the parent check? Every undirected edge can be traversed in both directions. Seeing the parent again is normal, not necessarily a cycle.

```java
public static boolean hasCycle(
        int n,
        int[][] edges
) {
    Map<Integer, List<Integer>> graph = new HashMap<>();

    for (int i = 0; i < n; i++) {
        graph.put(i, new ArrayList<>());
    }

    for (int[] edge : edges) {
        int u = edge[0];
        int v = edge[1];

        graph.get(u).add(v);
        graph.get(v).add(u);
    }

    Set<Integer> visited = new HashSet<>();

    // Handles disconnected graphs too
    for (int node = 0; node < n; node++) {
        if (!visited.contains(node)) {
            if (hasCycleDfs(graph, node, -1, visited)) {
                return true;
            }
        }
    }

    return false;
}

private static boolean hasCycleDfs(
        Map<Integer, List<Integer>> graph,
        int node,
        int parent,
        Set<Integer> visited
) {
    visited.add(node);

    for (int neighbor :
            graph.getOrDefault(node, List.of())) {
        if (!visited.contains(neighbor)) {
            if (hasCycleDfs(
                    graph, neighbor, node, visited
            )) {
                return true;
            }
        } else if (neighbor != parent) {
            return true;
        }
    }

    return false;
}
```

### Common mistakes

- Treating every visited neighbor as a cycle.
- Forgetting the parent parameter.
- Ignoring the recursive call's boolean result.
- Checking only one component of a disconnected graph.

**Complexity:** O(V + E) time and O(V + E) space including the graph.

**Note:** This parent-based implementation assumes a standard undirected graph without parallel edges. If parallel edges are allowed, cycle detection needs to account for edge identity.

---

# 8. Shortest Path — Unweighted Graph

**Problem:** Find the minimum number of edges needed to travel from `source` to `destination`.

## Why BFS?

BFS explores all nodes at distance 0, then distance 1, then distance 2, and so on. Therefore, the first time it reaches the destination, it has found a shortest path.

## Implementation

```java
public static int shortestPath(
        int n,
        int[][] edges,
        int source,
        int destination
) {
    Map<Integer, List<Integer>> graph = new HashMap<>();

    for (int i = 0; i < n; i++) {
        graph.put(i, new ArrayList<>());
    }

    for (int[] edge : edges) {
        int u = edge[0];
        int v = edge[1];

        graph.get(u).add(v);
        graph.get(v).add(u);
    }

    int[] distance = new int[n];
    Arrays.fill(distance, -1);

    Queue<Integer> queue = new ArrayDeque<>();

    distance[source] = 0;
    queue.offer(source);

    while (!queue.isEmpty()) {
        int node = queue.poll();

        if (node == destination) {
            return distance[node];
        }

        for (int neighbor : graph.get(node)) {
            if (distance[neighbor] == -1) {
                distance[neighbor] = distance[node] + 1;
                queue.offer(neighbor);
            }
        }
    }

    // Destination is unreachable
    return -1;
}
```

### Why use `distance[]` instead of a separate `visited` set?

`distance[neighbor] == -1` means the node has not been discovered. Assigning its distance when enqueuing it also marks it visited.

This prevents duplicate queue entries.

### Distance versus path

The method above returns the **distance**, not the actual path. To reconstruct the path, maintain a `parent[]` array and walk backward from the destination.

**Complexity:** O(V + E) time and O(V + E) space including the graph.

**Important:** Ordinary BFS finds shortest paths in unweighted graphs. For weighted graphs with non-negative edge weights, Dijkstra's algorithm is a common choice.

---

# 9. Course Schedule — Directed Cycle Detection

**Problem:** Given `numCourses` and prerequisite pairs `[course, prerequisite]`, determine whether all courses can be completed.

Each pair means:

`prerequisite → course`

If the dependency graph has a directed cycle, the courses in that cycle cannot all be completed.

## Three-state DFS

| State | Meaning |
|---|---|
| `0` | Unvisited |
| `1` | Visiting — currently on the recursion path |
| `2` | Processed — DFS finished exploring this node |

**Key rule:** Reaching a node in state `1` means a directed cycle exists.

Reaching state `2` is safe: that node has already been fully explored.

## Implementation

```java
public static boolean canFinish(
        int numCourses,
        int[][] prerequisites
) {
    Map<Integer, List<Integer>> graph = new HashMap<>();

    for (int i = 0; i < numCourses; i++) {
        graph.put(i, new ArrayList<>());
    }

    for (int[] prerequisite : prerequisites) {
        int course = prerequisite[0];
        int required = prerequisite[1];

        graph.get(required).add(course);
    }

    int[] state = new int[numCourses];

    for (int course = 0; course < numCourses; course++) {
        if (state[course] == 0) {
            if (hasDirectedCycle(graph, course, state)) {
                return false;
            }
        }
    }

    return true;
}

private static boolean hasDirectedCycle(
        Map<Integer, List<Integer>> graph,
        int course,
        int[] state
) {
    if (state[course] == 1) {
        return true;
    }

    if (state[course] == 2) {
        return false;
    }

    state[course] = 1;

    for (int neighbor : graph.get(course)) {
        if (hasDirectedCycle(graph, neighbor, state)) {
            return true;
        }
    }

    state[course] = 2;
    return false;
}
```

### Why this differs from undirected cycle detection

| Undirected graph | Directed graph |
|---|---|
| Track visited nodes and parent | Track three DFS states |
| Ignore the parent edge | Detect a node already being visited |
| Visited non-parent neighbor signals a cycle | State `1` signals a cycle |
| State can be represented with a set | State `0`, `1`, `2` distinguishes active and finished nodes |

**Complexity:** O(V + E) time and O(V + E) space including the graph.

### Connection to Topological Sorting

If the directed graph has no cycle, it is a DAG (Directed Acyclic Graph), and its nodes can be ordered so that prerequisites come before dependent courses.

Topological sorting is the next topic to study. Two important approaches are:

1. DFS finishing order.
2. Kahn's algorithm using indegrees and a queue.

---

# 10. Pattern Recognition Cheat Sheet

| If the question asks… | Think… |
|---|---|
| Is there a path between two nodes? | DFS or BFS |
| How many disconnected groups exist? | Outer loop + DFS/BFS |
| How many islands are in a grid? | Grid DFS/BFS |
| Recolor a connected region | Flood Fill |
| Does an undirected graph contain a cycle? | DFS + parent |
| Does a directed graph contain a cycle? | DFS states: 0, 1, 2 |
| Minimum edges in an unweighted graph | BFS + distance |
| Can all prerequisites be completed? | Directed cycle detection |
| Order tasks subject to dependencies | Topological sorting |
| Shortest path in a weighted graph with non-negative weights | Dijkstra's algorithm |

---

# 11. Common Graph Mistakes

- Forgetting to mark a node visited.
- Marking nodes only after dequeuing them in BFS.
- Ignoring recursive return values when searching for paths or cycles.
- Confusing an undirected edge back to the parent with a cycle.
- Using undirected cycle-detection logic on a directed graph.
- Forgetting disconnected components in the outer loop.
- Building an adjacency list when a grid already represents the graph.
- Forgetting to preserve the original color in Flood Fill.
- Forgetting that modifying a grid changes the input.
- Using BFS for weighted shortest paths without checking whether ordinary BFS is appropriate.

---

# 12. Complexity Cheat Sheet

Let V = vertices, E = edges, R = rows, and C = columns.

| Pattern | Time | Auxiliary space |
|---|---|---|
| DFS / BFS traversal | O(V + E) | O(V) |
| Connected components | O(V + E) | O(V), excluding graph |
| Number of Islands | O(R × C) | O(R × C) worst case |
| Flood Fill | O(R × C) | O(R × C) worst case |
| Undirected cycle detection | O(V + E) | O(V), excluding graph |
| Shortest path using BFS | O(V + E) | O(V), excluding graph |
| Course Schedule using DFS | O(V + E) | O(V), excluding graph |

The total space for graph-based implementations includes O(V + E) for the adjacency list.

---

# 13. Revision Checklist

## Fundamentals
- [ ] Explain directed versus undirected graphs.
- [ ] Build an adjacency list.
- [ ] Explain DFS versus BFS.
- [ ] Explain why visited nodes must be tracked.

## Traversal and components
- [x] Path existence using DFS.
- [x] BFS traversal and path existence.
- [x] Count connected components.
- [x] Number of Islands.
- [x] Flood Fill.

## Cycle detection and shortest paths
- [x] Cycle detection in an undirected graph.
- [x] Shortest distance in an unweighted graph.
- [x] Course Schedule using directed cycle detection.

## Upcoming topics
- [ ] Topological sorting with DFS.
- [ ] Kahn's algorithm and indegrees.
- [ ] Practice more graph problems.
- [ ] Revisit graph patterns without looking at notes.

---

# 14. Level 11 Progress Tracker

| Part | Topic | Status |
|---|---|---|
| Part 1 | Core Patterns | Covered; revisit sliding-window intuition and monotonic deque |
| Part 2 | Binary Search & Linked Lists | Covered; revisit Remove Nth Node From End |
| Part 3 | Trees & BST | Covered |
| Part 4 | Heap / Priority Queue | Covered; revisit Merge K Sorted Lists |
| Part 5 | Graphs | In progress |

**Graph checkpoint:** The core traversal, grid traversal, undirected cycle detection, unweighted shortest path, and Course Schedule patterns are covered. Topological sorting is next.

**Study rule:** Understand the pattern, implement it once without notes, and revisit only the concepts where the reasoning or implementation still feels unclear.
