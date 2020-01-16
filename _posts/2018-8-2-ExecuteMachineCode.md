---
layout: post
title: Executing arbitrary machine code in C
tags: [blog, research, opcodes]
---

This is part of a series of blog posts on undocumented opcode fuzzing. When I started trying to test undocumented opcodes, I struggled to find a technique for integrating machine code into higher-level code. I wanted to create a C program automating the testing and analysis of millions of undocumented instructions, so it had to be machine code rather than assembly as there were no mnemonics for them. You can of course create a pure hex file and convert it into a binary, but that's a nightmare if you want to code the equivalent of thousands of lines of C! I hope you find these techniques for executing machine code in user mode and kernel mode useful. The user mode technique is adapted from several examples on GitHub (unfortunately I can't now find the original posts) and can also be implemented in [Python](https://stackoverflow.com/questions/6143042/how-can-i-call-inlined-machine-code-in-python-on-linux).

## Post Outline
* [User mode](#user-mode)
* [Handling exceptions in user mode](#handling-exceptions)
* [Kernel mode](#kernel-mode)

# User Mode

## Function pointer and mprotect

To execute our opcode we first create an unsigned char array (the distinction between unsigned/signed char is important when working with hex) called `code`. This array can be executed with `((void(*)())code)()`, which creates a void function pointer to the array and then calls that pointer. However, thanks to memory protection the stack (where our array lives at runtime) isn't executable, so if we run this our program will segfault and die. Luckily on Linux there is an easy workaround for this: we can use `mprotect` to make the memory page containing our array executable. Note that the pointer you pass to `mprotect` must be aligned to a page boundary, or it will fail (**always** remember to check system calls for error return values!). As an alternative, you could compile your code with the `-z execstack` GCC option instead. Neither option is good from a security perspective, so do bear that in mind if you're thinking of using this as part of a large project; if you don't need to modify your machine code at runtime, you could change the memory permissions to RX instead of RWX so that it's no longer writable.

Using an array like this for our code means we don't need to worry about ASLR; although the memory address of the array will change each time the program is run, we don't need to know this address as we can just reference the array by name in our program. We can also easily modify the array's values. However, the rest of our code is static: if you want truly self-modifying code, [libjit](https://eli.thegreenplace.net/2013/10/17/getting-started-with-libjit-part-1/) is a C library specifically designed for Just-In-Time (JIT) programs generating and executing code at runtime.

## Function prologue and epilogue

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

* **Prologue:** we push the frame pointer to the stack (because by the calling convention it needs to be preserved across function calls) and copy the stack pointer to the base pointer so that `rbp` points to the base of this function's stack frame.
* **Our code:** We then run the four NOPs.
* **Epilogue:** we move 0 into `eax` (because main should have an integer return value; our function returns void so doesn't need this), pop the frame pointer back off the stack, return from the function and align the code with a long NOP (the compiler is probably inserting this for [performance reasons](http://john.freml.in/amd64-nopl); I haven't yet compared the performance of OpcodeTester with this included vs. without, and I suspect it'll be tricky to implement correctly as the instructions I test vary in length).

You might choose to include all of this for completeness; for my research I was trying to measure performance statistics of opcodes as accurately as possible, so I wanted the minimum amount of padding to avoid a segfault. If you experiment you'll find that this bare minimum consists of 3 padding opcodes: **0x55**, then the instructions you want to execute, finished with **0x5d** and **0xc3** (it makes sense really that you need to return from the function, doesn't it!).

If you're not familiar with assembly and none of that made sense, [this post](https://www.recurse.com/blog/7-understanding-c-by-learning-assembly) is fantastic.

Here's the completed program (tested on Ubuntu 16.10, 17.10, 18.04 - all on Intel 64-bit):

```c
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdint.h>

int main(){
  unsigned char code[7] = {0x55, 0x90, 0x90, 0x90, 0x90, 0x5d, 0xc3};

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

## Handling exceptions
If you're only running machine code with valid opcodes (and you've written it correctly!), you don't need to do exception handling - getting an exception at runtime is a sign something's wrong with your code and you need to fix it. If you're doing automated testing of undocumented opcodes, however, an unfortunate fact of life is that most of them will fail. There are a range of possible Intel CPU exceptions, but in my testing I mostly only ever saw UD (undefined instruction) and GP (general protection) exceptions. In user mode, the kernel delivers these to our program as signals: UD becomes SIG_ILL and GP becomes SIG_SEGV.

Intercepting a signal is as simple as registering a signal handler for that specific signal; the default behaviour (e.g. the program dying) will no longer occur. But this means we need to handle restoring execution in our signal handler - if we don't, the program will just hang! The code below shows one possible approach.

I use the `instrCurrentlyExecuting` variable to first check the signal is an expected signal which we should be handling. If it didn't occur whilst an instruction was being tested, something has gone wrong - possibly a previously tested instruction corrupted the program state - so we restore the default signal handlers to clean up and kill the program. Similarly, we check we haven't had more than 3 signals for the same instruction: this means we are stuck in a signal loop. In testing, this is fairly rare but does still occur. I haven't yet determined what causes these loops; there's something that I'm not handling correctly!

If these checks are passed, we then get the execution context, change the instruction pointer to the address of our `resume` global assembly label (thus skipping the instruction which threw the exception), and reset the sign flag; the program continues executing normally. (This technique is taken from Chris Domas' [Sandsifter tool](https://github.com/xoreaxeaxeax/sandsifter).) The `sig` number and `siginfo` struct provide useful information if you want to log details about the signal before returning from the signal handler. Note that signal handlers should be as fast and light as possible: setting a variable is a better way to store information than using `printf`, for example.

```c
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdint.h>
#include <signal.h>
extern char resume;

void signalHandler(int sig, siginfo_t* siginfo, void* context){
	/*abort if couldn't restore context - instructionFailed increasing
	means we are stuck in a loop */
	if(instrCurrentlyExecuting){
		if(instructionFailed > 3) exit(1);
		else{
	  instructionFailed++;
		//get execution context, skip faulting instr, reset sign flag
		mcontext_t* mcontext = &((ucontext_t*)context)->uc_mcontext;      
		mcontext->gregs[IP]=(uintptr_t)&resume; 					
		mcontext->gregs[REG_EFL]&=~0x100;       					
		}
	}
	else{	//unexpected signal, restore default handlers
		signal(SIGILL, SIG_DFL); signal(SIGFPE, SIG_DFL);
		signal(SIGSEGV, SIG_DFL); signal(SIGBUS, SIG_DFL);
		signal(SIGTRAP, SIG_DFL); signal(SIGABRT, SIG_DFL);
	}
}

int main(){

  //register signal handler for all likely signals
  struct sigaction handler;
  memset(&handler, 0, sizeof(handler));
  handler.sa_sigaction = signalHandler;
  handler.sa_flags = SA_SIGINFO;
  if (sigaction(SIGILL, &handler, NULL) < 0   || \
  	sigaction(SIGINT, &handler, NULL) < 0 	|| \
  	sigaction(SIGFPE, &handler, NULL) < 0   || \
  	sigaction(SIGSEGV, &handler, NULL) < 0  || \
  	sigaction(SIGBUS, &handler, NULL) < 0   || \
  	sigaction(SIGTRAP, &handler, NULL) < 0  || )
  {
  	perror("sigaction");
  	return 1;
  }

  unsigned char code[7] = {0x55, 0x90, 0x90, 0x90, 0x90, 0x5d, 0xc3};

  //pointer passed to mprotect must be aligned to page boundary
  size_t pagesize = sysconf(_SC_PAGESIZE);
  uintptr_t pagestart = ((uintptr_t) &code) & -pagesize;

  if(mprotect((void*) pagestart, pagesize, PROT_READ|PROT_WRITE|PROT_EXEC)){
    perror("mprotect");
    return 1;
  }

  instrCurrentlyExecuting = true;
  ((void(*)())code)();  //execute instruction
  __asm__ __volatile__ ("\
		.global resume   \n\
		resume:          \n\
		"
		);
	;
	instrCurrentlyExecuting = false;

  printf("Successfully executed!\n");

  return 0;
}
```

# Kernel Mode
Executing machine code within a kernel driver is very similar. We need to use `__vmalloc` and `vfree` instead of `mprotect`, and `printk` instead of `printf`. `printk` outputs to the kernel log, which you can view with the terminal command `dmesg`. However, don't use `printk` too liberally! The system logs have no size limit by default, so if you were to *hypothetically* `printk` several lines each for several million instructions, and then repeat that quite a few times on a system that was running out of hard disk space anyway, you might *hypothetically* find yourself unable to boot until you cleared some space via the recovery console....(Totally not speaking from experience here.) If doing a lot of experimentation in a kernel driver it's definitely worth deleting old kernel logs regularly to free up space, and/or changing the config settings.

```c
static unsigned char *code;
opcode = __vmalloc(15, GFP_KERNEL, PAGE_KERNEL_EXEC);
memset(opcode, 0, 15);
opcode[0] = 0x55;
opcode[1] = 0x90;
opcode[2] = 0x90;
opcode[3] = 0x90;
opcode[4] = 0x90;
opcode[5] = 0x55;
opcode[6] = 0xc3;
((void(*)(void))code)();
printk(KERN_INFO "Successfully executed!\n");
vfree(opcode);
```

This is the absolute bare minimum code required - there's a lot of boilerplate required to create an entire driver.

Just like the initial user mode code above, this code doesn't handle exceptions at all. You might think an exception in a kernel driver would be catastrophic, but the kernel actually seems to handle it pretty well. In testing on Ubuntu (16.10, 17.10, 18.04) I never managed to completely crash the system just by throwing a UD or GP exception; the exception was logged and then my user mode program interacting with the driver was killed. Once I started trying to *handle* exceptions however (so that my user mode program wouldn't die all the time), I managed to cause lots of chaos - handling exceptions in kernel mode is possible, but a bit trickier, so I've explained it in a [separate post](/ExceptionsKernelDriver).
