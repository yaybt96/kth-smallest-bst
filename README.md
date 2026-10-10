# kth smallest element in a BST: the inorder trick interviewers expect, plus O(1)-space and follow-up answers

LeetCode 230 is one of those problems where the entire solution fits in a single sentence: an inorder traversal of a binary search tree visits nodes in ascending order, so the k-th node you visit is the k-th smallest value. Most candidates know that sentence. Fewer can implement it without an off-by-one bug, explain the space cost correctly, or answer the follow-up question about a tree that keeps changing.

This is a walkthrough of the problem itself — recursive, iterative, Morris, and the augmented-tree follow-up — plus the parts that actually decide whether the interview goes well.

## The one property that solves this problem

A BST has a structural guarantee: everything in a node's left subtree is smaller than the node, and everything in its right subtree is larger. Push that guarantee through a left-node-right traversal and the output is sorted. Not approximately sorted. Exactly sorted.

So the problem reduces to "return the value of the k-th node visited during an inorder walk." Two flags worth calling out before you write anything:

- `k` is 1-indexed. `k = 1` is the smallest value, not the second smallest.
- You do not need to sort anything. Collecting all values into an array and calling `sort()` also produces the right answer in `O(n log n)`, and it throws away the entire reason a BST was given to you. Interviewers read that as a missed signal, even when the tests pass.

## Recursive inorder with an early stop

The naive recursive version walks the whole tree and returns `values[k-1]`. You can do better: count visits as you go and bail out the moment the counter hits `k`.

python
def kthSmallest(root, k):
    count = 0
    answer = None

    def inorder(node):
        nonlocal count, answer
        if node is None or answer is not None:
            return
        inorder(node.left)
        count += 1
        if count == k:
            answer = node.val
            return
        inorder(node.right)

    inorder(root)
    return answer


The guard `if answer is not None: return` is what makes the early exit actually work. Without it, the recursion keeps unwinding into right subtrees you no longer care about. The code still returns a correct value, which is exactly why the bug survives local testing and shows up as a confused interviewer asking "why are you still traversing?"

### Complexity, stated honestly

Say it this way: the traversal visits exactly `k` nodes before stopping, so time is `O(k)` with the early exit, `O(n)` if you drop it. Space is the height of the tree, `O(h)`, because the call stack holds one frame per level on the current path — `O(log n)` for a balanced tree, `O(n)` for a degenerate one that looks like a linked list.

That last part matters. Candidates who say "space is `O(log n)`" without qualifying it are assuming a balanced input the problem never promised. Naming the skewed case before the interviewer asks is one of the cheapest credibility points available in a 45-minute screen.

## The iterative version with an explicit stack

Recursion is fine until the interviewer asks for it without recursion, or until you're staring at a tree that could be 10,000 nodes deep. The stack version simulates the same walk:

python
def kthSmallest(root, k):
    stack, cur = [], root
    while stack or cur:
        while cur:                # dive to the leftmost node
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()         # visit
        k -= 1
        if k == 0:
            return cur.val
        cur = cur.right           # then go right
    return -1


Two things to notice. The early exit is free here — you just return from inside the loop instead of threading a flag through recursive calls. And decrementing `k` in place is the cleanest way to count, because it removes the separate counter variable that causes most off-by-one mistakes.

## Morris traversal: O(1) extra space

If the interviewer pushes on the `O(h)` space, Morris traversal answers with constant space by temporarily rewiring the tree. The idea: for a node with a left child, find its inorder predecessor (the rightmost node in the left subtree) and point that predecessor's `right` back at the current node. That thread lets you return to the parent without a stack. You remove the thread on the second pass.

python
def kthSmallest(root, k):
    cur = root
    while cur:
        if cur.left is None:
            k -= 1
            if k == 0:
                return cur.val
            cur = cur.right
        else:
            pre = cur.left
            while pre.right and pre.right is not cur:
                pre = pre.right
            if pre.right is None:      # build the thread
                pre.right = cur
                cur = cur.left
            else:                      # thread already there: tear it down
                pre.right = None
                k -= 1
                if k == 0:
                    return cur.val
                cur = cur.right
    return -1


Time is `O(n)`, not `O(k)` — reaching the k-th node requires walking the predecessor chains, and that cost is proportional to the number of nodes even though you stop early. Space is `O(1)`. The tradeoff is real: Morris temporarily mutates a caller's tree, which is impolite in production code and fine in an interview where space is the stated constraint.

