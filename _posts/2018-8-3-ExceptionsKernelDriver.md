---
layout: post
title: How to handle CPU exceptions in a Linux kernel driver
tags: [blog, research, undocumentedCPU]
---

This is part of a series of blog posts on my Undocumented x86-64 Opcodes [research project](/research). Previous research into undocumented opcodes (Chris Domas' [Sandsifter project](https://github.com/xoreaxeaxeax/sandsifter)) had tested them in ring 3 (user mode), but not in ring 0 (kernel mode). I was very keen to test out opcodes in ring 0 because many of them failed with #GP exceptions in ring 0, which can be an indication that an opcode exists but is privileged (although there are lots of other causes of #GP). I was convinced I was going to discover some really interesting undocumented instructions if only I could run them in ring 0. Spoiler alert: sadly this wasn't the case. Along the way I spent many hours digging around in the Linux kernel figuring out how to handle #UD exceptions (to test millions of instructions at a time), and if you happen to have similarly misguided aims as me ('why of course I'll be deliberately running illegal instructions in the kernel') then I have a solution for you! The only resources I could find suggested modifying the Interrupt Descriptor Table (IDT) or modifying the kernel code and rebuilding it; this solution requires neither and can all be done in C from within your kernel driver.

## Post Outline
* [Linux kernel UD exception handling](#linux-kernel-ud-exception-handling)
* [Die notifier solution for UD](#die-notifier-solution-for-ud)
* [Handling GP and PF](#handling-gp-and-pf)

# Linux kernel UD exception handling

When an #UD exception occurs, the kernel switches into the exception execution context (see [entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)) and then calls `do_invalid_op`.



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

Intriguingly, however, these two functions are right below `notify_die` in kernel/notifier.c:

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

# Die notifier solution for UD

# Handling GP and PF
Handling these exceptions is a work in progress - I haven't found a reliable method for resolving these yet, but I'm optimistic that it can be done.
