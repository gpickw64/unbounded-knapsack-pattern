# unbounded knapsack: the forward-loop DP pattern behind coin change, rod cutting and word break

The phrase "unbounded knapsack" usually shows up in someone's search bar in one of two moods: they're staring at a DP problem where reusing items seems allowed, or they're staring at a 1D array wondering why their loop iterates forward here and backward in the 0/1 version. Both moods point at the same thing. The only thing that makes a knapsack problem "unbounded" is that every item type has infinite copies, and that single change is what flips the loop direction.

## What actually changes when items are unbounded

Three knapsack variants get mixed up constantly, and the difference is entirely about how many copies of each item you're allowed to take.

| Variant | Copies per item type | Typical 1D loop over capacity |
| --- | --- | --- |
| 0/1 knapsack | At most 1 | Backward, from capacity down to weight |
| Bounded knapsack | A fixed limit per item | Split into 0/1 items (binary splitting) |
| Unbounded knapsack | Unlimited | Forward, from weight up to capacity |

The formal statement: you have n item types, type i has weight wᵢ and value vᵢ, and a knapsack of capacity W. Maximize total value without exceeding W, where the quantity of each type is any non-negative integer. It's a classic NP-hard combinatorial problem, though only weakly NP-complete, which is why a pseudo-polynomial DP exists and is what you'd actually write in an interview.

The word "unbounded" describes the supply, not the answer. You still can't exceed capacity. You just can't run out of the item you're taking.

## The recurrence, in plain English

Define K(b) as the best value you can pack into a bag of capacity b. Two facts are enough to build everything:

- K(0) = 0. A bag with no room holds nothing.
- For b > 0, try every item type that fits. Taking one copy of item k leaves you a smaller version of the exact same problem, capacity b − wₖ.

That gives:


K(b) = max over k where w[k] <= b of ( v[k] + K(b - w[k]) )


Notice what's missing compared to 0/1 knapsack: there's no index tracking "how many item types have I considered". In the 0/1 world you must decide on each item exactly once, so the state needs both the item index and the remaining capacity. Here, because any item can be taken again later, the subproblem is only about remaining capacity. That state reduction is why the answer fits in a single array.

Memoize K by capacity and you have a top-down solution with O(n·W) states and O(n) work per state, which is the same total complexity as the bottom-up version, just with recursion overhead and a call stack.

## Bottom-up, in one array

The iterative version fills capacity values in increasing order, because K(b) only depends on values smaller than b:

python
def unbounded_knapsack(capacity, weights, values):
    dp = [0] * (capacity + 1)
    for c in range(1, capacity + 1):
        for i in range(len(weights)):
            if weights[i] <= c:
                dp[c] = max(dp[c], values[i] + dp[c - weights[i]])
    return dp[capacity]


Two loops, O(n·W) time, O(W) space. The inner loop can be over items or over capacity, and for the plain optimization version it doesn't matter which one you put outside, because each capacity value gets the max over all items either way. This is the point where people relax, and then the counting variants punish them for it. More on that below.

If you keep the 2D table instead of collapsing it, the transition is:


dp[i][c] = max(dp[i-1][c], v[i] + dp[i][c - w[i]])


Read the right side carefully. The second term references dp[i], the same row, not dp[i-1]. That's the unbounded allowance showing up in the indices: after deciding to take item i, item i is still on the table.

## Why the loop direction is the whole trick

Here's the 0/1 version that everyone writes:

python
for i in range(n):
    for c in range(capacity, weights[i] - 1, -1):
        dp[c] = max(dp[c], values[i] + dp[c - weights[i]])


Backward iteration is a load-bearing detail, not a style choice. When you go from high capacity to low, `dp[c - weights[i]]` hasn't been touched yet in this round, so it still represents a state built without item i. Item i gets used at most once.

Flip the loop forward and the same code silently becomes the unbounded solution: `dp[c - weights[i]]` has already been updated in this round, so it may already contain item i, and you're free to stack copies.

