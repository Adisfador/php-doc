# LeetCode паттерны и шаблоны

## 1. Two Pointers (Два указателя)

### Когда использовать
- Отсортированный массив
- Поиск пары элементов
- Reverse, палиндромы
- Удаление дубликатов

### Шаблон

```php
function twoSum(array $nums, int $target): array {
    $left = 0;
    $right = count($nums) - 1;
    
    while ($left < $right) {
        $sum = $nums[$left] + $nums[$right];
        
        if ($sum === $target) {
            return [$left, $right];
        } elseif ($sum < $target) {
            $left++;
        } else {
            $right--;
        }
    }
    
    return [];
}
// Time: O(n), Space: O(1)
```

### Примеры задач
- Two Sum II (sorted array)
- 3Sum
- Container With Most Water
- Remove Duplicates from Sorted Array
- Valid Palindrome

### Вариации

```php
// Встречные указатели (reverse)
function reverseString(array &$s): void {
    $left = 0;
    $right = count($s) - 1;
    
    while ($left < $right) {
        [$s[$left], $s[$right]] = [$s[$right], $s[$left]];
        $left++;
        $right--;
    }
}

// Один быстрый, один медленный (remove duplicates)
function removeDuplicates(array &$nums): int {
    $slow = 0;
    
    for ($fast = 1; $fast < count($nums); $fast++) {
        if ($nums[$fast] !== $nums[$slow]) {
            $slow++;
            $nums[$slow] = $nums[$fast];
        }
    }
    
    return $slow + 1;
}
```

---

## 2. Sliding Window (Скользящее окно)

### Когда использовать
- Подмассивы/подстроки
- Max/min в окне фиксированного размера
- Поиск pattern в строке

### Шаблон - Fixed Size Window

```php
function maxSumSubarray(array $nums, int $k): int {
    $maxSum = $windowSum = 0;
    
    // Первое окно
    for ($i = 0; $i < $k; $i++) {
        $windowSum += $nums[$i];
    }
    $maxSum = $windowSum;
    
    // Двигаем окно
    for ($i = $k; $i < count($nums); $i++) {
        $windowSum += $nums[$i] - $nums[$i - $k];
        $maxSum = max($maxSum, $windowSum);
    }
    
    return $maxSum;
}
// Time: O(n), Space: O(1)
```

### Шаблон - Variable Size Window

```php
function lengthOfLongestSubstring(string $s): int {
    $maxLen = $start = 0;
    $seen = [];
    
    for ($end = 0; $end < strlen($s); $end++) {
        $char = $s[$end];
        
        // Сжимаем окно если дубликат
        if (isset($seen[$char]) && $seen[$char] >= $start) {
            $start = $seen[$char] + 1;
        }
        
        $seen[$char] = $end;
        $maxLen = max($maxLen, $end - $start + 1);
    }
    
    return $maxLen;
}
// Time: O(n), Space: O(min(n, charset))
```

### Примеры задач
- Maximum Sum Subarray of Size K
- Longest Substring Without Repeating Characters
- Minimum Window Substring
- Longest Repeating Character Replacement
- Permutation in String

---

## 3. Fast & Slow Pointers (Быстрый и медленный)

### Когда использовать
- Linked List с циклами
- Найти середину списка
- Cycle detection

### Шаблон

```php
class ListNode {
    public $val;
    public $next;
    public function __construct($val = 0, $next = null) {
        $this->val = $val;
        $this->next = $next;
    }
}

// Обнаружение цикла (Floyd's Cycle Detection)
function hasCycle(?ListNode $head): bool {
    $slow = $fast = $head;
    
    while ($fast !== null && $fast->next !== null) {
        $slow = $slow->next;       // +1
        $fast = $fast->next->next; // +2
        
        if ($slow === $fast) {
            return true;  // Цикл найден
        }
    }
    
    return false;
}
// Time: O(n), Space: O(1)

// Найти середину списка
function findMiddle(?ListNode $head): ?ListNode {
    $slow = $fast = $head;
    
    while ($fast !== null && $fast->next !== null) {
        $slow = $slow->next;
        $fast = $fast->next->next;
    }
    
    return $slow;  // Середина
}
```