## All four approaches side by side

`h` is the tree height. `n` is the node count.

| Approach | Time | Extra space | Reach for it when |
| --- | --- | --- | --- |
| Recursive inorder, stop at k-th visit | O(k) visits | O(h) call stack | Default answer; balanced tree or small `k` |
| Iterative inorder with explicit stack | O(k) visits | O(h) explicit stack | No recursion allowed, or the tree may be very deep |
| Morris traversal | O(n) | O(1) | Space is the hard constraint |
| Subtree-size augmentation | O(h) per query | O(h) to update counts | Follow-up: frequent inserts/deletes and repeated queries |
| Max-heap of size k | O(n log k) | O(k) | Input is a plain binary tree, or values arrive as a stream |

The heap row is worth remembering for a specific reason: it is the correct answer when the input is a binary tree and not a search tree. A lot of candidates freeze when a variant swaps "BST" for "binary tree," because the inorder trick evaporates. Maintaining a max-heap of size `k` while walking every node gives you the k-th smallest in `O(n log k)` with `O(k)` memory, and it works regardless of ordering.

## The follow-up question: what if the tree keeps changing?

The problem statement ends with the question that separates a memorized solution from an understood one:

> If the BST is modified often (insert/delete) and you need to find the k-th smallest frequently, how would you optimize?

The answer is to store a count of subtree size on each node. Then finding the k-th smallest becomes a descent instead of a traversal: look at the size of the left subtree. If it is at least `k`, the answer is in the left subtree. If it equals `k - 1`, the current node is the answer. Otherwise, subtract `left_size + 1` from `k` and descend right.

| Operation | With subtree counts | Notes |
| --- | --- | --- |
| `kthSmallest` | O(h) | Each step moves down one level |
| `insert` | O(h) to find the spot + O(h) to fix counts up the path | Counts update on the way back up |
| `delete` | O(h) | Same count-fix pass |

The catch that interviewers listen for: `h` is only `O(log n)` if the tree stays balanced. A plain BST with sorted inserts degrades into a linked list and your "optimized" query is back to `O(n)`. To actually deliver `O(log n)` you need a self-balancing structure — a red-black tree, an AVL tree, or an order-statistic tree that maintains subtree sizes through rotations. Mentioning rotations is usually enough to close the topic.

There's a second, related family of problems worth knowing: "k-th smallest in a sorted matrix," "k-th smallest in two sorted arrays," and "k-th smallest pair distance." Those are binary-search-on-the-answer problems, not tree problems, and they live in a different pattern bucket entirely.

## Common bugs that cost you the interview

- **Preorder or postorder.** Only inorder gives sorted output on a BST. Preorder on `[5,3,6,2,4,null,null,1]` with `k = 3` hands you `5`, which is wrong.
- **Treating `k` as a zero-based index.** Mixing `values[k]` logic with a counter that starts at 1 is the single most common source of wrong answers here.
- **Not stopping.** Correct output, wasted work, and a follow-up question you now have to answer badly.
- **Misstating space complexity.** `O(h)`, and say what `h` can be.
- **Forgetting `null` nodes.** Constraints guarantee `1 <= k <= n <= 10^4`, so `k` is always valid — but in a real codebase you would validate rather than trust the caller.
- **Assuming the tree is balanced.** Nothing in the constraints says so.

## A sample answer script for the whiteboard

If you want a structure to rehearse, this order tends to land well:

1. "The key invariant is that inorder traversal of a BST yields ascending order, so this is a traversal problem, not a sorting problem."
2. "I'll do an iterative inorder walk with an explicit stack, because I can return immediately when the count hits `k` and I don't have to worry about recursion depth."
3. Walk through the leftmost descent on the sample tree out loud, then pop, decrement, check.
4. "Time is `O(k)` visits, space is `O(h)` for the stack — `O(log n)` balanced, `O(n)` skewed."
5. If asked for constant space, "Morris traversal gets `O(1)` space by threading predecessors, at the cost of `O(n)` time and temporarily mutating the tree."
6. If asked about frequent updates, "subtree-size counts turn the query into an `O(h)` descent; keeping `h` logarithmic needs a balanced tree."

