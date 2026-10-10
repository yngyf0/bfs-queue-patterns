# breadth first search: the queue template, level-order idioms, and how to spot it in coding interviews

Most people meet breadth first search twice. First in a data structures class, where it's a short paragraph about a queue and a visited set. Then, a year later, in a coding interview, where it shows up disguised as "minimum number of moves," "shortest path in a grid," or "the nearest gate," and suddenly the paragraph isn't enough.

This is written for the second meeting. The mechanics take ten minutes. What takes longer is recognizing when BFS is the right tool, which of the two common idioms to use, and why the version you write under pressure is often subtly worse than the version you know.

## What breadth first search actually does

BFS explores a graph in rings. Start at a source node, visit everything one edge away, then everything two edges away, and so on. The FIFO queue is what enforces that order: nodes leave in the order they arrived, so you never process a distance-2 node before a distance-1 node.

That single property is why BFS finds shortest paths on unweighted graphs. When you first reach a node, the distance you reached it at is the minimum, because anything shorter would have been in the queue ahead of you.

python
from collections import deque

def bfs(start):
    queue = deque([start])
    visited = {start}
    while queue:
        node = queue.popleft()
        if is_target(node):          # check the node you just dequeued
            return node
        for neighbor in neighbors(node):
            if neighbor not in visited:
                visited.add(neighbor)  # mark at enqueue time, not dequeue
                queue.append(neighbor)
    return None


Two details in those seven lines cause most of the bugs people hit:

**Mark visited when you enqueue, not when you dequeue.** If you wait until the node comes off the queue, the same node can be appended by several different parents before any of them processes it. The queue stops representing distinct work, and the runtime quietly degrades from O(V + E) to something much worse, because you keep re-examining neighbors you already scheduled.

**Check the node before expanding it.** The start node is a valid answer in a lot of problems ("the source is already the target"), and a template that only checks children silently misses it. This is a real point of confusion in AlgoMonster's own BFS intro discussion, where a reader points out that the root node never gets checked in the published template.

## The three shapes of BFS problems

Almost every BFS question is one of three shapes, and each has a slightly different muscle memory.

### Tree level order

Trees have no cycles, so you technically don't need a visited set. What you do need is a way to keep levels separate, and that's the `len(queue)` snapshot:

python
def level_order(root):
    if root is None:
        return []
    result, queue = [], deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):  # freeze the size of this level
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result


The snapshot is doing real work. Children get appended behind the current level, so without freezing the count, the boundary between depth 1 and depth 2 dissolves the moment you enqueue anything. Variants like right-side view, maximum value per row, and zigzag traversal are all the same outer loop with a different inner body.

### Grids and implicit graphs

Here the graph is implied by the input rather than handed to you. Cells are nodes, and moves are edges. A distance counter travels in the queue entry:

python
def shortest_path(grid):
    rows, cols = len(grid), len(grid[0])
    if grid[0][0] == 1 or grid[rows - 1][cols - 1] == 1:
        return -1
    directions = [(-1, 0), (1, 0), (0, -1), (0, 1)]
    queue = deque([(0, 0, 1)])
    visited = {(0, 0)}
    while queue:
        r, c, dist = queue.popleft()
        if (r, c) == (rows - 1, cols - 1):
            return dist
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if (0 <= nr < rows and 0 <= nc < cols
                    and grid[nr][nc] == 0
                    and (nr, nc) not in visited):
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))
    return -1


Word ladder, rotten oranges, knight moves, and open-the-lock all reduce to this, with a different `neighbors` function and a different goal test.

### Explicit graphs

Adjacency lists, course prerequisites, social networks, connected components. Same template, but now the visited set is mandatory and the complexity becomes O(V + E) rather than a function of the input grid alone.

| Input shape | Time | Space |
| --- | --- | --- |
| Graph, adjacency list, V nodes / E edges | O(V + E) | O(V) |
| Tree with N nodes | O(N) | O(N) in the worst case for the queue |
| R × C grid, 4-directional moves | O(R × C) | O(R × C) for visited |

## Multi-source BFS, and why it's free

Problems like "shortest distance from each cell to the nearest gate" or "rotting oranges" look like they need BFS from every source. They don't. Seed the queue with all starting points at distance 0 at once, then run the identical loop. Every node still gets dequeued exactly once, so the complexity is unchanged, and the distance you record is the distance to the closest source by construction.

This is one of the few pattern upgrades that costs nothing to learn and shows up often enough to be worth remembering.

## When breadth first search is the wrong answer

Knowing when to reject BFS matters as much as recognizing it.

- **Weighted edges.** BFS minimizes the number of edges, not total cost. A two-hop path with expensive edges can beat a three-hop path of cheap ones. Use Dijkstra for non-negative weights, or Bellman-Ford if negatives are possible.
- **Weights of only 0 and 1.** A plain queue breaks here. Use a deque and push zero-weight neighbors to the front and one-weight neighbors to the back.
- **Cycle detection and topological sort.** BFS can do them, but DFS is usually cleaner to write and easier to defend out loud.
- **"Return any path" instead of "the shortest path."** DFS does this with less bookkeeping.
- **Extra constraints that aren't edge weights.** "Shortest path with at most k turns" or "with fuel stops" isn't plain BFS. The state has to grow to include the constraint, so the nodes become `(position, remaining_fuel)` pairs. BFS still applies, just to a larger state space.
- **Exponential state spaces with no pruning.** Enumerating subsets or permutations with BFS will blow up. That's a backtracking problem.

