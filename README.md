# linked list cycle detection: Floyd's tortoise and hare explained, why the fast pointer always catches up, and how to return the cycle's start node

A singly linked list ends somewhere. Every node points to the next one, and the last node points to null. That's the version you learn first, and it's the version that never breaks your code.

The version that shows up in interviews is the other one: a node somewhere in the middle points back to an earlier node, and now your traversal never terminates. If you're here because you're staring at LeetCode 141 or 142 and the two-pointer code looks like magic, it isn't. It's arithmetic, and once you see it, you can reproduce it from scratch in about thirty seconds.

There are really two questions bundled inside "linked list cycle detection," and they have different answers:

1. **Does a cycle exist?** (LeetCode 141, `hasCycle`, returns true or false)
2. **Where does the cycle begin?** (LeetCode 142, `detectCycle`, returns the node)

Question one is the one everybody learns. Question two is the one interviewers actually care about, because it's where the reasoning shows.

## What actually counts as a cycle

Here's the detail that trips people up before they write a single line of code: a cycle is about **node identity**, not node values.

A list like `1 → 2 → 1 → null` contains the value `1` twice and has no cycle at all. Two separate nodes happen to hold the same number. If you detect cycles by storing visited *values* in a set, this input gives you a wrong answer.

A real cycle means the traversal returns to the same node reference, the same object in memory. In Python that's `is`, in Java that's `==` on references (not `.equals()`), in C++ it's the raw pointer comparison. Every correct solution depends on that distinction, whether it's a hash set of node objects or two pointers chasing each other.

Worth being precise about a second thing too: when slow and fast meet, that meeting point is *inside* the cycle, not at its entrance. Meeting is proof that a loop exists; it says nothing about where the loop starts. Those are two separate passes of the pointer game.

## Floyd's tortoise and hare, step by step

The algorithm is usually attributed to Robert W. Floyd, and it goes like this:

1. Put two pointers at the head, both starting at the first node.
2. Each iteration, move `slow` forward one node and `fast` forward two.
3. If `fast` (or `fast.next`) becomes null, the list ends normally. No cycle, return false.
4. If `slow` and `fast` land on the same node, there's a cycle.

That's the whole thing. The reason it feels like a trick is that step 4 arrives without warning the first time you trace it, so let's trace it.

Take `1 → 2 → 3 → 4 → 5 → 6` where node 6 points back to node 3. The cycle runs `3 → 4 → 5 → 6 → 3`, so the loop is four nodes long, and there are two nodes (1 and 2) sitting before the entrance.

| Step | slow | fast |
| --- | --- | --- |
| 0 | 1 | 1 |
| 1 | 2 | 3 |
| 2 | 3 | 5 |
| 3 | 4 | 3 |
| 4 | 5 | 5 |
| 5 | 6 | 3 |
| 6 | 3 | 5 |
| 7 | 4 | 3 |
| 8 | 5 | 5 ← meet |

The fast pointer laps the slow one and lands exactly on it at node 5. Note that node 5 is not where the cycle starts, which is the point I flagged above.

### The code

Python:

python
def has_cycle(head) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False


Java:

java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}


The loop condition is the part people get wrong under pressure. You need to check `fast` **and** `fast.next` before you dereference `fast.next.next`. Checking only `fast` gives you a null pointer exception on an even-length list; checking only `fast.next` when `fast` is already null gives you the same crash one line earlier.

## Why the pointers are guaranteed to meet

You don't need the proof to write the code, but knowing it stops the algorithm from feeling arbitrary — and interviewers ask.

Let `μ` be the number of nodes from the head to the cycle's entrance, and `λ` be the cycle length. Once both pointers are inside the loop, think of their positions modulo `λ`. Every iteration, `fast` gains exactly one node on `slow`, because it moves two while `slow` moves one. In a loop of finite length, a gap that shrinks by one each step has nowhere to hide: after at most `λ` iterations the gap is zero and they occupy the same node.

That's the guarantee. It holds no matter how long the tail is, how short the loop is, or how many times `fast` has already lapped around before `slow` even arrives.

