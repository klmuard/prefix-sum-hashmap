# subarray sum equals k: prefix sum + hashmap explained line by line, with Python and Java code

Subarray Sum Equals K (LeetCode 560) is the problem that quietly splits people into two groups: those who reach for a sliding window, watch it fail the negative-number test cases, and then find the prefix-sum trick; and those who have already memorized the prefix-sum trick and finish in four minutes. This walks through both paths, including the code in Python, Java, and C++, the trace you can follow without running anything, and the details that decide whether your submission gets accepted or fails on a hidden test case.

## What the problem is actually asking

Given an array of integers `nums` and an integer `k`, return the number of contiguous, non-empty subarrays whose elements sum to exactly `k`.

Two examples from the problem statement:

- `nums = [1,1,1], k = 2` → `2`. The subarrays `[1,1]` at indices 0–1 and `[1,1]` at indices 1–2.
- `nums = [1,2,3], k = 3` → `2`. The subarrays `[1,2]` and `[3]`.

The constraints are the part people skim past:

- `1 <= nums.length <= 2 * 10^4`
- `-1000 <= nums[i] <= 1000`
- `-10^7 <= k <= 10^7`

Everything important hides in those three lines. The array length caps you at roughly an `O(n log n)` or `O(n)` solution if you want the tests to finish, and the negative bound on `nums[i]` is the reason the technique you *think* applies doesn't.

## Why sliding window fails here

Sliding window (grow right, shrink left) works on problems like Minimum Size Subarray Sum because of one assumption: adding an element never decreases the window sum. With all-positive numbers, that assumption holds, so when your sum exceeds the target you can safely drop elements from the left and never look back.

The moment negatives enter the array, that monotonicity is gone. Adding `-20` to a window can push the sum *down* past `k`, and shrinking the window can push it back *up*. Any two-pointer logic built on "too big, move left" starts discarding valid subarrays, and no amount of patching fixes it for the general case.

python
# temptation: this passes [1,1,1] and [1,2,3], then dies on negatives
# because "sum > k -> shrink" is not a valid rule when nums[i] can be negative


That's the pivot point of the problem. The array can contain negative numbers, so you need a technique that doesn't care about ordering or monotonicity.

## Brute force first, so you know what you're improving

The naive version checks every `(start, end)` pair and re-sums the range, which is `O(n^3)` if you call `sum()` naively. Carrying a running total as you extend the end index drops it to `O(n^2)`:

python
def subarraySum(nums, k):
    count = 0
    n = len(nums)
    for i in range(n):
        total = 0
        for j in range(i, n):
            total += nums[j]
            if total == k:
                count += 1
    return count


With `n = 20,000`, that's 200 million iterations in the worst case. In Python it will time out; in C++ it might squeak by, it might not. Either way, brute force is worth writing down first, because it tells you exactly what information you're recomputing and what a faster solution needs to cache.

## The prefix-sum insight, and the two-sum problem hiding inside it

A prefix sum is the total of everything from index 0 up to (but not including) some index. If `prefix[j]` is the sum of `nums[0..j-1]`, then the sum of the subarray from `i` to `j-1` is:


sum(nums[i..j-1]) = prefix[j] - prefix[i]


Substituting the condition `sum(nums[i..j-1]) == k`:


prefix[j] - prefix[i] = k
prefix[i] = prefix[j] - k


Read that second line slowly, because it's the whole solution. As you scan left to right, you maintain the current prefix sum `prefix[j]`. The question you ask at each step is: *how many earlier prefix sums equaled `prefix[j] - k`?* Each one of them marks a subarray ending at the current position whose sum is exactly `k`.

That's the complement lookup from Two Sum, with prefix sums in the role of the values. Two Sum stores each number you've seen; here you store each prefix sum you've seen, along with how many times it has occurred.

## Counting with a hashmap: the version that passes

python
from collections import defaultdict

def subarraySum(nums, k):
    counts = defaultdict(int)
    counts[0] = 1        # the empty prefix, sum 0, seen once
    total = 0
    result = 0
    for num in nums:
        total += num
        result += counts[total - k]
        counts[total] += 1
    return result


