# Go Slices and Backing Arrays

## Mental model

A Go slice is **not an array** and is not just a pointer.

A slice value can be thought of as a small descriptor containing:

```text
pointer to backing array
length
capacity
```

Conceptually:

```text
slice
┌─────────────────────┐
│ pointer to elements │
│ length              │
│ capacity            │
└─────────────────────┘
```

The actual elements live in a **backing array**.

For example:

```go
a := []int{1, 2, 3, 4}
```

creates backing storage for the elements and a slice `a` referring to it.

```text
a ──> [1][2][3][4]
      len=4, cap=4
```

The backing array does not need to be declared explicitly.

---

## Creating another slice

```go
b := a[1:3]
```

does not copy the elements.

`b` is a new slice descriptor referring to part of the same backing array.

```text
backing array
[1][2][3][4]
    ↑
    b

a = [1 2 3 4]
b = [2 3]
```

Slice bounds are half-open:

```text
[start:end]
```

`start` is included, `end` is not.

---

## Length and capacity

**Length** is how many elements the slice currently exposes.

**Capacity** is how many elements are available from the slice's starting position to the end of its current backing storage.

Example:

```go
a := []int{1, 2, 3, 4, 5}
b := a[2:4]
```

Then:

```text
b = [3 4]
len(b) = 2
cap(b) = 3
```

because the backing array still has space for `[3 4 5]` starting from `b[0]`.

---

## Mutating shared data

If two slices refer to the same backing array, modifying an element through one slice changes the shared data:

```go
a := []int{1, 2, 3, 4}
b := a[1:3]

b[0] = 99
```

Result:

```text
a = [1 99 3 4]
b = [99 3]
```

Neither slice was copied. Both still refer to the same storage.

---

## `append` and capacity

If there is enough capacity, `append` can reuse the existing backing array.

```go
a := []int{1, 2, 3, 4}
b := a[:2]

b = append(b, 10)
```

`b` had spare capacity, so the backing array becomes:

```text
[1][2][10][4]
```

Therefore:

```text
a = [1 2 10 4]
b = [1 2 10]
```

This can overwrite elements visible through other slices sharing the same backing array.

### When capacity is exhausted

If the slice cannot grow further in its current backing array:

```go
b = append(b, 20)
```

Go allocates new backing storage, copies **the elements belonging to `b`**, appends the new value, and returns a slice referring to the new storage.

Conceptually:

```text
old:
a ──> [1][2][10][4]

new:
b ──> [1][2][10][20]...
```

After that, changes to `b` no longer affect `a`.

Do not rely on the exact capacity chosen for the new allocation; the growth strategy is an implementation detail.

---

## Assignment vs `copy`

Plain assignment copies the slice descriptor:

```go
b := a
```

Now `a` and `b` initially refer to the same backing array.

To create independent element storage:

```go
b := make([]int, len(a))
copy(b, a)
```

Now changing `b[i]` does not change `a[i]`.

---

## Passing a slice to a function

Slices are passed **by value**.

That means the slice descriptor is copied, not the backing array.

```go
func change(s []int) {
    s[0] = 999
}

a := []int{1, 2, 3}
change(a)
```

The function's `s` and the caller's `a` have separate descriptors, but both initially point to the same backing array.

Result:

```text
a = [999 2 3]
```

### Changing the slice descriptor itself

This only changes the function's local copy:

```go
func shrink(s []int) {
    s = s[:1]
}
```

Calling:

```go
a := []int{1, 2, 3}
shrink(a)
```

does not change `len(a)` in the caller.

The same issue applies to `append`.

```go
func add(s []int) {
    s = append(s, 4)
}
```

The function may get a new length, capacity, or even a new backing array, but the caller's slice descriptor is unchanged.

If the caller needs the updated slice, return it:

```go
func add(s []int) []int {
    return append(s, 4)
}

a = add(a)
```

---

## Key distinction

Keep these three things separate:

1. **Backing array** — owns the actual element storage.
2. **Slice** — a value describing a view into backing storage.
3. **Slice descriptor** — conceptually contains pointer + length + capacity.

Element mutation:

```go
s[i] = value
```

changes the backing array.

Changing the slice itself:

```go
s = s[:n]
s = append(s, value)
```

changes the slice descriptor and may also mutate or replace the backing array depending on available capacity.

That distinction explains most surprising slice behaviour.
