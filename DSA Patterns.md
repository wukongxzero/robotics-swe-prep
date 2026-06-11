---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [dsa, neetcode, interview-prep]

# DSA Patterns

> [!star] This is your active interview-prep front — NeetCode 150 in C++ Not robotics knowledge — the coding-screen layer. The named plan: NeetCode 150 in C++, in pattern order, timed, spoken aloud. This note is the pattern index to drill against, not a place to expand into a textbook.

> [!question] Explain it cold
> 
> - Name the core patterns and the signal that triggers each.
> - For your top-3 weak patterns, what's the template?
> - What's the time/space complexity you'd state for each?

---

## Core idea

Most interview problems are one of ~15 patterns wearing a costume. The skill isn't memorizing 150 solutions — it's recognizing _which pattern_ a problem is from its signal words, then applying the template. NeetCode 150 is organized by these patterns on purpose; drill by pattern, not randomly.

## The pattern index (the trigger → pattern map)

- **Arrays/Hashing** — "seen before?", counts, dedup → hash map/set. O(n).
- **Two Pointers** — sorted array, pair/triplet, palindrome → converging pointers. O(n).
- **Sliding Window** — "longest/shortest substring/subarray with property" → expand/contract window. O(n).
- **Stack** — matching/nesting, "next greater", monotonic → stack. O(n).
- **Binary Search** — sorted, or "minimize the max / search the answer space" → binary search. O(log n).
- **Linked List** — reversal, cycle (Floyd's fast/slow), merge → pointer manipulation.
- **Trees** — DFS (recursion) / BFS (queue), BST property → traversal. O(n).
- **Tries** — prefix/word lookups → trie.
- **Heap / Priority Queue** — "top-k", "k largest/smallest", streaming median → heap. O(n log k).
- **Backtracking** — permutations/combinations/subsets, "all possible" → recursive choose/explore/unchoose. Exponential.
- **Graphs** — connectivity, shortest path, islands → BFS/DFS/Union-Find; Dijkstra (weighted), topological sort (DAG/ordering).
- **Dynamic Programming** — "number of ways", "min/max cost", overlapping subproblems → memoize/tabulate. 1D, 2D, knapsack, LIS.
- **Greedy** — local optimal → global (intervals, scheduling).
- **Intervals** — overlapping/merge/insert → sort by start, sweep.
- **Bit Manipulation** — XOR tricks, masks.

## Method (the named plan)

- **C++** (your interview language), in **pattern order** (not random), **timed**, **spoken aloud** — articulating the approach is the actual bottleneck, not the coding. Treat every problem as a mock: state the pattern, the approach, the complexity, _then_ code.
- Track which patterns are slow/shaky here; those are the drill targets.
- **Gate**: mock interviews start once NeetCode ~50% done and ROS2 executors / FK / IK are fluently articulable.

## RSE coding screen — what actually gets asked

RSE roles (Boston Dynamics, Waymo, Figure, Cruise, etc.) test **medium** difficulty, in this priority order:

- **Graphs / BFS / DFS** — #1 most relevant. Path planning, grid traversal, connectivity. Often wrapped in robotics language ("robot on a grid", "shortest path between waypoints") — still BFS underneath.
- **Arrays / Hashing** — sensor data, dedup, counting
- **Sliding Window** — time series, signal buffers
- **Binary Search** — search on answer space
- **Trees, Heap** — standard, show up regularly
- **Hard DP, tries, bit manipulation** — rare, not the focus

**Drill order for RSE:** Graphs first → Arrays/Hashing → Sliding Window → Binary Search.

---

## Graph templates (memorize these)

```python
# BFS — level-by-level, shortest path in unweighted graph
from collections import deque
def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# DFS — recursive, depth-first
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```

**Graph drill order (NeetCode):**
1. Number of Islands — easy, BFS/DFS on grid
2. Clone Graph — medium
3. Course Schedule — medium, topological sort

**Concept checklist before drilling:**
- [ ] What a graph is: nodes, edges, adjacency list vs matrix
- [ ] BFS: queue + visited set, level-by-level
- [ ] DFS: stack/recursion + visited set, depth-first

---

## Grid BFS — shortest path on a grid (RSE #1 pattern)

**Signal words:** "shortest path", "minimum steps/moves", "robot on a grid", "reach the target", grid with walls/obstacles.

**Why BFS and not DFS?** BFS explores level-by-level — every node at distance 1 before distance 2, etc. The first time you reach the target, that path is guaranteed shortest. DFS can find _a_ path but not the _shortest_ one.

### How to think about it (before writing any code)

1. **Model the grid as a graph.** Each cell `(r, c)` is a node. Edges connect to valid neighbors (up/down/left/right, not a wall, not out of bounds).
2. **BFS from start.** Use a queue. Mark cells visited as you enqueue them (not as you dequeue — or you'll enqueue duplicates).
3. **Track distance.** Either store `(row, col, distance)` in the queue, or use a separate `dist` grid.
4. **Stop when you dequeue the target.** Return the distance. If the queue empties without hitting the target, return -1 (no path).

### Step-by-step spoken approach (say this out loud first)

> "This is a shortest-path-on-a-grid problem — BFS, because BFS gives shortest path in unweighted graphs. I'll treat each cell as a node, use a queue seeded with the start, mark visited as I enqueue, and track distance. When I dequeue the target I return the distance. Time: O(R×C), space: O(R×C) for visited."

### Template — Python

```python
from collections import deque

def shortest_path_grid(grid, start, end):
    ROWS, COLS = len(grid), len(grid[0])
    sr, sc = start
    er, ec = end
    
    # 4-directional moves
    directions = [(0,1),(0,-1),(1,0),(-1,0)]
    
    visited = set()
    visited.add((sr, sc))
    queue = deque([(sr, sc, 0)])  # (row, col, distance)
    
    while queue:
        r, c, dist = queue.popleft()
        
        if r == er and c == ec:
            return dist          # first time we reach end = shortest
        
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and          # in bounds
                0 <= nc < COLS and          # in bounds
                grid[nr][nc] != 1 and       # not a wall (1 = wall)
                (nr, nc) not in visited):
                visited.add((nr, nc))       # mark on enqueue, not dequeue
                queue.append((nr, nc, dist + 1))
    
    return -1   # no path found
```

### Template — C++ (your interview language)

```cpp
#include <queue>
#include <vector>
#include <tuple>
using namespace std;

int shortestPathGrid(vector<vector<int>>& grid, pair<int,int> start, pair<int,int> end) {
    int ROWS = grid.size(), COLS = grid[0].size();
    auto [sr, sc] = start;
    auto [er, ec] = end;
    
    vector<vector<bool>> visited(ROWS, vector<bool>(COLS, false));
    vector<pair<int,int>> dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    
    queue<tuple<int,int,int>> q;   // (row, col, dist)
    q.push({sr, sc, 0});
    visited[sr][sc] = true;
    
    while (!q.empty()) {
        auto [r, c, dist] = q.front(); q.pop();
        
        if (r == er && c == ec) return dist;
        
        for (auto [dr, dc] : dirs) {
            int nr = r + dr, nc = c + dc;
            if (nr >= 0 && nr < ROWS &&
                nc >= 0 && nc < COLS &&
                grid[nr][nc] != 1 &&
                !visited[nr][nc]) {
                visited[nr][nc] = true;
                q.push({nr, nc, dist + 1});
            }
        }
    }
    return -1;
}
```

### Complexity
- **Time:** O(R × C) — each cell visited at most once.
- **Space:** O(R × C) — visited array + queue.

### Common traps
- Marking visited on **dequeue** instead of **enqueue** → the same cell gets added to the queue multiple times → exponential blowup.
- Forgetting bounds check before the wall check → array out-of-bounds crash.
- Using DFS for shortest path → finds _a_ path, not the _shortest_ one.
- 8-directional vs 4-directional — read the problem; default is 4.

---

## A* — heuristic shortest path (robotics-relevant)

**Signal words:** shortest path on a **weighted** grid or graph where you know the goal location in advance and can estimate remaining distance. Common in robotics: Nav2 global planner, motion planning, any grid-based path planning.

**Mental model:** Dijkstra + a heuristic `h(n)` that estimates remaining cost to goal. Priority queue ordered by `f(n) = g(n) + h(n)` — actual cost so far + estimated remaining. Expands toward the goal instead of fanning out evenly.

### The three terms

| Term | Meaning |
|------|---------|
| `g(n)` | Actual cost from start to `n` |
| `h(n)` | Heuristic — estimated cost from `n` to goal (never overestimate for optimality) |
| `f(n)` | `g(n) + h(n)` — priority queue key |

**Admissible heuristic:** `h(n)` never overestimates true cost → A* is guaranteed optimal.

**Grid heuristics:**
- 4-directional movement → Manhattan distance: `|r - gr| + |c - gc|`
- 8-directional movement → Chebyshev distance: `max(|r - gr|, |c - gc|)`
- Continuous space → Euclidean distance

### BFS vs Dijkstra vs A*

| Algorithm | Queue type | When to use |
|-----------|-----------|-------------|
| BFS | FIFO queue | Unweighted graph — all edges cost 1 |
| Dijkstra | Min-heap on `g(n)` | Weighted graph, goal unknown or many goals |
| A* | Min-heap on `f(n) = g(n) + h(n)` | Weighted graph, single goal, can estimate distance |

**Key insight:** BFS fans out in rings. Dijkstra fans out by cost. A* fans out *toward the goal* — it skips dead-ends behind you. In practice explores far fewer nodes than BFS/Dijkstra on large open grids.

### Degenerate cases
- `h = 0` → A* = Dijkstra
- `h = perfect` → A* only expands nodes on the optimal path
- `h` inadmissible (overestimates) → faster but may miss optimal

### Template — Python

```python
import heapq

def astar(grid, start, goal):
    ROWS, COLS = len(grid), len(grid[0])
    sr, sc = start
    gr, gc = goal

    def h(r, c):
        return abs(r - gr) + abs(c - gc)   # Manhattan distance

    dirs = [(0,1),(0,-1),(1,0),(-1,0)]
    g = {(sr, sc): 0}
    pq = [(h(sr, sc), 0, sr, sc)]   # (f, g, row, col)

    while pq:
        f, cost, r, c = heapq.heappop(pq)

        if r == gr and c == gc:
            return cost

        if cost > g.get((r, c), float('inf')):
            continue   # stale entry

        for dr, dc in dirs:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS and grid[nr][nc] != 1):
                new_g = cost + 1
                if new_g < g.get((nr, nc), float('inf')):
                    g[(nr, nc)] = new_g
                    heapq.heappush(pq, (new_g + h(nr, nc), new_g, nr, nc))

    return -1
```

### Template — C++

```cpp
#include <queue>
#include <vector>
#include <tuple>
#include <unordered_map>
using namespace std;

int astar(vector<vector<int>>& grid, pair<int,int> start, pair<int,int> goal) {
    int ROWS = grid.size(), COLS = grid[0].size();
    auto [sr, sc] = start;
    auto [gr, gc] = goal;

    auto h = [&](int r, int c) { return abs(r - gr) + abs(c - gc); };

    // (f, g_cost, row, col)
    using T = tuple<int,int,int,int>;
    priority_queue<T, vector<T>, greater<T>> pq;
    vector<vector<int>> g(ROWS, vector<int>(COLS, INT_MAX));

    g[sr][sc] = 0;
    pq.push({h(sr, sc), 0, sr, sc});

    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!pq.empty()) {
        auto [f, cost, r, c] = pq.top(); pq.pop();

        if (r == gr && c == gc) return cost;
        if (cost > g[r][c]) continue;   // stale

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS && grid[nr][nc] != 1) {
                int new_g = cost + 1;
                if (new_g < g[nr][nc]) {
                    g[nr][nc] = new_g;
                    pq.push({new_g + h(nr, nc), new_g, nr, nc});
                }
            }
        }
    }
    return -1;
}
```

### Complexity
- **Time:** O((V + E) log V) — same asymptotic as Dijkstra, but fewer nodes expanded in practice.
- **Space:** O(V) for the g-cost table and priority queue.

### Common traps
- Using inadmissible heuristic → finds *a* path, not the shortest one.
- Forgetting the stale-entry check → processes outdated entries with higher cost.
- Euclidean heuristic on a 4-directional grid → inadmissible (diagonal shortcuts don't exist). Use Manhattan.
- For unweighted interview grids, BFS is simpler and equally correct — A* shines when weights vary or the grid is huge.

### Robotics connection
A* is the default global planner in **Nav2** (`NavFn` plugin uses Dijkstra; `SmacPlanner` uses A*). Also used in SLAM occupancy grid planning. The heuristic choice matters in real maps: inflation layers make the cost non-uniform, so the planner is doing weighted A*.

---

## Sliding Window — longest/shortest subarray with a property

**Signal words:** "longest/shortest substring/subarray", "at most k distinct", "max sum subarray of size k", "minimum window containing...".

**Mental model:** Two pointers `left` and `right` defining a window. Expand `right` to include more. When the window violates the constraint, shrink from `left` until it's valid again. The window slides forward — each element enters once and leaves once → O(n).

### Step-by-step spoken approach

> "Sliding window — I have a subarray constraint. I'll expand right one step at a time, update my window state, then shrink left until the window is valid again. I track the best answer while the window is valid. O(n) time."

### Template — variable-size window (find longest valid)

```python
def longest_valid_window(s, k):
    left = 0
    window_state = {}   # track whatever constraint needs tracking
    best = 0
    
    for right in range(len(s)):
        # 1. Add s[right] to window
        window_state[s[right]] = window_state.get(s[right], 0) + 1
        
        # 2. Shrink from left while window is INVALID
        while len(window_state) > k:   # example: at most k distinct chars
            window_state[s[left]] -= 1
            if window_state[s[left]] == 0:
                del window_state[s[left]]
            left += 1
        
        # 3. Window is now valid — update answer
        best = max(best, right - left + 1)
    
    return best
```

```cpp
// C++ version
int longestValidWindow(string s, int k) {
    unordered_map<char,int> window;
    int left = 0, best = 0;
    
    for (int right = 0; right < s.size(); right++) {
        window[s[right]]++;
        
        while ((int)window.size() > k) {
            window[s[left]]--;
            if (window[s[left]] == 0) window.erase(s[left]);
            left++;
        }
        best = max(best, right - left + 1);
    }
    return best;
}
```

### Complexity
- **Time:** O(n) — right and left each move forward at most n times total.
- **Space:** O(k) for the window state map.

### Common traps
- Off-by-one: window size = `right - left + 1`, not `right - left`.
- Shrinking too aggressively — only shrink until the window is _just_ valid, not past that.
- Fixed-size window variant: instead of a `while` shrink, just always remove `s[right - k]` when `right >= k`.

---

## Heap / Priority Queue — top-k, k-th largest, streaming

**Signal words:** "k largest/smallest", "top-k frequent", "find the median from a data stream", "merge k sorted lists".

**Mental model:** A heap keeps the extreme element at the top in O(log n) per insert/remove. For "k largest", maintain a **min-heap of size k** — the top is the smallest of the k largest, so anything smaller gets kicked out.

### Step-by-step spoken approach

> "Top-k pattern — heap of size k. I'll push each element; when the heap exceeds size k, pop the min. What remains is the k largest. O(n log k) time."

### Template — k largest elements

```python
import heapq

def k_largest(nums, k):
    min_heap = []
    for num in nums:
        heapq.heappush(min_heap, num)
        if len(min_heap) > k:
            heapq.heappop(min_heap)   # remove the smallest
    return list(min_heap)   # these are the k largest
```

```cpp
#include <queue>
#include <vector>
using namespace std;

vector<int> kLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;  // min-heap
    for (int num : nums) {
        minHeap.push(num);
        if ((int)minHeap.size() > k) minHeap.pop();
    }
    vector<int> result;
    while (!minHeap.empty()) { result.push_back(minHeap.top()); minHeap.pop(); }
    return result;
}
```

### Heap cheatsheet

| Goal | Heap type | Size |
|------|-----------|------|
| k largest | min-heap | k |
| k smallest | max-heap | k |
| k-th largest | min-heap | k, top = answer |
| Median stream | min-heap + max-heap | balanced halves |

### Complexity
- **Time:** O(n log k) — n pushes, each O(log k).
- **Space:** O(k).

### Common traps
- Python's `heapq` is a **min-heap only**. For max-heap, push `-num` and negate when popping.
- C++ `priority_queue` defaults to **max-heap**. For min-heap: `priority_queue<int, vector<int>, greater<int>>`.
- Off-by-one: "k-th largest" means keep exactly k elements — the top of the min-heap IS the k-th largest.

---

## Dijkstra — weighted shortest path

**Signal words:** "shortest path", "minimum cost", **with edge weights** (if unweighted → plain BFS). "Cheapest flight", "minimum effort path", "network delay time".

**Mental model:** BFS with a priority queue instead of a plain queue. Always expand the node with the smallest known distance first. Once a node is popped from the heap, its distance is finalized (never update again).

### Step-by-step spoken approach

> "Dijkstra — weighted shortest path. Min-heap seeded with (0, start). Pop the cheapest node, relax its neighbors — if new distance is better, push (new_dist, neighbor). Skip a node if we've already finalized it. O((V+E) log V)."

### Template — Python

```python
import heapq

def dijkstra(graph, start, end):
    # graph: {node: [(neighbor, weight), ...]}
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    min_heap = [(0, start)]   # (distance, node)
    
    while min_heap:
        d, node = heapq.heappop(min_heap)
        
        if d > dist[node]:    # already found a better path — skip
            continue
        if node == end:
            return d
        
        for neighbor, weight in graph[node]:
            new_dist = d + weight
            if new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                heapq.heappush(min_heap, (new_dist, neighbor))
    
    return float('inf')   # unreachable
```

### Template — C++

```cpp
#include <queue>
#include <vector>
#include <climits>
using namespace std;

int dijkstra(vector<vector<pair<int,int>>>& graph, int start, int end, int n) {
    // graph[u] = {(v, weight), ...}
    vector<int> dist(n, INT_MAX);
    dist[start] = 0;
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, start});
    
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        
        if (d > dist[u]) continue;    // stale entry
        if (u == end) return d;
        
        for (auto [v, w] : graph[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return -1;
}
```

### Complexity
- **Time:** O((V + E) log V) — each edge relaxed once, heap operations O(log V).
- **Space:** O(V + E).

### BFS vs Dijkstra decision

| Condition | Use |
|-----------|-----|
| Unweighted graph (all edges cost 1) | BFS — simpler, O(V+E) |
| Weighted graph, non-negative weights | Dijkstra |
| Negative weights | Bellman-Ford (not asked at RSE screens) |

### Common traps
- Using BFS on a weighted graph → wrong answer (BFS doesn't account for edge cost).
- Not skipping stale heap entries → processes old (higher) distances again.
- Negative weights break Dijkstra — the greedy assumption (popped node is final) fails.

---

## Binary Search — sorted or "search the answer space"

**Signal words:** sorted array + "find target / position", "minimum/maximum value that satisfies a condition", "minimize the maximum", "koko eating bananas", "capacity to ship packages".

**Two flavors:**
1. **Search in array** — classic, find a target value.
2. **Search on answer** — the answer itself is a number in a range; binary search on that range with a `feasible(mid)` check.

### Step-by-step spoken approach

> "Binary search — sorted data / answer-space search. Lo and hi bracket the range. Mid = lo + (hi-lo)/2 to avoid overflow. If mid satisfies the condition, record it and move the boundary to find better; if not, move the other way. O(log n)."

### Template — search in sorted array

```python
def binary_search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

### Template — search on answer space (leftmost valid)

```python
def search_answer_space(lo, hi):
    # find the smallest value in [lo, hi] where feasible(mid) is True
    result = -1
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if feasible(mid):
            result = mid    # record — but keep searching left for smaller valid
            hi = mid - 1
        else:
            lo = mid + 1
    return result

def feasible(mid):
    # return True if mid satisfies the constraint
    pass
```

```cpp
// C++ — search on answer space
int searchAnswerSpace(int lo, int hi) {
    int result = -1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (feasible(mid)) {
            result = mid;
            hi = mid - 1;   // search left for smaller valid answer
        } else {
            lo = mid + 1;
        }
    }
    return result;
}
```

### Recognizing answer-space problems

The tell: the problem asks "what is the minimum X such that Y is possible?" — X is your search space, Y is your `feasible` check.

Examples:
- "Minimum speed for Koko to eat all bananas in h hours" → binary search on speed (1 to max_pile).
- "Minimum ship capacity to ship packages in d days" → binary search on capacity.
- "Minimum days to make m bouquets" → binary search on days.

### Complexity
- **Time:** O(log n) for array search; O(log(range) × cost_of_feasible) for answer-space.
- **Space:** O(1).

### Common traps
- `mid = (lo + hi) / 2` overflows for large integers in C++ — use `lo + (hi - lo) / 2`.
- Off-by-one on bounds: loop condition is `lo <= hi` for exact search; `lo < hi` for some left/right boundary variants.
- Not knowing when to move `lo` vs `hi` — write it out: "if mid works, can I do better (smaller/larger)? Then move boundary toward better."

---

## Arrays/Hashing — solved problems

### Valid Anagram (#242)

**Pattern:** Arrays/Hashing — character frequency count.
**Signal:** "are these two strings anagrams?" → count characters, compare counts.

**Approach 1 — Sort (O(n log n))**
Sort both strings, compare. Simple but slower.

```cpp
bool isAnagram(string s, string t) {
    if (s.length() != t.length()) return false;
    sort(s.begin(), s.end());
    sort(t.begin(), t.end());
    return s == t;
}
```

**Approach 2 — Hashmap (O(n))**
Count characters in `s` (increment), decrement for `t`. If any count != 0, not an anagram.

```cpp
bool isAnagram(string s, string t) {
    if (s.length() != t.length()) return false;
    unordered_map<char, int> count;
    for (char c : s) count[c]++;
    for (char c : t) count[c]--;
    for (auto& p : count) if (p.second != 0) return false;
    return true;
}
```

| Approach | Time | Space |
|----------|------|-------|
| Sort | O(n log n) | O(1) |
| Hashmap | O(n) | O(n) |

---

### Valid Sudoku (#36)

**Pattern:** Arrays/Hashing — one set per constraint dimension.
**Signal:** "no duplicates across rows, columns, and boxes" → one set per row, col, and box.

**Key insight:** Three independent constraint dimensions → declare three arrays of 9 sets. Single pass, check-then-insert for all three at each cell.

**Box index formula:** `box_id = (i/3)*3 + (j/3)` — maps any cell (i,j) to 0–8.

```cpp
bool isValidSudoku(vector<vector<char>>& board) {
    unordered_set<char> rows[9], cols[9], boxes[9];
    for (int i = 0; i < 9; i++) {
        for (int j = 0; j < 9; j++) {
            if (board[i][j] != '.') {
                char c = board[i][j];
                int box_id = (i/3)*3 + (j/3);
                if (rows[i].count(c) || cols[j].count(c) || boxes[box_id].count(c))
                    return false;
                rows[i].insert(c); cols[j].insert(c); boxes[box_id].insert(c);
            }
        }
    }
    return true;
}
```

**Complexity:** O(1) time and space — fixed 9×9 grid, 27 sets of max 9 chars.

---

### Group Anagrams (#49)

**Pattern:** Arrays/Hashing — sorted string as hashmap key.
**Signal:** "group strings that are anagrams of each other" → need a canonical key that all anagrams share.

**Key insight:** All anagrams of a word sort to the same string. Use that sorted string as the hashmap key.

```cpp
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> res;
    for (const auto& s : strs) {
        string sortedS = s;
        sort(sortedS.begin(), sortedS.end());
        res[sortedS].push_back(s);
    }
    vector<vector<string>> result;
    for (auto& pair : res) result.push_back(pair.second);
    return result;
}
```

**Complexity:** O(n · k log k) — n strings, each sorted in O(k log k) where k = string length.

---

## Two Pointers — solved problems

### Two Sum II (#167)

**Pattern:** Two Pointers — sorted array, pair sum.
**Signal:** sorted input + find pair summing to target → two pointers, O(1) space.

**Key insight:** Sorted array means moving left up increases sum, moving right down decreases it. Converge until you hit the target.

```cpp
vector<int> twoSum(vector<int>& numbers, int target) {
    int left = 0, right = numbers.size() - 1;
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target) return {left+1, right+1};  // 1-indexed
        else if (sum < target) left++;
        else right--;
    }
    return {};
}
```

**Complexity:** O(n) time, O(1) space.

---

### Three Sum (#15)

**Pattern:** Two Pointers — fix one element, two-pointer scan for the pair.
**Signal:** find all unique triplets summing to zero → sort + outer loop + two pointers.

**Key insight:** Sort first. Fix `nums[i]`, then two-pointer on the rest. Skip duplicates at both the outer loop and after finding a valid triplet.

```cpp
vector<vector<int>> threeSum(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> result;
    for (int i = 0; i < nums.size(); i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;  // skip outer duplicates
        int left = i+1, right = nums.size()-1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.push_back({nums[i], nums[left], nums[right]});
                left++; right--;
                while (left < right && nums[left] == nums[left-1]) left++;
                while (left < right && nums[right] == nums[right+1]) right--;
            } else if (sum < 0) left++;
            else right--;
        }
    }
    return result;
}
```

**Complexity:** O(n²) time, O(1) space excluding output.

**Gotcha:** Duplicate skips needed in two places — outer loop (`i > 0 && nums[i] == nums[i-1]`) AND after each valid triplet for both left and right pointers.

---


## Interview follow-ups (meta)

- **Q:** How do you approach a problem you haven't seen?
    - **A:** Identify the pattern from the signal (sorted? substring property? top-k? ways to count?), state the approach and complexity out loud, confirm with the interviewer, then code. Pattern recognition first, code second.
- **Q:** Two Sum variants?
    - **A:** Unsorted → hash map (O(n)). Sorted → two pointers (O(n), O(1) space). The data's structure picks the pattern.


## Links

- Related: [[Modern C++ for Robotics]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Valid Anagram — what are the two approaches and their complexities? ? (1) Sort both strings, compare — O(n log n) time, O(1) space. (2) Hashmap frequency count — O(n) time, O(n) space.

Valid Anagram hashmap approach — what are the three steps? ? (1) Check lengths equal. (2) Increment count for each char in s, decrement for each char in t. (3) Loop through map — if any value != 0, return false. Return true.

Group Anagrams — what is the key insight? ? All anagrams sort to the same string. Use the sorted string as the hashmap key → group all originals under it.

Group Anagrams — what is the time complexity and why? ? O(n · k log k) — n strings, each sorted in O(k log k) where k is string length.

What's the signal → pattern for "longest substring with some property"? ? Sliding window — expand to include, contract to maintain the property. O(n).

What's the signal → pattern for "top-k" or "k largest/smallest"? ? Heap / priority queue of size k. O(n log k).

Unsorted Two Sum vs sorted Two Sum — which pattern each? ? Unsorted → hash map (O(n) time/space). Sorted → two pointers (O(n) time, O(1) space). The data structure picks the pattern.

What's the first thing you do on a new problem (the spoken-aloud habit)? ? Identify the pattern from the signal, then state the approach and time/space complexity out loud before coding — pattern recognition first, code second.

Grid BFS — what are the 4 steps before writing any code? ? (1) Model: each cell is a node, edges = valid neighbors. (2) BFS from start with a queue. (3) Mark visited on enqueue, not dequeue. (4) Return distance when target is dequeued; -1 if queue empties.

Why BFS and not DFS for shortest path on a grid? ? BFS explores level-by-level — the first time you reach the target is guaranteed shortest. DFS finds a path but not the shortest one.

Grid BFS time and space complexity? ? O(R×C) time and space — each cell visited at most once.

What is the visited-on-enqueue rule and why does it matter? ? Mark a cell visited when you add it to the queue, not when you pop it. If you mark on dequeue, the same cell gets enqueued multiple times → exponential blowup.

Sliding window — what are the two pointer moves? ? Expand right by one each iteration (add element to window). Shrink left while the window violates the constraint. Window size = right - left + 1.

Sliding window time complexity and why? ? O(n) — right advances n times total, left advances at most n times total. Each element enters and leaves the window at most once.

Heap for k-largest: which heap type and why? ? Min-heap of size k. The top is the smallest of the k largest — anything smaller gets evicted. What remains after processing all elements is the k largest.

Python heapq default, and how to get a max-heap? ? heapq is min-heap only. For max-heap: push -num and negate when popping.

C++ priority_queue default, and how to get a min-heap? ? Defaults to max-heap. For min-heap: priority_queue<int, vector<int>, greater<int>>.

Dijkstra vs BFS — when to use each? ? BFS for unweighted graphs (all edges cost 1), O(V+E). Dijkstra for weighted non-negative graphs, O((V+E) log V). Negative weights → Bellman-Ford.

Dijkstra stale-entry rule? ? When you pop (d, node) from the heap, check if d > dist[node]. If so, skip — a shorter path was already found and processed.

Binary search overflow-safe mid formula? ? mid = lo + (hi - lo) / 2. Never (lo + hi) / 2, which overflows for large integers in C++.

How do you recognize an answer-space binary search problem? ? "Minimum X such that Y is possible" or "maximum X such that Y holds." Binary search on X; write a feasible(mid) function that checks if mid satisfies Y.

What is the A* priority queue key and what are its two components? ? f(n) = g(n) + h(n). g(n) = actual cost from start to n; h(n) = heuristic estimate of remaining cost to goal.

What makes a heuristic admissible and why does it matter for A*? ? Admissible means h(n) never overestimates the true remaining cost. Guarantees A* finds the optimal path. If h overestimates, A* may skip the optimal path to expand "promising-looking" but wrong nodes.

What heuristic for A* on a 4-directional grid? ? Manhattan distance: |r - goal_r| + |c - goal_c|. Never overestimates because you can't move diagonally.

BFS vs Dijkstra vs A* — when to use each? ? BFS: unweighted (all edges cost 1), O(V+E). Dijkstra: weighted non-negative, goal unknown or many goals, O((V+E) log V). A*: weighted, single known goal, can estimate distance, O((V+E) log V) but explores fewer nodes in practice.

What does A* degenerate to when h=0? ? Dijkstra — no heuristic guidance, expands by actual cost only.

A* stale-entry rule (same as Dijkstra)? ? When you pop (f, cost, r, c), check if cost > g[r][c]. If so, skip — a cheaper path to this cell was already found and processed.

Where does A* appear in robotics? ? Nav2 global planner (SmacPlanner uses A*). SLAM occupancy grid planning. Any grid-based motion planning where start and goal are known.

Valid Sudoku — what data structure and how many instances? ? Three arrays of 9 unordered_sets: rows[9], cols[9], boxes[9]. One set per row, column, and 3×3 box.

Valid Sudoku — what is the box index formula? ? box_id = (i/3)*3 + (j/3). Maps any cell (i,j) to 0–8.

Valid Sudoku — what is the time and space complexity? ? O(1) time and space — fixed 9×9 grid, 27 sets of max 9 chars each.

Two Sum II — what pattern and why? ? Two pointers on sorted input. Sorted means left++ increases sum, right-- decreases it. O(n) time, O(1) space vs hashmap's O(n) space.

Two Sum II — what do you return and why the +1? ? {left+1, right+1} — the problem is 1-indexed.

Three Sum — what are the two places you skip duplicates? ? (1) Outer loop: if (i > 0 && nums[i] == nums[i-1]) continue. (2) After pushing a valid triplet: skip duplicate left and right values with while loops.

Three Sum — why sort first? ? Enables two pointers (sorted order lets you steer sum by moving pointers) and makes duplicate skipping trivial (equals check on adjacent elements).

Three Sum time complexity? ? O(n²) — outer loop O(n) × two-pointer scan O(n). Sorting is O(n log n) but dominated.

---

### Container With Most Water (#11)

**Pattern:** Two Pointers — converging from both ends.
**Signal:** array of heights, find two lines that hold the most water.

**Key insight:** Area = `min(height[left], height[right]) * (right - left)`. Start with the widest window (both ends). The bottleneck is always the shorter wall — moving the taller pointer inward can only make things worse, so always move the shorter one.

```cpp
int maxArea(vector<int>& height) {
    int left = 0, right = height.size() - 1;
    int max_area = 0;
    while (left < right) {
        int area = min(height[left], height[right]) * (right - left);
        max_area = max(max_area, area);
        if (height[left] < height[right]) left++;
        else right--;
    }
    return max_area;
}
```

**Complexity:** O(n) time, O(1) space.

**Gotcha:** Don't try all pairs — that's O(n²). The key is that moving the shorter pointer is the only move that could find a larger area; moving the taller one never helps.

---

### Trapping Rain Water (#42)

**Pattern:** Two Pointers — running left/right maximums.
**Signal:** array of heights, how much water is trapped between bars.

**Key insight:** Water at position `i` = `min(max_left, max_right) - height[i]`. Brute force computes this in O(n²) by scanning both sides. Two-pointer does it in O(1) space: if `height[left] < height[right]`, `left_max` is the bottleneck — process the left side and move inward. You don't need to know `right_max` exactly because any bar to the right is taller than `height[left]`.

```cpp
int trap(vector<int>& height) {
    int left = 0, right = height.size() - 1;
    int left_max = 0, right_max = 0;
    int water = 0;

    while (left < right) {
        left_max  = max(left_max,  height[left]);
        right_max = max(right_max, height[right]);

        if (height[left] < height[right]) {
            water += left_max - height[left];
            left++;
        } else {
            water += right_max - height[right];
            right--;
        }
    }
    return water;
}
```

**Complexity:** O(n) time, O(1) space.

**Common traps:**
- Using `left_max - height[right]` on the else branch (wrong max applied to wrong side).
- Confusing with Container With Most Water — that problem picks the best pair; this one sums water at every position.
- Brute force `min(max_left, max_right) - height[i]` is correct logic but O(n²) — the two-pointer just avoids recomputing the maxes.

---

## Graphs — solved problems

### Clone Graph (#133)

**Pattern:** DFS + hashmap as clone registry.
**Signal:** "deep copy a graph" → traverse every node, track visited to avoid infinite loops and duplicate clones.

**Key insight:** The hashmap does two jobs: (1) visited check — don't recurse into a node twice, (2) clone registry — `original→clone`, so when multiple nodes point to the same neighbor they all get the same clone back. Register the clone **before** recursing into neighbors, so back-edges find it in the map instead of looping forever.

```cpp
Node* dfs(Node* node, unordered_map<Node*, Node*>& cloned) {
    if(cloned.count(node)) return cloned[node];

    Node* copy = new Node(node->val);
    cloned[node] = copy;   // register BEFORE recursing

    for(Node* neighbor : node->neighbors)
        copy->neighbors.push_back(dfs(neighbor, cloned));

    return copy;
}

