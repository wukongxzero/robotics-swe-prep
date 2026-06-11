---
type: concept
domain: SWE & math foundations
status: drafted
last-reviewed:
tags: [interview, dsa, meta-skill, deep]
---

# Interview Problem-Solving Framework

> [!star] This is the meta-skill — how you behave in the room
> Knowing the pattern isn't enough. Interviewers evaluate how you think out loud, how you handle being stuck, and whether you communicate before coding. This note is the script.


---

## The universal 5-step framework

Use this on every problem, every time. Do not skip steps, do not reorder them.

### Step 1 — Understand the problem out loud (1–2 min)

Before anything else, restate the problem in your own words and clarify constraints.

**Say:**
> "So I'm given [X], and I need to return [Y]. Let me make sure I understand — [restate the example]. A few quick questions: can the input be empty? Can values be negative? What's the expected input size?"

**Why:** Interviewers hide edge cases in the problem description. Restating shows you read carefully. Asking about size tells you if O(n²) is acceptable or if you need O(n log n).

---

### Step 2 — Identify the pattern (30 sec)

Map the problem to one of the 15 patterns. Say the pattern name out loud.

**The signal → pattern map (quick reference):**

| Signal words | Pattern |
|---|---|
| Grid, shortest path, minimum steps | **BFS on grid** |
| Connectivity, islands, components | **BFS or DFS** |
| Ordering, prerequisites, dependencies | **Topological sort** |
| Shortest path with weights | **Dijkstra** |
| Longest/shortest subarray with property | **Sliding window** |
| Top-k, k-th largest/smallest | **Heap** |
| Sorted array + find target/position | **Binary search** |
| "Minimize the maximum" or "feasible?" | **Binary search on answer** |
| Counts, "seen before?", dedup | **Hash map/set** |
| All combinations/permutations/subsets | **Backtracking** |
| Overlapping subproblems, min/max cost | **DP** |

**Say:**
> "This looks like a [pattern] problem because [signal word or structure]. I'll use [data structure] to solve it."

---

### Step 3 — State the approach and complexity (1 min)

Describe your approach in plain English — no code yet. Then state time and space complexity.

**Say:**
> "My approach: I'll [do X], then [do Y], tracking [Z]. The time complexity is O([...]) because [reason]. Space is O([...]) because [reason]. Does that sound right before I start coding?"

**Why:** This is where you catch a wrong approach before spending 10 minutes coding it. The interviewer will redirect you here if you're off track.

---

### Step 4 — Code it (10–15 min)

Now write the code. Talk as you code — narrate what each piece does.

**Say while coding:**
> "I'm initializing the queue with the start node..."
> "Here I'm checking bounds before accessing the grid..."
> "This visited check prevents infinite loops..."

**Rules:**
- Write a helper if it cleans up the main function.
- Use meaningful variable names — `row, col` not `i, j` for grids.
- Don't silence yourself. Dead air is bad. If you're thinking, say "let me think about the edge case here..."

---

### Step 5 — Test and walk through (2–3 min)

After coding, trace through the example by hand. Then check edge cases.

**Say:**
> "Let me trace through the example... [walk through it]. Now edge cases: empty input returns [X], single element returns [Y], no path returns [Z]."

**Why:** Bugs caught here don't count against you. Bugs found by the interviewer after you claim you're done do.

---

## When you're stuck — the exact script

Being stuck is fine. Staying silent is not. Use this exact progression:

**Level 1 — think out loud:**
> "I know I need to get from X to Y. Let me think about what changes between states..."

**Level 2 — try a simpler version:**
> "Let me think about a smaller example first — what if the grid was 2×2?"

**Level 3 — ask for a hint without sounding lost:**
> "I'm torn between [approach A] and [approach B]. Is there a constraint I should be using to choose?"

**Level 4 — state what you do know:**
> "I know the brute force is O(n²) — do a nested loop over all pairs. I'm trying to figure out how to bring that down to O(n). Can I use some structure in the data?"