### Примеры задач
- Linked List Cycle
- Linked List Cycle II (найти начало цикла)
- Happy Number
- Middle of the Linked List
- Palindrome Linked List

---

## 4. Merge Intervals (Слияние интервалов)

### Когда использовать
- Работа с интервалами/диапазонами
- Календари, расписания
- Перекрывающиеся диапазоны

### Шаблон

```php
function merge(array $intervals): array {
    if (empty($intervals)) return [];
    
    // Сортировка по началу
    usort($intervals, fn($a, $b) => $a[0] <=> $b[0]);
    
    $merged = [$intervals[0]];
    
    for ($i = 1; $i < count($intervals); $i++) {
        $last = &$merged[count($merged) - 1];
        $current = $intervals[$i];
        
        if ($current[0] <= $last[1]) {
            // Перекрываются - объединяем
            $last[1] = max($last[1], $current[1]);
        } else {
            // Не перекрываются - добавляем новый
            $merged[] = $current;
        }
    }
    
    return $merged;
}
// Time: O(n log n), Space: O(n)

// Пример: [[1,3],[2,6],[8,10],[15,18]] → [[1,6],[8,10],[15,18]]
```

### Примеры задач
- Merge Intervals
- Insert Interval
- Meeting Rooms
- Meeting Rooms II (min conference rooms)
- Employee Free Time

---

## 5. Cyclic Sort (Циклическая сортировка)

### Когда использовать
- Числа в диапазоне [1, n]
- Найти missing/duplicate numbers
- In-place сортировка

### Шаблон

```php
function cyclicSort(array &$nums): void {
    $i = 0;
    
    while ($i < count($nums)) {
        $correctIndex = $nums[$i] - 1;
        
        if ($nums[$i] !== $nums[$correctIndex]) {
            // Swap на правильное место
            [$nums[$i], $nums[$correctIndex]] = 
                [$nums[$correctIndex], $nums[$i]];
        } else {
            $i++;
        }
    }
}
// Time: O(n), Space: O(1)

// Найти пропущенное число
function findMissingNumber(array $nums): int {
    $i = 0;
    $n = count($nums);
    
    while ($i < $n) {
        $correctIndex = $nums[$i];
        if ($correctIndex < $n && $nums[$i] !== $nums[$correctIndex]) {
            [$nums[$i], $nums[$correctIndex]] = 
                [$nums[$correctIndex], $nums[$i]];
        } else {
            $i++;
        }
    }
    
    for ($i = 0; $i < $n; $i++) {
        if ($nums[$i] !== $i) {
            return $i;
        }
    }
    
    return $n;
}
```

### Примеры задач
- Missing Number
- Find All Numbers Disappeared in an Array
- Find the Duplicate Number
- Find All Duplicates in an Array
- First Missing Positive

---

## 6. In-place Reversal of Linked List

### Когда использовать
- Reverse linked list
- Reverse подсписок
- Rotate list

### Шаблон

```php
function reverseList(?ListNode $head): ?ListNode {
    $prev = null;
    $current = $head;
    
    while ($current !== null) {
        $next = $current->next;     // Сохранить следующий
        $current->next = $prev;     // Развернуть указатель
        $prev = $current;           // Двигаем prev
        $current = $next;           // Двигаем current
    }
    
    return $prev;  // Новая голова
}
// Time: O(n), Space: O(1)

// Reverse между позициями left и right
function reverseBetween(?ListNode $head, int $left, int $right): ?ListNode {
    $dummy = new ListNode(0, $head);
    $prev = $dummy;
    
    // Дойти до left
    for ($i = 0; $i < $left - 1; $i++) {
        $prev = $prev->next;
    }
    
    // Reverse между left и right
    $current = $prev->next;
    for ($i = 0; $i < $right - $left; $i++) {
        $next = $current->next;
        $current->next = $next->next;
        $next->next = $prev->next;
        $prev->next = $next;
    }
    
    return $dummy->next;
}
```

### Примеры задач
- Reverse Linked List
- Reverse Linked List II
- Reverse Nodes in k-Group
- Rotate List
- Swap Nodes in Pairs

---