The same arithmetic gives you the second half of the trick. Suppose they meet after `slow` has traveled `μ + k` steps, where `k` is somewhere in `[0, λ)`. Then `fast` has traveled `2(μ + k)` steps, and the extra distance `μ + k` must be a whole number of laps around the loop:


μ + k = n · λ     →     μ = n · λ − k


Read that last line as a statement about positions. Walking `μ` steps from the head and walking `μ` steps from the meeting point puts you on the same node — the entrance. Which means you can find where the cycle starts without counting anything or storing anything.

That's Floyd's cycle-*finding* algorithm, and it's why the second-pointer-reset never needs a second data structure.

## Finding where the cycle starts

Pick up right where the meeting happened. Leave `fast` where it is, reset `slow` to the head, then advance both one node at a time. The node they meet on is the cycle's entrance.

Continuing the trace above: they met at node 5, so `fast` stays at 5 and `slow` restarts at node 1.

- Step 1: slow = 2, fast = 6
- Step 2: slow = 3, fast = 3 ← both on node 3

Node 3 is exactly where the cycle begins. Two steps, no extra memory.

python
def detect_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            slow = head                 # reset one pointer
            while slow is not fast:     # walk both at the same speed
                slow = slow.next
                fast = fast.next
            return slow                 # cycle entrance
    return None


One thing that bites people here: don't reset both pointers. If you move both back to the head, they'll walk together forever and never separate. Exactly one pointer resets.

## Edge cases that break naive implementations

The two-pointer version handles the ugly inputs without special-casing, which is part of why it's the standard answer.

- **Empty list.** `head` is null, the while condition fails immediately, return false. Nothing to guard.
- **Single node pointing to itself.** After one iteration both pointers land on the same node. One step, cycle confirmed.
- **Single node pointing to null.** `fast.next` is null, loop exits, return false.
- **Repeated values, no cycle.** Handled correctly, as long as you compare references rather than values.
- **Odd vs. even length.** The `fast and fast.next` combination covers both; the fast pointer simply runs off the end of whichever parity the list happens to have.

The one genuinely dangerous mistake isn't an edge case at all — it's a relaxed loop condition. Most submitted failures on this problem are a null dereference, not a wrong idea.

## Other ways to detect a cycle, and what they cost

Floyd's algorithm isn't the only option. It's the one with the best space guarantee, but it's worth knowing the alternatives so you can say something sensible when an interviewer asks "what else?"

| Approach | Time | Extra space | Catch |
| --- | --- | --- | --- |
| Fast/slow pointers (Floyd) | O(n) | O(1) | Needs the two-sweep trick to find the entrance |
| Hash set of visited nodes | O(n) | O(n) | Simple, but memory scales with the list |
| Marking visited nodes | O(n) | O(1) | Mutates the list; often disallowed, and breaks if values are signed or immutable |
| Brent's algorithm | O(n) | O(1) | Telemetering with exponential jumps; same asymptotic space, usually fewer evaluations in practice |

Brent's algorithm is the interesting one for people who've seen Floyd's a dozen times. It's based on exponential search and, according to the cycle-detection literature, both use a constant number of memory cells while taking a number of steps proportional to the distance from the start of the sequence to the first repetition. Cycle detection itself reaches well beyond linked lists — the same idea is used to test pseudorandom number generators and hash functions, to detect infinite loops in programs, and to catch deadlocks in database transaction management.

For a LeetCode-style interview, write Floyd's. Mention the hash set to show you know the tradeoff. Mention Brent's only if you can actually describe it.

## How this shows up in interviews

AlgoMonster's own statistical breakdown of interview problems puts linked list alongside array, hash map, stack, queue, and sorting in the "high ROI" bucket of basic data structures, and it names cycle detection specifically as one of the classics that "get asked repeatedly," next to reversing a list and finding the middle node.

