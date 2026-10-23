# Java Level 11 — DSA & Problem Solving

## Part 1 — Core Patterns

### Status

**Level 11: In Progress**

Covered in this part:

- ✅ Big-O Complexity
- ✅ Arrays & Hashing
- ✅ Two Pointers
- ✅ Sliding Window
- ✅ Prefix Sum
- ✅ Stack
- ✅ Monotonic Stack
- ✅ Queue / Deque Basics
- 🟡 Monotonic Deque / Sliding Window Maximum — Revisit

---

# 1. Big-O Complexity

Big-O describes how an algorithm's resource usage grows as the input size increases.

For interviews, focus mainly on:

- Time Complexity
- Space Complexity

## Common Complexities

| Complexity | Typical Example |
|---|---|
| `O(1)` | HashMap lookup |
| `O(log n)` | Binary Search |
| `O(n)` | Single traversal |
| `O(n log n)` | Efficient sorting |
| `O(n²)` | Nested loops |
| `O(2^n)` | Some recursive/backtracking solutions |

Ignore constants:

```text
O(2n)      → O(n)
O(5n + 10) → O(n)
O(n² + n)  → O(n²)
```

## Interview Habit

Always be ready to explain:

```text
Time Complexity
Space Complexity
```

---

# 2. Arrays + Hashing

HashMap and HashSet are commonly used to replace repeated searching with approximately `O(1)` lookup.

## Recognition Cues

```text
Need fast lookup?
→ HashMap / HashSet

"Have I seen X?"
→ HashSet

"Where did I see X?"
→ HashMap<Value, Index>

"How many X?"
→ HashMap<Value, Count>
```

---

# 3. Two Sum — HashMap Pattern

## Problem

Given:

```text
nums = [2,7,11,15]
target = 9
```

Return indices:

```text
[0,1]
```

because:

```text
2 + 7 = 9
```

## Key Idea

For every number:

```text
complement = target - current
```

Ask:

```text
Have I already seen complement?
```

## Implementation

```java
public static int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];

        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }

        map.put(nums[i], i);
    }

    return new int[]{-1, -1};
}
```

Complexity:

```text
Time  → O(n)
Space → O(n)
```

### Important

Check before inserting the current value.

This correctly handles:

```text
nums = [3,3]
target = 6
```

---

# 4. Valid Anagram — Frequency Map

## Problem

```text
"anagram"
"nagaram"

→ true
```

## Pattern

Count characters from the first string.

Then decrement/remove them while processing the second.

```java
public static boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) {
        return false;
    }

    Map<Character, Integer> map = new HashMap<>();

    for (char c : s.toCharArray()) {
        map.put(c, map.getOrDefault(c, 0) + 1);
    }

    for (char c : t.toCharArray()) {
        if (!map.containsKey(c)) {
            return false;
        }

        if (map.get(c) > 1) {
            map.put(c, map.get(c) - 1);
        } else {
            map.remove(c);
        }
    }

    return true;
}
```

Complexity:

```text
Time  → O(n)
Space → O(k)
```

where `k` is the number of distinct characters.

---

# 5. Two Pointers

Two Pointers uses two indices that move according to some condition.

Common form:

```text
left →             ← right
```

## Recognition Cues

Think Two Pointers when you see:

- Sorted array
- Pair with target
- Palindrome
- Compare opposite ends
- Reverse
- Remove duplicates
- Optimization involving two boundaries

---

# 6. Valid Palindrome

```java
public static boolean isPalindrome(String s) {
    int left = 0;
    int right = s.length() - 1;

    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }

        left++;
        right--;
    }

    return true;
}
```

Complexity:

```text
Time  → O(n)
Space → O(1)
```

---

# 7. Two Sum II — Sorted Array

Because the array is sorted:

```text
sum < target
→ need bigger number
→ left++

sum > target
→ need smaller number
→ right--
```

## Implementation

