# Greedy Algorithms

## 1. What & Why

A greedy algorithm makes the locally optimal choice at each step, hoping to reach a globally optimal solution. No backtracking, no exhaustive search — just take the best available option right now and move on.

When greedy works, it is elegant and fast — often O(n log n) for problems where DP would be O(n²). The challenge is knowing when greedy works and when it doesn't. Greedy fails on the 0/1 knapsack problem, many coin change instances, and other problems where a locally suboptimal choice now enables a globally better result later.

---

## 2. Core Concepts

### Greedy Choice Property

A problem has the greedy choice property if making the locally optimal choice at each step leads to a globally optimal solution. This must be **proven** — not assumed.

### Optimal Substructure

After making a greedy choice, the remaining problem is an instance of the same problem. The greedy choice reduces the problem size.

### Proving Greedy Correctness: Exchange Argument

The standard proof technique:
1. Assume an optimal solution exists that differs from the greedy solution
2. Show you can "exchange" a non-greedy choice for the greedy choice without worsening the solution
3. Conclude the greedy solution is at least as good as any optimal solution — therefore it is optimal

### When Greedy Fails

- **0/1 Knapsack:** taking the highest value/weight ratio item first doesn't lead to optimal packing (you can't use fractions)
- **Coin change with arbitrary denominations:** coins = [1, 3, 4], amount = 6. Greedy gives 4+1+1 = 3 coins. Optimal: 3+3 = 2 coins.
- **Shortest path with negative weights:** Dijkstra (greedy) fails; need Bellman-Ford
- **Matrix chain multiplication:** greedy order doesn't minimize multiplications

---

## 3. How It Works

```
Activity Selection (interval scheduling):
Activities: [(1,4), (3,5), (0,6), (5,7), (3,9), (5,9), (6,10), (8,11), (8,12), (2,14), (12,16)]
Goal: select maximum number of non-overlapping activities.

Greedy: sort by end time, always pick the next activity that starts after the last selected ends.

Sorted by end: [(1,4),(3,5),(0,6),(5,7),(3,9),(5,9),(6,10),(8,11),(8,12),(2,14),(12,16)]
Select (1,4): end=4
Next starts >= 4: (3,5)? No. (5,7)? starts=5 >=4. Select (5,7): end=7
Next starts >= 7: (8,11)? starts=8 >=7. Select (8,11): end=11
Next starts >= 11: (12,16)? starts=12 >=11. Select (12,16): end=16

Result: 4 activities. Optimal.
```

---

## 4. Code Examples (TypeScript)

