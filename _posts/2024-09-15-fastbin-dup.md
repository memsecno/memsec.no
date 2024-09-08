---
title: "Heap Exploitation : Fast Bin Dup Technique"
author: "Bjørn-Ivar Bekkevold"
date: "2024-09-15"
layout: post
categories: post
---
# Heap Exploitation: Fastbin Dup Technique

This writeup explores the fastbin duplication challenge, based on the Linux Heap Exploitation course by [Max Kemper](https://www.udemy.com/user/max-kamper/). We exploit an x86 64-bit ELF binary, leveraging a **Double-Free vulnerability** found in **glibc 2.30 (without tcache)**, invoking a shell as a proof-of-concept for arbitrary code execution.



### Short summary
The exploit is created by forming two fastbin duplicates and tampering with the metadata of the **Main Arena**. This process leads to the creation of a fake chunk on the arena itself. The chunk is then used to overwrite the **Top Chunk Pointer** with a fake top chunk, enabling memory allocation from an arbitrary location within the virtual memory space. We use this fake top chunk to redirect a **Malloc hook** to an address of our own choice. By assigning a 1-gadget to the hook's address, we successfully invoke a shell via the binary.

#### Required Tools:
- Pwndbg
- Pwntools (Python Library)
- x86 64-bit Linux Environment
- GNU toolchain


### Malloc introduction
Fastbin dup is a technique exploiting a double-free vulnerability in heap memory allocation, targeting specific versions of glibc's malloc implementation.

Heaps are contiguous blocks of dynamic memory allocated to an executing program. In the glibc malloc implementation, the heap is represented in chunks. A chunk consists of several fields, including a size field and metadata. Each chunk starts at 32 bytes and increases in increments of 0x10. Each chunk also includes at least 16 bytes of metadata when in use (See table below).

The heap will instantiate chunks from what is called a Top Chunk. Claiming a part of the size reserve of a Top Chunk. The Top Chunk size field represents total available heap memory allocated to a process. Where the creation and allocation of a chunk subtracts the requested chunk size from the Top Chunk size field. 


<table style="font-size: 10px; width: 50%;">
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

<table style="font-size: 10px; width: 70%;">
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



In a Linux process, Malloc manages dynamic memory allocation through a structure called an Arena. Arenas are used to keep track and manage heap memory in different threads, to aid in performance
and manage scalability.

An arena can be seen as a simple table structure, and keeps an overview of the allocated chunks via a Malloc primitive called bins. Bins can be considered simple linked list primitives formed as stack queues.
Where the elements in the queue points to free chunks of different sizes.
Bins are divided into categories, primarily based their size and usage, and are labeled as fastbin, smallbin, largebin and unsortedbin.

The table below depicts the overall structure of an arena, with parts of the table pointing to the different fastbin queues. Each fastbin is assigned to chunks of a respective size. Used for fast allocation and deallocation of chunks already instantiated. And can in some ways be considered a caching system for managing dynamic memory.


<table style="font-size: 10px; width: 70%;">
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

The Malloc implementation in glibc utilizes chunks of different sizes when allocating heap memory to a Linux process. When a program requests heap memory through Mallocs `malloc()` function, the allocator checks the appropriate bin for a free chunk, matching the requested size.
If successful, it pops it from the bin, and returns a pointer to the chunk.

When a chunk is freed, using the Mallocs `free()` function, the chunk is placed back to its bin based on its respective size. Waiting to be used again.

## Exploitation
### Target
The binary is compiled with different exploitation mitigations, to avoid misuse. Running in an ASLR enabled environment.

<table style="font-size: 10px; width: 70%;">
  <thead>
    <tr>
      <th style="background-color:#2F4F4F; color:white;">Mitigation</th>
      <th style="background-color:#4F4F4F; color:white;">Description</th>
      <th style="background-color:#696969; color:white;">Purpose</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Stack Canary</strong></td>
      <td style="background-color:#4F4F4F; color:white;">A random value placed on the stack between local variables and the return address, checked before returning from a function.</td>
      <td style="background-color:#696969; color:white;">Prevents buffer overflows that overwrite the return address, to avoid control flow hijacking.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>Full RELRO</strong></td>
      <td style="background-color:#4F4F4F; color:white;">Marks the Global Offset Table (GOT) as read-only after dynamic linking to prevent overwriting function pointers.</td>
      <td style="background-color:#696969; color:white;">Prevents attackers from hijacking function calls through GOT overwrites.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>NX Bit</strong></td>
      <td style="background-color:#4F4F4F; color:white;">Marks certain memory regions, such as the stack or heap, as non-executable.</td>
      <td style="background-color:#696969; color:white;">Prevents code injection attacks, disallows execution of code in writable regions.</td>
    </tr>
    <tr>
      <td style="background-color:#2F4F4F; color:white;"><strong>PIE (Position Independent Executable)</strong></td>
      <td style="background-color:#4F4F4F; color:white;">Compiles programs so that they can be loaded at a random memory address each time they run.</td>
      <td style="background-color:#696969; color:white;">Prevents code reuse attacks (like ROP) by making memory addresses unpredictable.</td>
    </tr>
  </tbody>
</table>

The challenge binary lets us request chunks through malloc, and release them again with free. The binary only allows us to request fast chunks, and forbids allocation of 0x70 chunks.
As an aid, the binary leaks the libc `puts()` symbol address,
Enabling us to uncover the internal memory map and relative distances to the remainder libc functions.

### Attack Chain
The attack begins by exploiting the absence of integrity checks in **glibc 2.30**, allowing repeated freeing of heap chunks to form a circular chain.

#### Fastbin Duplication
 **Allocating and Freeing Chunks:** We request two 0x50-sized chunks, A and B, on the heap. After freeing A, then B, then A again, we create a double-free on the 0x50 fastbin, forming a circular link between the chunks.

Meaning Chunk A (top of the stack) points to Chunk B, by linking the FD pointed from A to B, but now the FD of B also points back to A,
forming a circular link between the chunks.


<pre>
+-----------------------------+
|       Fastbin Head          |
+-----------------------------+
| Forward Pointer (fd)        |
| (Points to first free chunk)|
+-----------------------------+
          |
          v
+-----------------------------+          +-----------------------------+          +-----------------------------+
|      Chunk 1 (Freed)         |-------->|      Chunk 2 (Freed)         |-------->|      Chunk 3 (Freed)         |
+-----------------------------+          +-----------------------------+          +-----------------------------+
| Forward Pointer (fd)         |          | Forward Pointer (fd)         |          | Forward Pointer (fd)         |
| (Points to Chunk 2)          |          | (Points to Chunk 3)          |          | (Points to NULL)             |
+-----------------------------+          +-----------------------------+          +-----------------------------+
| Backward Pointer (bk)        |          | Backward Pointer (bk)        |          | Backward Pointer (bk)        |
| (Unused in fastbin)          |          | (Unused in fastbin)          |          | (Unused in fastbin)          |
+-----------------------------+          +-----------------------------+          +-----------------------------+
|        User Data             |          |        User Data             |          |        User Data             |
|    (Not accessible)          |          |    (Not accessible)          |          |    (Not accessible)          |
+-----------------------------+          +-----------------------------+          +-----------------------------+
                                                  Fastbin Queue
</pre>
<pre>
+-----------------------------+
|       Fastbin Head          |
+-----------------------------+
| Forward Pointer (fd)        |
| (Points to first free chunk)|
+-----------------------------+
          |
          v
+-----------------------------+           +-----------------------------+
|                             |           |                             |
|       Chunk A (Freed)       |           |       Chunk B (Freed)       |
|                             |           |                             |
+-----------------------------+           +-----------------------------+
|                             |           |                             |
|    Forward Pointer (fd)     |<--------->|    Forward Pointer (fd)     |
|  (Points to Chunk B)        |           |  (Points back to Chunk A)   |
+-----------------------------+           +-----------------------------+
|                             |           |                             |
|          User Data          |           |          User Data          |
|                             |           |                             |
+-----------------------------+           +-----------------------------+
                        Fastbin Chunk Duplication
</pre>


Now we have made **Malloc** repurpose a free chunk twice onto the fastbin.
We take advantage of this behavior, by manipulating the chunk FD to become a fake size field.

#### Malforming the Main Arena
We request a new chunk through the `malloc` function, of the same 0x50 size. 
We apply a set of non-printing characters /0x99/0xfe.../0xdc to form the size 0x61 in
the user data section. Making it look like a valid size field of a heap chunk.

```python
# Request two 0x50-sized chunks.
chunk_A = malloc(0x48, b"A"*8)
chunk_B = malloc(0x48, b"B"*8)
# Free the first chunk, then the second, then the first again
# A -> B -> A
free(chunk_A)
free(chunk_B)
free(chunk_A)
# Tamper with 0x50 fastbin metadata
# Make chunk fd become a fake size field in Arena
malloc(0x48, p64(0x61))
# Request chunks B & A, leaving fake sz field at head of 0x50 fastbin (first double free)
malloc(0x48, b"C"*8)
malloc(0x48, b"D"*8)
```
We then allocate chunks B and A again. Which leaves the fake size field 0x61 at the top of the 0x50 fastbin stack.

The fastbin has now been repurposed to a fake size field of 0x61. We can continue down this path to rearrange metadata in the main arena, to be interpreted differently.
 
#### Overwriting the Top Chunk
We continue with a second double-free, now targeting the 0x60 fastbin.
Since we know the address of puts() from the memory leak. We can navigate our way through the relative address distances to
the other libc functions that the binary relies on. 

We repeat the pattern of allocating two chunks in the 0x60 fastbin, we call these J and K. We free chunk J, then K, then J again, forming our second double-free. This time we repurpose the FD of chunk J to point to the address of main_arena + 0x20, meaning the fifth fastbin queue.
When we do a third `malloc`, the user data of the chunk in queue (holding the address of main arena + 0x20) will be interpreted as
the next FD (available free chunk) in the queue. This works because the main arena 0x50 fast bin holds the value 0x61, and gets
wrongly interpreted as a valid chunk size field.

Remember that for a reallocation from a fastbin, the current chunk will either hold a cleared user data area of 0x00s,
but if not, this is assumed to be a valid FD. The fastbin will then assume that the queue still holds freed chunks.

Now that we have prepared the metadata in the arena, we do a fourth allocation, this time to overwrite the top chunk pointer, residing in the 13th field of the main arena.

If we are able to create our own fake top chunk pointer, we can allocate memory from any arbitrary region within the memory space. Remember that this binary was built with a **glibc 2.30** dependency, meaning it includes the **2.29 Glibc** introduction of a Top Chunk Size integrity check.
This means we need to ensure that the size of a Top Chunk is within the system memory range reserved for the process.
![Fake Top Chunk within Arena](./images/fast_bin_dup_challenge/top_chunk_fake.png)

#### Activating a fake Malloc hook
 Our target for the fake top chunk pointer, is to activate one of Mallocs hooks. These hooks are used for purposes such as implementing ones own memory allocator, or to assess performance of a binary regarding its memory allocation operations. We want to overwrite this hook, currently deactivated by holding a value of all 0x00s, and repurpose it to an address of our own.When activated the hook will be called by the malloc function, we want to make the hook point to a valid gadget within the program. 

**Gadgets** are a set of assembly instructions of a specific ISA internal to the target binary. A gadget conducts some specific operation, and by chaining these together can be used to maliciously alter the behavior of an executing binary. A 1-gadget can be considered the last step of these, usually where the operation involves invoking a shell.

![One-Gagdets](./images/fast_bin_dup_challenge/one_gadget.png)

Now to be able to overwrite the `__malloc_hook` by creating a fake Top Chunk, we need to find a location close to the hook that can resemble a valid Top Chunk size field. For this we make use of the `find_fake_fast` command within **pwndbg**. Remember that neither the x86 ISA nor the glibc Malloc implementation enforces instruction alignment, explaining why the address found by `find_fake_fast` is a valid address. This would for instance not be valid on most RISC architectures (like ARM) requiring instruction alignment.


For the identification of a valid gadget, we use the 1-gadget tool referenced in the course, proposing a set of different gadgets 
found within the binary or one the **shared object** (.so) files linked to it.

![Previous chunks existing on Stack](./images/fast_bin_dup_challenge/stack_argv.png)

#### Invoking a shell
We use the last candidate proposed by 1-gadget. Note that one of the requirements is to control the `argv` input arguments of `sh`, within the specified stack offset of RSP + 0x50 and onwards. For this binary, this is taken care of by the fact that initial chunks allocated through the program, A, B, J and K are all existing along the stack. This means we can control what goes on to the stack offsets needed by this 1-gadget.

```python
# link fastbin addr to chunk_J FD
# Make 0x60 allocate a chunk that is actually part of the Arena
malloc(0x58, p64(libc.sym.main_arena + 0x20))

#pop J from fastbin stack
malloc(0x58, b"-s\0")
#pop K from fastbin stack
malloc(0x58, b"B")
## now fake FD is at top of 0x60 fastbin head.

# REPLACE TOP CHUNK ADDR IN MAIN ARENA:
malloc(0x58, bX*8*6 + p64(libc.sym.__malloc_hook - 35))
## top chunk is now 35 bytes before malloc hook
# the closest memory region with a value that can emulate a fake size field
#
# malloc and plant the gadget address onto the malloc_hook region
malloc(0x28, bX*19 + p64(libc.address + 0xe1fa1))

# malloc whatever to trigger malloc hook and execute gadget
malloc(0x18, b'')
```


For `sh` we apply the `-s` argument to indicate `stdin`input, which should make data succeeding RSP + 0x50 irrelevant (Remember x86 calling convention pushes args onto 
stack in reverse). 

![Shell Access](./images/fast_bin_dup_challenge/shell_pop.png)

#### Root privileges through `suid`
The challenge mentions this as a good example of how the `suid` access property can be misused. For those unaware, `suid` is a user access setting for the Linux file system. Applied to an executable, making it run with the effective `uid` belonging to the owner of the file instead of the initiating user. This is often seen in conjunction with scripts that require root privileges, where a non-admin user can call a script and execute it with higher privileges than the user would have otherwise.

![Root Shell Access](./images/fast_bin_dup_challenge/root_shell_pop.png)

This binary when run with this setting facilitates a way an attacker can use the double free vulnerability to gain root on a device. All thanks to this access control property. To conduct this, we will change the `-s`argument of `sh` to `-p`to avoid resetting the effective `uid` to the calling user. 


## Conclusion

In this challenge, we successfully exploited a 64-bit x86 binary by creating two fastbin duplicates. We bypassed several exploit mitigations, including Stack Canary, Full RELRO, NX Bit, and PIE. Using a libc leak, we manipulated the main arena to overwrite the Top Chunk Pointer, activate a Malloc hook, and invoke a shell using a 1-gadget.

In this challenge we successfully exploited a 64-bit x86 binary by creating two fastbin duplicated. We leveraged a double-free Malloc vulnerability in the 2.30 version of glibc. Achieving arbitrary code execution through a 1-gadget.

The binary ran with multiple exploit mitigations which we bypassed:
- Stack Canary
- Full RELRO
- NX bit
- PIE

The binary leaked the position of libc symbol `Puts()` allowing us to identify the remainder relative positioning of libc symbols despite ASLR (enabled by PIE). We used the libc leak to locate the **Main Arena** and `__malloc_hook`. Using the two instances of fastbin dups to tamper with the 0x50 and 0x60 fastbins.

We then formed a fake chunk onto the arena, which allowed us to overwrite the Top Chunk pointer. The fake Top Chunk was then further used to activate and overwrite the `__malloc_hook` with a glibc gadget that invoked a shell from the binary, bypassing the NX bit and Stack Canary.

Additionally, we demonstrated how `suid` can be misused to gain root access, leveraging the same exploitation chain.