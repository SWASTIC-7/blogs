# ELF — Executable and Linkable Format Explained

ELF is the standard binary format on Linux, BSD, and many UNIX-like systems. When you compile a C/C++/Rust program, the compiler and linker produce an ELF file that the OS loader reads to set up process memory. This post breaks down the structure of an ELF file: headers, sections, segments, and how your source code maps to `.text`, `.data`, `.bss`, `.rodata`, and more.

---

## Why ELF?

ELF provides a flexible, extensible format for:

- **Executables** — programs you run directly.
- **Shared objects (.so)** — dynamically linked libraries.
- **Relocatable objects (.o)** — intermediate compiler output before linking.
- **Core dumps** — process memory snapshots for debugging.

Understanding ELF helps with debugging, reverse engineering, binary hardening, and low-level systems programming.

---

## High-level ELF structure

```
+---------------------------+
|       ELF Header          |  <- magic, arch, entry point, offsets
+---------------------------+
|   Program Header Table    |  <- segments (used at runtime)
+---------------------------+
|                           |
|        Sections           |  <- .text, .data, .bss, .rodata, ...
|                           |
+---------------------------+
|   Section Header Table    |  <- section metadata (used by linker/debugger)
+---------------------------+
```

- **ELF Header**: always at offset 0; identifies the file as ELF, specifies architecture, entry point, and offsets to the header tables.
- **Program Header Table**: describes *segments* — contiguous memory regions the loader maps into the process (used at runtime).
- **Sections**: logical groupings of code/data (`.text`, `.data`, etc.) — used by the linker and debugger.
- **Section Header Table**: metadata about each section (name, type, size, flags).

---

## ELF Header

The ELF header (first 52 or 64 bytes for 32/64-bit) contains:

| Field | Description |
|-------|-------------|
| `e_ident` | Magic bytes (`\x7fELF`), class (32/64), endianness, OS ABI |
| `e_type` | Object file type: `ET_EXEC` (executable), `ET_DYN` (shared object), `ET_REL` (relocatable) |
| `e_machine` | Target architecture (x86, x86-64, ARM, RISC-V, etc.) |
| `e_entry` | Virtual address of program entry point (`_start` or similar) |
| `e_phoff` | Offset to program header table |
| `e_shoff` | Offset to section header table |
| `e_phnum` | Number of program headers |
| `e_shnum` | Number of section headers |
| `e_shstrndx` | Index of section header string table (names of sections) |

**Inspect with:**

```bash
readelf -h <binary>
```

---

## Sections vs Segments

| Concept | Purpose | Used by |
|---------|---------|---------|
| **Section** | Logical grouping (code, data, symbols, debug info) | Linker, debugger, `objdump`, `readelf` |
| **Segment** | Contiguous memory region to load | OS loader at runtime |

Multiple sections can be grouped into one segment. For example, `.text` and `.rodata` may share a single read-only, executable segment.

---

## Common Sections

### `.text` — Executable code

Machine instructions of your program. Marked **read-only** and **executable**.

```c
int add(int a, int b) { return a + b; }
// ↑ compiled code goes into .text
```

### `.rodata` — Read-only data

String literals, `const` globals, jump tables.

```c
const char *msg = "Hello";   // "Hello" stored in .rodata
```

### `.data` — Initialized writable data

Global and static variables initialized to non-zero values.

```c
int counter = 42;            // stored in .data
```

### `.bss` — Uninitialized data (Block Started by Symbol)

Global and static variables that are zero-initialized or uninitialized. The section occupies **no space in the file** but is allocated (and zeroed) at load time.

```c
static int buffer[1024];     // goes into .bss (zero-initialized)
int uninit;                  // also .bss
```

### `.symtab` / `.dynsym` — Symbol tables

Maps symbol names (functions, variables) to addresses. `.symtab` is the full table (often stripped); `.dynsym` is the dynamic symbol table for shared libraries.

### `.strtab` / `.dynstr` — String tables

Null-terminated strings referenced by symbol and section tables.

### `.rel.text` / `.rela.text` — Relocation entries

Tell the linker how to patch addresses (e.g., when linking object files or loading shared libraries).

### `.plt` / `.got` — Procedure Linkage Table / Global Offset Table

Used for lazy binding of external functions in dynamically linked executables.

### `.init` / `.fini` — Initialization and termination

Code run before `main()` (constructors) and after `main()` returns (destructors).

### `.eh_frame` / `.eh_frame_hdr` — Exception handling

Unwind information for C++ exceptions and stack traces.

### `.debug_*` — DWARF debug info

Source-level debugging information (line numbers, variable locations). Present when compiled with `-g`.

---

## Section Flags

Each section has flags indicating permissions and behavior:

| Flag | Meaning |
|------|---------|
| `SHF_WRITE` | Section is writable |
| `SHF_ALLOC` | Section occupies memory at runtime |
| `SHF_EXECINSTR` | Section contains executable code |

**Typical mappings:**

| Section | Flags |
|---------|-------|
| `.text` | `ALLOC + EXECINSTR` |
| `.rodata` | `ALLOC` |
| `.data` | `ALLOC + WRITE` |
| `.bss` | `ALLOC + WRITE` |

---

## Program Headers (Segments)

