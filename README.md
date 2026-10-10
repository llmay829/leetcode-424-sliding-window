# longest repeating character replacement: Solve LeetCode 424 With One Sliding Window Pass (and the maxFreq Trick That Trips People Up)

You find the problem, read it twice, and it sounds simple enough. Replace up to `k` characters, return the longest run of identical letters you can produce. Then you try to write the loop and discover you can't just slide a window forward and call it done, because sometimes the answer sits on the left edge of your window, sometimes on the right, and sometimes the letter you should be matching isn't the one you counted first.

LeetCode 424 is rated Medium and its acceptance rate sits around 55%, which tells you something: roughly half of submissions have a bug in the validity check, the pointer movement, or the `maxFreq` bookkeeping. The good news is that all three problems have the same fix, and it's one line of arithmetic.

## What the problem actually asks

Given a string `s` of uppercase English letters and an integer `k`, you may change any character to any other uppercase letter, at most `k` times total. Return the length of the longest substring that contains a single repeated letter after those changes.

Two examples from the problem statement:

- `s = "ABAB", k = 2` → `4`. Flip both A's to B's, or both B's to A's.
- `s = "AABABBA", k = 1` → `4`. Change the single A at index 3 to B, giving `"AABBBBA"`, and take the substring `"BBBB"`.

Constraints worth noting: `1 <= s.length <= 10^5`, and `0 <= k <= s.length`.

That upper bound of 100,000 characters is the whole reason this problem is interesting. A solution that checks every substring dies. You need something linear.

## Why the obvious approach fails

The brute force version is easy to write and easy to understand. For every starting index, extend the substring one character at a time, keep a count of each letter inside that substring, and ask: is `(substring length - count of most frequent letter) <= k`?

That difference is the number of characters you'd have to replace to make everything in the substring identical. Pick the most common letter, convert everything else into it. If the cost is within `k`, the substring is valid.

The logic is correct. The runtime is O(n²), and at n = 100,000 that's 10 billion operations. Not happening.

There's a second, subtler trap that catches people who try to fix brute force by picking a target letter up front. Suppose you decide "the answer is all A's" and run a window over A's only. On `"AABABBA"` with k=1 that works fine. But consider `"ABBBBBA"`: the best answer is the middle B-run, and a window that starts anchored on A never finds it. You'd need to run the whole scan once per letter in the alphabet. That's O(26n) and it does pass, but it's more code than you need and it's not the solution your interviewer is fishing for.

## The one insight that makes this linear

Here's the entire idea: a window is valid when


window length - (count of the window's most frequent character) <= k


Notice what's missing. There's no "target letter" anywhere in that condition. You never commit to making the window all A's or all B's. You just track the most common character inside the current window and ask whether the leftovers fit inside your replacement budget.

This works because the answer is monotonic in a useful way. If a window of size 5 is valid, every smaller window contained inside it is also valid. And if a window is invalid, extending it further to the right can never make it valid again, since adding characters only increases the count of non-majority letters. That monotonicity is exactly the property sliding window needs.

## The template, step by step

1. Keep a frequency array for the 26 uppercase letters.
2. Move `right` across the string, incrementing the count for `s[right]`.
3. Update `max_freq` if this character just became the most frequent one seen so far.
4. If `(right - left + 1) - max_freq > k`, the window is too expensive. Shrink from the left: decrement `count[s[left]]` and advance `left`.
5. Record the best window size you've seen.

python
def characterReplacement(s: str, k: int) -> int:
    count = {}
    left = 0
    max_freq = 0
    best = 0

    for right, ch in enumerate(s):
        count[ch] = count.get(ch, 0) + 1
        max_freq = max(max_freq, count[ch])

        while (right - left + 1) - max_freq > k:
            count[s[left]] -= 1
            left += 1

        best = max(best, right - left + 1)

    return best


Same thing in Java, with an int array instead of a map:

java
class Solution {
    public int characterReplacement(String s, int k) {
        int[] count = new int[26];
        int left = 0, maxFreq = 0, best = 0;

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            maxFreq = Math.max(maxFreq, ++count[c - 'A']);

            while (right - left + 1 - maxFreq > k) {
                count[s.charAt(left++) - 'A']--;
            }
            best = Math.max(best, right - left + 1);
        }
        return best;
    }
}


