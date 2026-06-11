---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [dsa, neetcode, interview-prep]

# DSA Patterns

> [!star] This is your active interview-prep front — NeetCode 150 in C++ Not robotics knowledge — the coding-screen layer. The named plan: NeetCode 150 in C++, in pattern order, timed, spoken aloud. This note is the pattern index to drill against, not a place to expand into a textbook.


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


## Links

- Related: [[Modern C++ for Robotics]]
- Parent: [[00 Knowledge Map]]

---