The program header table describes segments the loader maps into memory:

| Type | Description |
|------|-------------|
| `PT_LOAD` | Loadable segment (code, data) |
| `PT_DYNAMIC` | Dynamic linking information |
| `PT_INTERP` | Path to dynamic linker (`/lib64/ld-linux-x86-64.so.2`) |
| `PT_NOTE` | Auxiliary information |
| `PT_GNU_STACK` | Stack executability (NX enforcement) |
| `PT_GNU_RELRO` | Read-only after relocation (security hardening) |

Each `PT_LOAD` segment specifies:

- **Virtual address** — where to map in memory.
- **File offset / size** — where in the ELF file.
- **Memory size** — may be larger than file size (for `.bss`).
- **Flags** — `PF_R` (read), `PF_W` (write), `PF_X` (execute).

**Inspect with:**

```bash
readelf -l <binary>
```

---

## How source code maps to ELF sections

```c
#include <stdio.h>

const char *greeting = "Hello";   // "Hello" → .rodata
int global_init = 100;            // → .data
int global_uninit;                // → .bss
static int file_static;           // → .bss

void say_hello(void) {            // → .text
    static int call_count = 0;    // → .data (initialized static)
    call_count++;
    printf("%s\n", greeting);
}

int main(void) {                  // → .text
    say_hello();
    return 0;
}
```

After compiling (`gcc -o hello hello.c`), use:

```bash
objdump -t hello | grep -E "greeting|global_init|global_uninit|file_static|call_count|say_hello|main"
```

You'll see each symbol's section (`.text`, `.data`, `.bss`, `.rodata`).

---

## Inspecting ELF files

### `readelf` — detailed ELF info

```bash
readelf -h hello          # ELF header
readelf -S hello          # section headers
readelf -l hello          # program headers (segments)
readelf -s hello          # symbol table
readelf -r hello          # relocations
readelf -d hello          # dynamic section
```

### `objdump` — disassembly and sections

```bash
objdump -h hello          # section headers with sizes
objdump -d hello          # disassemble .text
objdump -s -j .rodata hello   # dump .rodata contents
```

### `nm` — symbol listing

```bash
nm hello                  # list symbols
nm -C hello               # demangle C++ names
```

### `size` — section sizes summary

```bash
size hello
# text    data    bss     dec     hex     filename
# 1234    256     8       1498    5da     hello
```

---

## Linker scripts and custom sections

You can define custom sections using GCC attributes:

```c
__attribute__((section(".my_section")))
int special_data = 0xDEAD;
```

Linker scripts (`.ld` files) control section placement, useful for embedded systems and kernel development.

---

## Security features in ELF

| Feature | Description |
|---------|-------------|
| **RELRO** | Mark GOT read-only after relocation (`-Wl,-z,relro,-z,now`) |
| **NX / No-exec stack** | `PT_GNU_STACK` with no execute flag |
| **PIE** | Position Independent Executable (full ASLR) |
| **Stack canaries** | Compiler-inserted stack protectors (`-fstack-protector`) |
| **FORTIFY_SOURCE** | Runtime bounds checking for common functions |

Check binary hardening:

```bash
checksec --file=hello
```

---

## Typical ELF memory layout at runtime

When the loader maps an ELF executable:

```
High addresses
+---------------------------+
|         Stack             |  ← grows down
+---------------------------+
|     Memory-mapped region  |  ← shared libs, mmap
+---------------------------+
|          Heap             |  ← grows up (brk/sbrk)
+---------------------------+
|          .bss             |  ← zero-initialized
+---------------------------+
|         .data             |  ← initialized globals
+---------------------------+
|        .rodata            |  ← read-only data
+---------------------------+
|         .text             |  ← executable code
+---------------------------+
Low addresses
```

---

## Quick reference: ELF section summary

| Section | Contents | In file? | Writable? | Executable? |
|---------|----------|----------|-----------|-------------|
| `.text` | Code | Yes | No | Yes |
| `.rodata` | Constants, strings | Yes | No | No |
| `.data` | Initialized globals | Yes | Yes | No |
| `.bss` | Uninitialized globals | No (zero-fill) | Yes | No |
| `.symtab` | Symbols | Yes | — | — |
| `.strtab` | Strings | Yes | — | — |
| `.plt`/`.got` | Dynamic linking | Yes | Varies | Varies |
| `.debug_*` | Debug info | Yes | — | — |

---

## Summary

ELF organizes your compiled program into:

1. **Headers** — metadata for the loader and linker.
2. **Sections** — logical units (`.text`, `.data`, `.bss`, `.rodata`, etc.).
3. **Segments** — memory regions mapped at runtime.

Understanding ELF helps you:

- Debug linking errors and missing symbols.
- Inspect and harden binaries.
- Work with embedded systems and custom loaders.
- Reverse engineer and analyze binaries.

---

## Further reading

- `man elf` — ELF format specification.
- [ELF specification (PDF)](https://refspecs.linuxfoundation.org/elf/elf.pdf)
- `readelf`, `objdump`, `nm` man pages.
- "Linkers and Loaders" by John R. Levine.
- DWARF debugging format specification.

---

*See also: [memory_layout.md](memory_layout.md) for how memory is structured at runtime.*
