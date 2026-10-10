# Course Schedule II: Solve LeetCode 210 Without Memorizing It, From In-Degrees to Cycle Detection

You already know this one is topological sort. That's not the hard part. The hard part is that Course Schedule II has two failure modes that don't show up in the examples: you return a partial ordering when the graph contains a cycle, or you build the edges backwards and pass the sample case anyway.

This is Course Schedule (#207) with one sentence changed. Instead of "is it possible," you return the order. That one sentence is where most wrong submissions come from, so let's walk the whole thing properly.

## What Course Schedule II is actually asking

You get `numCourses` courses numbered `0` to `numCourses - 1`, plus a list of prerequisite pairs. Each pair is `[a, b]`, meaning you must finish `b` before you can take `a`.

Return an ordering that lets you finish every course. If several orderings work, any of them is accepted. If no ordering exists — meaning the prerequisites form a cycle — return an empty array.


Input:  numCourses = 4, prerequisites = [[1,0],[2,0],[3,1],[3,2]]
Output: [0,1,2,3]  or  [0,2,1,3]


Constraints are small: up to 2000 courses, and prerequisites can be dense since each pair is distinct. A brute-force search over permutations is out of the question, but anything linear in nodes plus edges is comfortable.

The phrase "any of them is accepted" is doing more work than it looks like. It means you never need to match a specific expected array, and it also means you shouldn't waste interview minutes trying to produce a canonical answer unless you're explicitly asked for one.

## Where the direction of the edge gets flipped

The pair `[1, 0]` means course 1 depends on course 0. So the dependency points from `0` to `1`: finish 0, then 1 is available. In graph terms you add a directed edge `0 → 1`, and course 1's in-degree goes up by one.

In-degree is just "how many prerequisites does this course still have." A course with in-degree 0 has no unmet prerequisites and can be taken right now.

Get this backwards — treating `[1,0]` as an edge `1 → 0` — and you'll often still pass the first sample, because `[[1,0]]` with two courses produces a valid order in the flipped graph too. The bug shows up later, on a bigger test, and costs you twenty minutes of confused staring.

## Course Schedule vs Course Schedule II

|  | Course Schedule (#207) | Course Schedule II (#210) |
| --- | --- | --- |
| Return value | `True` / `False` | A full ordering, or `[]` |
| Cycle handling | Return `False` | Return `[]` |
| Kahn's algorithm output | Just count processed nodes | Count **and** record them |
| Extra work required | None | Store the dequeue order |

If you can write 207, you're roughly four lines away from 210. The trap is that those four lines are easy to attach to the wrong place.

## Approach 1: Kahn's algorithm (BFS)

Kahn's algorithm peels off courses with no remaining prerequisites, one layer at a time. The order in which they leave the queue *is* the topological order.

1. Build an adjacency list: for each pair `[a, b]`, add `a` to the list of courses unlocked by `b`.
2. Track each course's in-degree.
3. Push every course with in-degree 0 into a queue.
4. Pop a course, append it to the result, then decrement the in-degree of everything it unlocks. Anything that drops to 0 goes into the queue.
5. At the end, if the result contains all `numCourses` courses, return it. Otherwise a cycle exists — return `[]`.

python
from collections import defaultdict, deque

def findOrder(numCourses, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * numCourses

    for course, prereq in prerequisites:
        graph[prereq].append(course)   # prereq -> course
        indegree[course] += 1

    queue = deque(i for i in range(numCourses) if indegree[i] == 0)
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for nxt in graph[node]:
            indegree[nxt] -= 1
            if indegree[nxt] == 0:
                queue.append(nxt)

    return order if len(order) == numCourses else []


Running it on `numCourses = 4, prerequisites = [[1,0],[2,0],[3,1],[3,2]]`:


in-degrees:  0 -> 0,  1 -> 1,  2 -> 1,  3 -> 2
queue: [0]

pop 0  -> order [0],  1 drops to 0, 2 drops to 0   queue: [1, 2]
pop 1  -> order [0,1], 3 drops to 1                queue: [2]
pop 2  -> order [0,1,2], 3 drops to 0              queue: [3]
pop 3  -> order [0,1,2,3]                          queue: []


Four courses processed, four in the order, so `[0,1,2,3]` is returned. If you had popped 2 before 1, you'd get `[0,2,1,3]` — equally correct, and that's exactly why you can't compare your output to a hardcoded array.

## Approach 2: DFS with three-state coloring

The DFS version is shorter to write once you trust it, and it detects cycles as a side effect. Each node carries one of three states:

- `0` — not visited yet
- `1` — currently on the recursion stack (we're still inside this node's call)
- `2` — fully processed

If you ever reach a node marked `1`, you've walked back into something you're already inside. That's a cycle.

The ordering falls out of the reverse post-order: a course gets appended to the list only after everything it unlocks has been appended, so reversing the list gives a valid order.

python
def findOrder(numCourses, prerequisites):
    graph = [[] for _ in range(numCourses)]
    for course, prereq in prerequisites:
        graph[prereq].append(course)

    state = [0] * numCourses
    post_order = []
    has_cycle = False

    def dfs(node):
        nonlocal has_cycle
        if has_cycle or state[node] == 2:
            return
        if state[node] == 1:
            has_cycle = True
            return
        state[node] = 1
        for nxt in graph[node]:
            dfs(nxt)
        state[node] = 2
        post_order.append(node)

    for i in range(numCourses):
        dfs(i)
        if has_cycle:
            return []

    return post_order[::-1]


One detail worth noticing: `state[node] = 2` and the append both happen *after* the loop over neighbors. Move either of them earlier and the ordering breaks in a way that's invisible on small inputs.

## The two bugs that actually cost you the submission

**Returning a partial order instead of an empty array.** When a cycle exists, Kahn's algorithm still processes every course that isn't trapped in the cycle. Your `order` list comes back with, say, 3 of 5 courses — and it looks like a plausible answer. The guard is one line:

python
return order if len(order) == numCourses else []


Getting this wrong is the single most common way this problem gets marked wrong.

**Leaving the recursion unguarded in DFS.** Without the `1` state, a cycle makes DFS spin forever or blow the stack on larger inputs. Two states ("visited / not visited") are not enough for cycle detection — you need to distinguish "done with this" from "currently inside this."

There's a third, softer trap: the answer isn't unique. Some people burn time trying to match an expected output they saw in a solution post. The judge accepts any valid topological ordering, and so does an interviewer, as long as you can defend the order.

> If your solution looks right on the provided examples but fails on hidden tests, check the cycle guard first, then check your edge direction. Those two account for nearly all of it.

## Complexity, and the follow-ups that usually come next

Both approaches are **O(V + E)** time — every course and every prerequisite edge is touched a constant number of times — and **O(V + E)** space for the adjacency list plus the queue or recursion stack. With `numCourses ≤ 2000`, recursion depth isn't a practical problem in Python or Java, though an iterative DFS or the BFS version sidesteps the question entirely if you'd rather not defend it.

Interviewers tend to extend the same graph rather than write a new problem. Expect the "safe states" variant, where the question becomes which courses can reach a cycle at all, or a scheduling variant where you're asked for the minimum number of rounds instead of any valid order. Same in-degrees, same peeling process — only the thing you track changes.

That's the argument for learning this as a pattern instead of a solution. Course Schedule II, Course Schedule, and the reachability variants share one skeleton: build the graph, track in-degrees, process what's free.

## Practicing topological sort so it holds up under pressure

If you're working through graph problems and keep landing on "it's topological sort, I think," that's a recognition gap, not a coding gap. It usually means you've solved the problem but haven't internalized the signal that tells you to reach for it: dependencies between items, a required ordering, or a question about whether an ordering exists at all.

[👉 AlgoMonster](https://algo.monster?fpr=li69) organizes its curriculum around that exact idea. It's built by former Google engineers and structured as 48 coding patterns rather than a problem list — each pattern gets a conceptual explanation, a reusable code template, and a handful of problems that apply it with variations. Topological sort sits inside the graph section alongside DFS and BFS traversal.

The parts that matter for this specific problem:

- **Decision flowcharts** that walk you from "the problem says 'prerequisites'" to "this is a topological sort" instead of leaving you to pattern-match on vibes
- **Roughly 670+ illustrations and diagrams** — graph traversal is one of the few topics where seeing in-degrees decrement is genuinely faster than reading about it
- **In-browser editor with template code** in Python, Java, JavaScript, C++, Go, and Haskell
- **Speedrun mode** for timed review, which is closer to interview conditions than untimed practice

A few things it doesn't do, and these are worth knowing before you pay:

- **No videos at all.** Everything is text plus interactive diagrams. If you learn by watching someone else code, this will feel dry.
- **231 lessons and problems** total, versus LeetCode's catalog of thousands. The patterns are the product; the problem count is deliberately small. Plan to supplement with LeetCode volume if you're targeting companies that pull from a huge question pool.
- **System design coverage is introductory.** Senior loops need a dedicated resource.
- **No monthly plan.** It's annual or lifetime only.

## AlgoMonster plans and current pricing

AlgoMonster runs three tiers. There is no month-to-month option, and the paid prices move around with sales, so treat the numbers below as what the subscribe page is showing now rather than a permanent rate.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [ Try AlgoMonster free](https://algo.monster/subscribe?fpr=li69) |
| Pro — Yearly | All 48 patterns, 231 lessons and problems, decision flowcharts, Speedrun mode, interactive editor, 12 months of access | $99/year (listed at $198) | Billed annually | [ Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in Yearly, plus lifetime updates, one-time payment | $189 (listed at $378) | Billed once | [ Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

The math on Yearly versus Lifetime depends entirely on whether you'll come back. The curriculum is designed to be finished in roughly 4–8 weeks at an hour or two a day. If this is one job search and you're done, the yearly plan is the smaller commitment. If you expect to interview again in two or three years — which is normal for most engineers — the lifetime plan costs about the same as two years of renewals and you stop thinking about it.

Worth flagging: prices on the subscribe page shift with promotions, and third-party coupon sites list numbers that don't always match what you'll see at checkout. The page is the source of truth.

## Common questions about Course Schedule II

**Does the order have to match the example?** No. Any valid topological ordering is accepted, including `[0,2,1,3]` for the standard four-course example.

**Why does my solution return an empty array when a cycle clearly isn't there?** Nine times out of ten you're comparing `len(order)` against the wrong value, or incrementing in-degrees for the prerequisite instead of the dependent course. Both produce a false cycle signal.

**Can I avoid recursion?** Yes. Kahn's algorithm is fully iterative, and DFS can be written with an explicit stack, though tracking the "currently on stack" state takes a little care.

**Is this a "medium" worth grinding?** It's a foundational one. The pattern shows up constantly in scheduling, dependency resolution, and build-order questions, and the cycle-detection guard is a detail that transfers directly to those.

The shortest path here is to write Kahn's algorithm from memory until the `len(order) == numCourses` check feels automatic, then do the DFS version once to understand why post-order reversal works. Two hours of that beats another dozen problems you'll forget by Friday.
