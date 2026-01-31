# Структуры данных на PHP

## Array (Массив)

### Особенности PHP Array
PHP array это hybrid: и array, и hash map одновременно.

```php
// Indexed array
$arr = [1, 2, 3, 4, 5];
$arr[0];  // O(1) доступ
$arr[] = 6;  // O(1) append

// Associative array (hash map)
$map = ['key' => 'value', 'name' => 'John'];
$map['key'];  // O(1) lookup

// Operations
count($arr);           // O(1) - размер
array_push($arr, 6);   // O(1) - добавить в конец
array_pop($arr);       // O(1) - удалить с конца
array_shift($arr);     // O(n) - удалить с начала
array_unshift($arr, 0);// O(n) - добавить в начало
in_array(3, $arr);     // O(n) - поиск значения
array_search(3, $arr); // O(n) - поиск индекса
isset($arr[0]);        // O(1) - проверка существования

// Time Complexity:
// Access:  O(1)
// Search:  O(n)
// Insert:  O(1) в конец, O(n) в начало/середину
// Delete:  O(1) с конца, O(n) с начала/середины
```

---

## Stack (Стек) - LIFO

```php
class Stack {
    private array $items = [];
    
    public function push($item): void {
        $this->items[] = $item;  // O(1)
    }
    
    public function pop() {
        if ($this->isEmpty()) return null;
        return array_pop($this->items);  // O(1)
    }
    
    public function peek() {
        return $this->items[count($this->items) - 1] ?? null;  // O(1)
    }
    
    public function isEmpty(): bool {
        return empty($this->items);  // O(1)
    }
    
    public function size(): int {
        return count($this->items);  // O(1)
    }
}

// Использование
$stack = new Stack();
$stack->push(1);
$stack->push(2);
$stack->push(3);
echo $stack->pop();  // 3
echo $stack->peek(); // 2

// SplStack (built-in)
$stack = new SplStack();
$stack->push(1);
$stack->push(2);
$stack->pop();  // 2
```

### Применение Stack
- Function call stack
- Undo/Redo операции
- Проверка сбалансированных скобок
- DFS (Depth-First Search)
- Обратная польская нотация

---

## Queue (Очередь) - FIFO

```php
class Queue {
    private array $items = [];
    
    public function enqueue($item): void {
        $this->items[] = $item;  // O(1)
    }
    
    public function dequeue() {
        if ($this->isEmpty()) return null;
        return array_shift($this->items);  // O(n) - проблема!
    }
    
    public function peek() {
        return $this->items[0] ?? null;  // O(1)
    }
    
    public function isEmpty(): bool {
        return empty($this->items);
    }
}

// SplQueue (built-in, эффективнее)
$queue = new SplQueue();
$queue->enqueue(1);
$queue->enqueue(2);
$queue->enqueue(3);
echo $queue->dequeue();  // 1 (FIFO)

// Эффективная очередь через двойные указатели
class EfficientQueue {
    private array $items = [];
    private int $head = 0;
    
    public function enqueue($item): void {
        $this->items[] = $item;  // O(1)
    }
    
    public function dequeue() {
        if ($this->isEmpty()) return null;
        
        $item = $this->items[$this->head];
        unset($this->items[$this->head]);  // O(1)!
        $this->head++;
        
        // Очистка для экономии памяти
        if ($this->head > 100 && $this->head > count($this->items)) {
            $this->items = array_values($this->items);
            $this->head = 0;
        }
        
        return $item;
    }
    
    public function isEmpty(): bool {
        return !isset($this->items[$this->head]);
    }
}
```

### Применение Queue
- BFS (Breadth-First Search)
- Обработка задач (job queues)
- Scheduling
- Cache (FIFO eviction)

---

## Linked List (Связный список)

