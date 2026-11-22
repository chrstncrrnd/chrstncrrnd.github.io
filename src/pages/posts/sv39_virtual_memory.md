---
layout: ../../layouts/PostLayout.astro
title: "Sv39 Virtual Memory Explained"
description: "An introduction to RISC-V's Sv39 virtual memory system."
creation: "22 November 2026"
latest_edit: "22 November 2026"
tags:
  - "RISC-V"
  - "C"
  - "OS"
  - "Virtual Memory"
---

# Introduction 

In this article I will go over the basics of how Sv39 (and similar virtual
memory systems) in RISC-V work, why they exist and what a simple implementation
could look like. I won't dive too deep into specifics, the main point of this
article is to gain intuition for how this system works.

# 1 - Basics
## Why do we Need Virtual Memory?

Virtual memory exists to solve two problems. Firstly, how do we stop two user
programs from accessing each other's memory, and secondly, how can we run a
program larger than the system's memory.

If two programs could access each other's memory, this would be terrible for
security since we can imagine that a malicious program could read your passwords
straight from your browser's memory. It could also be that your calculator app
has a bug where it writes to an arbitrary memory address, if this is where
another program has its instructions, we can see that things could go badly
quickly where one faulty program can now affect perfectly functional programs.

For the second point, when we're actually running a program, the CPU only sees
the current instruction and maybe a 64 bit value it pulls from memory. The point
is that, there is no reason why we can't run programs larger than the system's
memory (given, of course, they fit on secondary media attached to the computer).

## So, What is Virtual Memory?

Essentially, virtual memory is just the concept of there existing a mapping between a 
virtual memory space and the physical memory space.

In simple terms, we give a program running on a machine its own memory space
called its virtual memory space. We can think of it as the program running alone
on a computer which has as very large amount of RAM. Our program effectively
sees that it has it's own memory which contains only it's own stuff. This memory
is as large as the virtual memory space, usually defined as $2^n$ bytes where
$n$ is the number of bits in our virtual memory address, in Sv39 $n = 39$
(that's where the 39 comes from!)

Effectively, when a program requests data from memory, we take the address it's
virtual address its referring to and translate it to a physical address, one we
can actually use to index our physical RAM.

## What is paging?

Paging is a system for virtual memory.