```typescript
// --- Activity Selection / Interval Scheduling ---
// Maximize number of non-overlapping intervals
function activitySelection(intervals: [number, number][]): [number, number][] {
  // Sort by end time — greedy: always take the activity that ends earliest
  const sorted = [...intervals].sort((a, b) => a[1] - b[1]);
  const selected: [number, number][] = [];
  let lastEnd = -Infinity;

  for (const [start, end] of sorted) {
    if (start >= lastEnd) {
      selected.push([start, end]);
      lastEnd = end;
    }
  }
  return selected;
}

// --- Minimum Number of Meeting Rooms ---
// Given meeting intervals, find the minimum number of rooms needed
function minMeetingRooms(intervals: [number, number][]): number {
  const starts = intervals.map(i => i[0]).sort((a, b) => a - b);
  const ends = intervals.map(i => i[1]).sort((a, b) => a - b);

  let rooms = 0, maxRooms = 0;
  let s = 0, e = 0;

  while (s < starts.length) {
    if (starts[s] < ends[e]) {
      rooms++;      // a meeting starts before another ends → need new room
      s++;
    } else {
      rooms--;      // a meeting ended → room freed
      e++;
    }
    maxRooms = Math.max(maxRooms, rooms);
  }
  return maxRooms;
}

// --- Jump Game: can you reach the last index? ---
// Greedy: track the maximum index reachable from current position
function canJump(nums: number[]): boolean {
  let maxReach = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > maxReach) return false; // can't reach this position
    maxReach = Math.max(maxReach, i + nums[i]);
  }
  return true;
}

// --- Jump Game II: minimum jumps to reach the last index ---
function jump(nums: number[]): number {
  let jumps = 0, currentEnd = 0, farthest = 0;

  for (let i = 0; i < nums.length - 1; i++) {
    farthest = Math.max(farthest, i + nums[i]);
    if (i === currentEnd) {
      // Must make a jump — take the one that reaches farthest
      jumps++;
      currentEnd = farthest;
    }
  }
  return jumps;
}

// --- Gas Station: circular route ---
// Find the starting gas station from which you can complete the circuit
function canCompleteCircuit(gas: number[], cost: number[]): number {
  let totalGas = 0, currentGas = 0, start = 0;

  for (let i = 0; i < gas.length; i++) {
    const net = gas[i] - cost[i];
    totalGas += net;
    currentGas += net;

    if (currentGas < 0) {
      // Can't reach station i+1 from current start — try starting after i
      start = i + 1;
      currentGas = 0;
    }
  }

  return totalGas >= 0 ? start : -1; // if total >= 0, solution exists
}

// --- Fractional Knapsack: maximize value, can take fractions ---
// Greedy by value/weight ratio — works for fractional, NOT 0/1 knapsack
function fractionalKnapsack(items: { weight: number; value: number }[], capacity: number): number {
  const sorted = [...items].sort((a, b) => b.value / b.weight - a.value / a.weight);
  let totalValue = 0, remaining = capacity;

  for (const item of sorted) {
    if (remaining === 0) break;
    const take = Math.min(item.weight, remaining);
    totalValue += take * (item.value / item.weight);
    remaining -= take;
  }
  return totalValue;
}

// --- Task Scheduler: minimum CPU intervals (with cooling time n) ---
// Must wait n intervals before running the same task again
function leastInterval(tasks: string[], n: number): number {
  const freq: Record<string, number> = {};
  for (const t of tasks) freq[t] = (freq[t] ?? 0) + 1;

  const maxFreq = Math.max(...Object.values(freq));
  const maxCount = Object.values(freq).filter(f => f === maxFreq).length;

  // Formula: max((maxFreq - 1) * (n + 1) + maxCount, tasks.length)
  // The formula accounts for idle slots needed between most frequent tasks
  return Math.max((maxFreq - 1) * (n + 1) + maxCount, tasks.length);
}

// --- Assign Cookies: maximize number of satisfied children ---
// Each child i has greed factor g[i], each cookie has size s[j]
// A child is satisfied if s[j] >= g[i]. Each cookie assigned to one child.
function findContentChildren(g: number[], s: number[]): number {
  g.sort((a, b) => a - b); // sort by greed ascending
  s.sort((a, b) => a - b); // sort by size ascending

  let child = 0, cookie = 0;
  while (child < g.length && cookie < s.length) {
    if (s[cookie] >= g[child]) child++; // satisfy this child
    cookie++; // use this cookie regardless
  }
  return child;
}

// --- Partition Labels: partition string into max parts where each letter appears in only one part ---
function partitionLabels(s: string): number[] {
  // For each character, find its last occurrence
  const lastOccurrence = new Map<string, number>();
  for (let i = 0; i < s.length; i++) lastOccurrence.set(s[i], i);

  const partitions: number[] = [];
  let start = 0, end = 0;

  for (let i = 0; i < s.length; i++) {
    end = Math.max(end, lastOccurrence.get(s[i])!); // extend partition to include all of this char
    if (i === end) {
      // Current partition is complete
      partitions.push(end - start + 1);
      start = i + 1;
    }
  }
  return partitions;
}
// "ababcbacadefegdehijhklij" → [9, 7, 8]

// --- Minimum Number of Arrows to Burst Balloons ---
// Balloons: [xstart, xend]. Arrow at x bursts all balloons with xstart <= x <= xend
function findMinArrowShots(points: [number, number][]): number {
  if (points.length === 0) return 0;
  const sorted = [...points].sort((a, b) => a[1] - b[1]); // sort by end
  let arrows = 1;
  let arrowPos = sorted[0][1]; // first arrow at first balloon's end

  for (let i = 1; i < sorted.length; i++) {
    if (sorted[i][0] > arrowPos) {
      // This balloon starts after current arrow — need a new arrow
      arrows++;
      arrowPos = sorted[i][1];
    }
  }
  return arrows;
}

// --- Huffman Encoding (concept) ---
// Build optimal prefix-free binary code: more frequent characters get shorter codes
// Implementation uses a min-heap (see heaps.md)
interface HuffmanNode {
  char?: string;
  freq: number;
  left?: HuffmanNode;
  right?: HuffmanNode;
}

function buildHuffmanCodes(freq: Map<string, number>): Map<string, string> {
  // Min-heap by frequency
  const heap: HuffmanNode[] = [...freq.entries()]
    .map(([char, f]) => ({ char, freq: f }))
    .sort((a, b) => a.freq - b.freq);

  while (heap.length > 1) {
    const left = heap.shift()!;  // lowest freq
    const right = heap.shift()!; // second lowest freq
    const merged: HuffmanNode = { freq: left.freq + right.freq, left, right };
    // Insert merged node in sorted position
    const insertPos = heap.findIndex(n => n.freq >= merged.freq);
    if (insertPos === -1) heap.push(merged);
    else heap.splice(insertPos, 0, merged);
  }

  const codes = new Map<string, string>();
  function traverse(node: HuffmanNode, code: string): void {
    if (node.char !== undefined) { codes.set(node.char, code); return; }
    if (node.left) traverse(node.left, code + "0");
    if (node.right) traverse(node.right, code + "1");
  }
  traverse(heap[0], "");
  return codes;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Applying greedy when only DP is correct.** The most dangerous mistake. Greedy for 0/1 knapsack or coin change with arbitrary denominations gives wrong answers. Always verify the greedy choice property holds before using greedy.

> ⚠️ **Sorting by the wrong metric.** In interval scheduling, sorting by start time gives wrong results (longer intervals block more). Sort by end time. In activity scheduling for maximum profit, sort by deadline. Getting the sort key wrong defeats the greedy approach.

> ⚠️ **Confusing fractional knapsack (greedy works) with 0/1 knapsack (greedy fails).** Fractional: you can take 0.5 of an item. 0/1: you either take or don't. The greedy value/weight ratio works for fractional, fails for 0/1.

> ⚠️ **Gas station: not checking total fuel.** If total gas < total cost, no solution exists regardless of starting point. Check this first before running the greedy search.

---

## 6. When to Use / Not Use

**Greedy works well for:**
- Interval scheduling (sort by end time)
- Huffman encoding (optimal prefix codes)
- Dijkstra's algorithm (shortest path, non-negative weights)
- Prim's / Kruskal's MST
- Fractional knapsack
- Problems where locally optimal = globally optimal (provable via exchange argument)

**Greedy fails for:**
- 0/1 knapsack (need DP)
- Coin change with non-canonical denominations (need DP)
- Shortest path with negative edges (need Bellman-Ford)
- Any problem where a suboptimal choice now enables a better global result later

**Comparison with DP:**
- Greedy: O(n log n) typical, makes irrevocable choices, works when greedy property holds
- DP: O(n²) or O(n³) typical, explores all options, always correct for applicable problems
- Always prefer greedy when provably correct — it's simpler and faster

---

## 7. Real-World Scenario

A meeting scheduler needs to find the minimum number of conference rooms:

```typescript
interface Meeting {
  title: string;
  start: number; // minutes since midnight
  end: number;
}