```php
class ListNode {
    public $val;
    public ?ListNode $next;
    
    public function __construct($val = 0, ?ListNode $next = null) {
        $this->val = $val;
        $this->next = $next;
    }
}

class LinkedList {
    private ?ListNode $head = null;
    private int $size = 0;
    
    // Добавить в начало O(1)
    public function prepend($val): void {
        $newNode = new ListNode($val, $this->head);
        $this->head = $newNode;
        $this->size++;
    }
    
    // Добавить в конец O(n)
    public function append($val): void {
        if ($this->head === null) {
            $this->head = new ListNode($val);
            $this->size++;
            return;
        }
        
        $current = $this->head;
        while ($current->next !== null) {
            $current = $current->next;
        }
        $current->next = new ListNode($val);
        $this->size++;
    }
    
    // Удалить по значению O(n)
    public function delete($val): bool {
        if ($this->head === null) return false;
        
        if ($this->head->val === $val) {
            $this->head = $this->head->next;
            $this->size--;
            return true;
        }
        
        $current = $this->head;
        while ($current->next !== null) {
            if ($current->next->val === $val) {
                $current->next = $current->next->next;
                $this->size--;
                return true;
            }
            $current = $current->next;
        }
        
        return false;
    }
    
    // Поиск O(n)
    public function find($val): ?ListNode {
        $current = $this->head;
        while ($current !== null) {
            if ($current->val === $val) {
                return $current;
            }
            $current = $current->next;
        }
        return null;
    }
    
    // Reverse O(n)
    public function reverse(): void {
        $prev = null;
        $current = $this->head;
        
        while ($current !== null) {
            $next = $current->next;
            $current->next = $prev;
            $prev = $current;
            $current = $next;
        }
        
        $this->head = $prev;
    }
    
    public function toArray(): array {
        $result = [];
        $current = $this->head;
        while ($current !== null) {
            $result[] = $current->val;
            $current = $current->next;
        }
        return $result;
    }
}

// Time Complexity:
// Access:  O(n)
// Search:  O(n)
// Insert:  O(1) в начало, O(n) в конец
// Delete:  O(n)
```

---

## Hash Map (Hash Table)

В PHP это обычный associative array.

```php
// Hash Map operations
$map = [];

// Insert/Update O(1)
$map['key'] = 'value';

// Get O(1)
$value = $map['key'] ?? null;

// Delete O(1)
unset($map['key']);

// Contains O(1)
$exists = isset($map['key']);
$exists = array_key_exists('key', $map);

// Iterate O(n)
foreach ($map as $key => $value) {
    echo "$key: $value\n";
}

// Размер O(1)
$size = count($map);

// Практический пример: подсчет частоты
function countFrequency(array $arr): array {
    $freq = [];
    foreach ($arr as $item) {
        $freq[$item] = ($freq[$item] ?? 0) + 1;
    }
    return $freq;
}

// array_count_values() - built-in
$freq = array_count_values([1, 2, 2, 3, 3, 3]);
// [1 => 1, 2 => 2, 3 => 3]
```

---

## Set (Множество)

```php
// PHP не имеет встроенного Set, используем array keys
$set = [];

// Add O(1)
$set['value'] = true;

// Contains O(1)
$exists = isset($set['value']);

// Remove O(1)
unset($set['value']);

// Size O(1)
$size = count($set);

// Iterate O(n)
foreach (array_keys($set) as $value) {
    echo $value;
}

// Set operations
$set1 = ['a' => true, 'b' => true, 'c' => true];
$set2 = ['b' => true, 'c' => true, 'd' => true];

// Union
$union = $set1 + $set2;  // или array_merge()

// Intersection
$intersection = array_intersect_key($set1, $set2);

// Difference
$diff = array_diff_key($set1, $set2);

// SplObjectStorage для объектов
$set = new SplObjectStorage();
$obj1 = new stdClass();
$set->attach($obj1);
$set->contains($obj1);  // true
```

---

## Binary Tree (Бинарное дерево)

