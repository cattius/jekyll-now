---
layout: page
title: Research
permalink: /research/
---
My research interests span **cybersecurity, neuroscience, deep learning, and AGI (artificial general intelligence)**.

I'm particularly fascinated by how these areas interact. There's a deep symbiosis between neuroscience and deep learning, and as we progress towards developing AGI ensuring it is secure and resistant to adversarial input will become paramount. Researchers have already carried out side-channel attacks on the brain, and an unprotected digital AGI would be far more accessible to attackers than a silly human which needs an EEG attached to it first. But I also believe that neuroscience, deep learning and AGI connect on a much more fundamental level: **computation is about more than just circuits and digital computers**. The behaviour of any complex system can be interpreted as 'computation' - even a puddle of water! I believe that if we ever do find the 'secret' to intelligence it'll be something shared by all complex systems (see Stephen Wolfram's [principle of computational equivalence](http://www.wolframscience.com/nks/chap-12--the-principle-of-computational-equivalence/) for more on this).

Our good old circuit-based digital computers are also full of secrets. Once you get down to the low-level software and hardware implementation details there's a lot of "unexpected" behaviour (unexpected by other developers...or sometimes even by the original developers/engineers!), and this almost always leads to security vulnerabilities - from Rowhammer, Meltdown and the Spectre class of vulnerabilities to the shadowy unknown workings of Intel ME / AMD PSP and undocumented CPU instructions and MSRs. I'm especially interested in **undocumented CPU behaviour, side channels, and information leakage**.

# Research Projects

## Undocumented CPU Behaviour:
* Undocumented x86-64 opcodes
    * *'Automated Analysis of Undocumented Intel x86-64 Instructions'* (2018): [slides](https://github.com/cattius/opcodetester/blob/master/presentation.pdf), [paper](https://github.com/cattius/opcodetester/blob/master/thesis.pdf), [OpcodeTester tool](https://github.com/cattius/opcodetester)
    * [How to execute undocumented opcodes (or arbitrary machine code) in C on Linux](/#)
    * [How to handle CPU exceptions in a Linux kernel driver](/#)
    * [Exploring the vast search space of undocumented x86-64 opcodes](/#)
    * [Introducing OpcodeTester](/#)
* Further investigation (MSRs, Intel ME, exception and decoding behaviour) ongoing