If you're practicing under time pressure, that template transfers almost verbatim to Max Consecutive Ones III and a handful of other window problems. Recognizing which problem wants this template is the skill worth drilling. 👉 [Practice the sliding window pattern with structured lessons and diagrams](https://algo.monster/subscribe?fpr=li69)

## Tracing it on "AABABBA" with k = 1

Line-by-line traces are usually where these explanations get useful, so here's the actual state of every variable at each step.

| right | char | counts (A, B) | max_freq | window size | cost | action | best |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | A | 1, 0 | 1 | 1 | 0 | valid | 1 |
| 1 | A | 2, 0 | 2 | 2 | 0 | valid | 2 |
| 2 | B | 2, 1 | 2 | 3 | 1 | valid | 3 |
| 3 | A | 3, 1 | 3 | 4 | 1 | valid | 4 |
| 4 | B | 3, 2 | 3 | 5 | 2 | shrink left (drops index 0) | 4 |
| 5 | B | 2, 3 | 3 | 5 | 2 | shrink left (drops index 1) | 4 |
| 6 | A | 2, 3 | 3 | 5 | 2 | shrink left (drops index 2) | 4 |

Final answer: 4. At index 4 the window `"AABAB"` would need two replacements, so the left edge moves forward and the window stops growing. From that point on it just walks the same length across the string. That walking behavior is the reason the `while` loop can be replaced with an `if`.

## The maxFreq question everyone gets wrong

This is the part of the problem that generates the most confused comment threads, and it deserves a straight answer.

When you shrink the window from the left, `count[s[left]]` goes down. But `max_freq` does not. It only ever increases, which means after a shrink it can be *stale* — it may report a frequency higher than the actual maximum inside the current window. That makes the validity check more permissive than it should be for a window that isn't really valid.

So why does the answer stay correct?

Because the window only ever grows by one and shrinks by one. Any window the algorithm reports as its best was genuinely valid at the moment it was measured, back when `max_freq` was accurate for that window. A stale `max_freq` can make the current window look valid when it isn't, but it can never produce a length larger than one that was already achievable. The recorded maximum is a lower bound that happens to be exact.

The practical upshot: don't recalculate `max_freq` by scanning all 26 counts after every shrink. Doing that is still O(26n) so it passes, but it adds a loop and slows your code down for no reason. Leave `max_freq` alone.

The flip side matters too. Set `max_freq = count[ch]` instead of `max(max_freq, count[ch])` and you break the algorithm on inputs like `"ABAB"` with k=2, because the maximum frequency collapses the moment the window moves past the majority character.

A second clean formulation, if you prefer your variables obvious:

python
def characterReplacement(s: str, k: int) -> int:
    count = [0] * 26
    left = 0
    max_freq = 0
    for right, ch in enumerate(s):
        idx = ord(ch) - ord('A')
        count[idx] += 1
        max_freq = max(max_freq, count[idx])
        if (right - left + 1) - max_freq > k:
            count[ord(s[left]) - 'A'] -= 1
            left += 1
    return len(s) - left


Here the window size never decreases, and the answer falls out of where `left` ends up: `len(s) - left`. It's the same algorithm with the bookkeeping folded into the return statement. Some people find it cleaner. Other people stare at `len(s) - left` for a minute and reach for the `while` version. Both are fine.

## Edge cases that break sloppy implementations

- `k = 0`: the answer is the longest run of identical characters in the original string. Your validity check has to handle `cost <= k`, not `< k`, or you'll shorten valid windows.
- `k >= s.length`: the whole string is replaceable, so return `s.length` without doing anything clever.
- Single character string: `"A"` with any `k` returns 1. A loop that assumes at least two characters will index out of bounds.
- Shrinking without decrementing: if you advance `left` before subtracting `count[s[left]]`, your counts drift upward and the window keeps validating itself. This is the single most common bug in submitted solutions.
- Using a `HashMap` where an array of 26 is expected: it works, and interviewers generally accept it, but `int[26]` signals that you noticed the fixed alphabet.

## Complexity

Time is O(n). The `right` pointer visits each character once, and `left` only moves forward, so across the entire run it advances at most n times total. No character re-enters the window.

Space is O(1). The frequency structure holds at most 26 entries, no matter how long the string is.

## How it sits next to similar problems

424 rarely appears alone in a study plan, because the pattern it teaches shows up under a dozen costumes.

