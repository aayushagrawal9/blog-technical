---
title: "Building a Simple Memory Allocator in Rust"
date: 2025-10-22 09:00:00 -0700
categories: [Programming, Rust]
tags: [rust, memory, systems-programming, allocator]
---

Memory allocation is one of those things most developers take for granted. Let's build a simple bump allocator in Rust to understand what happens under the hood when you call `Box::new()`.

## What is a Bump Allocator?

A bump allocator is the simplest possible allocation strategy. You maintain a pointer to the next free byte, and every allocation just "bumps" that pointer forward. Deallocation is a no-op -- you free everything at once when the allocator is dropped.

```rust
use std::alloc::{GlobalAlloc, Layout};
use std::cell::UnsafeCell;
use std::ptr;

const ARENA_SIZE: usize = 128 * 1024; // 128 KB

struct BumpAllocator {
    arena: UnsafeCell<[u8; ARENA_SIZE]>,
    next: UnsafeCell<usize>,
}

unsafe impl Sync for BumpAllocator {}
```

## Implementing GlobalAlloc

To use our allocator with Rust's standard library, we implement the `GlobalAlloc` trait:

```rust
unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let next = &mut *self.next.get();
        let arena = &mut *self.arena.get();

        // Align the next pointer
        let aligned = (*next + layout.align() - 1) & !(layout.align() - 1);
        let new_next = aligned + layout.size();

        if new_next > ARENA_SIZE {
            ptr::null_mut() // Out of memory
        } else {
            *next = new_next;
            arena.as_mut_ptr().add(aligned)
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // Bump allocators don't deallocate individual allocations
    }
}
```

## Why Bump Allocators Matter

While you wouldn't use a bump allocator for a general-purpose application, they're incredibly useful in specific scenarios:

- **Game engines**: Per-frame allocations that get freed all at once
- **Compilers**: AST nodes allocated during a single compilation pass
- **Parsers**: Temporary structures built during parsing

The key insight is that **allocation patterns matter**. If you know your deallocation pattern in advance, you can choose a dramatically simpler (and faster) allocator.

## Performance Comparison

Here's a rough benchmark comparing our bump allocator to the system allocator for 10,000 small allocations:

| Allocator | Time |
|-----------|------|
| System (glibc malloc) | ~450 us |
| Bump allocator | ~12 us |

The bump allocator wins by a wide margin because each allocation is just an addition and a comparison -- no free lists, no coalescing, no locking.

## Going Further

From here, you could explore:
- **Arena allocators** that support typed allocations
- **Pool allocators** for fixed-size objects
- **Slab allocators** used in the Linux kernel
- **jemalloc** and **mimalloc** for production-grade alternatives

Understanding allocation strategies gives you a powerful tool for performance optimization, even if you rarely need to write a custom allocator in practice.
