# Zig File Reading, Allocators, and Memory Ownership

## Goal

Build a small Zig program that:

- reads a text file,
- stores its contents in allocator-backed memory,
- counts bytes, lines, and words,
- prints the result,
- explicitly frees the allocated memory.

The main purpose of the exercise was **not** the text statistics themselves.

The important part was understanding:

- explicit allocation,
- memory ownership,
- slice lifetime,
- file/resource lifetime,
- the difference between a file handle and file contents,
- how Zig differs from Go's garbage-collected model.

---

## Final Program

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.gpa;

    const contents = try std.Io.Dir.cwd().readFileAlloc(
        init.io,
        "./src/test.txt",
        allocator,
        .limited(1024 * 1024),
    );
    defer allocator.free(contents);

    var lines: usize = 0;
    var words: usize = 0;
    var in_word = false;

    for (contents) |byte| {
        if (byte == '\n') {
            lines += 1;
        }

        const is_whitespace = switch (byte) {
            ' ', '\t', '\n', '\r' => true,
            else => false,
        };

        if (is_whitespace) {
            in_word = false;
        } else if (!in_word) {
            words += 1;
            in_word = true;
        }
    }

    if (contents.len > 0 and contents[contents.len - 1] != '\n') {
        lines += 1;
    }

    std.debug.print(
        "Contents:\n{s}\n\nBytes:\t{d}\nLines:\t{d}\nWords:\t{d}\n",
        .{ contents, contents.len, lines, words },
    );
}
```

---

# 1. File on Storage vs Data in Memory

A file exists persistently on storage.

When the program reads it, the program works with a copy of those bytes in memory.

Conceptually:

```text
file on storage
    ↓ read
allocated memory in RAM
    ↓
[]u8 slice
```

Changing the in-memory bytes does **not** automatically modify the original file.

Writing changes back to the file is a separate operation.

---

# 2. A File Handle Is Not the File Contents

An early incorrect attempt treated `openFile()` as if it returned the bytes inside the file.

The important distinction is:

```text
"test.txt"
    ↓
path

openFile(...)
    ↓
File
    ↓
OS-backed file handle

read from file
    ↓
[]u8
    ↓
actual bytes
```

A `File` represents an open operating-system resource.

It does not directly contain an `.items` field with the whole file contents.

---

# 3. File Resource Lifetime vs Memory Lifetime

Two different resources may need cleanup.

## File handle

If the file is opened explicitly:

```zig
const file = try ...openFile(...);
defer file.close(init.io);
```

then `close()` releases the OS-level file resource.

## Allocated memory

If file contents are allocated through an allocator:

```zig
const contents = try ...readFileAlloc(...);
defer allocator.free(contents);
```

then `free()` releases the allocated RAM.

These are different kinds of cleanup:

```text
file.close()
    → release OS file resource

allocator.free(contents)
    → release allocated memory
```

---

# 4. Explicit Allocators

Zig does not have a garbage collector.

Instead, allocation is explicit.

In this exercise:

```zig
const allocator = init.gpa;
```

provides an allocator.

Then:

```zig
const contents = try std.Io.Dir.cwd().readFileAlloc(
    init.io,
    "./src/test.txt",
    allocator,
    .limited(1024 * 1024),
);
```

asks Zig to allocate enough memory to hold the file contents.

The code that receives that allocated memory is responsible for eventually releasing it:

```zig
defer allocator.free(contents);
```

This establishes the ownership relationship:

```text
allocator
    ↓
creates allocation
    ↓
contents refers to it
    ↓
program uses contents
    ↓
allocator.free(contents)
```

---

# 5. Slice Does Not Own Memory by Itself

`contents` is a slice.

Conceptually:

```text
contents
├── ptr ─────────────> allocated bytes
└── len
```

The slice itself is only a description of memory.

It does not automatically manage the lifetime of the allocation.

The allocator created the backing memory.

Our program is responsible for releasing it.

This is similar to the earlier Go slice mental model:

> a slice is a descriptor/view over backing storage.

The difference is that in Zig the backing allocation can be explicitly freed by the programmer.

---

# 6. Lifetime and Dangling References

The slice is valid only while the allocation it refers to is valid.

Conceptually:

```text
allocate
    ↓
contents is valid
    ↓
use contents
    ↓
free allocation
    ↓
contents must not be used anymore
```

Calling:

```zig
allocator.free(contents);
```

does **not** automatically turn `contents.ptr` into `null`.

The slice may still contain the old address and length.

But the memory no longer belongs to the program.

Using the slice after that point is invalid.

This is a dangling reference.

Important rule:

> After freeing an allocation, every pointer or slice referring to that allocation must be considered unusable.

---

# 7. `defer`

This exercise also used:

```zig
defer allocator.free(contents);
```

This means:

> Run this cleanup when the current function returns.

This keeps the cleanup close to the allocation site and reduces the chance of forgetting it.

The conceptual sequence is:

```text
allocate
↓
register deferred free
↓
use allocation
↓
main returns
↓
free executes
```

---

# 8. Arena vs GPA

An earlier snippet used:

```zig
init.arena.allocator()
```

An arena has a different lifetime policy.

It is useful when many allocations share the same lifetime and can all be discarded together.

That version did not demonstrate explicit per-allocation cleanup clearly enough for this exercise.

The final version therefore used:

```zig
init.gpa
```

and explicitly called:

```zig
allocator.free(contents);
```

The lesson was not that arenas are wrong.

The lesson was that different allocators represent different lifetime/ownership strategies.

---

# 9. Temporary Buffers vs Allocated File Contents

An earlier experiment used:

```zig
var input_buffer: [1024]u8 = undefined;
```

That buffer is different from the final `contents`.

A fixed local array like this:

```zig
[1024]u8
```

has storage tied to the function's local lifetime.

It does not require `allocator.free()`.

By contrast:

```zig
readFileAlloc(...)
```

creates allocator-backed memory whose lifetime must be managed explicitly.

Conceptually:

```text
local fixed buffer
    → lifetime tied to function scope