That's a useful framing if you're deciding where to spend your time. Two pointers, DFS, and BFS sit at the top of AlgoMonster's ROI table, and cycle detection is a two-pointer problem in a topic that gets recycled constantly. It's short enough to fully master in an afternoon, and it shows up as a sub-step inside harder problems — detecting a loop in a functional graph, checking a linked structure for a deadlock, or as the first phase of a cycle-removal question.

There's a limit worth acknowledging: cycle detection is one pattern, and passing an interview at a company that reinvents problems (Google, per AlgoMonster's own notes on the topic) means being fluent in many of them, not one. Anyone selling you a single trick as the whole answer is overselling.

## Practicing it without grinding 300 problems

The reason people get this wrong at interview time isn't that the algorithm is hard. It's that they've read it once, nodded, and never traced pointers across a table by hand. Cycle detection rewards one careful pass of manual tracing more than ten problems of pattern-matching.

AlgoMonster is built around the opposite of the giant problem bank: it teaches named patterns with explanations, and the linked list material follows the same approach. Its "Linked List Cycle" problem asks you to determine whether a linked list from the first node contains a cycle, with constraints up to 10^5 nodes, and explicitly offers constant space as a bonus objective — which is the version you should actually aim for. The platform is text and diagram based rather than video, so if you learn by reading code and tracing visualizations instead of watching someone talk through them, it fits better than video-first courses do.

👉 [Try the linked list cycle problem on AlgoMonster](https://algo.monster/problems/linked_list_cycle?fpr=li69)

Everything past the free sample content needs a paid plan. Here's what the pricing page currently shows.

| Plan | What you get | Price | Billing | Best for | Link |
| --- | --- | --- | --- | --- | --- |
| Free | Limited access to a sample of patterns and problems | $0 | — | Checking whether the text-and-diagram teaching style works for you | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; questions from top tech companies refreshed monthly; 3,000+ editorials; member discount on live bootcamps | $99/year (shown as $8.25/mo) | Billed annually, 12 months of access | Preparing for one round of interviews over the next few months | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | One-time payment | Expecting to interview again in a few years | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two honest notes about that table. AlgoMonster runs sales regularly, and the numbers above are what the site was showing when I checked, so confirm the figure at checkout before you pay — the annual plan has also displayed a short trial period before the annual charge kicks in. And the lifetime plan only makes sense on repeat use: at $189 versus $99 a year, it pays for itself early in the second year, which is a good deal if you job-hop and a wasteful one if you finish the curriculum in six weeks and never return.

Neither plan is a substitute for writing code. If you read the AlgoMonster editorial on cycle detection and skip the pointer tracing, you'll be exactly where you started.

## FAQ

**Is Floyd's algorithm really O(n) time?**

Yes. With no cycle, `fast` hits null after roughly n/2 iterations. With a cycle, `slow` enters the loop after μ steps and the gap closes within at most λ more, and μ + λ ≤ n. Cycles are detected in a single pass, not repeated passes.

**Why not just use a hash set? It's easier.**

It is easier, and it's a legitimate answer — O(n) time, O(n) space. But "do this with constant space" is often the actual question, and the two-pointer version is the one that passes that bar. Candidates who reach for a set first usually get asked to redo it.

**Can duplicates cause a false positive?**

No, as long as you compare node references instead of values. Storing values in a set will produce false positives on inputs like `1 → 2 → 1`, which is a real and commonly tested trap.

**Do I need the math proof for interviews?**

Rarely for the boolean version. For the version that returns the cycle's starting node, yes — you'll be asked why resetting one pointer to the head works, and "it just does" won't survive a follow-up. Know that μ + k is a multiple of λ and you can reconstruct the argument on the spot.

**Do I need to handle a cycle that includes the head?**

That case works without any extra code. If the entrance is the head, `μ` is zero, the second loop exits immediately because the reset pointer is already at the entrance, and you return the head. No null checks needed, since the reset pointer never goes past the entrance.

The whole algorithm is about fifteen lines in any language. The part worth your time isn't memorizing those lines — it's being able to trace both pointers across a table and explain, without notes, why the gap in the loop can only ever close.
