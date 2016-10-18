master branch: [![Build Status (gcc/clang/tcc)](https://travis-ci.org/faragon/libsrt.svg?branch=master)](https://travis-ci.org/faragon/libsrt) [![Build Status (vs win32)](https://ci.appveyor.com/api/projects/status/bw52jnfgv7qff6x4/branch/master?svg=true)](https://ci.appveyor.com/project/faragon/libsrt) [![Build Status](https://drone.io/github.com/faragon/libsrt/status.png)](https://drone.io/github.com/faragon/libsrt/latest)

master branch analysis: [![Build Status](https://scan.coverity.com/projects/5366/badge.svg)](https://scan.coverity.com/projects/5366)

documentation (generated by doc/mk\_doc.sh): [HTML](https://faragon.github.io/libsrt.html)

libsrt has been included into Paul Hsieh's [String Library Comparisons](http://bstring.sourceforge.net/features.html) table. What a great honor! :-)

libsrt: Safe Real-Time library for the C programming language
===

libsrt is a C library that provides string, vector, bit set, and map handling. It's been designed for avoiding explicit memory management, allowing safe and expressive code, while keeping high performance. It covers basic needs for writing high level applications in C without worrying about managing dynamic size data structures. It is also suitable for low level and hard real time applications, as functions are predictable in both space and time (asuming OS and underlying C library is also real-time suitable).

Key points:

* Easy: write high-level-like C code. Write code faster and safer.
* Fast: using O(n)/O(log n)/O(1) state of the art algorithms.
* Useful: UTF-8 strings, vector, tree, and map structures.
* Efficient: space optimized, minimum allocation calls (heap and stack support).
* Compatible: OS-independent (e.g. built-in space-optimized UTF-8 support).
* Predictable: suitable for microcontrollers and hard real-time compliant code.
* Unicode: UTF-8 support for strings, including binary/raw data cases.

Generic advantages
===

* Ease of use
 * Use strings in a similar way to higher level languages.

* Space-optimized
 * Dynamic one-block linear addressing space.
 * Internal structures use indexes instead of pointers (i.e. similar memory usage in both 32 and 64 bit mode).
 * More information: doc/benchmarks.md

* Time-optimized
 * Buffer direct access
 * Preallocation hints (reducing memory allocation calls)
 * Heap and stack memory allocation support
 * More information: doc/benchmarks.md

* Predictable (suitable for hard and soft real-time)
 * Predictable execution speed: all API calls have documented time complexity. Also space complexity, when extra space involving dynamic memory is required.
 * Hard real-time: allocating maximum size (strings, vector, trees, map) will imply calling 0 or 1 times to the malloc() function (C library). Zero times if using the stack as memory or externally allocated memory pool, and one if using the heap. This is important because malloc() implementation has both memory and time overhead, as internally manages a pool of free memory (which could have O(1), O(log(n)), or even O(n), depending on the compiler provider C library implementation).
 * Soft real-time: logarithmic time memory allocation (when not doing preallocation): for 10^n new elements just log(n) calls to the malloc() function. This is not a guarantee for hard real time, but for soft real time (being careful could provide almost same results, except in the case of very poor malloc() implementation in the C library being used -not a problem with modern compilers-).

* RAM, ROM, and disk operation
 * Data structures can be stored in ROM memory.
 * Data structures are suitable for memory mapped operation, and disk store/restore (full when not using pointers but explicit data)

* Known edge case behavior
 * Allowing both "carefree code" and per-operation error check. I.e. memory errors and UTF8 format error can be checked after every operation.

* Implementation
 * Simple internal structures, with linear addressing. It allows to reduce memory allocation calls to the minimum (using the stack it is possible to avoid heap usage).

* Test-covered
 * Implementation kept as simple as possible/known.
 * No duplicated code for handling the different data structures: at logic level are the same, and differences are covered using a minimal abstraction for accessing the meta-information (string size, alloc size, flags, etc.).
 * Small code, suitable for static linking.
 * Coverage, profiling, and memory leak/overflow/uninitialized checks (Valgrind, GNU gprof -not fully automated into the build, yet-).

* Compatibility
 * C99 (and later) compatible
 * C++11 (and later) compatible
 * CPU-independent: endian-agnostic, aligned memory accesses
 * POSIX builds (Linux, BSD's) require GNU Make (e.g. on FreeBSD use 'gmake' instead of 'make')

Generic disadvantages/limitations
===

* Double pointer usage: because of using just one allocation, wrte operations require to address a double pointer, so in the case of reallocation the source pointer could be changed.

* Without the resizing heuristic one-by-one increment would be slow, as it makes to copy data content (with the heuristic the cost it is not only amortized, but much cheaper than allocating things one-by-one).

* Concurrent read-only operations is safe, but concurrent read/write must be protected by the user (e.g. using mutexes or spinlocks). That can be seen as a disadvantage or as a "feature" (it is faster).

* It may work in C89 mode only if following language extensions are available:
 * \_\_VA\_ARGS\_\_ macro
 * alloca()
 * type of bit-field in 'struct'
 * %S printf extension (only for unit testing)
 * Format functions (\*printf) rely on system C library, so be aware if you write multi-platform software before using compiler-specific extensions or targetting different C standards).
 * Allow mixed code and variable declaration.

String-specific advantages (ss\_t)
===

* Unicode support
 * Internal representation is in UTF-8
 * Search and replace into UTF-8 data is supported
 * Full and fast Unicode lowercase/uppercase support without requiring "setlocale" nor hash tables.
* Binary data support
 * I.e. strings can have zeros in the middle
 * Search and replace into binary data is supported
* Efficient raw and Unicode (UTF-8) handling. Unicode size is tracked, so resulting operations with cached Unicode size, will keep that, keeping the O(1) for getting that information afterwards.
 * Find/search: O(n), one pass.
 * Replace: O(n), one pass. Worst case overhead is limited to a realloc and a copy of the part already processed.
 * Concatenation: O(n), one pass for multiple concatenation. I.e. Optimal concatenation of multiple elements require just one allocation, which is computed before the concatination. When concatenating ss\_t strings the allocation size compute time is O(1).
 * Resize: O(n) for worst case (when requiring reallocation for extra space. O(n) for resize giving as cut indicator the number of Unicode characters. O(1) for cutting raw data (bytes).
 * Case conversion: O(n), one pass, using the same input if case conversion requires no allocation over current string capacity. If resize is required, in order to keep O(n) complexity, the string is scanned for computing required size. After that, the conversion outputs to the secondary string. Before returning, the output string replaces the input, and the input becomes freed.
 * Avoid double copies for I/O (read access, write reserve)
 * Avoid re-scan (e.g. commands with offset for random access)
 * Transformation operations are supported in all dup/cpy/cat functions, in order to both increase expressiveness and avoid unnecessary copies (e.g. tolower, erase, replace, etc.). E.g. you can both convert to lower a string in the same container, or copy/concatenate to another container.
* Space-optimized
 * Using just 4 byte overhead for strings with size <= 255 bytes
 * Using sizeof(size\_t) * 5 byte overhead for strings with size >= 256 bytes (e.g. 20 bytes for a 32-bit CPU, 40 for 64-bit)
 * Data structure has no pointers, i.e. just one allocation is required for building a string. Or zero, if using the stack.
 * No additional memory allocation for search.
 * Extra memory allocation may be required for: UTF-8 uppercase/lowercase and replace.
 * Strings can grow from 0 bytes to ((size\_t)~0 - metainfo\_size)
* String operations
 * copy, cat, tolower/toupper, find, split, printf, cmp, etc.
 * allocation, buffer pre-reserve,
 * Raw binary content is allowed, including 0's.
 * "Wide char" and "C style" strings R/W interoperability support.
 * I/O helpers: buffer read, reserve space for async write
 * Aliasing suport, e.g. ss\_cat(&a, a) is valid
* Focus on reducing verbosity:
 * ss\_cat(&t, s1, ..., sN);
 * ss\_cat(&t, s1, s2, ss\_printf(&s3, "%i", cnt), ..., sN);
 * ss\_free(&s1, &s2, ..., &sN);
 * Expressive code without explicit memory handling
* Focus on reducing errors, e.g.
 * If a string operation fails, the string is kept in the last successful state (e.g. ss\_cat(&a, b, huge\_string, other))
 * String operations always return valid strings, e.g.
		This is OK:
			ss\_t *s = NULL;
			ss_cpy_c(&s, "a");
		Same behavior as:
			ss\_t *s = ss_dup_c("a");
 * ss\_free(&s1, ..., &sN);  (no manual set to NULL is required)

String-specific disadvantages/limitations
===

* No reference counting support. Rationale: simplicity.
* ss\_t has not string precomputed hash support, yet. So searching on trees when keys are strings is not yet optimal (it will be almost as fast as searching for an integer).

Vector-specific advantages (sv\_t)
===

* Variable-length concatenation and push functions.
* Allow explicit size for allocation (8, 16, 32, 64 bits) with size-agnostic generic signed/unsigned functions (easier coding).
* Allow variable-size generic element.
* Sorting.

Vector-specific disadvantages/limitations
===

* No insert function. Rationale: insert is slow (O(n)). Could be added, if someone asks for it.

Map-specific advantages (sm\_t)
===

* Abstraction over Red-Black tree implementation using linear memory pool with just 8 byte per node overhead, allowing up to (2^32)-1 nodes (for both 32 an 64 bit compilers). E.g. one million 32 bit key, 32 bit value map will take just 16MB of memory (16 bytes per element \-8 byte metadata, 4 + 4 byte data\-).
* Keys: integer (8, 16, 32, 64 bits) and string (ss\_t)
* Values: integer (8, 16, 32, 64 bits), string (ss\_t), and pointer
* O(1) for allocation
* O(1) for deleting maps without strings (one or zero calls to 'free' C function)
* O(n) for deleting maps with strings (n + one or zero calls to 'free' C function)
* O(n) for map copy (in case of maps without strings, would be as fast as a memcpy())
* O(log n) insert, search, delete
* O(n) sorted enumeration (amortized O(n log n))
* O(n) unsorted enumeration (faster than the sorted case)
* O(n) copy: tree structure is copied as fast as a memcpy(). For map types involving strings, additional allocation is used for duplicating strings.

Map-specific disadvantages/limitations
===

* Because of being implemented as a tree, it is slower than a hash-map, on average. However, in total execution time is not that bad, as because of allocation heuristics a lot of calls to the allocator are avoided.
* There is room for node deletion speed up (currently deletion is a bit slower than insertion, because of an additional tree search used for avoiding having memory fragmentation, as implementation guarantees linear/compacted memory usage, it could be optimized with a small LRU for cases of multiple delete/insert operation mix).

Test-covered platforms
===

| ISA | Word size | Endianess | Unaligned memory access HW support | OS | Compilers | Code analysis | Test coverage |
| --- | --------- | --------- | ---------------------------------- | --- | --------- | ------------- | ------------- |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | Linux Ubuntu 12.04/14.04 | gcc, g++, tcc, clang, clang++ | Valgrind, clang, Coverity | Travis CI (automatic, every public commit) |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | Windows | Visual Studio Express 2013, AppVeyor's VS | VS | AppVeyor (automatic, every public commit) |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | FreeBSD 10.2 | gcc, g++, clang, clang++ | Valgrind clang | manual |
| ARMv5 (ARM926EJ-S) | 32 | little | no | Arch Linux | gcc, g++, clang, clang++ | none | manual |
| ARMv5 (Feroceon) | 32 | little | no | Linux Debian 7.0 "Wheezy" | gcc, g++ | none | manual |
| ARMv6 (ARM1176JZF-S) | 32 | little | yes | Linux Raspbian | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| ARMv7-A (Krait 400) | 32 | little | yes | Linux Android 5.1.1 + BusyBox | gcc, g++ | none | manual |
| ARMv8-A (Cortex A53) | 64 | little | yes | Debian 8.5 "Jessie" | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| MIPS, MIPS64 (Octeon) | 32, 64 | big | yes | EdgeOS v1.6.0 (Linux Vyatta-based using Debian 7 "Wheezy" packages) | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| PowerPC (G4) | 32 | big | yes | Linux Ubuntu 12.04 | gcc, g++ | none | manual |

Coming soon (systems that require some changes in the test, e.g. because RAM/ROM constraints, etc.):

| ISA | Word size | Endianess | Unaligned memory access HW support | OS | Compilers | Code analysis | Test coverage |
| --- | --------- | --------- | ---------------------------------- | --- | --------- | ------------- | ------------- |
| ARMv6-M (Cortex M0) | 32 | little | no | none | gcc | none | manual |
| ARMv7-M (Cortex M3) | 32 | little | no | RT-Thread OS | gcc | none | manual |
| MIPS32 r2, MIPS16e M14K (PIC32-EMZ64) | 32 | little | no | none | gcc | none | manual |
| MIPS32 (mips24k) | 32 | big | no | OpenWrt (Chaos Calmer, r47374) | gcc | none | manual |

License
===

Copyright (c) 2015-2016, F. Aragon. All rights reserved.
Released under the BSD 3-Clause License (see the doc/LICENSE file included).

Contact
===

email: faragon.github (GMail account, add @gmail.com)

Other
===

Status
---

Beta. API still can change.

"to do" list
---

Check [doc/todo.md](https://github.com/faragon/libsrt/blob/master/doc/todo.md)


Acknowledgements and references
---

Check [doc/references.md](https://github.com/faragon/libsrt/blob/master/doc/references.md)

