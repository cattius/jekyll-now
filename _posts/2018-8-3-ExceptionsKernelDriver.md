---
layout: post
title: Handling CPU exceptions in a Linux kernel driver
tags: [blog, research, opcodes]
---

This is part of a series of blog posts on undocumented opcode fuzzing. Previous research into undocumented opcodes such as Chris Domas' [Sandsifter project](https://github.com/xoreaxeaxeax/sandsifter) had tested them in ring 3 (user mode), but not in ring 0 (kernel mode). I was very keen to test out opcodes in ring 0 because many of them failed with GP exceptions in ring 0. GP can be an indication that an opcode exists but is privileged, although there are lots of other causes too. I was convinced I was going to discover some really interesting undocumented instructions if only I could run them in ring 0. Spoiler alert: sadly this wasn't the case. But along the way I spent a lot of time digging around in the Linux kernel and figured out how to handle UD exceptions to test millions of instructions at a time. If you happen to have similarly misguided aims as me ('*why of course I'll be deliberately running illegal instructions in the kernel*') then I have a solution for you! The only resources I could find suggested modifying the Interrupt Descriptor Table (IDT) or modifying the kernel code and rebuilding it; this solution requires neither and can all be done in C from within your kernel driver.

Note: exception-handling is architecture specific, so the deep dive section below is only valid for x86/x86-64.

**Update: since writing this post, I've since discovered that the canonical method for exception-handling in the kernel is to register fixup addresses in the exception table. See [here](https://www.kernel.org/doc/Documentation/x86/exception-tables.txt) and [here](https://kernelnewbies.org/FAQ/TestWpBit) for details.**

## Post Outline
* [Deep dive into UD exception handling](#deep-dive-into-ud-exception-handling)
* [Die notifier solution for UD](#die-notifier-solution-for-ud)
* [What's going on with the instruction pointer?](#whats-going-on-with-the-instruction-pointer)

# Deep dive into UD exception handling
The code quoted in this section is from [arch/x86/kernel/traps.c](https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/traps.c) and [kernel/notifier.c](https://elixir.bootlin.com/linux/latest/source/kernel/notifier.c) in v4.17.12 of the Linux kernel.

When the processor detects an interrupt or exception, it stops execution of the code it was running and executes the relevant interrupt/exception handler. The handler deals with the interrupt/exception and then returns from the interrupt execution context - it may return to the code that was running, or may return somewhere completely different if it can't recover from the exception (e.g. it decides to kill the program that was running, or a kernel panic is triggered if the exception's completely unrecoverable).

Handlers are defined by the operating system and their addresses in memory are listed in the Interrupt Descriptor Table (IDT). The initial assembly code used to switch into the interrupt execution context isn't relevant here - check out the [Linux Insides guide](https://0xax.gitbooks.io/linux-insides/Interrupts/linux-interrupts-5.html) for details. For the purposes of this post, we can jump straight to the main functionality of `do_invalid_op`, the Linux x86 UD exception handler. This handler is defined with the DO_ERROR macro and is actually just a call to `do_error_trap` with arguments specific to UD. At some stage this `do_error_trap` function kills our program, which is what we want to avoid. So let's check it out and see how we might be able to hijack it.

```c
static void do_error_trap(struct pt_regs *regs, long error_code, char *str,
			  unsigned long trapnr, int signr)
{
	siginfo_t info;

	RCU_LOCKDEP_WARN(!rcu_is_watching(), "entry code didn't wake RCU");

	/*
	 * WARN*()s end up here; fix them up before we call the
	 * notifier chain.
	 */
	if (!user_mode(regs) && fixup_bug(regs, trapnr))
		return;

	if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, signr) !=
			NOTIFY_STOP) {
		cond_local_irq_enable(regs);
		do_trap(trapnr, signr, str, regs, error_code,
			fill_trap_info(regs, signr, trapnr, &info));
	}
}
```

So, if `notify_die` doesn't return `NOTIFY_STOP`, we re-enable local interrupts and then call `do_trap`:

```c
static void
do_trap(int trapnr, int signr, char *str, struct pt_regs *regs,
	long error_code, siginfo_t *info)
{
	struct task_struct *tsk = current;


	if (!do_trap_no_signal(tsk, trapnr, str, regs, error_code))
		return;
	/*
	 * We want error_code and trap_nr set for userspace faults and
	 * kernelspace faults which result in die(), but not
	 * kernelspace faults which are fixed up.  die() gives the
	 * process no chance to handle the signal and notice the
	 * kernel fault information, so that won't result in polluting
	 * the information about previously queued, but not yet
	 * delivered, faults.  See also do_general_protection below.
	 */
	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = trapnr;

	if (show_unhandled_signals && unhandled_signal(tsk, signr) &&
	    printk_ratelimit()) {
		pr_info("%s[%d] trap %s ip:%lx sp:%lx error:%lx",
			tsk->comm, tsk->pid, str,
			regs->ip, regs->sp, error_code);
		print_vma_addr(KERN_CONT " in ", regs->ip);
		pr_cont("\n");
	}

	force_sig_info(signr, info ?: SEND_SIG_PRIV, tsk);
}
NOKPROBE_SYMBOL(do_trap);
```

`do_trap` calls `do_trap_no_signal`, which turns out to be what kills our program:

```c
static nokprobe_inline int
do_trap_no_signal(struct task_struct *tsk, int trapnr, char *str,
		  struct pt_regs *regs,	long error_code)
{
	if (v8086_mode(regs)) {
		/*
		 * Traps 0, 1, 3, 4, and 5 should be forwarded to vm86.
		 * On nmi (interrupt 2), do_trap should not be called.
		 */
		if (trapnr < X86_TRAP_UD) {
			if (!handle_vm86_trap((struct kernel_vm86_regs *) regs,
						error_code, trapnr))
				return 0;
		}
		return -1;
	}

	if (!user_mode(regs)) {
		if (fixup_exception(regs, trapnr))
			return 0;

		tsk->thread.error_code = error_code;
		tsk->thread.trap_nr = trapnr;
		die(str, regs, error_code);
	}

	return -1;
}
```

If there's a way to make `notify_die` return `NOTIFY_STOP` then we can bypass this! Let's take a look at it:

```c
int notrace notify_die(enum die_val val, const char *str,
	       struct pt_regs *regs, long err, int trap, int sig)
{
	struct die_args args = {
		.regs	= regs,
		.str	= str,
		.err	= err,
		.trapnr	= trap,
		.signr	= sig,

	};
	RCU_LOCKDEP_WARN(!rcu_is_watching(),
			   "notify_die called but RCU thinks we're quiescent");
	return atomic_notifier_call_chain(&die_chain, val, &args);
}
NOKPROBE_SYMBOL(notify_die);
```

Hmm, so it returns the result of the `atomic_notifier_call_chain`. What might that be? Intriguingly, these two functions are right below `notify_die` in kernel/notifier.c:

```c
int register_die_notifier(struct notifier_block *nb)
{
	vmalloc_sync_all();
	return atomic_notifier_chain_register(&die_chain, nb);
}
EXPORT_SYMBOL_GPL(register_die_notifier);

int unregister_die_notifier(struct notifier_block *nb)
{
	return atomic_notifier_chain_unregister(&die_chain, nb);
}
EXPORT_SYMBOL_GPL(unregister_die_notifier);
```

A die notifier turns out to be the closest equivalent to a signal handler for a kernel driver! In fact, all we need to do to hijack the handler and stop it killing our program is register a die notifier (the only die notifier in the chain, in this case) in which we re-enable local interrupts (as we skip that line of very crucial code...), modify the instruction pointer so we can skip the faulting instruction, and return `NOTIFY_STOP`. We should also check we're not stuck in an exception loop - this is an indication we've failed to handle the exception correctly and will cause a system hang or kernel panic.

# Die notifier solution for UD
The code below demonstrates a simple die notifier for handling UD exceptions. There's lots of potential to make mistakes (i.e. crash your computer) here, but thankfully I already made all the mistakes for you so you don't have to - pay close attention to the comments in the code! This isn't a complete driver, just the code required for the die notifier. The instruction pointer incrementation to skip past the faulting instruction is exceptionally weird and may be **microarchitecture-specific** - it's only been tested on an Intel Broadwell processor, so test with caution in a VM first before trying on bare metal! If it's wrong for your system, you'll probably get an immediate CPU hang - restart asap and try with different incrementation values. If you get different results from me then please get in touch! I'm really intrigued why this odd behaviour with the instruction pointer occurs. See the next section for further discussion of why it's so weird.

```c
#include <linux/kdebug.h> //for die_notifier
static int opcodeByteCount = 0;

/* by returning NOTIFY_STOP we skip the call to die() in do_error_trap()
but this means we miss important things like re-enabling interrupts!
so we must do all these things here, otherwise the system freezes */
static int opcode_die_event_handler (struct notifier_block *self, unsigned long event, void *data){
    struct die_args* args = (struct die_args *)data;
    if (prevExceptionsHandled < 2){

    	if(args->trapnr == 6){ // UD exception
        prevExceptionsHandled++;
        if(opcodeByteCount > 4) args->regs->ip += (opcodeByteCount - 2);
        else if(opcodeByteCount > 1) args->regs->ip += (opcodeByteCount);
        //this works for 1-3 bytes as long as the instr doesn't page fault
        //(shorter instrs more likely to - I assume missing expected addressing modes/operands)
        else return 0;
        //below is equiv of unexported cond_local_irq_enable(args->regs);
        if(args->regs->flags & X86_EFLAGS_IF){
          local_irq_enable();
        }
    	  return NOTIFY_STOP;
    	}

    	else if(args->trapnr == 13 || args->trapnr == 14){
        //TODO: PF and GP handling. Note GP does not use do_error_trap - slightly different handler
        return 0;
    	}
    	else return 0;
    }

    else {
    	//if too many exceptions or an event we cannot handle, play it safe and let the kernel handle the event and kill the program
    	return 0;
    }
}

//to set up notifier call chain
static __read_mostly struct notifier_block opcode_die_notifier = {
  .notifier_call = opcode_die_event_handler,
  .next = NULL,
  .priority = 0
};

/* this code below should go in a relevant function as required */
register_die_notifier (&opcode_die_notifier);
/* code for executing opcode goes here - see previous post.
   set opcodeByteCount to the length of the faulting instruction
   (ignore GCC function prologue/epilogue - just the instruction itself,
   e.g. opcode = {0x55, 0x90, 0x5d, 0xc3}, opcodeByteCount = 1).
   don't forget to vfree(opcode) after to avoid a memory leak
   */
unregister_die_notifier(&opcodeTesterKernel_die_notifier);  //this is crucial on exit to avoid crash when reloading driver
```

# What's going on with the instruction pointer?
Let's take a closer look at two of the more mysterious lines of code from the solution.

```c
if(opcodeByteCount > 4) args->regs->ip += (opcodeByteCount - 2);
else if(opcodeByteCount > 1) args->regs->ip += (opcodeByteCount);
```

This indicates something very strange I discovered by trial-and-error (ft. lots and lots of CPU hangs) whilst developing the solution. According to the Intel Software Developer's manual, UD exceptions are thrown during decoding (before execution) of an instruction, and the return IP address is the address of the start of the instruction. So to skip past the instruction, we *should* simply need to increment the IP by opcodeByteCount, and this is the case if the instruction is 1-4 bytes in length. But if it's 5 bytes or longer incrementing by opcodeByteCount causes crashes - the return IP address provided by the CPU appears to be *2 bytes into the instruction*, and I have no idea why. This is something I want to investigate further - I'm not sure whether it's a quirk of Intel's instruction decoder or whether Linux's interrupt handler is somehow reporting the wrong return IP.

**Note: since a BIOS update I can no longer reproduce this behavior on the CPU I was testing or on other Intel microarchitectures. I believe this may have been a microarchitecture-specific issue fixed by a microcode patch. The instruction pointer should now always be incremented by opcodeByteCount.**