This is a genuinely useful habit in an interview. If you've written 0/1 knapsack a hundred times, the fact that "just reverse the loop" converts it to unbounded is a one-line answer to a question that otherwise sounds hard.

## Coin change is unbounded knapsack wearing a hat

The most common place this pattern appears in interviews is coin change, and it's not a loose analogy. AlgoMonster's own Coin Change lesson opens by saying the problem is an unbounded knapsack problem, with each denomination usable unlimited times. Mapping the two is a rename: coin = item, denomination = weight, target amount = capacity.

The optimization goal flips direction, though. Knapsack maximizes value; minimum-coin-change minimizes count. And there's a boundary difference worth remembering: knapsack asks for the best solution not exceeding capacity, while coin change usually asks for a solution hitting the amount exactly.

python
def coin_change(coins, amount):
    INF = amount + 1
    dp = [INF] * (amount + 1)
    dp[0] = 0
    for a in range(1, amount + 1):
        for coin in coins:
            if coin <= a:
                dp[a] = min(dp[a], dp[a - coin] + 1)
    return dp[amount] if dp[amount] != INF else -1


Using `amount + 1` as the "impossible" sentinel instead of `float('inf')` or `INT_MAX` is the standard trick. The maximum number of coins you'd ever need for amount a is a (all ones), so any value above that is provably invalid, and you avoid overflow when adding 1. If dp[amount] is still the sentinel at the end, the amount can't be made and you return -1.

The same shape, with different sentinels and transitions, covers rod cutting, integer break, perfect squares, minimum cost for tickets, and word break. Learn the loop structure once and you're reusing it across half the DP question bank.

## Where it gets subtle: counting instead of optimizing

Counting problems look identical and are not. This is the failure mode that actually costs offers.

Coin Change II asks how many combinations of coins make the amount. Combination Sum IV asks how many sequences do, where [1,2] and [2,1] count separately. The code differs by where the loops sit.

python
# Combinations: order does not matter
dp = [0] * (amount + 1)
dp[0] = 1
for coin in coins:
    for a in range(coin, amount + 1):
        dp[a] += dp[a - coin]

# Permutations: order matters
dp = [0] * (amount + 1)
dp[0] = 1
for a in range(1, amount + 1):
    for coin in coins:
        if coin <= a:
            dp[a] += dp[a - coin]


Items outside, capacity inside: each coin gets fully processed before the next is introduced, so combinations can only be built in one canonical order. Capacity outside, items inside: every capacity value reconsiders every coin as a possible last move, which counts orderings.

Neither version is wrong. Confusing the two is. If your answer is off by a factor or a specific test case fails with a symmetric input, check the loop order first, before you check anything else.

## Two mistakes worth naming

**The greedy instinct.** The highest value-to-weight ratio first feels right and is not. As a University of Texas Arlington lecture on unbounded knapsack points out bluntly, the ratio-based "put as many of the best item as fit, then move on" approach is the greedy version and does not return the optimal answer for all instances. It works for fractional knapsack because you can cut an item. With whole items and integer capacities, a slightly worse ratio item can fill the leftover space better and win overall. If a problem statement is coin change and a candidate starts sorting by denomination, that's the tell.

**Treating complexity as polynomial.** O(n·W) looks like polynomial time in two variables, and it is polynomial in the numeric value of W, not in the input size. W is given in binary, so its bit length is log W, which makes the runtime exponential in input length. That's what "pseudo-polynomial" means, and it's the standard answer to "isn't knapsack NP-hard?" Both things are true: the decision problem is weakly NP-complete, and the DP is the practical solution for the W values that appear in real tests.

## Drilling the pattern instead of memorizing it

Reading this once won't hold. The pattern only sticks after you've written the forward loop, gotten a wrong answer from a loop-order mistake, and fixed it yourself.

A sensible order of attack, roughly matching how the knapsack section of a structured course sequences it: weight-only knapsack (partition equal subset sum) first, then unbounded knapsack introduction, then coin change optimization, then knapsack with weight plus value, then 0/1 knapsack, and bounded knapsack with its binary-splitting reduction last. Doing weight-only first means you isolate the "can we fill exactly this capacity" question before adding values into the mix.

