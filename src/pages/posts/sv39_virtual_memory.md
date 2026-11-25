---
layout: ../../layouts/PostLayout.astro
title: "Sv39 Virtual Memory Explained"
description: "An introduction to RISC-V's Sv39 virtual memory system."
creation: "23 November 2026"
latest_edit: "23 November 2026"
tags:
  - "RISC-V"
  - "C"
  - "OS"
  - "Virtual Memory"
---

# Introduction

In this article I will go over the basics of how Sv39 (and similar virtual
memory systems) in RISC-V work, why they exist and what a simple implementation
could look like. The goal of this article is intuition rather than completeness,
so I deliberately omit certain details for the sake of simplicity. You should be
comfortable with C and with binary and hexadecimal; no OS background is needed. 

<!-- toc -->

# 1 - Basics
## Why do we Need Virtual Memory?

Virtual memory is motivated by two main problems. Firstly, how do we stop two user
programs from accessing each other's memory, and secondly, how can we run a
program which doesn't fit entirely in the system's RAM.

Two programs accessing each other's memory would be a serious security problem.
A malicious program could read your passwords straight from your web browser's
memory; one bad actor would compromise an entire system. A perfectly legitimate
program could also accidentally overwrite another program. Now one faulty
program could cause a myriad of problems for everything else running on the same
system.

For the second point, when we're actually running a program, we don't need it
*all* in memory at once. We execute programs instruction-by-instruction, the CPU
only needs the instruction it is currently executing and the data that
instruction touches. As far as the CPU is concerned, it doesn't care if the
entirety of the program is stored in RAM. It only needs to receive the correct
information when requested. In fact, in almost all program executions, many
lines of code will go without executing and plenty of assets will never be
rendered.

## So, What is Virtual Memory?

Virtual memory is a "fake" view of memory we show to a program. This view of
memory is isolated from all other programs and can be as large as the address
space permits.

