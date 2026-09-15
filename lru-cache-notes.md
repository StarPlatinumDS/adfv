# LRU Cache in Go — From Naive Implementation to O(1)

## Goal

The purpose of this exercise was to understand how an **LRU cache** works and why its common implementation uses both:

- a hash map
- a doubly linked list

The main learning objective was not to memorize the canonical solution, but to first build a simpler version, observe its limitations, and then improve it.

---

## What is a cache?

A cache is temporary storage used to avoid repeating more expensive work.

For example, instead of querying a database every time:

```go
GetUser("123")
```

we can first check whether the result is already stored in memory.

A cache lookup can result in:

- **cache hit** — the value is already stored
- **cache miss** — the value is not stored and must be obtained from the original source

Because cache capacity is limited, some eviction policy is needed.

---

## What does LRU mean?

**LRU = Least Recently Used.**

When the cache is full and a new item needs to be inserted, the item that has not been used for the longest time is removed.

Example:

```text
capacity = 3

Put(a)
Put(b)
Put(c)

usage:
c -> b -> a

Get(a)

usage:
a -> c -> b

Put(d)
```

`b` is now the least recently used item, so it is evicted:

```text
d -> a -> c
```

---

# Version 1 — Counter-Based LRU

The first implementation used a map and a monotonically increasing counter.

```go
type Entry struct {
    Value    int
    LastUsed int
}

type Cache struct {
    Capacity int
    Counter  int
    Data     map[string]Entry
}
```

The map stores entries:

```text
"a" -> {Value: 100, LastUsed: 4}
"b" -> {Value: 200, LastUsed: 2}
"c" -> {Value: 300, LastUsed: 3}
```

The entry with the smallest `LastUsed` value is the least recently used.

---

## Get

A cache lookup:

1. checks whether the key exists
2. returns a miss if it does not
3. increments the counter
4. updates `LastUsed`
5. writes the modified struct back into the map

Example:

```go
func (c *Cache) Get(key string) (int, bool) {
    entry, ok := c.Data[key]
    if !ok {
        return 0, false
    }

    c.Counter++
    entry.LastUsed = c.Counter

    c.Data[key] = entry

    return entry.Value, true
}
```

### Important Go detail

For:

```go
entry, ok := c.Data[key]
```

`entry` is a **copy** of the value stored in the map.

Therefore:

```go
entry.LastUsed = 10
```

does not modify the map by itself.

The changed value must be written back:

```go
c.Data[key] = entry
```

---

## Put

`Put` has two main cases.

### Existing key

If the key already exists:

- update its value
- update its recency
- do not evict anything

### New key

If the key is new and the cache is full:

- scan all entries
- find the entry with the smallest `LastUsed`
- delete its key
- insert the new entry

Conceptually:

```text
if key exists:
    update it
    return

if cache is full:
    find least recently used key
    delete it

insert new key
```

---

## Complexity of version 1

Most operations are cheap:

```text
Get                  O(1) average
Put existing key     O(1) average
Put with free space  O(1) average
```

But eviction requires scanning every cached item:

```text
Put when full        O(n)
```

The expensive part was:

```go
for k, v := range c.Data {
    if first || v.LastUsed < minLastUsed {
        ...
    }
}
```

This became the reason to look for a better data structure.

---

# Testing the Cache

Before optimizing the implementation, tests were added using Go's standard `testing` package.

A test function has the form:

```go
func TestSomething(t *testing.T) {
    // arrange
    // act
    // assert
}
```

Example:

```go
func TestCacheGet(t *testing.T) {
    cache := initCache(3)

    cache.Put("a", 100)

    value, ok := cache.Get("a")
    if !ok {
        t.Fatal("expected a to exist")
    }

    if value != 100 {
        t.Fatalf("expected 100, got %d", value)
    }
}
```

The tests covered:

- storing and retrieving a value
- cache misses
- LRU eviction
- updating without eviction
- zero-capacity cache behavior
- recency changes after `Get`
- recency changes after updating an existing key

The tests focused on **observable cache behavior**, not internal implementation details.

That matters because the implementation could later change completely while the same behavior should still pass the tests.

---

# Why a Doubly Linked List?

The problem with version 1 was:

> How do we know which item is least recently used without scanning every entry?

If recency is stored as an ordered structure:

```text
HEAD                         TAIL
most recent            least recent
    A <-> C <-> B
```

then the least recently used item is always:

```go
list.tail
```

No scan is required.

But we also need to move arbitrary items when they are accessed.

For example:

```text
A <-> B <-> C
```

after accessing `B`:

```text
B <-> A <-> C
```

This is where a doubly linked list becomes useful.

---

# Nodes and the List

A node stores:

```go
type Node struct {
    key   string
    value int
    prev  *Node
    next  *Node
}
```

The list itself only stores pointers to its ends:

```go
type LList struct {
    head *Node
    tail *Node
}
```

The structure of the list exists through the pointers inside each node.

Example:

```text
HEAD
 ↓
 A <-> B <-> C
             ↑
            TAIL
```

---

## Important mental model

The list does not contain nodes like an array contains values.

Instead, nodes exist independently and reference each other.

The list only knows:

```text
head -> first node
tail -> last node
```

When a node is removed from the list, it is only **disconnected from the linked structure**.

It is not manually freed from memory.

In Go, an object becomes eligible for garbage collection when nothing reachable references it anymore.

This distinction is important because the LRU cache often removes a node from one position and immediately reuses the same node elsewhere.

---

# AddBeginning

To insert a node at the front:

```go
func (l *LList) AddBeginning(n *Node) {
    if l.head == nil {
        l.head = n
        l.tail = n
        n.next = nil
        n.prev = nil
        return
    }

    n.prev = nil
    n.next = l.head
    l.head.prev = n
    l.head = n
}
```

Example:

```text
before:

A <-> B

after AddBeginning(C):

C <-> A <-> B
↑             ↑
head          tail
```

---

# Remove

A node can be removed in O(1) if we already have a pointer to it.

For a middle node:

```text
A <-> B <-> C
```

and `n == B`:

```go
n.prev.next = n.next
n.next.prev = n.prev
```

becomes:

```text
A <-> C
```

No search is necessary.

The complete implementation must also handle:

- empty list
- only node
- head
- tail
- middle node

Example:

```go
func (l *LList) Remove(n *Node) {
    if l.head == nil {
        return
    }

    if l.head == l.tail {
        l.head = nil
        l.tail = nil
        n.prev = nil
        n.next = nil
        return
    }

    if l.head == n {
        l.head = n.next
        l.head.prev = nil
        n.next = nil
        return
    }

    if l.tail == n {
        l.tail = n.prev
        l.tail.next = nil
        n.prev = nil
        return
    }

    n.prev.next = n.next
    n.next.prev = n.prev

    n.prev = nil
    n.next = nil
}
```

Useful invariants:

```text
empty list:
head == nil
tail == nil

non-empty list:
head.prev == nil
tail.next == nil
```

---

# MoveToBeginning

Once `Remove` and `AddBeginning` exist, moving an existing node becomes simple:

```go
func (l *LList) MoveToBeginning(n *Node) {
    l.Remove(n)
    l.AddBeginning(n)
}
```

This is useful for the cache because every successful `Get` makes that entry the most recently used.

---

# Version 2 — Map + Doubly Linked List

The optimized cache uses:

```go
type Cache struct {
    Capacity int
    Data     map[string]*Node
    List     *LList
}
```

The map does **not** store a separate copy of the cached data.

Instead, it stores a pointer to the same node that is part of the linked list.

Conceptually:

```text
map:

"a" -----> Node A
"b" -----> Node B
"c" -----> Node C


list:

HEAD
 C <-> A <-> B
             TAIL
```

This combination solves two different problems.

The map gives:

```text
find arbitrary key quickly
```

The linked list gives:

```text
maintain recency order quickly
```

---

# Optimized Get

The map immediately returns the exact node:

```go
node, ok := c.Data[key]
```

If the key exists:

- move the node to the beginning
- return its value

```go
func (c *Cache) Get(key string) (int, bool) {
    node, ok := c.Data[key]
    if !ok {
        return 0, false
    }

    c.List.MoveToBeginning(node)

    return node.value, true
}
```

No counter is needed.

No list search is needed.

---

# Optimized Put

For an existing key:

```text
update node value
move node to beginning
```

For a new key:

```text
if full:
    tail is LRU
    remove tail from list
    remove its key from map

create new node
store its pointer in map
add it to beginning of list
```

Example:

```go
func (c *Cache) Put(key string, value int) {
    if c.Capacity <= 0 {
        return
    }

    node, ok := c.Data[key]
    if ok {
        node.value = value
        c.List.MoveToBeginning(node)
        return
    }

    if len(c.Data) >= c.Capacity {
        lru := c.List.tail

        c.List.Remove(lru)
        delete(c.Data, lru.key)
    }

    node = &Node{
        key:   key,
        value: value,
    }

    c.Data[key] = node
    c.List.AddBeginning(node)
}
```

---

