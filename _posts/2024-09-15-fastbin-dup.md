---
title: "Heap Exploitation : Fast Bin Dup Technique"
author: "Bjørn-Ivar Bekkevold"
date: "2024-09-15"
layout: post
categories: post
---
# Heap Exploitation Series: Fastbin Dup Technique

This is a writeup for the fastbin duplication challenge. Based on the Linux Heap Exploitation course by [Max Kemper](www.udemy.com)


#### Required Tools:
- Pwndbg
- Python library Pwntools
- x86 64-bit Linux Environment
- GNU toolchain


### Malloc introduction

Fastbin dup is a technique that utilizes a double free vulnerability in heap memory allocation. Targeting the malloc implementation in certain versions of glibc.

Heaps are contigious blocks of dynamic memory allocated to an executing program.
In the glibc Malloc implementation, the heap is reprented in chunks. And can be considered the basis primitive of raw heap memory.
Detailed in the table below, a chunk can comprise of different sizes starting from 32 bytes
as its smallest unit.
Chunks increase in size in 0x10 increments. And each chunk has atleast 16 bytes of metadata when in use.
Meaning a smallest possible unit of a chunk will have 16 bytes of user data available for a requesting program.
The size field, comprises of 8 bytes, with the least significant nibble, being reserved for flags indicating the state that the chunk is in. 

The heap will instantiate chunks from what is called a Top Chunk. claiming a part of the size reserve of a Top Chunk, which has its very own size field.
This represents the total amount of available heap memory allocated to a process. Where the creation and allocation of a chunk subtracts the requested chunk size from the Top Chunk size field. 

<table>
  <thead>
    <tr>
      <th style="background-color:#2F4F4F; color:white;">Field</th>
      <th style="background-color:#4F4F4F; color:white;">Size (Bytes)</th>
      <th style="background-color:#696969; color:white;">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Prev Size (optional)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Size of the previous chunk (only in free chunks, not in fastbins).</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Size</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Size of the current chunk</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Forward Pointer (fd)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the next free chunk in the bin (only in free chunks).</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Backward Pointer (bk)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the previous free chunk in the bin (only in free chunks).</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>User Data</strong></td>
      <td style="background-color:#4F4F4F; color:white;">Varies</td>
      <td style="background-color:#696969; color:white;">Memory space available to the program.</td>
    </tr>
  </tbody>
</table>

<table>
  <thead>
    <tr>
      <th style="background-color:#2F4F4F; color:white;">Flag Name</th>
      <th style="background-color:#4F4F4F; color:white;">Bit Position</th>
      <th style="background-color:#696969; color:white;">Size (Bits)</th>
      <th style="background-color:#808080; color:white;">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>PREV_INUSE</strong></td>
      <td style="background-color:#4F4F4F; color:white;">0</td>
      <td style="background-color:#696969; color:white;">1 bit</td>
      <td style="background-color:#808080; color:white;">Indicates whether the previous chunk is in use (allocated). If set, the previous chunk is allocated; if unset, it is free.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>M</strong></td>
      <td style="background-color:#4F4F4F; color:white;">1</td>
      <td style="background-color:#696969; color:white;">1 bit</td>
      <td style="background-color:#808080; color:white;">Indicates whether the chunk is part of a non-main arena (used in multithreaded programs).</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>IS_MMAPPED</strong></td>
      <td style="background-color:#4F4F4F; color:white;">2</td>
      <td style="background-color:#696969; color:white;">1 bit</td>
      <td style="background-color:#808080; color:white;">Indicates that the chunk was allocated via <code>mmap</code> instead of the usual heap allocation methods (e.g., <code>sbrk</code>).</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>SIZE_BITS</strong></td>
      <td style="background-color:#4F4F4F; color:white;">3 and higher</td>
      <td style="background-color:#696969; color:white;">Remaining bits</td>
      <td style="background-color:#808080; color:white;">The remaining bits represent the actual size of the chunk, excluding the metadata. This value is aligned to a multiple of 8 or 16 bytes.</td>
    </tr>
  </tbody>
</table>



In an executing Linux process, Malloc manages dynamic memory allocation through a structure called an Arena.
Arenas are used to keep track and manage heap allocations in different threads, and aids in improving performance
and scalability, avoiding possible heap contention in multithreaded applications.

An arena can be seen as a table structure, and keeps an overview of the allocated chunks via another structure called bins.
Bins can be considered simple linked list primitives based as a stack queue structure.
With the elements in the queue pointing to free chunks of different sizes.
Bins are divided into categories, primarily based their size and usage, and are labeled as fastbin, smallbin, largebin and unsortedbin.

The table below depicts the overall structure of an arena, with parts of the table pointing to the different 
fastbin queues. Each fastbin is dedicated to chunks of a respective size. Used for fast allocation and deallocation of chunks already instantiated. And in some ways can be considered a caching system for managing dynamic memory.


<table>
  <thead>
    <tr>
      <th style="background-color:#2F4F4F; color:white;">Field</th>
      <th style="background-color:#4F4F4F; color:white;">Size (Bytes)</th>
      <th style="background-color:#696969; color:white;">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Mutex (mutex)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Lock to ensure thread safety during memory operations.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Next Arena (next)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the next arena in the linked list of arenas.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Fastbin Array</strong></td>
      <td style="background-color:#4F4F4F; color:white;">80 bytes (10 pointers)</td>
      <td style="background-color:#696969; color:white;">Array of pointers to heads of fastbin lists, managing small chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 0 (16 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 16-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 1 (24 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 24-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 2 (32 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 32-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 3 (40 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 40-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 4 (48 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 48-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 5 (56 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 56-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 6 (64 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 64-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 7 (72 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 72-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 8 (80 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 80-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;">&nbsp;&nbsp;&nbsp;- Fastbin 9 (88 bytes)</td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes (pointer)</td>
      <td style="background-color:#696969; color:white;">Pointer to the fastbin for 88-byte chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Top Chunk (top)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the top chunk, the largest free chunk in the arena.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Last Remainder (last_remainder)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the last remainder chunk, a leftover from a previous allocation.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Bins (bins)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">2032 bytes (254 pointers)</td>
      <td style="background-color:#696969; color:white;">Array of pointers to heads of bins, managing larger free chunks.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Binmap (binmap)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">32 bytes (4 pointers)</td>
      <td style="background-color:#696969; color:white;">Bitmap tracking which bins are currently in use.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Next Unsorted Chunk (next_chunk)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the next unsorted chunk, a recently freed chunk.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Next Non-main Arena (next_non_main_arena)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Pointer to the next non-main arena, used in multithreaded programs.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>System Memory (system_mem)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">8 bytes</td>
      <td style="background-color:#696969; color:white;">Tracks the total amount of memory allocated from the system.</td>
    </tr>
  </tbody>
</table>



#### Allocation flow:

When a program requests heap memory through Mallocs malloc() function, the allocator checks the appropriate bin for a free chunk, matching the requested size.
If successful, pops it from the bin, and returns a pointer to the chunk.

#### Deallocation flow:
When a chunk is freed, using the Mallocs free() function, the chunk gets placed back to its respective bin based on its size. 

## Exploitation
The technique relies on the lack of integrity checks when repeatedly freeing chunks of heap memory.
The Malloc implementation utilizes chunks of different sizes when allocating heap memory to a Linux process.