Through this fake (or rather, virtual) view, the program can see and access only
the data belonging to it. This view is $2^n$ bytes large, where n is the number of bits
in our virtual memory address. In Sv39, n = 39 (that's where the 39 comes from!)

When a program requests data from memory, we translate its virtual address which 
refers to memory in its virtual view, to a physical address; one we can use to 
index RAM.

## What is Paging?

Paging is a way of implementing virtual memory.

When translating virtual to physical addresses, we need to store these
translations (which I'll call maps) somewhere in memory.
If we stored these maps for every translation pair, we would end up consuming
more memory on the mapping than the data itself. Hence we store translation chunks.

A chunk is defined by a starting address and a length. Every address inside a
chunk is translated using the same mapping, just shifted by its offset within
the chunk.

However, variable length chunks carry the problem of finding a free space large
enough for them in memory -- this is called external fragmentation. Instead,
we consider only fixed size chunks. This is paging: our system of chunks but of
the fixed size: 4096 bytes.

We break up the virtual and physical memory space into blocks, or pages, which
are 4096 bytes long. Then, we map 4096 virtual addresses starting at address V
to the 4096 physical addresses in memory starting at P. RISC-V uses this 
system for virtual memory.

Note that some textbooks refer to the physical pages as "frames".

# 2 - Virtual Memory in RISC-V

In RISC-V, the CPU produces virtual addresses. The MMU (memory management unit),
sitting between the CPU and memory, is responsible for translating those virtual
addresses to physical addresses. 

A fundamental question remains: how do we actually translate between virtual
and physical memory addresses? So far we know that this process exists and we can use
it to solve the aforementioned problems. But how does the MMU actually take a virtual
address and produce a physical address? Who tells it which maps to which?

An important consideration is that we want to be the ones to tell the MMU *what*
the mappings are. If it did it automatically, we would lose a lot of control
over this system and too much complexity would be delegated to the hardware.

Therefore, our job is to tell the MMU what the mappings are and where to find them.

## The `satp` Register

We use the `satp` register to tell the MMU where in memory to find our mappings.
It acts as an interface for the CPU to talk to the MMU. Since we are dealing with
64 bit RISC-V, it is a 64 bit register with the following fields: 

$$
\overset{\scriptstyle 63\,\cdots\,60}{\underset{\scriptstyle 4}{\boxed{\hspace{0.6em}\text{MODE}\hspace{0.6em}}}}\overset{\scriptstyle 59\,\cdots\,44}{\underset{\scriptstyle 16}{\boxed{\hspace{3.4em}\text{ASID}\hspace{3.4em}}}}\overset{\scriptstyle 43\,\cdots\,0}{\underset{\scriptstyle 44}{\boxed{\hspace{6em}\text{PPN}\hspace{6em}}}}
$$


The `MODE` bits specify which mode to use, e.g. Sv39, Sv48, Sv57 or bare.
The difference between the Sv modes is how many bits they support for virtual
addresses. This article uses Sv39, which gives us a $2^{39}$ byte = 512 GiB
virtual address space. The bare mode implies we don't use any form of virtual
memory. This is the default mode.


The `ASID` bits specify the address space identifier. This is out of scope for
the purposes of this article.

Finally, the `PPN` (physical page number) bits specify in which physical page
the root page table is located. We will discover what a page table is in the next section.

We can use a function like this to compute the value to write to the `satp` register.
```c
#define SATP_SV39_MODE (8ULL << 60)

// Note that `page_table_root` must be aligned to the page size.
static inline uint64_t pagetable_root_2_satp(uint64_t page_table_root) {
  return (SATP_SV39_MODE | (page_table_root >> 12));
}
```

The register is written to using the `csrw` instruction. The following is
some assembly to do that, where `a0` contains the value computed using the above
function: 
```assembly 
csrw satp, a0
sfence.vma zero, zero # Flush the TLB
```

You can find out more about this register in the RISC-V specification
[here](https://docs.riscv.org/reference/isa/v20260120/priv/supervisor.html#satp).


## Page tables

Before we get to how to create mappings and how the MMU finds them, we must
first define the basic data structure used: the page table.

A page table is a page filled with a list of page table entries. In our system,
a page table entry is a 64 bit = 8 byte unsigned integer, hence we can fit 4096/8 = 512 page
table entries in one page.

A page table entry is a pointer to a page with some extra metadata, the following is a
diagram of the format:

$$
\small
\overset{\scriptstyle 63\,\cdots\,54}{\underset{\scriptstyle 10}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.5em}\text{Reserved}\hspace{0.5em}}}}\overset{\scriptstyle 53\,\cdots\,28}{\underset{\scriptstyle 26}{\boxed{\vphantom{\text{Ag[]}}\hspace{2.2em}\text{PPN[2]}\hspace{2.2em}}}}\overset{\scriptstyle 27\,\cdots\,19}{\underset{\scriptstyle 9}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.6em}\text{PPN[1]}\hspace{0.6em}}}}\overset{\scriptstyle 18\,\cdots\,10}{\underset{\scriptstyle 9}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.6em}\text{PPN[0]}\hspace{0.6em}}}}\overset{\scriptstyle 9\,\cdots\,8}{\underset{\scriptstyle 2}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.1em}\text{RSW}\hspace{0.1em}}}}\overset{\scriptstyle 7}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{D}\hspace{0.2em}}}}\overset{\scriptstyle 6}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{A}\hspace{0.2em}}}}\overset{\scriptstyle 5}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{G}\hspace{0.2em}}}}\overset{\scriptstyle 4}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{U}\hspace{0.2em}}}}\overset{\scriptstyle 3}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{X}\hspace{0.2em}}}}\overset{\scriptstyle 2}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{W}\hspace{0.2em}}}}\overset{\scriptstyle 1}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{R}\hspace{0.2em}}}}\overset{\scriptstyle 0}{\underset{\scriptstyle 1}{\boxed{\vphantom{\text{Ag[]}}\hspace{0.2em}\text{V}\hspace{0.2em}}}}
$$

The final 8 bits are used as flags, for example the `R`, `W` and `X` bits
specify if a page is readable, writable or executable. The `V` flag indicates
if the page table entry is valid, and the `U` flag indicates if the page
is accessible from user mode.