## 7. Tree BFS (Breadth-First Search)

### Когда использовать
- Level-order traversal
- Min depth
- Zigzag traversal

### Шаблон

```php
class TreeNode {
    public $val;
    public $left;
    public $right;
    public function __construct($val = 0, $left = null, $right = null) {
        $this->val = $val;
        $this->left = $left;
        $this->right = $right;
    }
}

function levelOrder(?TreeNode $root): array {
    if ($root === null) return [];
    
    $result = [];
    $queue = [$root];
    
    while (!empty($queue)) {
        $levelSize = count($queue);
        $currentLevel = [];
        
        for ($i = 0; $i < $levelSize; $i++) {
            $node = array_shift($queue);
            $currentLevel[] = $node->val;
            
            if ($node->left !== null) {
                $queue[] = $node->left;
            }
            if ($node->right !== null) {
                $queue[] = $node->right;
            }
        }
        
        $result[] = $currentLevel;
    }
    
    return $result;
}
// Time: O(n), Space: O(n)
```

### Примеры задач
- Binary Tree Level Order Traversal
- Binary Tree Zigzag Level Order Traversal
- Minimum Depth of Binary Tree
- Level Order Successor
- Connect Level Order Siblings

---

## 8. Tree DFS (Depth-First Search)

### Когда использовать
- Все пути от root до leaf
- Path sum
- Diameter, height

### Шаблон

```php
// Preorder: root → left → right
function preorder(?TreeNode $root, array &$result = []): array {
    if ($root === null) return $result;
    
    $result[] = $root->val;
    preorder($root->left, $result);
    preorder($root->right, $result);
    
    return $result;
}

// Inorder: left → root → right (BST в отсортированном порядке)
function inorder(?TreeNode $root, array &$result = []): array {
    if ($root === null) return $result;
    
    inorder($root->left, $result);
    $result[] = $root->val;
    inorder($root->right, $result);
    
    return $result;
}

// Postorder: left → right → root
function postorder(?TreeNode $root, array &$result = []): array {
    if ($root === null) return $result;
    
    postorder($root->left, $result);
    postorder($root->right, $result);
    $result[] = $root->val;
    
    return $result;
}

// Path Sum
function hasPathSum(?TreeNode $root, int $targetSum): bool {
    if ($root === null) return false;
    
    if ($root->left === null && $root->right === null) {
        return $root->val === $targetSum;
    }
    
    $remaining = $targetSum - $root->val;
    return hasPathSum($root->left, $remaining) || 
           hasPathSum($root->right, $remaining);
}
```

### Примеры задач
- Path Sum
- Path Sum II (все пути)
- Diameter of Binary Tree
- Maximum Depth of Binary Tree
- Invert Binary Tree

---

## 9. Two Heaps (Две кучи)

### Когда использовать
- Найти медиану в stream
- Sliding window median

### Шаблон

```php
class MedianFinder {
    private SplMaxHeap $maxHeap;  // Левая половина
    private SplMinHeap $minHeap;  // Правая половина
    
    public function __construct() {
        $this->maxHeap = new SplMaxHeap();
        $this->minHeap = new SplMinHeap();
    }
    
    public function addNum(int $num): void {
        // Добавить в maxHeap
        $this->maxHeap->insert($num);
        
        // Балансировка: largest from maxHeap → minHeap
        $this->minHeap->insert($this->maxHeap->extract());
        
        // Размеры должны быть равны или maxHeap на 1 больше
        if ($this->maxHeap->count() < $this->minHeap->count()) {
            $this->maxHeap->insert($this->minHeap->extract());
        }
    }
    
    public function findMedian(): float {
        if ($this->maxHeap->count() > $this->minHeap->count()) {
            return (float)$this->maxHeap->top();
        }
        
        return ($this->maxHeap->top() + $this->minHeap->top()) / 2;
    }
}
```

### Примеры задач
- Find Median from Data Stream
- Sliding Window Median
- IPO

---

## 10. Subsets (Подмножества)

### Когда использовать
- Все комбинации
- Все подмножества
- Permutations

### Шаблон - BFS подход

