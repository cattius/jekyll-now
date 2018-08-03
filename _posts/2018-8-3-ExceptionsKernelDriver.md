---
layout: post
title: How to handle CPU exceptions in a Linux kernel driver
tags: [blog, research, undocumentedCPU]
---

This is part of a series of blog posts on my Undocumented x86-64 Opcodes [research project](/research). Previous research into undocumented opcodes (Chris Domas' [Sandsifter project](https://github.com/xoreaxeaxeax/sandsifter)) had tested them in ring 3 (user mode), but not in ring 0 (kernel mode). I was very keen to test out opcodes in ring 0 because many of them failed with #GP exceptions in ring 0, which can be an indication that an opcode exists but is privileged (although there are lots of other causes of #GP). I was convinced I was going to discover some really interesting undocumented instructions if only I could run them in ring 0. Spoiler alert: sadly this wasn't the case. Along the way I spent many hours digging around in the Linux kernel figuring out how to handle #UD exceptions (to test millions of instructions at a time), and if you happen to have similarly misguided aims as me ('why of course I'll be deliberately running illegal instructions in the kernel') then I have a solution for you! The only resources I could find suggested modifying the Interrupt Descriptor Table (IDT) or modifying the kernel code and rebuilding it; this solution requires neither and can all be done in C from within your kernel driver.

Note: exception-handling is architecture specific, so the deep dive section below is only valid for x86/x86-64. The odd observed instruction pointer behaviour used to implement the die notifier solution is likely microarchitecture-specific and has only been tested on an Intel x86-64 Broadwell processor; test with caution in a VM first before trying on bare metal! If you get it wrong, you'll probably get an immediate CPU hang - restart asap. If you get very different results then please get in touch! I'm keen to understand why this odd behaviour with the instruction pointer occurs.

## Post Outline
* [Deep dive into UD exception handling](#deep-dive-into-ud-exception-handling)
* [Die notifier solution for UD](#die-notifier-solution-for-ud)
* [Handling GP and PF](#handling-gp-and-pf)

# Deep dive into UD exception handling
When an #UD exception occurs, the kernel switches into the exception execution context (see [entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)) and then calls `do_invalid_op`.

The code quoted in this section is from [arch/x86/kernel/traps.c](https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/traps.c) and ... in v4.17.12 of the Linux kernel.

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

Hmm, so it returns the result of the `atomic_notifier_call_chain`. Intriguingly, these two functions are right below `notify_die` in kernel/notifier.c:

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

A die notifier turns out to be the closest equivalent to a signal handler for a kernel driver. 



# Die notifier solution for UD

# Handling GP and PF
Handling these exceptions is a work in progress - I haven't found a reliable method for resolving these yet, but I'm optimistic that it can be done.