Node* clone_graph(Node* node) {
    if(!node) return nullptr;
    unordered_map<Node*, Node*> cloned;
    return dfs(node, cloned);
}
```

**Complexity:** O(V + E) time and space — every node and edge visited once.

**Common traps:**
- `return copy` inside the for loop — exits after cloning only the first neighbor.
- Global hashmap — persists between calls, causes wrong results. Keep it local in the wrapper.
- Forgetting the `nullptr` check — crashes on empty graph input.

---

### Number of Islands (#200)

**Pattern:** DFS/BFS on a grid — flood fill.
**Signal:** count connected components of '1's in a binary grid.

**Key insight:** For each unvisited '1', increment the island count and DFS to mark the entire connected island as '0' (visited). When the DFS returns, you've consumed the whole island.

```cpp
class Solution {
    void dfs(vector<vector<char>>& grid, int r, int c) {
        int rows = grid.size(), cols = grid[0].size();
        if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '0')
            return;
        grid[r][c] = '0';   // mark visited by sinking the island
        dfs(grid, r+1, c);
        dfs(grid, r-1, c);
        dfs(grid, r, c+1);
        dfs(grid, r, c-1);
    }
public:
    int numIslands(vector<vector<char>>& grid) {
        int count = 0;
        for (int r = 0; r < grid.size(); r++)
            for (int c = 0; c < grid[0].size(); c++)
                if (grid[r][c] == '1') { count++; dfs(grid, r, c); }
        return count;
    }
};
```

**Complexity:** O(R×C) time and space — each cell visited at most once.

**Common traps:**
- Missing the column bounds check `c >= cols` in the base case.
- Placing `return` before the recursive calls — the flood fill never runs.
- Modifying the grid in-place is fine here; if you can't, use a `visited` bool grid instead.

---

Trapping Rain Water — what is the key insight for the O(n) solution? ? Track running left_max and right_max. If height[left] < height[right], left_max is the bottleneck — water at left = left_max - height[left], then left++. The right side mirrors this. O(n) time, O(1) space.

Trapping Rain Water vs Container With Most Water — what's the difference? ? Container picks the best single pair of bars — answer is one area. Trapping Rain Water sums water trapped at every position across the whole array. Same two-pointer structure but different accumulation logic.

Container With Most Water — what pattern and key insight? ? Two pointers. Start at both ends — the bottleneck is the shorter wall. Always move the shorter pointer inward; moving the taller one can only make things worse.

Container With Most Water — time and space complexity? ? O(n) time, O(1) space.

Clone Graph — why does the hashmap need to be registered before recursing? ? So back-edges (neighbor pointing back to a node already being processed) find the existing clone in the map instead of recursing infinitely.

Clone Graph — what are the two jobs of the hashmap? ? (1) Visited check — don't recurse into a node twice. (2) Clone registry — original→clone, so multiple nodes pointing to the same neighbor all get the same clone back.

Number of Islands — pattern and approach? ? DFS/BFS on a grid. For each unvisited '1', increment count and flood-fill it to '0' so it's never counted again.

Number of Islands — time and space complexity? ? O(R×C) time and space — each cell visited at most once.