```java
public static int[] twoSumSorted(int[] nums, int target) {
    int left = 0;
    int right = nums.length - 1;

    while (left < right) {
        int sum = nums[left] + nums[right];

        if (sum == target) {
            return new int[]{left, right};
        }

        if (sum < target) {
            left++;
        } else {
            right--;
        }
    }

    return new int[]{-1, -1};
}
```

Complexity:

```text
Time  → O(n)
Space → O(1)
```

---

# 8. Container With Most Water

Area:

```text
width  = right - left
height = min(height[left], height[right])

area = width × height
```

## Critical Rule

Always move the **shorter wall**.

Why?

The shorter wall limits the current area.

Moving the taller wall decreases width without giving us a chance to increase the limiting height.

```java
public static int maxArea(int[] height) {
    int left = 0;
    int right = height.length - 1;

    int maxArea = 0;

    while (left < right) {
        int width = right - left;

        int h = Math.min(
                height[left],
                height[right]
        );

        maxArea = Math.max(
                maxArea,
                width * h
        );

        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }

    return maxArea;
}
```

Complexity:

```text
Time  → O(n)
Space → O(1)
```

---

# 9. Sliding Window

Sliding Window is useful for **contiguous** sections of arrays or strings.

Recognition:

```text
Array/String
+
Subarray/Substring/Contiguous
+
Condition/calculation
```

Then ask:

> Can I reuse information from the previous window instead of recalculating everything?

There are two major types:

```text
Fixed Sliding Window
Variable Sliding Window
```

---

# 10. Fixed Sliding Window

Given:

```text
nums = [2,1,5,1,3,2]
k = 3
```

Windows:

```text
[2,1,5] → 8
[1,5,1] → 7
[5,1,3] → 9
[1,3,2] → 6
```

Instead of recalculating every window:

```text
newSum =
oldSum
- leaving element
+ entering element
```

## Maximum Sum Subarray of Size K

```java
public static int maxSumSubarray(int[] nums, int k) {
    int windowSum = 0;

    for (int i = 0; i < k; i++) {
        windowSum += nums[i];
    }

    int maxSum = windowSum;

    for (int end = k; end < nums.length; end++) {
        windowSum += nums[end];
        windowSum -= nums[end - k];

        maxSum = Math.max(maxSum, windowSum);
    }

    return maxSum;
}
```

Remember:

```text
Build first window

Then repeatedly:

+ entering
- leaving

Update answer
```

Complexity:

```text
Time  → O(n)
Space → O(1)
```

---

# 11. Maximum Average Subarray

Same fixed-window pattern.

```java
public static double maxAverage(int[] nums, int k) {
    int currentWindowSum = 0;

    for (int i = 0; i < k; i++) {
        currentWindowSum += nums[i];
    }

    int maxWindowSum = currentWindowSum;

    for (int end = k; end < nums.length; end++) {
        currentWindowSum += nums[end];
        currentWindowSum -= nums[end - k];

        maxWindowSum = Math.max(
                maxWindowSum,
                currentWindowSum
        );
    }

    return (double) maxWindowSum / k;
}
```

---

# 12. Variable Sliding Window

Variable window does not have a predetermined size.

Mental model:

```text
EXPAND RIGHT
     ↓
Check condition
     ↓
SHRINK LEFT when necessary
     ↓
Update answer
     ↓
Continue
```

General template:

```java
int left = 0;

for (int right = 0; right < n; right++) {

    // Add right to window

    while (windowNeedsShrinking()) {
        // Remove left
        left++;
    }

    // Update answer
}
```

---

# 13. Longest Substring Without Repeating Characters

Problem:

```text
"abcabcbb"

→ "abc"
→ 3
```

Use:

```text
Sliding Window + HashSet
```

The HashSet represents characters currently inside the window.

```java
public static int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();

    int left = 0;
    int maxLength = 0;

    for (int right = 0; right < s.length(); right++) {

        while (window.contains(s.charAt(right))) {
            window.remove(s.charAt(left));
            left++;
        }

        window.add(s.charAt(right));

        maxLength = Math.max(
                maxLength,
                right - left + 1
        );
    }

    return maxLength;
}
```