That sequence answers the original question, the space question, and the follow-up before being asked, which is roughly the whole point of the exercise.

## Practicing this pattern instead of memorizing this problem

LeetCode 230 is one instance of a pattern that shows up repeatedly: a tree's structure gives you an ordering for free, and you exploit it with a traversal that stops early. The same reasoning solves "validate a BST," "k-th largest element in a BST," and a pile of range-query problems.

That's the framing **AlgoMonster** is built around. It's a text-and-diagram interview prep platform from ex-Google engineers and competitive programmers, organized by reusable coding patterns rather than by problem count. For this topic specifically, the relevant pieces are the depth-first search and BST material, plus a decision flowchart that includes a literal "kth smallest/largest?" branch — the platform's whole pitch is that you learn to pick the pattern from the problem statement instead of grinding until recognition happens by accident.

What the Pro content includes, per the site: 48 patterns, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, template code, unlimited Speedrun drill mode, unlimited AI assistant use, company-specific question sets updated monthly, and two bonus courses (a CS foundations course and a build-an-LLM course). The honest limitations, which third-party reviews consistently flag: everything is text and diagrams with no video, the problem bank is around 231 lessons and problems rather than thousands, and the system design and behavioral content is introductory rather than senior-level. If you learn from video walkthroughs, this format will annoy you.

👉 [Work through the BST and DFS patterns on AlgoMonster](https://algo.monster?fpr=li69)

### Plans and pricing

There is no monthly tier. The subscribe page shows a free tier plus two paid plans, and promotional pricing moves around between visits — the same page displayed `$99/year` and `$189` lifetime on one check, and `$69.99/year` and `$89` lifetime on another, with list prices shown as struck-through `$198` and `$378`.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the teaching format | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions; 12 months of full access | $99/year displayed (list $198); deeper promos have shown $69.99/year | Renews annually, cancel anytime | [Check the current Pro Yearly price](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, paid once | $189 one-time (list $378); a deeper sale showed $89 | One-time payment | [See the Pro Lifetime deal](https://algo.monster?fpr=li69) |

Because the pricing is genuinely promotional and shifts, confirm the number on the checkout page before paying rather than trusting any quoted figure — including the ones above. Third-party write-ups quote lifetime prices anywhere from `$89` to `$459`, which is a good signal that the sale tier rotates rather than that any one number is current.

There's no formal free trial beyond the free tier, so the sensible move is to spend an hour with the free content and decide whether reading text and diagrams is how you actually learn. Several reviews also repeat an offer guarantee tied to completing the curriculum; treat that as marketing copy you'd want to read the terms of, not a verified outcome.

If you already have fluency in one language and your real problem is that you can't identify the right approach in the first ten minutes of a problem, the pattern-first structure is a reasonable fit. If you need volume and frequency data above all else, a problem bank is the better buy, and plenty of people run both.

👉 [Compare the plans and current discounts](https://algo.monster?fpr=li69)

## FAQ

**Is the answer `O(n)` or `O(log n)`?**
Neither, as usually stated. Reaching the smallest node costs `O(h)`, and the traversal visits `O(k)` nodes with the early exit, so `O(k)` time and `O(h)` space. `O(log n)` only appears if you assume a balanced tree.

**Does the same solution work on a plain binary tree?**
No. Without the ordering guarantee, inorder traversal gives you an arbitrary sequence. Use a max-heap of size `k` for `O(n log k)` time and `O(k)` space, or sort all values if you don't care about efficiency.

**What about the k-th largest?**
Mirror the traversal: right subtree, node, left subtree. First visit is the largest, k-th visit is the k-th largest. Same code, swapped children.

**Can it be done without recursion?**
Yes — an explicit stack, or Morris traversal if you also need constant space.

**What if `k` is out of range?**
The problem constraints guarantee `1 <= k <= n`, so no handling is required for the graded solution. Mention that in production you'd validate and return an explicit error rather than a sentinel like `-1`.

## Bottom line

The k-th smallest element in a BST is a traversal problem wearing a data-structure costume. Say the inorder invariant out loud, implement the iterative walk with an early exit, state `O(k)` time and `O(h)` space without pretending the tree is balanced, and have the subtree-size answer ready for the follow-up. That covers everything LeetCode 230 is actually testing — and the same reasoning is what makes the pattern worth practicing rather than the problem worth memorizing.