**Never say:** "I don't know." Say what you *do* know and where you're stuck. That's what interviewers want to hear.

---

## Grid BFS — the full spoken script

This is the script for the most common RSE interview problem type. Memorize the structure.

**Problem:** "Given a grid with walls, find the shortest path from start to end."

**What you say, step by step:**

> **Step 1 — Understand:**
> "So I have a 2D grid where 1 is a wall and 0 is open. I need to find the minimum number of moves from the start cell to the end cell, moving up/down/left/right. If there's no path I return -1. Is that right?"

> **Step 2 — Pattern:**
> "This is a shortest-path-on-a-grid problem. The signal is 'minimum moves' on an unweighted grid — that's BFS. BFS explores level by level, so the first time I reach the end cell is guaranteed to be the shortest path."

> **Step 3 — Approach + complexity:**
> "My approach: seed a queue with the start cell at distance 0. Pop a cell, check its 4 neighbors — if in bounds, not a wall, not visited — enqueue with distance+1. Mark visited on enqueue. The moment I pop the end cell, return its distance. Time: O(R×C) since each cell is visited at most once. Space: O(R×C) for the visited set and queue."

> **Step 4 — Code it.** *(write the BFS template, narrating as you go)*

> **Step 5 — Trace:**
> "Let me trace through... start at (0,0), enqueue (0,1) and (1,0) at distance 1..." *(walk through a few steps)*. "Edge case: if start equals end, return 0. If start is a wall, return -1."

---

## Dijkstra spoken script (weighted grid / graph)

**Problem:** "Find the minimum cost path from start to end, where each cell or edge has a different cost."

> **Step 2 — Pattern:**
> "The graph has weights, so BFS doesn't work — BFS treats all edges as cost 1. I need Dijkstra. It's like BFS but with a min-heap instead of a plain queue, so I always expand the cheapest known node first."

> **Step 3 — Approach:**
> "Seed a min-heap with (cost=0, start). Pop the cheapest node. For each neighbor, if current cost + edge cost is better than known distance, update and push. Skip stale heap entries where the popped distance is worse than the recorded best. Time: O((V+E) log V)."

---

## Sliding window spoken script

**Problem:** "Find the longest substring/subarray with at most k distinct characters (or some similar constraint)."

> **Step 2 — Pattern:**
> "Signal is 'longest subarray with property' — that's sliding window. Two pointers, left and right."

> **Step 3 — Approach:**
> "Expand right by one each iteration, adding the element to my window state. While the window violates the constraint, shrink from left. After each shrink, the window is valid — update the best answer. Each element enters and leaves at most once, so O(n) time."

---

## Binary search spoken script

**Problem:** "What's the minimum speed such that Koko can eat all bananas in h hours?"

> **Step 2 — Pattern:**
> "This is a 'minimize X such that something is feasible' problem — that's binary search on the answer space. The answer is some speed between 1 and max_pile. I'll binary search on speed and write a feasible(speed) function."

> **Step 3 — Approach:**
> "feasible(speed) checks: given this speed, can Koko eat all bananas in h hours? That's O(n). Binary search runs O(log(max_pile)) times. Total: O(n log max_pile)."

---

## Common interview mistakes to avoid

| Mistake | Fix |
|---|---|
| Silent thinking | Narrate every thought, even "I'm not sure yet" |
| Coding before stating approach | Always say the approach out loud first — get interviewer buy-in |
| Saying "I don't know" | Say what you know and where you're stuck |
| Forgetting edge cases | Always check: empty input, single element, no valid answer |
| Wrong complexity claim | State it and justify it — "each cell visited once → O(R×C)" |
| Not testing your code | Always trace through the example after writing |

---

## Links

- Related: [[DSA Patterns]], [[Modern C++ for Robotics]]
- Parent: [[00 Knowledge Map]]

---