```php
function subsets(array $nums): array {
    $result = [[]];
    
    foreach ($nums as $num) {
        $newSubsets = [];
        foreach ($result as $subset) {
            $newSubsets[] = array_merge($subset, [$num]);
        }
        $result = array_merge($result, $newSubsets);
    }
    
    return $result;
}
// Time: O(2ⁿ), Space: O(2ⁿ)

// [1,2,3] → [[], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]]
```

### Шаблон - Backtracking

```php
function subsetsBacktrack(array $nums): array {
    $result = [];
    
    function backtrack($start, $current) use ($nums, &$result) {
        $result[] = $current;
        
        for ($i = $start; $i < count($nums); $i++) {
            backtrack($i + 1, array_merge($current, [$nums[$i]]));
        }
    }
    
    backtrack(0, []);
    return $result;
}

// Permutations
function permute(array $nums): array {
    $result = [];
    
    function backtrack($current) use ($nums, &$result) {
        if (count($current) === count($nums)) {
            $result[] = $current;
            return;
        }
        
        foreach ($nums as $num) {
            if (in_array($num, $current)) continue;
            backtrack(array_merge($current, [$num]));
        }
    }
    
    backtrack([]);
    return $result;
}
```

### Примеры задач
- Subsets
- Subsets II (с дубликатами)
- Permutations
- Permutations II
- Combination Sum

---

## 11. Modified Binary Search

### Когда использовать
- Отсортированный массив
- Rotated sorted array
- Поиск в диапазоне

### Шаблон

```php
function binarySearch(array $nums, int $target): int {
    $left = 0;
    $right = count($nums) - 1;
    
    while ($left <= $right) {
        $mid = $left + (int)(($right - $left) / 2);
        
        if ($nums[$mid] === $target) {
            return $mid;
        } elseif ($nums[$mid] < $target) {
            $left = $mid + 1;
        } else {
            $right = $mid - 1;
        }
    }
    
    return -1;
}

// Rotated Sorted Array
function searchRotated(array $nums, int $target): int {
    $left = 0;
    $right = count($nums) - 1;
    
    while ($left <= $right) {
        $mid = (int)(($left + $right) / 2);
        
        if ($nums[$mid] === $target) return $mid;
        
        // Левая половина отсортирована
        if ($nums[$left] <= $nums[$mid]) {
            if ($nums[$left] <= $target && $target < $nums[$mid]) {
                $right = $mid - 1;
            } else {
                $left = $mid + 1;
            }
        } else {
            // Правая половина отсортирована
            if ($nums[$mid] < $target && $target <= $nums[$right]) {
                $left = $mid + 1;
            } else {
                $right = $mid - 1;
            }
        }
    }
    
    return -1;
}
```

### Примеры задач
- Binary Search
- Search in Rotated Sorted Array
- Find Minimum in Rotated Sorted Array
- Search in a Sorted Array of Unknown Size

---

## 12. Top K Elements

### Когда использовать
- K largest/smallest элементов
- K frequent elements
- Kth largest element

### Шаблон

```php
// Top K Frequent Elements
function topKFrequent(array $nums, int $k): array {
    // Подсчет частоты
    $freq = array_count_values($nums);
    
    // Min heap по частоте
    $heap = new SplMinHeap();
    
    foreach ($freq as $num => $count) {
        $heap->insert(['count' => $count, 'num' => $num]);
        
        if ($heap->count() > $k) {
            $heap->extract();
        }
    }
    
    $result = [];
    while (!$heap->isEmpty()) {
        $result[] = $heap->extract()['num'];
    }
    
    return $result;
}
// Time: O(n log k), Space: O(n)

// Kth Largest Element (QuickSelect)
function findKthLargest(array $nums, int $k): int {
    return quickSelect($nums, 0, count($nums) - 1, count($nums) - $k);
}

function quickSelect(array &$nums, int $left, int $right, int $k): int {
    $pivot = partition($nums, $left, $right);
    
    if ($pivot === $k) {
        return $nums[$pivot];
    } elseif ($pivot < $k) {
        return quickSelect($nums, $pivot + 1, $right, $k);
    } else {
        return quickSelect($nums, $left, $pivot - 1, $k);
    }
}

function partition(array &$nums, int $left, int $right): int {
    $pivot = $nums[$right];
    $i = $left;
    
    for ($j = $left; $j < $right; $j++) {
        if ($nums[$j] < $pivot) {
            [$nums[$i], $nums[$j]] = [$nums[$j], $nums[$i]];
            $i++;
        }
    }
    
    [$nums[$i], $nums[$right]] = [$nums[$right], $nums[$i]];
    return $i;
}
```