| Problem | What changes | Same core idea? |
| --- | --- | --- |
| Max Consecutive Ones III (1004) | Flip at most k zeros | Yes, identical structure |
| Longest Subarray of 1's After Deleting One Element (1493) | Exactly one deletion | Close, one deleted element |
| Longest Substring with At Most K Distinct (340) | Constraint is distinct count | Same window, different validity test |
| Find the Longest Equal Subarray (2831) | Answer can be non-contiguous | Needs the index-shifting trick |
| Minimum Window Substring (76) | Shrink until valid instead of expand | Inverted direction, same machinery |

If 424 felt hard on the first pass, the usual cause isn't the sliding window mechanics. It's that you hadn't seen the "length minus majority count" formulation before. It's a reusable trick, not a one-off.

## Where to practice it without wasting a week

Most people who struggle with 424 don't struggle with the code. They struggle with pattern selection: reading a problem, deciding within sixty seconds that it's a sliding window problem, and then picking the right validity condition. Grinding random problems builds volume but doesn't build that recognition.

AlgoMonster organizes its curriculum around exactly that step. Sliding window sits in as one of its core patterns, with the abstract template, worked examples, and a decision flowchart for choosing a pattern when a new problem shows up. The content is text-and-diagram based rather than video, written by engineers from Google and competitive programming backgrounds. The 424 walkthrough is one of the editorials in the free library, so you can read their version of this problem at no cost before deciding anything: 👉 [Read the free Longest Repeating Character Replacement explanation](https://algo.monster/liteproblems/424?fpr=li69)

Worth knowing before you pay: the platform is compact by design. A few hundred lessons and roughly 230 worked problems, compared to LeetCode's several thousand. It also doesn't cover video explanations, live coaching, or deep system design prep. If your gap is "I know what to study, I just need volume," LeetCode does that job. If your gap is "I don't know why I keep failing medium problems," the pattern framework is the more direct fix.

### Current plans

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and editorials, enough to test the format | $0 | — | [Start with the free content](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant and Speedrun; real interview questions updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Recurring annually | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, paid once | $189 | One-time | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

A couple of honest notes on those numbers. AlgoMonster runs sales regularly, and the listed prices move with them, so the figure you see on the subscribe page is the one that counts. Third-party reviews quote a wide range, from around $89/year up into the hundreds for lifetime, which is a good sign that promotional pricing fluctuates rather than that anyone is lying. The subscribe page has also displayed a monthly option at around $18.99/month at times. If you only need a few weeks of prep, check whether it's live before committing to an annual plan.

For most candidates with interviews in the next two to three months, the yearly plan is the practical choice. Lifetime makes sense if you expect to interview repeatedly over several years, or if you want to keep re-reading the templates between job searches.

## Questions people ask about LeetCode 424

**Does the answer ever require replacing characters outside the window?**

No. Replacements are hypothetical and confined to the substring you're evaluating. The characters you "change" don't affect the rest of the string, which is why the problem reduces to counting, not simulation.

**What if two characters are tied for the highest frequency in a window?**

It doesn't matter. Pick either one as the majority and replace the rest. Same cost, same window length.

**Is the window always exactly one character wide at the start?**

It starts at size 1 after processing the first character. If `k = 0`, the window grows only across runs of identical characters, which is the correct behavior.

**Why does my solution pass 40 test cases and then fail on `"ABAB"` with `k = 2`?**

Nine times out of ten it's `max_freq` being overwritten instead of compared, or the shrink step forgetting to decrement the outgoing character's count. Both bugs produce correct behavior on strings where the majority letter never changes and wrong behavior the moment it does.

## What to do next

Write the 40-line solution from scratch without looking at anything, then run it against `"ABAB"` with `k=2`, `"AABABBA"` with `k=1`, `"AABABBA"` with `k=0`, and a string of 100,000 identical characters. If all four pass, you understand the problem. If the `k=0` case fails, go back and check whether your validity condition uses `<=` or `<`.

Then spend the rest of your study session on the pattern rather than the problem. The value of 424 isn't the 40 lines of Python. It's learning to look at a string problem, spot that the constraint is monotonic, and reach for a window without hesitating. That reflex is what interviewers are actually testing, and it's the part that transfers. 👉 [See AlgoMonster's pattern-based curriculum and current pricing](https://algo.monster/subscribe?fpr=li69)