```php
class TreeNode {
    public $val;
    public ?TreeNode $left;
    public ?TreeNode $right;
    
    public function __construct($val = 0, ?TreeNode $left = null, ?TreeNode $right = null) {
        $this->val = $val;
        $this->left = $left;
        $this->right = $right;
    }
}

class BinaryTree {
    private ?TreeNode $root = null;
    
    // Inorder traversal: Left → Root → Right
    public function inorder(?TreeNode $node = null, array &$result = []): array {
        $node = $node ?? $this->root;
        if ($node === null) return $result;
        
        $this->inorder($node->left, $result);
        $result[] = $node->val;
        $this->inorder($node->right, $result);
        
        return $result;
    }
    
    // Preorder: Root → Left → Right
    public function preorder(?TreeNode $node = null, array &$result = []): array {
        $node = $node ?? $this->root;
        if ($node === null) return $result;
        
        $result[] = $node->val;
        $this->preorder($node->left, $result);
        $this->preorder($node->right, $result);
        
        return $result;
    }
    
    // Postorder: Left → Right → Root
    public function postorder(?TreeNode $node = null, array &$result = []): array {
        $node = $node ?? $this->root;
        if ($node === null) return $result;
        
        $this->postorder($node->left, $result);
        $this->postorder($node->right, $result);
        $result[] = $node->val;
        
        return $result;
    }
    
    // Level order (BFS)
    public function levelOrder(): array {
        if ($this->root === null) return [];
        
        $result = [];
        $queue = [$this->root];
        
        while (!empty($queue)) {
            $levelSize = count($queue);
            $currentLevel = [];
            
            for ($i = 0; $i < $levelSize; $i++) {
                $node = array_shift($queue);
                $currentLevel[] = $node->val;
                
                if ($node->left) $queue[] = $node->left;
                if ($node->right) $queue[] = $node->right;
            }
            
            $result[] = $currentLevel;
        }
        
        return $result;
    }
    
    // Max depth
    public function maxDepth(?TreeNode $node = null): int {
        $node = $node ?? $this->root;
        if ($node === null) return 0;
        
        return 1 + max(
            $this->maxDepth($node->left),
            $this->maxDepth($node->right)
        );
    }
}
```

---

## Binary Search Tree (BST)

```php
class BST {
    private ?TreeNode $root = null;
    
    // Insert O(log n) average, O(n) worst
    public function insert($val): void {
        $this->root = $this->insertNode($this->root, $val);
    }
    
    private function insertNode(?TreeNode $node, $val): TreeNode {
        if ($node === null) {
            return new TreeNode($val);
        }
        
        if ($val < $node->val) {
            $node->left = $this->insertNode($node->left, $val);
        } else {
            $node->right = $this->insertNode($node->right, $val);
        }
        
        return $node;
    }
    
    // Search O(log n) average, O(n) worst
    public function search($val): ?TreeNode {
        return $this->searchNode($this->root, $val);
    }
    
    private function searchNode(?TreeNode $node, $val): ?TreeNode {
        if ($node === null || $node->val === $val) {
            return $node;
        }
        
        if ($val < $node->val) {
            return $this->searchNode($node->left, $val);
        }
        
        return $this->searchNode($node->right, $val);
    }
    
    // Inorder дает отсортированный массив!
    public function toSortedArray(): array {
        $result = [];
        $this->inorder($this->root, $result);
        return $result;
    }
    
    private function inorder(?TreeNode $node, array &$result): void {
        if ($node === null) return;
        
        $this->inorder($node->left, $result);
        $result[] = $node->val;
        $this->inorder($node->right, $result);
    }
}
```

---

## Heap (Куча)

```php
// Min Heap (built-in)
$minHeap = new SplMinHeap();
$minHeap->insert(5);
$minHeap->insert(3);
$minHeap->insert(7);
$minHeap->insert(1);

echo $minHeap->extract();  // 1 (минимум)
echo $minHeap->top();      // 3 (peek)

// Max Heap
$maxHeap = new SplMaxHeap();
$maxHeap->insert(5);
$maxHeap->insert(3);
$maxHeap->insert(7);
$maxHeap->insert(1);

echo $maxHeap->extract();  // 7 (максимум)

// Priority Queue
$pq = new SplPriorityQueue();
$pq->insert('task1', 1);  // priority 1
$pq->insert('task2', 5);  // priority 5 (higher)
$pq->insert('task3', 3);  // priority 3

echo $pq->extract();  // task2 (highest priority)

// Custom comparison
class MaxHeapCustom extends SplHeap {
    protected function compare($a, $b): int {
        return $b <=> $a;  // max heap
    }
}

// Time Complexity:
// Insert: O(log n)
// Extract-Min/Max: O(log n)
// Peek: O(1)
// Build heap: O(n)
```