allocator-backed memory
    → lifetime controlled explicitly
```

---

# 10. `undefined`

The earlier reader code used:

```zig
var buffer: [1024]u8 = undefined;
```

`undefined` does not mean:

- empty,
- zero,
- null.

It means the initial bytes should not be relied upon.

This is fine for a buffer that will be overwritten before reading its contents.

---

# 11. Whole-File Reading vs Line-by-Line Reading

A line-by-line example used:

```zig
takeDelimiter('\n')
```

This reads data until a delimiter and returns the bytes without the delimiter.

Therefore code such as:

```zig
std.debug.print("{s}", .{line});
```

prints every line without restoring the newline.

This resulted in output similar to:

```text
some randomtextthat has to be printedliKE SO...
```

because the `\n` characters were consumed as delimiters.

For this exercise, line-by-line reading was unnecessary.

`readFileAlloc()` already loads the entire file into memory:

```text
file
    ↓
readFileAlloc
    ↓
contents: []u8
```

After that point, the filesystem part of the task is over.

The rest is simply processing bytes in memory.

---

# 12. Iterating Over File Contents

Because `contents` is a slice, it can be iterated directly:

```zig
for (contents) |byte| {
    ...
}
```

Each `byte` is one `u8`.

No additional file API is necessary.

---

# 13. Byte Count

A slice already stores its length.

Therefore the byte count is simply:

```zig
contents.len
```

No loop is required.

This directly reinforces the slice model:

```text
slice
├── pointer
└── length
```

---

# 14. Line Count

Newlines can be counted while iterating:

```zig
if (byte == '\n') {
    lines += 1;
}
```

But this alone misses the last line if the file does not end with `\n`.

Example:

```text
hello\n
world
```

There is one newline character but two lines.

Therefore:

```zig
if (contents.len > 0 and contents[contents.len - 1] != '\n') {
    lines += 1;
}
```

accounts for a final unterminated line.

---

# 15. Word Count

Counting whitespace does not correctly count words.

Example:

```text
hello     world
```

There are many spaces but only two words.

Instead, the program tracks whether it is currently inside a word:

```zig
var in_word = false;
```

Whitespace ends a word:

```zig
if (is_whitespace) {
    in_word = false;
}
```

A non-whitespace byte encountered while not already inside a word starts a new word:

```zig
else if (!in_word) {
    words += 1;
    in_word = true;
}
```

Conceptually:

```text
whitespace → non-whitespace
```

means:

```text
new word started
```

---

# 16. Whitespace Detection

For the exercise, whitespace was handled explicitly:

```zig
const is_whitespace = switch (byte) {
    ' ', '\t', '\n', '\r' => true,
    else => false,
};
```

This avoided adding another standard-library API merely for a small educational task.

The goal was to practice:

- slices,
- loops,
- byte comparisons,
- simple state.

---

# 17. Zig I/O Version Mismatch

One difficulty during the exercise was that current Zig I/O syntax differs significantly from many examples available online.

Old snippets may use APIs that no longer exist or have moved.

The exercise therefore deliberately avoided treating memorization of exact standard-library API names as the learning objective.

Important distinction:

> Understanding the operation is more important than remembering the current spelling of the API.

Useful conceptual sequence:

```text
file
↓
reader / read operation
↓
memory
↓
slice
↓
process data
↓
cleanup
```

The exact standard-library functions may evolve.

The ownership and lifetime questions remain important.

---

# 18. Relative Paths

`cwd()` means:

> current working directory of the running process.

It does **not** mean:

> directory containing `main.zig`.

Therefore:

```zig
"./src/test.txt"
```

worked because the program was run with the project root as the working directory and the file was inside `src`.

This distinction matters whenever file paths behave unexpectedly.

---

# Final Mental Model

The most important model from the exercise is:

```text
file on storage
        ↓
readFileAlloc
        ↓
allocator creates memory
        ↓
contents: []u8
├── pointer
└── length
        ↓
process bytes
        ↓
print result
        ↓
allocator.free(contents)
        ↓
memory is no longer valid
```

And separately:

```text
File
    = OS-backed resource

[]u8
    = view over bytes in memory

Allocator
    = mechanism that provides/releases memory
```

---

# Main Takeaways

1. A file handle and file contents are different things.
2. Reading a file brings bytes into memory.
3. `readFileAlloc()` returns a slice over allocator-backed memory.
4. A slice does not manage memory lifetime by itself.
5. Zig has no garbage collector.
6. Explicitly allocated memory must have a clear cleanup strategy.
7. `free()` does not automatically null existing pointers/slices.
8. Using a slice after freeing its backing memory is invalid.
9. `defer` is useful for pairing resource acquisition with cleanup.
10. Fixed local buffers and allocator-backed memory have different lifetimes.
11. Once the file is loaded, statistics can be computed by directly iterating over the byte slice.
12. Zig's I/O APIs change across versions, so the underlying ownership/lifetime model is more important than memorizing exact function names.

---

# Result

The assignment is complete.

The final program successfully:

- reads a file,
- allocates memory explicitly,
- exposes the file contents as a byte slice,
- counts bytes,
- counts lines,
- counts words,
- prints the data,
- releases the allocation explicitly.

The main learning outcome was a first concrete understanding of **explicit memory ownership and lifetime in Zig**, especially in contrast with Go's runtime-managed memory model.