### Примеры задач
- Kth Largest Element in an Array
- Top K Frequent Elements
- K Closest Points to Origin
- Kth Smallest Element in a Sorted Matrix

---

## 13. K-way Merge

### Когда использовать
- Merge K sorted arrays/lists
- Smallest range в K lists

### Шаблон

```php
function mergeKLists(array $lists): ?ListNode {
    $heap = new SplMinHeap();
    
    // Добавить головы всех списков
    foreach ($lists as $i => $head) {
        if ($head !== null) {
            $heap->insert([
                'val' => $head->val,
                'node' => $head,
                'listIndex' => $i
            ]);
        }
    }
    
    $dummy = new ListNode(0);
    $current = $dummy;
    
    while (!$heap->isEmpty()) {
        $item = $heap->extract();
        $node = $item['node'];
        
        $current->next = $node;
        $current = $current->next;
        
        if ($node->next !== null) {
            $heap->insert([
                'val' => $node->next->val,
                'node' => $node->next,
                'listIndex' => $item['listIndex']
            ]);
        }
    }
    
    return $dummy->next;
}
```

---

## 14. Dynamic Programming

### Шаблон - 1D DP

```php
// Climbing Stairs
function climbStairs(int $n): int {
    if ($n <= 2) return $n;
    
    $dp = array_fill(0, $n + 1, 0);
    $dp[1] = 1;
    $dp[2] = 2;
    
    for ($i = 3; $i <= $n; $i++) {
        $dp[$i] = $dp[$i - 1] + $dp[$i - 2];
    }
    
    return $dp[$n];
}

// Space optimized
function climbStairs(int $n): int {
    if ($n <= 2) return $n;
    
    $prev2 = 1;
    $prev1 = 2;
    
    for ($i = 3; $i <= $n; $i++) {
        $current = $prev1 + $prev2;
        $prev2 = $prev1;
        $prev1 = $current;
    }
    
    return $prev1;
}
```

### Шаблон - 2D DP

```php
// Longest Common Subsequence
function longestCommonSubsequence(string $text1, string $text2): int {
    $m = strlen($text1);
    $n = strlen($text2);
    $dp = array_fill(0, $m + 1, array_fill(0, $n + 1, 0));
    
    for ($i = 1; $i <= $m; $i++) {
        for ($j = 1; $j <= $n; $j++) {
            if ($text1[$i - 1] === $text2[$j - 1]) {
                $dp[$i][$j] = $dp[$i - 1][$j - 1] + 1;
            } else {
                $dp[$i][$j] = max($dp[$i - 1][$j], $dp[$i][$j - 1]);
            }
        }
    }
    
    return $dp[$m][$n];
}
```

### Примеры задач
- Climbing Stairs
- House Robber
- Coin Change
- Longest Increasing Subsequence
- Longest Common Subsequence

---

## Чек-лист паттернов

| Паттерн | Признаки | Сложность |
|---------|----------|-----------|
| Two Pointers | Sorted array, pairs | O(n) |
| Sliding Window | Subarray/substring | O(n) |
| Fast & Slow | Linked list cycle | O(n) |
| Merge Intervals | Overlapping ranges | O(n log n) |
| Cyclic Sort | [1..n] range | O(n) |
| Linked List Reversal | Reverse list | O(n) |
| Tree BFS | Level order | O(n) |
| Tree DFS | Paths, depth | O(n) |
| Two Heaps | Median, balance | O(n log n) |
| Subsets | Combinations | O(2ⁿ) |
| Binary Search | Sorted, search | O(log n) |
| Top K | K largest/smallest | O(n log k) |
| K-way Merge | Merge sorted | O(N log k) |
| DP | Overlapping subproblems | Varies |