👉 [Knapsack DP introduction on AlgoMonster](https://algo.monster/problems/dp_knapsack_intro?fpr=li69) walks from the 0/1 baseline into the unbounded case with diagrams of the state table, and its Coin Change lesson states the unbounded knapsack framing directly before going into code. There's also a discussion thread attached to the knapsack intro lesson where learners argue through specific table cells, which is the kind of thing that catches a misread state definition before it becomes a bad habit.

## What AlgoMonster is, and what it isn't

AlgoMonster is a coding interview prep platform built around explicit problem patterns rather than a large problem bank. It's text and diagram based: no video walkthroughs. Its paid tier advertises 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, all patterns and problems unlocked, unlimited Speedrun sessions, an AI assistant, questions sourced from top tech companies and refreshed monthly, and a member discount on live bootcamps.

Tradeoffs worth knowing before paying, from what third-party reviews report: the problem count is small relative to LeetCode's 3,000+, coverage of system design and behavioral rounds is introductory, and there's no live coaching or mock interviews. A 2026 review on Lodely makes the cost-per-problem comparison and lands on the same conclusion AlgoMonster would: you're paying for the pattern framework and decision flowcharts, not for volume.

There is no monthly plan and no enterprise or team tier. Access is annual or lifetime, and both are on one subscribe page.

## Plans and current pricing

Prices on this page move with promotions, so treat the table as a snapshot and confirm the number at checkout.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems, enough to test the teaching format | $0 | No card required | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, monthly-refreshed company questions, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps | $69.99/year, shown as $5.84/month on the annual billing page | Billed annually, 12 months of access, cancel or upgrade anytime | [Get Pro Yearly access](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates on all future content, one payment, no renewals | $89 one-time | Billed once, yours permanently | [Get Pro Lifetime access](https://algo.monster/subscribe?fpr=li69) |

One honest caveat on those numbers: AlgoMonster runs frequent sales and the price rendered on the subscribe page can differ from what older write-ups list. Published figures from other sources range from about $99 to $299 per year and up to $459 for lifetime when no sale is active. The only number that matters is the one on the checkout page when you buy, and it's usually the sale price.

The annual versus lifetime call is simple arithmetic. If you're prepping for one interview loop over the next couple of months, the annual plan covers it. If you expect to interview again in two or three years, the lifetime price is roughly the cost of one annual renewal, which makes it the cheaper path over a career. Nothing about the DP pattern above changes between plans; the paid tier decides how much guided practice you get, not whether you can solve coin change.

👉 [See the current plans and price on AlgoMonster](https://algo.monster/subscribe?fpr=li69)

## FAQ

**Is unbounded knapsack harder than 0/1 knapsack?**

The state is simpler, since you don't need the item index. The implementation is one loop direction away from 0/1. What trips people up is the counting variants, where loop order changes the meaning of the answer.

**Can I solve unbounded knapsack with one array?**

Yes. dp[c] for c from 1 to W, inner loop over item types, forward iteration, O(W) space.

**Why does forward iteration allow reuse?**

Because when you read dp[c - wᵢ] mid-round, that cell may already include item i from an earlier step in the same round. Backward iteration reads cells that are still holding pre-round values.

**Why is the greedy ratio approach wrong?**

Because whole items don't fill leftover capacity as neatly as fractions do. A lower-ratio item can produce a better total in a specific capacity, and greedy commits before it knows.

## Bottom line

Unbounded knapsack is one recurrence, one loop direction, and a family of problems that reshapes it. Get K(b) = max(v[k] + K(b − w[k])) and the forward loop into your hands, then differentiate minimum-count, exact-fill counting, and order-sensitive counting cases by which loop sits outside. Practice the weight-only version first, and check the wrong answers against the actual table instead of guessing.

If you're doing this alone and want the pattern sequence laid out with diagrams, 👉 [work through the knapsack section on AlgoMonster](https://algo.monster/problems/coin_change?fpr=li69) and see whether the text-and-diagram format fits how you learn before paying for it.