Bits 53 through 10 are used for the PPN (physical page number) of the page we
are pointing to. Taking `PPN << 12` we get the physical starting address of the
page in memory. Note that the left shift is of 12 bits since our pages are 4096
bytes long and $\log_2$(4096) = 12. Hence 12 bits are used for the byte offset.

In most diagrams explaining the page table entries, including this one,
you will see that the PPN is split into three: `PPN[2]`, `PPN[1]` and `PPN[0]`. 
This has to do with the idea of "higher level leaves". I will expand upon
this at the end of the article, for now we consider the PPN
as one contiguous number.


## The MMU's Walk

As of now, we know two things: mappings are stored in memory, and we can tell
the MMU where to start looking for the mappings through the `satp` register.
Next, we are going to find out how the MMU takes the `PPN` bits stored in the
`satp` register alongside a virtual address and finds the corresponding page
table entry which contains the starting physical page number of the corresponding 
physical address.


In order to see how the MMU uses a virtual address, we must first analyze 
the structure of one such address. Remember, a virtual address in Sv39 is
39 bits wide.
$$
\overset{\scriptstyle 38\,\cdots\,30}{\underset{\scriptstyle 9}{\boxed{\hspace{1em}\text{VPN[2]}\hspace{1em}}}}
\overset{\scriptstyle 29\,\cdots\,21}{\underset{\scriptstyle 9}{\boxed{\hspace{1em}\text{VPN[1]}\hspace{1em}}}}
\overset{\scriptstyle 20\,\cdots\,12}{\underset{\scriptstyle 9}{\boxed{\hspace{1em}\text{VPN[0]}\hspace{1em}}}}
\overset{\scriptstyle 11\,\cdots\,0}{\underset{\scriptstyle 12}
{\boxed{\hspace{2em}\text{page offset}\vphantom{\text{Ag[]}}\hspace{2em}}}}
$$

`VPN[n]` is the virtual page number at level `n`. The page offset is the
byte offset within the page.

Using a virtual address, the MMU performs the following walk:

1. Assign `root_table = satp.PPN << 12`, where `satp.PPN` are the PPN bits of the
  `satp` register, i.e. bits 43 through 0. This is an address in main memory containing
  a page table.
2. Index the page table at `VPN[2]`: `pte_l2 = root_table[VPN[2]]`. This page table
   entry points to the next page table: `table_l2 = pte_l2.PPN << 12` (i.e. bits 53 through
   10 of the `pte_l2` page table entry).
3. Repeat this indexing one level deeper: `pte_l1 = table_l2[VPN[1]]`. Once again find the
  table one level down: `table_l1 = pte_l1.PPN << 12`.
4. Finally, let `pte_l0 = table_l1[VPN[0]]`. The page starting at
   address `pte_l0.PPN << 12` contains our data!

All in all we obtain the final physical address: `pte_l0.PPN << 12 | page_offset`, where
`page_offset` is the lowest 12 bits of the virtual address. Return this address.


That was a lot of information; let's look at it from a high level. Over 3
iterations, we are indexing a page table to obtain the page table entry to the
next page. The next page is a page table, if we are at levels 2 or 1, and the
actual physical page if we are at level 0. We can see this as a tree with depth
3 and branching factor 512. At each iteration, the MMU looks at the current node
(page table) and choses the corresponding branch depending on `VPN[n]`. Once we
reach a leaf, at depth 3, we know the page our pte is pointing to is the one
containing our data. To compute the physical address, sum the page offset from
the virtual address to `pte_l0.PPN << 12`.

The MMU performs this walk for translation. Since accessing memory is costly,
the MMU usually uses something called the TLB (translation lookaside buffer).
The TLB caches translations so the MMU can avoid performing this walk on every
memory access.

The following is a simple implementation of this walk in C:

```c
// paddr_t is the type for a physical address, and vaddr_t is the type for a virtual address
typedef uint64_t paddr_t;
typedef uint64_t vaddr_t;
// ...
paddr_t walk(uint64_t *root_table, vaddr_t vaddr) {
  uint64_t *table = root_table;
  for (size_t level = 2; level > 0; --level) {
    // compute the corresponding VPN[level]
    uint64_t page_number = (vaddr >> (12 + 9 * level)) & 0x1FF;
    uint64_t pte = table[page_number];
    // go one level deeper
    table = (uint64_t *)(((pte >> 10) & ((1ULL << 44) - 1)) << 12);
  }
  uint64_t page_number = (vaddr >> 12) & 0x1FF;
  // last 12 bits 
  uint64_t page_offset = vaddr & 0xFFF;
  return ((table[page_number] >> 10) & ((1ULL << 44) - 1)) << 12 | page_offset;
}
```

## Storing Mappings

Now that we know how the MMU translates addresses by using our mappings,
we can figure out where and how to store them.

The process of mapping a page is almost identical to the walk the MMU performs.
The difference is that if a branch on our tree does not exist, we must create it 
by allocating a new node and updating the parent's page table. It's also
important to note that our `walk` function translates a virtual **address** to a physical
one, while our `map_page` function will create the mapping between a physical **page number** 
and a virtual one. Since we are using paging, we can't map individual addresses.

Once we reach a leaf, we simply write to it a page table entry whose PPN is that of 
the physical page we are mapping.

The following is a simple implementation in C:

```c
void map_page(uint64_t *root_table, vaddr_t vaddr, paddr_t paddr,
              uint64_t flags) {
  // we are mapping between pages, hence both the virtual address and physical address
  // must be aligned to the page size
  if (!(is_aligned(vaddr, PAGE_SIZE) && is_aligned(paddr, PAGE_SIZE))) {
    panic("Tried to map an unaligned page!");
  }
  uint64_t *table = root_table;
  for (size_t level = 2; level > 0; --level) {
    uint64_t page_number = (vaddr >> (12 + 9 * level)) & 0x1FF;
    uint64_t pte = table[page_number];
    // we don't have a valid page table entry, we must create it
    if (!(pte & PTE_V)) {
      // allocate a new page
      paddr_t new_page = alloc_zeroed_page();
      pte = ((new_page >> 12) << 10) | PTE_V;
      // write our new page table entry
      table[page_number] = pte;
      table = (uint64_t *)new_page;
      continue;
    }
    // go one level deeper
    table = (uint64_t *)(((pte >> 10) & ((1ULL << 44) - 1)) << 12);
  }
  // finally, write our pte to the lowest level
  uint64_t page_number = (vaddr >> 12) & 0x1FF;
  if (table[page_number] & PTE_V) {
    panic("Tried to remap an already mapped virtual page!");
  }
  table[page_number] = ((paddr >> 12) << 10) | flags | PTE_V;
}
```

And that's it! By storing mappings in this way and passing the root page table
to the `satp` register, the MMU can take the CPU's generated virtual addresses
and produce physical addresses to access memory.

# 3 - Final Notes

In the page tables section, I mentioned the concept of "higher level leaves". 
This is simply that in our tree we allow for leaves at shallower
depths. This lets us map much larger areas of contiguous memory at once,
sometimes referred to as "mega" and "giga" pages. Leaves in this case can be
identified by having either one of the `R`, `W` or `X` set to 1.

# 4 - Conclusion

While this article is very high level, I hope it helped build intuition for how
paging works in real systems. Please feel free to use any of the code here, most
of it is from my personal OS project, which you can find
[here](https://github.com/chrstncrrnd/c_os). Specifically, most of the code
comes from `kernel/paging.c`.

I hope this was a fun read! If you have any feedback or questions,
please do not hesitate to contact me.


# 5 - References
- [RISC-V Supervisor Level ISA](https://docs.riscv.org/reference/isa/v20260120/priv/priv-index.html)
- [Xv6 Operating System](https://github.com/mit-pdos/xv6-riscv/)
- [Virtual Memory - Stephen Marz's Blog](https://marz.utk.edu/my-courses/cosc130/lectures/virtual-memory/)