# Why the Map Stores `*Node`

A very important design choice is:

```go
map[string]*Node
```

rather than:

```go
map[string]Node
```

The map gives us a direct pointer to the exact node inside the linked structure.

For example:

```go
node := cache.Data["b"]
```

Now we already know the exact node to move.

Without this pointer, we might have to search the linked list to find `"b"`, bringing back O(n) behavior.

---

# Complexity of Version 2

### Get

```text
map lookup          O(1) average
remove node         O(1)
add to front        O(1)

total               O(1) average
```

### Put existing key

```text
map lookup          O(1) average
update value        O(1)
move node           O(1)

total               O(1) average
```

### Put new key when full

```text
find LRU            O(1)  // tail
remove from list    O(1)
delete from map     O(1) average
insert into map     O(1) average
add to front        O(1)

total               O(1) average
```

This removes the O(n) eviction scan from version 1.

---

# Important Things Learned

## 1. Build the simple version first

The counter-based implementation was inefficient, but useful.

It made the real bottleneck visible:

```text
finding the least recently used entry required scanning everything
```

Only after seeing that problem did the linked-list solution have an obvious reason to exist.

---

## 2. Data structures solve specific operation costs

The doubly linked list was not added because it is the "standard LRU solution."

It was added because the required operations were:

```text
find arbitrary cached item quickly
move arbitrary known item quickly
find oldest item quickly
remove oldest item quickly
```

A single map cannot provide all of those efficiently.

A map plus a doubly linked list can.

---

## 3. A pointer to a node is a direct handle to a list position

If a function receives:

```go
Remove(n *Node)
```

there is no reason to search from `head`.

The pointer already identifies the exact node.

That is what makes removal O(1).

---

## 4. Doubly linked lists allow direct reconnection

For:

```text
A <-> B <-> C
```

`B` knows both neighbors:

```text
B.prev -> A
B.next -> C
```

so removal can reconnect:

```text
A.next = C
C.prev = A
```

without traversing the list.

---

## 5. Removing a node is not the same as freeing memory

`Remove` means:

```text
disconnect node from linked-list structure
```

not:

```text
destroy node immediately
```

The node can still be referenced by:

- a local variable
- the cache map
- another object

Go's garbage collector reclaims it later only when it is no longer reachable.

---

## 6. Map values have Go value semantics

With:

```go
map[string]Entry
```

retrieving an entry returns a copy:

```go
entry := c.Data[key]
```

Mutating `entry` does not mutate the value already stored in the map.

It must be assigned back:

```go
c.Data[key] = entry
```

Changing the optimized cache to:

```go
map[string]*Node
```

means the map now stores pointers instead.

Mutating:

```go
node.value = 123
```

changes the same node shared with the linked list.

---

## 7. Test behavior, not implementation

Useful tests checked things such as:

```text
Does Get return the correct value?
Does a missing key return false?
Does Get affect eviction order?
Does updating a key affect recency?
Does the least recently used key disappear?
Does zero capacity store nothing?
```

They did not depend on:

```text
Counter
LastUsed
```

This allowed the internal implementation to change while preserving the same tests and external behavior.

---

# Final Mental Model

The optimized LRU cache can be summarized as:

```text
                 MAP
              key -> node
                   |
                   v

HEAD                                   TAIL
MRU                                    LRU
 ↓                                      ↓
Node <-> Node <-> Node <-> Node <-> Node
```

The map answers:

> Where is the item?

The linked list answers:

> How recently was it used?

The head is always the **most recently used** entry.

The tail is always the **least recently used** entry.

A cache hit moves the node to the head.

An eviction removes the tail.

Together, the two structures allow average O(1) `Get` and `Put`.

---

# Progress

Completed:

- understood basic cache behavior
- understood cache hits and misses
- understood LRU eviction
- built a naive counter-based LRU
- implemented capacity handling
- wrote automated Go tests
- identified O(n) eviction as the main bottleneck
- learned the basic structure of a doubly linked list
- implemented `AddBeginning`
- implemented O(1) node removal
- understood head/tail/list invariants
- understood node detachment vs garbage collection
- connected `map[string]*Node` with the linked list
- replaced counter-based recency with list ordering
- reached average O(1) `Get` and `Put`

## Possible follow-up improvements

Not necessary for the basic exercise:

- add tests for the linked-list implementation itself
- add a test proving updates change recency
- rename methods/types to more idiomatic Go names
- make the cache generic with `Cache[K comparable, V any]`
- consider thread safety with a mutex
- compare with `container/list`
- benchmark naive O(n) eviction against the linked-list version