`O(n)` time, `O(n)` space. Two lines carry all the subtlety:

1. `counts[0] = 1` seeds the map with the empty prefix. Without it you miss every subarray that starts at index 0, because those correspond to `prefix[i] = 0` and there's no earlier element to have produced that value.
2. `result += counts[total - k]` happens **before** `counts[total] += 1`. You must not count the current prefix against itself. Insert first and you'd count a zero-length subarray every time `k == 0`.

The same thing in Java, with `long` arithmetic so a large array can't overflow you:

java
public int subarraySum(int[] nums, int k) {
    Map<Long, Integer> seen = new HashMap<>();
    seen.put(0L, 1);
    long prefix = 0;
    int count = 0;
    for (int num : nums) {
        prefix += num;
        count += seen.getOrDefault(prefix - k, 0);
        seen.merge(prefix, 1, Integer::sum);
    }
    return count;
}


And in C++:

cpp
int subarraySum(vector<int>& nums, int k) {
    unordered_map<long long, int> seen{{0, 1}};
    long long prefix = 0;
    int count = 0;
    for (int num : nums) {
        prefix += num;
        count += seen[prefix - k];
        ++seen[prefix];
    }
    return count;
}


## A trace you can follow without running anything

Take `nums = [1, 2, 1, 2]`, `k = 3`. The map starts as `{0: 1}`.

| Step | num | prefix | prefix − k | map hits | answer | map after |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 1 | −2 | none | 0 | `{0:1, 1:1}` |
| 2 | 2 | 3 | 0 | 1 | 1 | `{0:1, 1:1, 3:1}` |
| 3 | 1 | 4 | 1 | 1 | 2 | `{0:1, 1:1, 3:1, 4:1}` |
| 4 | 2 | 6 | 3 | 1 | 3 | `{0:1, 1:1, 3:1, 4:1, 6:1}` |

Final answer: `3`, corresponding to `[1,2]` at indices 0–1, `[2,1]` at 1–2, and `[1,2]` at 2–3.

Step 2 is where the `counts[0] = 1` seed earns its place: `prefix - k` is `0`, and the only reason that's a hit is the pre-seeded empty prefix.

## The chain of mistakes that costs people hidden test cases

Five things break submissions on this problem, and they're all small:

- **No `{0: 1}` seed.** Passes when the answer happens to involve subarrays starting at index 1 or later, fails on anything that starts at 0.
- **Inserting before looking up.** Subtly corrupts results when `k == 0` and produces off-by-one counts elsewhere.
- **Storing a boolean instead of a frequency.** If the same prefix sum occurs three times, all three starting positions are valid subarrays. A set gives you one.
- **Indexing by value instead of by sum.** A map of `index → value` can't handle duplicate prefix sums, and duplicates are the norm rather than the exception in arrays with negatives or zeros.
- **`int` overflow in Java or C++ on pathological inputs.** The problem's own bounds keep values small, but related variants don't, and switching to `long` costs nothing.

One more, less a bug than a habit worth breaking: don't try to force the two-pointer version "since it's faster." It isn't faster, it's wrong for this input class.

## Variants interviewers ask immediately after

If you can solve 560 cleanly, you're about one adjustment away from these, and they come up in the same interview loop:

- **Return the indices, not the count** — store the earliest index for each prefix sum and return `[found + 1, current]` with an exclusive right boundary.
- **Maximum size subarray that sums to k** (LeetCode 325) — keep the *first* index each prefix sum was seen at, since earliest wins on length.
- **Binary subarrays with sum** (LeetCode 930) — the same pattern with two pointers or prefix counts over a 0/1 array.
- **Subarray sums divisible by k** (LeetCode 974) — hash on `prefix % k`, normalizing negative remainders.
- **Number of submatrices that sum to target** (LeetCode 1074) — the 2D version: fix a pair of rows, collapse them into a 1D column sum, then run 560 on that.
- **Count number of nice subarrays** (LeetCode 1248) — replace values with parity and it's the same complement lookup.