### Important Pitfall

When shrinking:

```java
window.remove(s.charAt(left));
```

NOT:

```java
window.remove(s.charAt(right));
```

The left pointer represents the element actually leaving the window.

### Why `while`?

One new character may require removing multiple characters before the window becomes valid again.

---

# 14. Minimum Size Subarray Sum

Given positive integers:

```text
nums = [2,3,1,2,4,3]
target = 7
```

Smallest qualifying subarray:

```text
[4,3]

length = 2
```

## Pattern

Expand using `right`.

Once:

```text
sum >= target
```

the window is valid.

Because we're looking for the **minimum**, keep shrinking while it remains valid.

```java
public static int minSubArrayLen(int target, int[] nums) {
    int left = 0;
    int sum = 0;

    int minLength = Integer.MAX_VALUE;

    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];

        while (sum >= target) {
            minLength = Math.min(
                    minLength,
                    right - left + 1
            );

            sum -= nums[left];
            left++;
        }
    }

    return minLength == Integer.MAX_VALUE
            ? 0
            : minLength;
}
```

## Important Distinction

### Longest valid window

```text
Expand

Invalid?
→ shrink until valid

Update MAX
```

### Smallest valid window

```text
Expand

Valid?
→ update MIN
→ shrink while still valid
```

This distinction is very important.

### Why positive integers?

Adding on the right only increases the sum.

Removing from the left only decreases it.

Negative numbers destroy this monotonic behavior, so this simple sliding-window approach does not generally work.

---

# 15. Prefix Sum

Prefix Sum stores cumulative sums so range sums can be calculated efficiently.

Given:

```text
nums = [2,4,1,7,3]
```

Prefer an extra leading zero:

```text
prefix = [0,2,6,7,14,17]
```

Build:

```java
public static int[] buildPrefixSum(int[] nums) {
    int[] prefix = new int[nums.length + 1];

    for (int i = 0; i < nums.length; i++) {
        prefix[i + 1] =
                prefix[i] + nums[i];
    }

    return prefix;
}
```

Then:

```text
sum(left...right)

= prefix[right + 1] - prefix[left]
```

Example:

```text
left = 1
right = 3

14 - 2 = 12
```

## Complexity

Build:

```text
O(n)
```

Each range query:

```text
O(1)
```

If there are many queries, build the prefix array once rather than rebuilding it for every query.

---

# 16. Prefix Sum + HashMap

Very important interview combination.

## Problem — Subarray Sum Equals K

```text
nums = [1,2,3]
k = 3

answer = 2
```

Subarrays:

```text
[1,2]
[3]
```

## Core Equation

Suppose:

```text
currentPrefix - previousPrefix = k
```

Rearrange:

```text
previousPrefix = currentPrefix - k
```

Therefore, at every position ask:

```text
Have I previously seen:

currentPrefix - k ?
```

Use:

```text
HashMap<PrefixSum, Frequency>
```

## Implementation

```java
public static int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount =
            new HashMap<>();

    prefixCount.put(0, 1);

    int prefixSum = 0;
    int count = 0;

    for (int num : nums) {
        prefixSum += num;

        count += prefixCount.getOrDefault(
                prefixSum - k,
                0
        );

        prefixCount.put(
                prefixSum,
                prefixCount.getOrDefault(
                        prefixSum,
                        0
                ) + 1
        );
    }

    return count;
}
```

Complexity:

```text
Time  → O(n)
Space → O(n)
```

## Why `{0 → 1}` initially?

It represents the empty prefix.

This allows us to detect subarrays starting from index `0`.

For example:

```text
prefixSum = k

prefixSum - k = 0
```

## Important Pitfall

Do NOT simply:

```java
count++;
```

when a previous prefix exists.

The same prefix sum may have appeared multiple times.

Instead:

```java
count += prefixCount.get(prefixSum - k);
```