function scheduleRooms(meetings: Meeting[]): Map<number, Meeting[]> {
  const sorted = [...meetings].sort((a, b) => a.start - b.start);
  const rooms = new Map<number, number>(); // roomId → when it's free
  const assignments = new Map<number, Meeting[]>();
  let nextRoomId = 0;

  for (const meeting of sorted) {
    // Find a room that's free (its previous meeting ends before this one starts)
    let assignedRoom = -1;
    for (const [roomId, freeAt] of rooms.entries()) {
      if (freeAt <= meeting.start) {
        assignedRoom = roomId;
        break;
      }
    }

    if (assignedRoom === -1) {
      // No free room — allocate a new one
      assignedRoom = nextRoomId++;
      assignments.set(assignedRoom, []);
    }

    rooms.set(assignedRoom, meeting.end);
    assignments.get(assignedRoom)!.push(meeting);
  }

  return assignments;
}
```

---

## 8. Interview Questions

**Q1: When does greedy fail and you need DP?**
A: When a locally optimal choice at one step prevents a better global outcome. Example: coin change with coins [1, 3, 4], amount = 6. Greedy picks 4 first (locally best), leaving 2 → needs 4+1+1 = 3 coins. DP finds 3+3 = 2 coins. The greedy choice (pick largest coin) violates the greedy choice property for this denomination set.

**Q2: How do you prove a greedy algorithm is correct?**
A: Exchange argument: assume any optimal solution O differs from the greedy solution G at some step. Show you can modify O to match G's choice at that step without increasing cost. By induction, G is as good as O at every step — so G is optimal.

**Q3: Solve the interval scheduling maximization problem.**
A: Sort intervals by end time. Greedily select the next interval that starts at or after the last selected interval ends. This always leaves the most room for future intervals. O(n log n) time.

**Q4: Explain the task scheduler problem.**
A: With cooling time n, the minimum intervals = max((maxFreq - 1) × (n + 1) + countOfMaxFreq, total tasks). The formula: arrange the most frequent task with gaps, count idles needed. If tasks fill the gaps, no idles are needed.

**Q5: What is the gas station greedy approach?**
A: Track cumulative gas surplus. When it goes negative, the current starting station is invalid — any station up to the current one is also invalid (they would have the same or worse deficit). Reset the start to the next station. If total gas ≥ total cost, a valid start exists.

---

## 9. Exercises

**Exercise 1:** Assign cookies. Given children's greed factors and cookie sizes, assign at most one cookie per child to maximize the number of satisfied children.
*Hint: sort both arrays, use two pointers.*

**Exercise 2:** Partition a string into as many parts as possible so that each letter appears in at most one part.
*Hint: for each character, record its last occurrence. A partition ends when the current index equals the maximum last occurrence of any character seen.*

**Exercise 3:** Find the minimum number of arrows to burst all balloons (each balloon is a range, an arrow at x bursts all balloons containing x).
*Hint: sort by end coordinate. One arrow can burst all balloons it passes through — place it at the end of the earliest-ending balloon.*

**Exercise 4:** Prove that activity selection by end time is optimal using an exchange argument.

**Exercise 5:** Given an array of non-negative integers, find the minimum number of steps to reach the end if from index i you can jump at most nums[i] steps.
*Hint: track the farthest reachable index and the boundary of the current jump.*

---

## 10. Further Reading

- CLRS Chapter 16 — Greedy Algorithms
- [Greedy algorithms — cp-algorithms.com](https://cp-algorithms.com/greedy/)
- [Exchange argument proof technique](https://cp-algorithms.com/greedy/exchange_argument.html)
- LeetCode: #455 Assign Cookies, #763 Partition Labels, #452 Minimum Arrows, #55 Jump Game, #134 Gas Station, #621 Task Scheduler