The pattern name is what matters here: *prefix sum plus hashmap*. Once you can name it, you stop re-deriving it from scratch each time.

## Practicing the pattern instead of the problem

Solving 560 once teaches you 560. Recognizing that it's a template — running total, frequency map, complement lookup — is what transfers to 325, 974, 1074, and the rest of the family. That distinction is the entire pitch behind AlgoMonster, which organizes its curriculum around explicit patterns rather than a wall of problems.

For this specific topic, the free material is usable without paying anything. The [Prefix Sum intro lesson](https://algo.monster/problems/prefix_sum_intro?fpr=li69) covers the running-totals transformation, the `n + 1` array length, and the off-by-one at the boundaries (`sum(arr[i..j-1]) = prefix[j] - prefix[i]` with an exclusive right index), then points at the hashmap upgrade as the next step. The 560 editorial walks the trace out value by value, which is more useful than a bare code block when you're trying to build the intuition.

What the paid tier adds is the surrounding structure: all patterns unlocked, unlimited Speedrun sessions for timed pattern recall, real questions from major tech companies refreshed monthly, and an AI assistant for hints when you're stuck. The content is written text with diagrams rather than video, so if you learn by watching, this will feel the wrong shape. Third-party writeups of the platform consistently describe it the same way, and also note that it's a small, focused library rather than a thousand-problem bank.

👉 [Try the prefix sum lesson and browse the full pattern library](https://algo.monster?fpr=li69)

## What AlgoMonster costs, and which plan fits

Pricing on AlgoMonster is not stable. There's no monthly plan and no true free trial beyond the free tier, and the paid tiers are almost always sitting inside some promotion, so the number you see depends on when you look. Two snapshots from the same period showed different figures.

| Plan | What you get | Price as listed | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems, enough to judge the teaching style | $0 | None | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 12 months of full access | $99/year displayed as $8.25/month, with a 3-day free trial on the page; promo snapshots have shown as low as $69.99/year | Annual, renews | [See current Pro Yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates, billed once; the page also lists an internal referral to a top tech company for qualified candidates | $189 one-time displayed; promo snapshots have shown as low as $89 | One payment | [See current Pro Lifetime price](https://algo.monster/subscribe?fpr=li69) |

Because discounts move around, check the checkout total before you commit rather than trusting a cached number from any review, including this one.

On which to pick: if your interview is within the next two or three months, yearly is the lower-risk outlay. Lifetime makes more sense if you expect to interview repeatedly over several years, since the platform's content updates are included. The honest counterpoint is that most people finish a pattern-based curriculum in a few weeks, and if you never come back, a lifetime payment is a lot for six weeks of reading. That's a scheduling question, not a product question, and only you can answer it.

## Quick answers to the follow-ups people search for

**Does the hashmap solution work with negative numbers?** Yes. That's precisely why it's the intended solution. The prefix-sum difference formula holds for any integers.

**Why initialize the map with `{0: 1}` rather than `{0: 0}` or nothing?** It represents the empty prefix before the first element. A subarray starting at index 0 has `prefix[i] = 0`, so you need that count to be present for the lookup to find it.

**What are the complexities?** `O(n)` time and `O(n)` space. The map holds at most `n + 1` distinct prefix sums.

**Is there an `O(1)` space solution?** Not for the general case with negatives. Constant-space sliding window works only when every element is non-negative, which this problem's constraints explicitly rule out.

**Can I use a sliding window anyway if I check for negatives first?** You can, and it's a reasonable hybrid: if every element is non-negative, a sliding window gives `O(n)` time and `O(1)` space. Just don't ship that as your default answer, since the general solution is shorter and always correct.

## The short version

Subarray Sum Equals K is a prefix-sum problem wearing a subarray costume. Track the running total, store how many times each total has occurred, and at every index add `counts[total - k]` to your answer. Seed the map with `{0: 1}`, look up before you insert, and use a frequency map rather than a set or an index map.

Everything else — the sliding window dead end, the `O(n²)` fallback, the sibling problems — is context that makes the trick stick. Worth twenty minutes of reading before you grind forty more variants.
