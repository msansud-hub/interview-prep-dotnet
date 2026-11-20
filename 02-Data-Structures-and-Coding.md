# Data structures, algorithms and hands-on coding

Explain your assumptions, inputs, output and edge cases before coding. Run through an example, state complexity, and discuss trade-offs. Examples below use C#; snippets are independent methods, not a complete application.

## Data structure questions

### 1. Array versus linked list?

**Answer:** An array supports O(1) indexing and good locality, but resizing usually requires copying. A linked list supports O(1) insertion/removal **when the node is already known**, but finding an item or index is O(n), and nodes add memory overhead.

**Example:**

```csharp
int[] items = { 10, 20 };
int second = items[1]; // 20

var list = new LinkedList<int>();
var node = list.AddLast(10);
list.AddAfter(node, 20); // Insert after the known node
```

### 2. List versus dictionary versus hash set?

**Answer:** `List<T>` is ordered and has O(1) indexed access; searching is O(n). `Dictionary<TKey,TValue>` maps keys to values; `HashSet<T>` stores unique values. Both normally offer average O(1) lookup, with worst-case O(n), and require consistent equality and hash codes.

**Example:**

```csharp
var names = new List<string> { "Ana" };
var scores = new Dictionary<string, int> { ["Ana"] = 90 };
var unique = new HashSet<int> { 1, 1 };

Console.WriteLine(unique.Count); // 1: duplicates are ignored
```

### 3. Stack versus queue?

**Answer:** A stack is last-in-first-out, useful for undo or depth-first traversal. A queue is first-in-first-out, useful for breadth-first traversal or work scheduling. Thread safety and durability are separate concerns; an in-memory queue is not a message broker.

**Example:**

```csharp
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
Console.WriteLine(stack.Pop()); // 2: last in, first out

var queue = new Queue<int>();
queue.Enqueue(1);
queue.Enqueue(2);
Console.WriteLine(queue.Dequeue()); // 1: first in, first out
```

### 4. Binary search requirements and complexity?

**Answer:** Data must be sorted according to the same comparison used by the search. Repeatedly halve the search range: O(log n) time and O(1) extra space for an iterative implementation.

**Example:**

```csharp
int[] sorted = { 1, 3, 5, 7 };
int index = Array.BinarySearch(sorted, 5); // 2
```

Binary search requires a sorted array.

### 5. BFS versus DFS?

**Answer:** Breadth-first search uses a queue and finds the shortest path by edge count in an unweighted graph. Depth-first search uses a stack or recursion and is useful for traversal, cycle analysis, and exploring components. With adjacency lists, both take O(V + E) time and O(V) auxiliary space.

**Example:** For edges `A -> B`, `A -> C`, `B -> D`, BFS visits `A, B, C, D`; DFS can visit `A, B, D, C`. Neighbor order affects the exact traversal order.

### 6. Explain Big O.

**Answer:** Big O describes an asymptotic upper bound as input grows, not exact runtime. For example, one scan is O(n), two nested scans are often O(n²), and a balanced binary search is O(log n). State whether space excludes the input or output. Expected hash-table lookup is different from worst-case lookup.

**Example:**

```csharp
foreach (var item in items)
{
    Console.WriteLine(item);
}
```

The loop performs n iterations, so its iteration count grows as O(n).

## Coding 1: Two sum

**Question:** Return indices of two different elements whose sum equals `target`; return `null` if there is no pair. Example: `[2,7,11,15]`, target `9` returns `(0,1)`.

**Answer:** Store earlier values and indices; for each element look up its complement. Using `long` for the complement avoids integer subtraction overflow.

```csharp
static (int, int)? TwoSum(int[] values, int target)
{
    var seen = new Dictionary<int, int>();
    for (int i = 0; i < values.Length; i++)
    {
        long complement = (long)target - values[i];
        if (complement >= int.MinValue && complement <= int.MaxValue &&
            seen.TryGetValue((int)complement, out int index))
            return (index, i);
        seen[values[i]] = i;
    }
    return null;
}
```

Average O(n) time and O(n) space. Test duplicates `[3,3]`, negative numbers, an empty array, and no solution.

## Coding 2: First non-repeating character

**Question:** Return the first non-repeating `char`, or `null` if none exists. Example: `swiss` returns `w`.

**Answer:** Count first, then scan in input order.

