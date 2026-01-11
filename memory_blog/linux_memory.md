# Linux Memory Management — Virtual to Physical

This guide explains how Linux manages memory: virtual address spaces, page tables, the MMU, and how programs are loaded and executed. We'll cover the journey from a virtual address in your program to the physical RAM location, kernel memory management, and practical tools for inspection.

---

## Table of Contents

1. [Why Virtual Memory?](#why-virtual-memory)
2. [Virtual vs Physical Addresses](#virtual-vs-physical-addresses)
3. [The Memory Management Unit (MMU)](#the-memory-management-unit-mmu)
4. [Page Tables and Address Translation](#page-tables-and-address-translation)
5. [Linux Process Memory Layout](#linux-process-memory-layout)
6. [How a Program is Loaded](#how-a-program-is-loaded)
7. [Memory Zones and Node Management](#memory-zones-and-node-management)
8. [Kernel Space vs User Space](#kernel-space-vs-user-space)
9. [Demand Paging and Copy-on-Write](#demand-paging-and-copy-on-write)
10. [Swapping and Page Cache](#swapping-and-page-cache)
11. [Inspecting Memory](#inspecting-memory)
12. [Performance Tuning](#performance-tuning)

---

## Why Virtual Memory?

Virtual memory provides:

- **Isolation**: Each process has its own address space; one process cannot corrupt another's memory.
- **Protection**: Read/write/execute permissions enforced at page granularity.
- **Abstraction**: Programs see a contiguous, private address space regardless of physical RAM layout.
- **Overcommitment**: Total virtual memory can exceed physical RAM (via swapping).
- **Shared memory**: Multiple processes can map the same physical pages (shared libraries, IPC).

---

## Virtual vs Physical Addresses

| Address Type | Description | Example |
|--------------|-------------|---------|
| **Virtual Address** | Address used by programs; CPU translates it via MMU | `0x00007fff12345678` |
| **Physical Address** | Actual RAM location | `0x00000001a8f00000` |

A program sees **only virtual addresses**. The kernel and MMU handle the mapping to physical RAM.

---

## The Memory Management Unit (MMU)

The MMU is hardware that translates virtual addresses to physical addresses using **page tables** set up by the kernel. On x86-64:

1. CPU generates a virtual address (VA).
2. MMU walks the page table hierarchy (PGD → PUD → PMD → PTE).
3. Finds the physical page frame number (PFN).
4. Combines PFN with the page offset to get the physical address (PA).
5. Accesses physical RAM.

If the page is not present (not in RAM), the MMU triggers a **page fault**, and the kernel loads the page from disk or allocates it on demand.

---

## Page Tables and Address Translation

### Page Size

Linux typically uses **4 KB pages** (configurable; some systems support huge pages: 2 MB, 1 GB).

A virtual address is split into:

```
 63          48 47     39 38     30 29     21 20     12 11           0
+-------------+---------+---------+---------+---------+---------------+
|   Sign Ext  |  PGD    |  PUD    |  PMD    |  PTE    |  Page Offset  |
+-------------+---------+---------+---------+---------+---------------+
```

- **PGD** (Page Global Directory) — level 4
- **PUD** (Page Upper Directory) — level 3
- **PMD** (Page Middle Directory) — level 2
- **PTE** (Page Table Entry) — level 1
- **Page Offset** — byte offset within the 4 KB page

### Multi-level Page Tables

x86-64 uses a **4-level page table** (5 levels with Intel's 5-level paging):

```
Virtual Address
       |
       v
+-------------+
|     CR3     |  <- points to PGD (per-process)
+-------------+
       |
       v
+-------------+
|     PGD     |  <- 512 entries (9 bits)
+-------------+
       |
       v
+-------------+
|     PUD     |  <- 512 entries
+-------------+
       |
       v
+-------------+
|     PMD     |  <- 512 entries
+-------------+
       |
       v
+-------------+
|     PTE     |  <- 512 entries (points to 4 KB physical page)
+-------------+
       |
       v
   Physical Page (4 KB)
```

Each PTE contains:

- **Physical Frame Number** (PFN)
- **Flags**: Present, Writable, User/Supervisor, Accessed, Dirty, NX (No-eXecute), etc.

---

## Linux Process Memory Layout

When a process runs, its virtual address space (on x86-64, typically 0x0000000000000000 to 0x00007fffffffffff for user space) looks like:

```
High Address (0x00007fffffffffff)
+-----------------------------------+
|           Stack                   |  ← grows downward
|         [frame, locals]           |
+-----------------------------------+
|          (guard page)             |
+-----------------------------------+
|        Memory-mapped region       |  ← mmap(), shared libs
|      (shared libraries, etc.)     |
+-----------------------------------+
|            Heap                   |  ← malloc(), grows upward
|        [dynamic allocs]           |
+-----------------------------------+
|            BSS                    |  ← uninitialized globals (zero-filled)
+-----------------------------------+
|            Data                   |  ← initialized globals/statics
+-----------------------------------+
|           Read-only data          |  ← .rodata (string literals, const)
+-----------------------------------+
|            Text                   |  ← program code (.text)
+-----------------------------------+
Low Address (0x0000000000000000)
```

**Kernel space** (0xffff800000000000 to 0xffffffffffffffff on x86-64) is reserved for the kernel and not directly accessible by user programs.

---

## How a Program is Loaded

### 1. exec() syscall

When you run a program, the shell calls `execve()`. The kernel:

1. **Validates the binary** (checks ELF header, architecture, permissions).
2. **Flushes the old address space** (if replacing a process).
3. **Allocates a new page table** for the process.

### 2. ELF Parsing

The kernel reads the ELF header and program headers (`PT_LOAD` segments):

- **Text segment**: maps `.text` (code) as read-only + executable.
- **Data segment**: maps `.data` and `.bss` as read-write.
- **Dynamic linker**: if present, maps the interpreter (e.g., `/lib64/ld-linux-x86-64.so.2`).

### 3. Memory Mapping (mmap)

Each `PT_LOAD` segment is mapped via `mmap()`:

```c
// kernel pseudo-code
mmap(vaddr, size, prot, MAP_PRIVATE|MAP_FIXED, fd, file_offset);
```

- **File-backed mapping**: `.text` and `.rodata` are backed by the ELF file on disk.
- **Anonymous mapping**: `.bss` is zero-filled, no file backing.

### 4. Demand Paging (Lazy Loading)

Pages are **not loaded immediately**. The kernel sets up page table entries marked **not present**. When the program accesses a page:

1. **Page fault** occurs.
2. Kernel checks the fault address.
3. If valid, kernel loads the page from disk (or zero-fills for `.bss`/heap).
4. Updates the PTE to point to the physical frame.
5. Resumes execution.

### 5. Stack Setup

The kernel allocates a stack (typically 8 MB default, configurable via `ulimit -s`), places `argc`, `argv`, `envp`, and auxiliary vectors at the top, and sets the stack pointer (`%rsp`).

### 6. Entry Point

The kernel sets the instruction pointer to the ELF entry point (usually `_start` in the C runtime, which calls `main()`).

---

## Memory Zones and Node Management

The kernel divides physical RAM into zones:

| Zone | Description |
|------|-------------|
| **ZONE_DMA** | Low memory for legacy DMA devices (< 16 MB on x86) |
| **ZONE_DMA32** | DMA-capable memory below 4 GB (x86-64) |
| **ZONE_NORMAL** | Directly mapped kernel memory |
| **ZONE_HIGHMEM** | High memory (on 32-bit systems with > ~896 MB RAM) |
| **ZONE_MOVABLE** | Pages that can be migrated (for memory hotplug) |

On NUMA systems, memory is divided into **nodes** (CPUs have local memory; accessing remote nodes is slower).

---

## Kernel Space vs User Space

| Region | Range (x86-64) | Description |
|--------|----------------|-------------|
| **User space** | `0x0000000000000000` – `0x00007fffffffffff` | Process virtual memory |
| **Kernel space** | `0xffff800000000000` – `0xffffffffffffffff` | Kernel code, data, direct-mapped RAM |

### Direct Mapping (Linear Mapping)

The kernel maintains a **direct mapping** of all physical RAM at a fixed offset (e.g., `0xffff888000000000` on x86-64). This allows the kernel to access any physical page by adding an offset:

```
physical_address = virtual_kernel_address - PAGE_OFFSET
```

### vmalloc

For non-contiguous allocations, the kernel uses `vmalloc()`, which allocates virtual addresses in a separate region and maps scattered physical pages.

---

## Demand Paging and Copy-on-Write

### Demand Paging

Pages are allocated **on first access**, not at `mmap()` time. Benefits:

- Faster process startup.
- Saves memory (unused code/data pages never loaded).

### Copy-on-Write (COW)

When `fork()` creates a child process:

1. Child gets a copy of the parent's page table.
2. All pages are marked **read-only** in both parent and child.
3. PTEs point to the **same physical pages**.
4. On write attempt, a **page fault** occurs:
   - Kernel copies the page.
   - Updates the faulting process's PTE to point to the new copy.
   - Marks the page writable.

This defers copying until necessary, saving memory and time.

---

## Swapping and Page Cache

### Swapping

When physical RAM is low, the kernel **swaps out** anonymous pages (heap, stack) to disk (swap partition or file). The PTE is updated to indicate the page is on disk. On access, a page fault brings it back.

### Page Cache

File-backed pages (executables, memory-mapped files) live in the **page cache**. When memory is tight, the kernel can **evict** clean pages (not modified) without writing them to disk (they can be re-read from the file).

---

## Virtual-to-Physical Translation Example

### Example: Translating `0x00007fff12345678`

Assume 4 KB pages (12-bit offset).

```
Virtual Address: 0x00007fff12345678

Binary (48-bit addressing):
   PGD index: bits 47-39 → 0x0FF (entry 255)
   PUD index: bits 38-30 → 0x1FC (entry 508)
   PMD index: bits 29-21 → 0x091 (entry 145)
   PTE index: bits 20-12 → 0x145 (entry 325)
   Offset:    bits 11-0  → 0x678 (byte 1656 within page)
```

1. CPU reads CR3 (PGD base address): `0x00000001a0000000`.
2. Index into PGD: `PGD[255]` → points to PUD at `0x00000001a1000000`.
3. Index into PUD: `PUD[508]` → points to PMD at `0x00000001a2000000`.
4. Index into PMD: `PMD[145]` → points to PTE at `0x00000001a3000000`.
5. Index into PTE: `PTE[325]` → PFN = `0x00000001b4567`.
6. Physical address = `(PFN << 12) | offset` = `0x00000001b4567000 | 0x678` = `0x00000001b4567678`.

---

## Inspecting Memory

### `/proc/<pid>/maps`

Shows memory regions for a process:

```bash
cat /proc/self/maps
```

Example output:

```
00400000-00401000 r-xp 00000000 08:01 12345  /bin/cat       # .text
00600000-00601000 r--p 00000000 08:01 12345  /bin/cat       # .rodata
00601000-00602000 rw-p 00001000 08:01 12345  /bin/cat       # .data
01a00000-01a21000 rw-p 00000000 00:00 0      [heap]
7fff12300000-7fff12321000 rw-p 00000000 00:00 0  [stack]
```

Columns: `start-end perms offset dev inode pathname`

### `/proc/<pid>/smaps`

Detailed per-region memory statistics (RSS, shared, private, swap).

### `/proc/<pid>/pagemap`

Exposes virtual-to-physical mappings (requires root):

```bash
sudo cat /proc/self/pagemap | od -An -tx8 -w8
```

Each 64-bit entry encodes PFN, flags, and swap information.

### `pmap`

High-level view of a process's memory:

```bash
pmap -x <pid>
```

### `gdb` and Memory Inspection

```bash
gdb -p <pid>
(gdb) info proc mappings
(gdb) x/16xb 0x00007fff12345678  # dump memory
```

---

## Page Faults

### Types

| Type | Cause | Handler |
|------|-------|---------|
| **Minor fault** | Page in page cache or not yet allocated (demand paging) | Kernel updates PTE, no I/O |
| **Major fault** | Page must be loaded from disk | Kernel reads from file/swap |
| **Invalid fault** | Access to unmapped or protected memory | Kernel sends SIGSEGV |

### Observing Faults

```bash
ps -o pid,min_flt,maj_flt,cmd <pid>
```

Or watch in real-time:

```bash
perf stat -e page-faults,minor-faults,major-faults -p <pid>
```

---

## Huge Pages

Normal pages are 4 KB. **Huge pages** (2 MB or 1 GB) reduce TLB pressure and page table overhead.

### Transparent Huge Pages (THP)

Kernel automatically promotes 4 KB pages to 2 MB:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
```

### Explicit Huge Pages

```bash
echo 1024 > /proc/sys/vm/nr_hugepages  # reserve 1024 × 2 MB
```

Applications use `mmap()` with `MAP_HUGETLB` or `shmget()` with `SHM_HUGETLB`.

---

## Memory Allocators and Kernel Interfaces

| Interface | Description |
|-----------|-------------|
| `brk()/sbrk()` | Grow/shrink the heap (legacy, used by `malloc`) |
| `mmap()` | Map files or anonymous memory |
| `munmap()` | Unmap memory |
| `mprotect()` | Change protection (read/write/exec) |
| `madvise()` | Hint kernel about usage patterns (e.g., `MADV_DONTNEED`) |
| `mlock()/mlockall()` | Lock pages in RAM (prevent swapping) |

---

## Memory Overcommit

Linux allows processes to allocate more virtual memory than available physical RAM (overcommit). Controlled by `/proc/sys/vm/overcommit_memory`:

| Mode | Behavior |
|------|----------|
| `0` | Heuristic (default): check available memory, allow reasonable overcommit |
| `1` | Always allow overcommit |
| `2` | Strict accounting (no overcommit beyond swap + `overcommit_ratio` of RAM) |

When physical memory runs out, the **OOM killer** terminates processes based on score (`/proc/<pid>/oom_score`).

---

## NUMA and Memory Policies

On NUMA systems, memory allocation policies affect performance:

```bash
numactl --membind=0 ./program    # bind to node 0
numactl --interleave=all ./program   # interleave across nodes
```

View NUMA layout:

```bash
numactl --hardware
```

---

## Kernel Memory Management Data Structures

| Structure | Purpose |
|-----------|---------|
| `struct task_struct` | Process descriptor (includes `mm_struct` pointer) |
| `struct mm_struct` | Process memory descriptor (page tables, VMAs) |
| `struct vm_area_struct` (VMA) | Describes a contiguous virtual memory region |
| `struct page` | Describes a physical page frame |

Each VMA represents a region like heap, stack, or mmap. Inspect with `/proc/<pid>/maps`.

---

## TLB (Translation Lookaside Buffer)

The TLB is a CPU cache for page table entries. Without it, every memory access would require multiple memory reads (page table walk). The kernel flushes the TLB on context switch or when page tables change.

### TLB Shootdown

On multi-core systems, changing a page table requires invalidating TLB entries on all CPUs (**TLB shootdown**), which can be expensive.

---

## Performance Tuning

### Monitor Memory Usage

```bash
free -h                  # RAM and swap usage
vmstat 1                 # virtual memory stats
cat /proc/meminfo        # detailed memory info
cat /proc/zoneinfo       # zone allocator info
```

### Reduce Page Faults

- Use `mlock()` for latency-sensitive pages.
- Pre-fault memory with `mmap(..., MAP_POPULATE)` or by touching pages.
- Use huge pages for large memory regions.

### Control Swappiness

```bash
cat /proc/sys/vm/swappiness   # 0-100 (default 60)
echo 10 > /proc/sys/vm/swappiness   # prefer keeping pages in RAM
```

### Memory Pressure Notifications

Use `cgroups` memory controller and eventfd for memory pressure notifications (useful for containers).

---

## Security and Isolation

### ASLR (Address Space Layout Randomization)

Randomizes stack, heap, mmap, and (with PIE) code addresses:

```bash
cat /proc/sys/kernel/randomize_va_space
# 0 = off, 1 = partial, 2 = full (default)
```

### Memory Protection

- **NX bit**: prevents execution of data pages (stack, heap).
- **SMEP/SMAP**: (x86) Supervisor Mode Execution/Access Prevention — kernel cannot execute/access user pages.
- **Kernel Page Table Isolation (KPTI)**: mitigates Meltdown by separating kernel and user page tables.

---

## Summary

Linux memory management is a multi-layer system:

1. **Programs see virtual addresses** in a contiguous address space.
2. **MMU and page tables** translate virtual to physical addresses.
3. **Demand paging** loads pages on first access; **COW** defers copying.
4. **Kernel manages physical RAM** via zones, buddy allocator, slab caches.
5. **Swapping and page cache** extend usable memory beyond physical RAM.
6. **Tools like `/proc/<pid>/maps`, `pmap`, `vmstat`** let you inspect and tune memory.

Understanding this architecture helps with debugging, performance optimization, and secure system design.

---

## References

- "Understanding the Linux Kernel" — Bovet & Cesati
- "Linux Kernel Development" — Robert Love
- `man 5 proc` — `/proc` filesystem
- [Linux Memory Management Documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/)
- Intel/AMD architecture manuals (paging, MMU)
- `man mmap`, `man mprotect`, `man madvise`

---

*See also:*
- [memory_layout.md](memory_layout.md) — process memory layout
- [elf.md](elf.md) — ELF binary structure
