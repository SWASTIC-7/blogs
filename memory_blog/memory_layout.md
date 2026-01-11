# Memory Layout of a Program — a concise guide

This post explains how a program's virtual memory is organized at runtime. It covers the major segments you'll encounter (text/code, data, BSS, heap, stack), how they grow, and practical tips for inspecting and reasoning about memory on different platforms.

**Why this matters:** understanding memory layout helps with debugging, performance tuning, security hardening (ASLR, NX), and diagnosing crashes (stack overflow, use-after-free).

--

**High-level layout (virtual address space)**

Virtual memory is typically arranged like this (addresses grow upward):

High addresses

```
+----------------------------+  <- Environment (argc, argv, env)
|        Program Data        |
+----------------------------+
|           Stack            |  <- frames pushed here (stack grows down)
|       [stack frames]       |
+----------------------------+
|      Unallocated Space     |  <- gap between stack and heap
+----------------------------+
|           Heap             |  <- dynamic allocations (malloc/new)
|   [malloc(), allocators]   |
+----------------------------+
| Block Started by Symbol (BSS) | <- uninitialized globals/statics
+----------------------------+
|         Data Segment      |  <- initialized globals, string literals
+----------------------------+
|       Text / Code Segment  |  <- program code (read-only / executable)
+----------------------------+
Low addresses
```

The exact order and addresses vary by OS, architecture, and loader options (PIE, shared libs). Modern systems add guard pages and map shared libraries, thread stacks, and kernel mappings at other parts of the space.

--

**Segments explained**

- **Text / Code**: contains machine instructions of your program and usually shared among processes that run the same binary. Marked executable and (often) read-only.

- **Data segment** (initialized data): holds global and static variables that are initialized to non-zero values at program start (e.g., `int x = 5;`). Contents are part of the binary image.

- **BSS** (Block Started by Symbol): holds uninitialized globals/statics (e.g., `static int counter;`). The OS zeroes this area before program start.

- **Heap**: dynamically allocated memory (malloc, new, vector growth). The heap grows upward (higher addresses) as the program requests more memory from the OS. Allocators manage sub-allocation inside the heap and may request large chunks (brk/sbrk or mmap on POSIX).

- **Stack**: per-thread call stack. Frames and local variables live here. The stack usually grows downward (toward lower addresses). Each thread gets its own stack region with guard pages to catch overflows.

- **Environment & Arguments**: often placed at high addresses right above the program data and stack; contains argv, envp and auxiliary vectors on UNIX.

--

**Growth and dynamic maps**

- Stack: grows downward. The kernel enforces a maximum stack size per process/thread (ulimit, thread creation flags). Exceeding it triggers a stack overflow (segfault/EXCEPTION_STACK_OVERFLOW).
- Heap: grows upward. The process may grow the heap with `brk`/`sbrk` or by mapping anonymous memory (`mmap`, `VirtualAlloc`). Allocator decisions (ptmalloc, jemalloc, tcmalloc) affect fragmentation and address usage.
- Shared libraries: loaded by the dynamic loader, often mapped into higher parts of the address space with their own code/data segments.

--

**Security and runtime features**

- **ASLR (Address Space Layout Randomization)**: randomizes base addresses (code, heap, stack, mmap) to make exploitation harder.
- **NX (No-eXecute) / DEP**: marks memory pages non-executable to prevent execution of injected code in data/heap/stack.
- **Stack canaries**: protect return addresses by placing a random value on the stack and checking it at function exit.
- **PIE (Position Independent Executable)**: allows the main binary itself to be randomized like shared libraries.

--

**Practical examples: printing addresses in C**

This small C program prints representative addresses for code, global data, heap, and stack. Compile and run it on your platform.