---

## Graph (Граф)

```php
// Adjacency List representation
class Graph {
    private array $adjList = [];
    
    public function addEdge($from, $to): void {
        if (!isset($this->adjList[$from])) {
            $this->adjList[$from] = [];
        }
        $this->adjList[$from][] = $to;
    }
    
    // DFS - рекурсивный
    public function dfs($start, array &$visited = []): array {
        $visited[$start] = true;
        $result = [$start];
        
        foreach ($this->adjList[$start] ?? [] as $neighbor) {
            if (!isset($visited[$neighbor])) {
                $result = array_merge($result, $this->dfs($neighbor, $visited));
            }
        }
        
        return $result;
    }
    
    // DFS - итеративный (stack)
    public function dfsIterative($start): array {
        $visited = [];
        $stack = [$start];
        $result = [];
        
        while (!empty($stack)) {
            $node = array_pop($stack);
            
            if (isset($visited[$node])) continue;
            
            $visited[$node] = true;
            $result[] = $node;
            
            foreach ($this->adjList[$node] ?? [] as $neighbor) {
                if (!isset($visited[$neighbor])) {
                    $stack[] = $neighbor;
                }
            }
        }
        
        return $result;
    }
    
    // BFS (queue)
    public function bfs($start): array {
        $visited = [$start => true];
        $queue = [$start];
        $result = [];
        
        while (!empty($queue)) {
            $node = array_shift($queue);
            $result[] = $node;
            
            foreach ($this->adjList[$node] ?? [] as $neighbor) {
                if (!isset($visited[$neighbor])) {
                    $visited[$neighbor] = true;
                    $queue[] = $neighbor;
                }
            }
        }
        
        return $result;
    }
}

// Использование
$graph = new Graph();
$graph->addEdge(0, 1);
$graph->addEdge(0, 2);
$graph->addEdge(1, 3);
$graph->addEdge(2, 4);

print_r($graph->dfs(0));  // [0, 1, 3, 2, 4]
print_r($graph->bfs(0));  // [0, 1, 2, 3, 4]
```

---

## Trie (Prefix Tree)

```php
class TrieNode {
    public array $children = [];
    public bool $isEndOfWord = false;
}

class Trie {
    private TrieNode $root;
    
    public function __construct() {
        $this->root = new TrieNode();
    }
    
    // Insert O(m) где m - длина слова
    public function insert(string $word): void {
        $node = $this->root;
        
        for ($i = 0; $i < strlen($word); $i++) {
            $char = $word[$i];
            
            if (!isset($node->children[$char])) {
                $node->children[$char] = new TrieNode();
            }
            
            $node = $node->children[$char];
        }
        
        $node->isEndOfWord = true;
    }
    
    // Search O(m)
    public function search(string $word): bool {
        $node = $this->searchNode($word);
        return $node !== null && $node->isEndOfWord;
    }
    
    // StartsWith O(m)
    public function startsWith(string $prefix): bool {
        return $this->searchNode($prefix) !== null;
    }
    
    private function searchNode(string $str): ?TrieNode {
        $node = $this->root;
        
        for ($i = 0; $i < strlen($str); $i++) {
            $char = $str[$i];
            
            if (!isset($node->children[$char])) {
                return null;
            }
            
            $node = $node->children[$char];
        }
        
        return $node;
    }
}

// Использование
$trie = new Trie();
$trie->insert('apple');
$trie->insert('app');
$trie->search('apple');     // true
$trie->search('app');       // true
$trie->search('appl');      // false
$trie->startsWith('app');   // true
```

---

## Сравнение структур данных

| Структура | Access | Search | Insert | Delete | Space |
|-----------|--------|--------|--------|--------|-------|
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| Linked List | O(n) | O(n) | O(1) | O(1) | O(n) |
| Stack | O(n) | O(n) | O(1) | O(1) | O(n) |
| Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| Hash Map | O(1) | O(1) | O(1) | O(1) | O(n) |
| BST (balanced) | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Heap | - | - | O(log n) | O(log n) | O(n) |
| Trie | - | O(m) | O(m) | O(m) | O(n×m) |

*m - длина строки/слова*