The recognition version of this, which is how AlgoMonster's public flowchart frames it: if the problem says "shortest," "minimum moves," or "fewest steps," and the graph is unweighted, it's BFS. If it asks for level-order or per-depth output, it's BFS with the level-size snapshot. If it asks for any path, cycle detection, or topological order, it's DFS.

## The five bugs worth checking before you say you're done

1. Root is null, and you dereference it anyway.
2. `list.pop(0)` instead of `deque.popleft()`. The first is O(n) per removal and turns the whole algorithm quadratic.
3. Visited set updated at dequeue time rather than enqueue time.
4. Missing level boundary in level-order problems, producing a flat list instead of nested levels.
5. Not checking the starting node, so single-node and zero-move cases return the wrong value.

Running that list takes about twenty seconds and catches most of what an interviewer would otherwise catch for you.

## Practicing BFS so it actually sticks

Reading the template once doesn't produce recall under pressure. A workable sequence, roughly a week of an hour a day:

1. Level order traversal, then right-side view, then zigzag. Same skeleton, three variations.
2. Number of islands, then max area of island. Same traversal, different aggregation.
3. Rotting oranges, then walls and gates. Multi-source seeding.
4. Word ladder. Implicit graph, where the `neighbors` function is the actual difficulty.
5. Shortest path in a binary matrix, then open the lock. Distance tracking plus a non-obvious state definition.

Do them from the template rather than from scratch, and write the complexity defense in a sentence each time. The interview version of "I know BFS" is being able to say, out loud, why the queue guarantees shortest paths and when it stops guaranteeing anything.

## Where AlgoMonster fits into BFS prep

If you already know the algorithm and just need volume, none of this is worth paying for. The case for a structured course is narrower: your gap is identifying which pattern a problem wants, and you'd rather read diagrams than watch a 40-minute video.

AlgoMonster is the platform most often recommended for exactly that gap. It's built by engineers who interviewed at Google, and the approach is explicitly pattern-first: instead of grinding problems and hoping the underlying structure becomes obvious, you learn a named pattern with a template, then practice problems mapped to that pattern.

Its breadth first search material sits inside that structure. The chapter moves from an intro that compares BFS and DFS trade-offs, through the queue template, into BFS on trees (level order traversal, right-side view, and relatives) and BFS on graphs. The format is text plus interactive diagrams, not video, and the diagrams are the part people tend to mention: roughly 1,000 illustrations across the platform, with an algorithm visualizer that animates pointers and code side by side so you can watch the frontier move.

A few other pieces are relevant to the practice loop above:

- **An inline IDE with an AI assistant** on the same page as the lesson, so you can write the template and ask about a failing edge case without switching tabs.
- **Speedrun mode**, a compressed pass through the pattern set. It's the feature that makes sense in the last week before an onsite, when you want recall rather than new material.
- **The decision flowchart**, which is essentially a formalized version of the "which pattern does this problem want?" question. Its BFS branch is one line: level-order or shortest-level answer.
- **Content in six languages**, which matters if your interview is in Java or C++ but you learned the patterns in Python.

The honest limits: there's no video at all, the problem bank is deliberately small compared with a platform built around volume, and system design and behavioral coverage are introductory rather than thorough. If a senior-level loop is the target, those are separate purchases.

### Plans and current prices

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Sampled lessons and problems, enough to test whether the text-and-diagram format works for you | $0 | — | [ Start with the free content](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions refreshed monthly; 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials; bonus Foundation and Build-an-LLM courses; member discount on live bootcamps | $99/year, shown as $8.25/month | Billed annually | [ See the yearly plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, paid once | $189 | One-time | [ Check the lifetime option](https://algo.monster?fpr=li69) |

Two things worth knowing before you decide. There's no monthly option, so the cheapest way in is the annual plan, and there's no team or enterprise tier. Some specialized content developed with outside partners, such as quant interview prep, is sold separately from the Pro subscription. Prices on this page move with promotions, so confirm the number on the checkout screen rather than trusting a screenshot.

If your prep window is a single interview cycle of six to ten weeks, the annual plan is the straightforward choice. The lifetime plan only pays for itself if you expect to do this again, which is common for engineers who move every two or three years, but is a real assumption rather than a safe one.

## A short FAQ

**Does BFS need a visited set on a tree?**
No. Trees are acyclic, so no node is reachable twice. You still want a level-size snapshot if the output is grouped by depth.

**Why not just use DFS and take the shortest path at the end?**
Because DFS returns some path, and finding the shortest one among all paths means enumerating them. BFS's level order gives you the shortest distance for free.

**What's the space complexity of BFS on a wide tree?**
O(N) in the worst case. The queue can hold an entire level, and the widest level in a balanced binary tree holds roughly half the nodes.

**Can BFS handle weighted graphs if all weights are equal?**
Yes, and that's the standard case. The moment weights differ, you need a priority queue instead of a FIFO queue, which is Dijkstra.

**Is one week enough to get comfortable with BFS?**
Enough to write the template from memory, yes. Enough to adapt it to an unfamiliar implicit graph under time pressure, usually not. Budget a second pass with the harder problems: word ladder, open the lock, and shortest path with a state constraint.

The algorithm itself is small. What you're really practicing is the recognition step and the discipline to write the seven-line template without the three defects that make it slow or wrong. Get those two things automatic, and breadth first search stops being a topic you study and becomes a tool you reach for.