```csharp
static char? FirstUnique(string text)
{
    var counts = new Dictionary<char, int>();
    foreach (char c in text)
        counts[c] = counts.GetValueOrDefault(c) + 1;
    foreach (char c in text)
        if (counts[c] == 1) return c;
    return null;
}
```

O(n) time and O(k) space for k distinct UTF-16 code units. Clarify case sensitivity and Unicode requirements: a `char` is not necessarily a full Unicode character.

## Coding 3: Valid brackets

**Question:** Given a string containing only `()[]{}`, determine whether brackets are properly nested. The empty string is valid.

**Answer:**

```csharp
static bool ValidBrackets(string text)
{
    var stack = new Stack<char>();
    foreach (char c in text)
    {
        if (c == '(' || c == '[' || c == '{') stack.Push(c);
        else
        {
            if (stack.Count == 0) return false;
            char open = stack.Pop();
            if (!((open == '(' && c == ')') ||
                  (open == '[' && c == ']') ||
                  (open == '{' && c == '}'))) return false;
        }
    }
    return stack.Count == 0;
}
```

O(n) time and O(n) space. Test `([])`, `([)]`, `]`, and `(`. For inputs outside the stated contract, agree on validation behavior.

## Coding 4: Binary search

**Question:** Find the index of a target in a sorted integer array, or return `-1`.

```csharp
static int BinarySearch(int[] values, int target)
{
    int left = 0, right = values.Length - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        if (values[mid] == target) return mid;
        if (values[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

O(log n) time and O(1) space. For duplicates this returns any matching index; finding the first requires a modified search. Test empty array, one item, and targets beyond both ends.

## Coding 5: Merge overlapping intervals

**Question:** Merge overlapping inclusive intervals. `[1,3], [2,6], [8,10]` becomes `[1,6], [8,10]`. Assume each interval has `Start <= End`.

```csharp
static List<(int Start, int End)> MergeIntervals(
    IEnumerable<(int Start, int End)> intervals)
{
    var sorted = intervals.OrderBy(x => x.Start).ThenBy(x => x.End).ToList();
    var result = new List<(int Start, int End)>();
    foreach (var item in sorted)
    {
        if (result.Count == 0 || item.Start > result[^1].End)
            result.Add(item);
        else
        {
            var last = result[^1];
            result[^1] = (last.Start, Math.Max(last.End, item.End));
        }
    }
    return result;
}
```

O(n log n) time for sorting and O(n) extra space. The stated inclusive convention merges intervals touching at an endpoint. Check empty, nested, and disjoint intervals.

## Coding 6: Reverse a singly linked list

**Question:** Reverse a linked list in place and return its new head.

```csharp
class Node
{
    public int Value;
    public Node? Next;
}

static Node? Reverse(Node? head)
{
    Node? previous = null;
    while (head != null)
    {
        Node? next = head.Next;
        head.Next = previous;
        previous = head;
        head = next;
    }
    return previous;
}
```

O(n) time and O(1) auxiliary space. Test empty, one node, and several nodes. Assumes an acyclic list.

## Additional practice questions and approaches

1. **Find a duplicate:** use a hash set; O(n) expected time, O(n) space. Example: `!seen.Add(value)` becomes true when `value` was already present.
2. **Longest substring without repeated characters:** sliding window plus last-seen indices; O(n) expected time. Example: `"abcabcbb"` has maximum length 3, from `"abc"`; move the left boundary only forward.
3. **Top K frequent values:** frequency dictionary plus heap, O(n log k), or bucket approach. Example: `[1,1,1,2,2,3]` with k=2 gives `[1,2]`.
4. **Level-order traversal of a binary tree:** queue-based BFS; O(n) time, O(width) queue space. Example: root `1` with children `2` and `3` gives levels `[1]`, then `[2,3]`.
5. **Detect a cycle in a linked list:** Floyd's slow/fast pointers; O(n) time, O(1) space. Example: `1 -> 2 -> 3 -> 2` eventually makes the two pointers meet.
6. **Find the nth highest salary:** clarify distinct salaries and ties; use `DENSE_RANK()`. Example: salaries `100,100,80` have second distinct salary `80`; see the SQL solution in file 07.
7. **Build a thread-safe order processor:** clarify concurrency and persistence first. Example: `ConcurrentQueue<Order>` protects in-memory enqueue/dequeue operations, but is not durable and cannot alone guarantee exactly-once business effects.