Every previous occurrence creates another valid subarray.

---

# 17. Stack

Stack follows:

```text
LIFO

Last In
First Out
```

In Java prefer:

```java
Deque<Integer> stack =
        new ArrayDeque<>();
```

Operations:

```java
stack.push(x);
stack.pop();
stack.peek();
stack.isEmpty();
```

Think Stack when dealing with:

- Matching parentheses
- Nested structures
- Undo/history
- Expression evaluation
- Most recent unresolved item
- Next/previous greater/smaller

---

# 18. Valid Parentheses

Problem:

```text
"{[]}" → true
"([)]" → false
```

Opening brackets go onto the stack.

Closing brackets must match the most recent opening bracket.

```java
public static boolean isValid(String s) {
    Deque<Character> stack =
            new ArrayDeque<>();

    for (char c : s.toCharArray()) {

        if (c == ')' || c == ']' || c == '}') {

            if (stack.isEmpty()) {
                return false;
            }

            char open = stack.pop();

            if ((c == ')' && open != '(') ||
                (c == ']' && open != '[') ||
                (c == '}' && open != '{')) {

                return false;
            }

        } else {
            stack.push(c);
        }
    }

    return stack.isEmpty();
}
```

Complexity:

```text
Time  → O(n)
Space → O(n)
```

### Important Edge Case

Always check:

```java
stack.isEmpty()
```

before `pop()`.

Otherwise:

```text
"]"
```

throws an exception.

---

# 19. Monotonic Stack

A Monotonic Stack maintains values in an intentional increasing or decreasing order.

Used for:

- Next Greater Element
- Next Smaller Element
- Previous Greater Element
- Previous Smaller Element
- Daily Temperatures
- Stock Span
- Histogram problems

Mental model:

```text
Stack = elements/indices still waiting for an answer
```

---

# 20. Next Greater Element

Given:

```text
nums = [2,1,4,3]
```

Result:

```text
[4,4,-1,-1]
```

Meaning:

```text
2 → 4
1 → 4
4 → none
3 → none
```

Store **indices**, not values.

```java
public static int[] nextGreaterElement(int[] nums) {
    int[] answer = new int[nums.length];

    Arrays.fill(answer, -1);

    Deque<Integer> stack =
            new ArrayDeque<>();

    for (int i = 0; i < nums.length; i++) {

        while (!stack.isEmpty()
                && nums[i] > nums[stack.peek()]) {

            int previousIndex = stack.pop();

            answer[previousIndex] = nums[i];
        }

        stack.push(i);
    }

    return answer;
}
```

Mental model:

```text
Current element arrives

Is it greater than something waiting
on the stack?

YES
→ current element is its answer
→ pop it

Repeat

Then current index starts waiting
```

## Why O(n)?

Although there's a `while` inside the `for`:

```text
each index pushed once
each index popped at most once
```

Therefore:

```text
O(n)
```

not `O(n²)`.

---

# 21. Next Smaller Element

Almost identical.

Only change:

```text
current > stackTop
```

to:

```text
current < stackTop
```

```java
public static int[] nextSmallerElement(int[] nums) {
    int[] answer = new int[nums.length];

    Arrays.fill(answer, -1);

    Deque<Integer> stack =
            new ArrayDeque<>();

    for (int i = 0; i < nums.length; i++) {

        while (!stack.isEmpty()
                && nums[i] < nums[stack.peek()]) {

            int previousIndex = stack.pop();

            answer[previousIndex] = nums[i];
        }

        stack.push(i);
    }

    return answer;
}
```

Complexity:

```text
Time  → O(n)
Space → O(n)
```

---

# 22. Queue

Queue follows:

```text
FIFO

First In
First Out
```

Java:

```java
Queue<Integer> queue =
        new ArrayDeque<>();
```

Operations:

```java
queue.offer(10);
queue.poll();
queue.peek();
```

Queues become particularly important later for:

```text
BFS
Level-order traversal
Graph traversal
Task processing
```