```c
#include <stdio.h>
#include <stdlib.h>

int global_init = 42;      // data
int global_uninit;         // bss

void foo(void) {
	int stack_var;
	void *heap = malloc(16);
	printf("&foo (code): %p\n", (void*)foo);
	printf("&global_init (data): %p\n", (void*)&global_init);
	printf("&global_uninit (bss): %p\n", (void*)&global_uninit);
	printf("malloc() (heap): %p\n", heap);
	printf("&stack_var (stack): %p\n", (void*)&stack_var);
}

int main(void) {
	foo();
	return 0;
}
```

What you'll see (addresses vary): code addresses often in a low region, data and bss near each other, heap somewhere in the middle, and the stack at a high address (on many UNIX-like systems).

--

**Inspecting memory mappings**

- Linux: `cat /proc/<pid>/maps` or `cat /proc/self/maps` to view current mappings and permissions. Example:

```
cat /proc/self/maps
```

- macOS: `vmmap <pid>` or use `vmmap -w`.
- Windows: use Sysinternals `VMMap`, Process Explorer, or `!address` in WinDbg. Visual Studio debugger shows memory windows for a process.

Note: the `maps` output shows ranges, permissions (rwx), and backing files (shared libs, anonymous, stack).

--

**Allocator behaviour and fragmentation**

- Modern allocators use arenas, bins, and mmap to reduce lock contention and fragmentation. Small allocations may be sub-allocated from larger chunks; large allocations may be individually `mmap`ed and returned to the OS independently.
- Observing fragmentation: many small allocations can fragment the heap internally. Using pooling, object pools, or sized arenas can reduce fragmentation.

--

**Debugging and tooling**

- Linux/macOS: `gdb` lets you inspect symbols and memory. Use `p &variable` and `info proc mappings` (gdb) or read `/proc/<pid>/maps`.
- Memory checkers: AddressSanitizer (ASan), Valgrind (Linux), and Dr. Memory (Windows) help find use-after-free, double-free, buffer overflow.
- Leak detectors: ASan, Valgrind's memcheck, or platform profilers.
- Windows tools: Application Verifier, VMMap (Sysinternals), and WinDbg for low-level inspection.

--

**Common pitfalls**

- Assuming stable addresses: ASLR and PIE mean addresses change between runs.
- Mixing stack and heap pointers across threads without synchronization.
- Forgetting that large heap allocations may be `mmap`-backed and not contiguous with the small-chunk allocator region.
- Relying on sizeof assumptions across architectures (padding, alignment).

--

**Quick checklist for investigating crashes**

1. Reproduce with debugging symbols enabled (`-g`).
2. Print addresses as in the example program to orient yourself.
3. Inspect mapped ranges (`/proc/self/maps`, `vmmap`, VMMap).
4. Run ASan or platform memory analyzers to catch common C/C++ memory bugs.
5. Check OS-level limits (stack size, ulimits) and allocator behavior.

--

**Platform-specific notes**

- Linux: `/proc/<pid>/maps`, `pmap`, `readelf -S` (for binary section offsets).
- macOS: `vmmap`, and `atos` for symbolization of addresses in crash reports.
- Windows: `VMMap` (Sysinternals), Process Explorer, `!address` in WinDbg.

--

**Summary**

The memory layout of a process is stable in concept but flexible in practice: OS loaders, security features, and allocators all influence exact mapping. Understanding the segments (text, data, BSS, heap, stack) and how they grow is fundamental for debugging, optimizing, and securing applications.

--

**References & further reading**

- The Linux Programming Interface — Michael Kerrisk (sections on process memory).
- man 5 proc (`/proc/<pid>/maps`) and `man mmap`.
- Valgrind, AddressSanitizer documentation.
- Windows Sysinternals (VMMap) and WinDbg docs.

--

If you'd like, I can:
- Add an illustrated SVG/PNG diagram matching the ASCII layout.
- Add a Rust version of the address-printing example.
- Add step-by-step instructions for inspecting a live process on Windows.

(Saved here: [memory_layout.md](memory_layout.md))


