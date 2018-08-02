---
layout: post
title: How to execute undocumented opcodes (or arbitrary machine code) in C on Linux
tags: [blog, research, undocumentedCPU]
---

This is part of a series of blog posts on my [Undocumented x86-64 Opcodes](/#) research project. When I started trying to test undocumented opcodes, I struggled to find a technique for integrating machine code into higher-level code (such as a C program automating the testing and analysis of millions of instructions). You can of course create a pure hex file and convert it into a binary, but that's a nightmare if you want to code the equivalent of thousands of lines of C! I hope you find these techniques for executing machine code in user mode and kernel mode useful. The user mode technique is adapted from several examples on GitHub (unfortunately I can't now find the original posts) and can also be implemented in [Python](https://stackoverflow.com/questions/6143042/how-can-i-call-inlined-machine-code-in-python-on-linux).

# User Mode

## Executing

To execute our opcode we first create an unsigned char array (the distinction between unsigned/signed char is important when working with hex) called `code`. This array can be executed with `((void(*)())code)()`, which creates a void function pointer to the array and then calls that pointer. However, thanks to memory protection the stack (where our array lives at runtime) isn't executable, so if we run this our program will segfault and die. Luckily on Linux there is an easy workaround for this: we can use `mprotect` to make the page containing our array executable. Note that the pointer you pass to `mprotect` must be aligned to a page boundary, or it will fail (**always** remember to check system calls for error return values!); and that the example code only allocates one page of memory (4096 bytes on most OSes), so if you have an exceptionally large amount of machine code to run you'll need more. TODO QUERY IF THE ARRAY IS ON THE STACK

Using an array like this for our code means we don't need to worry about ASLR; although the memory address of the array will change each time the program is run, we don't need to know this address as we can just reference the array by name in our program. We can also easily modify the array's values. However, the rest of our code is static: if you want truly self-modifying code, you do need to work around ASLR (see [this post](/#) by Evan Klitzke for more details).

So, fantastic. I make a `code` array of 4 unsigned chars, make it executable with `mprotect`, put 4 NOP opcodes in it (0x90 * 4), and then execute it using a void function pointer. But wait - it **still** segfaults! And that's because there's one more very important step: because we're executing the char array as if it's a function, GCC compiles it to assembly expecting it to behave like a function - we're missing the function prologue and epilogue. Let's try writing the simplest possible C function which does the same thing running 4 NOPs (thankfully because we're testing a documented opcode we can do this!) to find out what machine code 'padding' we need to avoid a segfault:

```c
int main(){
__asm__ __volatile__ ( "nop; nop; nop; nop;");
}
```

Compile with GCC and view the disassembly with objdump:

`gcc -o test test.c && objdump -S test`

Scroll down to the function we're interested in - main (yes, GCC adds a huge amount of other 'padding' functions to even such a simple program!):

```
00000000000005fa <main>:
 5fa:	55                   	push   %rbp
 5fb:	48 89 e5             	mov    %rsp,%rbp
 5fe:	90                   	nop
 5ff:	90                   	nop
 600:	90                   	nop
 601:	90                   	nop
 602:	b8 00 00 00 00       	mov    $0x0,%eax
 607:	5d                   	pop    %rbp
 608:	c3                   	retq   
 609:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
 ```

* Prologue: we push the frame pointer to the stack (because by the calling convention it needs to be preserved across function calls) and copy the stack pointer to the base pointer so that RBP points to the base of this function's stack frame.
* We then run the four NOPs.
* Epilogue: we move 0 into the EAX register (because main should have an integer return value; our function returns void so doesn't need this), pop the frame pointer back off the stack, return from the function and align the code with a long NOP (the compiler is probably inserting this for [performance reasons](http://john.freml.in/amd64-nopl); I haven't yet compared the performance of OpcodeTester with this included vs. without, and I suspect it'll be tricky to implement correctly as the instructions I test vary in length).

You might choose to include all of this for completeness; for my research I was trying to measure performance statistics of opcodes as accurately as possible, so I wanted the minimum amount of padding to avoid a segfault. If you experiment you'll find that this bare minimum consists of 3 padding opcodes: 0x55, then the instructions you want to execute, finished with 0x5d and 0xc3 (it makes sense really that you need to return from the function, doesn't it!).

If you're not familiar with assembly and none of that made sense, [this post](https://www.recurse.com/blog/7-understanding-c-by-learning-assembly) is fantastic.

Here's the completed program (tested on Ubuntu 16.10, 17.10, 18.04 - all on Intel 64-bit):

//get system's page size and calculate pagestart addr
size_t pagesize = sysconf(_SC_PAGESIZE);
uintptr_t start = (uintptr_t) &code;
uintptr_t end = start + sizeof(code);
uintptr_t pagestart = start & -pagesize;
if(mprotect((void*) pagestart, pagesize, PROT_READ|PROT_WRITE|PROT_EXEC)){
  perror("mprotect");
  return 1;
}

```c
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdint.h>

int main(){
  unsigned char code[4] = {0x55, 0x90, 0xx5d, 0xc3};

  //pointer passed to mprotect must be aligned to page boundary
  size_t pagesize = sysconf(_SC_PAGESIZE);
  uintptr_t pagestart = ((uintptr_t) &code) & -pagesize;

  if(mprotect((void*) pagestart, pagesize, PROT_READ|PROT_WRITE|PROT_EXEC)){
    perror("mprotect");
    return 1;
  }

  ((void(*)())code)();  //execute instruction
  printf("Successfully executed!\n");

  return 0;
}
```

## Handling exceptions (signals)

TODO

# Kernel Mode

TODO