---

# 23. Deque

Deque means:

```text
Double Ended Queue
```

We can operate from both ends.

```java
Deque<Integer> deque =
        new ArrayDeque<>();
```

Operations:

```java
deque.offerFirst(x);
deque.offerLast(x);

deque.pollFirst();
deque.pollLast();

deque.peekFirst();
deque.peekLast();
```

---

# 24. Monotonic Deque — Sliding Window Maximum

## Status: 🟡 Revisit

This pattern should be revisited during revision.

Problem:

```text
nums = [1,3,-1,-3,5,3,6,7]
k = 3
```

Windows:

```text
[1,3,-1]   → 3
[3,-1,-3]  → 3
[-1,-3,5]  → 5
[-3,5,3]   → 5
[5,3,6]    → 6
[3,6,7]    → 7
```

Result:

```text
[3,3,5,5,6,7]
```

## Core Idea

Deque stores **indices**.

Corresponding values remain in:

```text
decreasing order
```

Therefore:

```java
nums[deque.peekFirst()]
```

is always the maximum.

## Front vs Back

This distinction is the key:

```text
FRONT
→ current maximum
→ remove expired indices

BACK
→ remove smaller useless values
→ add current index
```

## Why remove smaller values?

Suppose:

```text
1 is waiting
```

and later:

```text
3 arrives
```

`1` can never become maximum while `3` remains in the same/future relevant window because `3` is:

```text
larger
AND
newer
```

So `1` is useless and can be discarded.

## Implementation

```java
public static int[] maxSlidingWindow(
        int[] nums,
        int k
) {
    Deque<Integer> deque =
            new ArrayDeque<>();

    int[] result =
            new int[nums.length - k + 1];

    for (int i = 0; i < nums.length; i++) {

        // Remove expired index
        if (!deque.isEmpty()
                && deque.peekFirst() <= i - k) {

            deque.pollFirst();
        }

        // Maintain decreasing values
        while (!deque.isEmpty()
                && nums[i] > nums[deque.peekLast()]) {

            deque.pollLast();
        }

        // Store current index
        deque.offerLast(i);

        // Record maximum once window is complete
        if (i >= k - 1) {
            result[i - k + 1] =
                    nums[deque.peekFirst()];
        }
    }

    return result;
}
```

Complexity:

```text
Time  → O(n)
Space → O(k)
```

## Revision Mental Model

Don't try to memorize the implementation initially.

Remember:

```text
For each new element:

1. FRONT
   Remove expired index

2. BACK
   Remove smaller values

3. BACK
   Add current index

4. FRONT
   Read maximum
```

This topic should be traced manually again before interviews.

---

# 25. Pattern Recognition Cheat Sheet

## HashMap / HashSet

```text
"Have I seen X?"
→ HashSet

"Where did I see X?"
→ HashMap<Value, Index>

"How many X?"
→ HashMap<Value, Frequency>
```

---

## Two Pointers

```text
Sorted array
Pair target
Palindrome
Compare ends
Boundary optimization
```

Think:

```text
left ↔ right
```

---

## Sliding Window

```text
Contiguous
Substring
Subarray
Longest/shortest
Fixed window size
```

Think:

```text
right expands
left shrinks
```

---

## Prefix Sum

```text
Range sum
Cumulative sum
Repeated subarray sums
```

Think:

```text
prefix[right + 1] - prefix[left]
```

---

## Prefix Sum + HashMap

```text
Number of subarrays
whose sum == k
```

Think:

```text
previousPrefix
=
currentPrefix - k
```

---

## Stack

```text
Nested structures
Matching
Most recent unresolved item
```

Think:

```text
LIFO
```

---

## Monotonic Stack

```text
Next greater
Next smaller
Previous greater
Previous smaller
```

Think:

```text
"Indices waiting for an answer"
```

---

## Queue

```text
FIFO
BFS
Level-by-level processing
```

---

## Monotonic Deque

```text
Sliding window maximum/minimum
```

Think:

```text
Front = answer

Back = maintain monotonic order
```

---

# 26. Common Mistakes From Practice

These are worth revising because they represent common interview bugs.

### Minimum calculation

Wrong:

```java
int min = Integer.MIN_VALUE;
min = Math.min(min, value);
```

Correct:

```java
int min = Integer.MAX_VALUE;
```

---

### Variable Sliding Window

When shrinking:

```java
remove(nums[left]);
left++;
```

Do not remove the current `right` element unless the algorithm specifically requires it.

---

### Prefix Sum

This representation:

```text
prefix size = n + 1
```

is usually cleaner because it eliminates the `left == 0` special case.

---

### Prefix Sum + HashMap

Wrong:

```java
if (map.containsKey(target)) {
    count++;
}
```

Correct:

```java
count += map.getOrDefault(target, 0);
```

Frequency matters.

---

### Stack

Never blindly:

```java
stack.pop();
```

when the stack may be empty.

Check:

```java
stack.isEmpty()
```

first.

---

### Monotonic Stack / Deque

Usually store:

```text
indices
```

rather than values.

Indices let you:

- update the correct answer position
- determine whether an element expired
- still access its value through `nums[index]`

---

# 27. Complexity Cheat Sheet

| Pattern | Typical Time | Typical Space |
|---|---:|---:|
| Hashing | `O(n)` | `O(n)` |
| Two Pointers | `O(n)` | `O(1)` |
| Fixed Sliding Window | `O(n)` | `O(1)` |
| Variable Sliding Window | `O(n)` | Depends |
| Prefix Sum Build | `O(n)` | `O(n)` |
| Prefix Range Query | `O(1)` | — |
| Prefix + HashMap | `O(n)` | `O(n)` |
| Stack | `O(n)` | `O(n)` |
| Monotonic Stack | `O(n)` | `O(n)` |
| Monotonic Deque | `O(n)` | `O(k)` |

---

# 28. Revision Checklist

Before moving on, you should eventually be able to recognize these without being told the pattern.

### Arrays / Hashing

- [x] Two Sum
- [x] Valid Anagram

### Two Pointers

- [x] Valid Palindrome
- [x] Two Sum II
- [x] Container With Most Water

### Sliding Window

- [x] Maximum Sum Subarray of Size K
- [x] Maximum Average Subarray
- [x] Longest Substring Without Repeating Characters
- [x] Minimum Size Subarray Sum

### Prefix Sum

- [x] Range Sum
- [x] Subarray Sum Equals K

### Stack

- [x] Valid Parentheses
- [x] Next Greater Element
- [x] Next Smaller Element

### Queue / Deque

- [x] Queue fundamentals
- [x] Deque fundamentals
- [ ] Sliding Window Maximum — revisit

---

# 29. Level 11 Progress

```text
✅ Big-O

✅ Arrays & Hashing

✅ Two Pointers

✅ Sliding Window

✅ Prefix Sum

✅ Stack

✅ Monotonic Stack

✅ Queue / Deque Basics

🟡 Monotonic Deque — revisit

⬜ Binary Search

⬜ Linked List

⬜ Trees / BST

⬜ Heap / Priority Queue

⬜ Graphs

⬜ BFS / DFS

⬜ Topological Sort

⬜ Union Find

⬜ Recursion

⬜ Backtracking

⬜ Greedy

⬜ Dynamic Programming

⬜ Intervals
```

---

# Final Mental Model

Don't memorize individual solutions.

Train yourself to map problem language to patterns:

```text
Need lookup?
→ Hashing

Sorted + pair/boundaries?
→ Two Pointers

Contiguous + longest/shortest?
→ Sliding Window

Repeated range/subarray sums?
→ Prefix Sum

Most recent unresolved item?
→ Stack

Next greater/smaller?
→ Monotonic Stack

FIFO / level-by-level?
→ Queue

Window maximum/minimum?
→ Monotonic Deque
```

That pattern recognition is the main goal of Level 11.