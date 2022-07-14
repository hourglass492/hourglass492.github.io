---
layout: post
title: A Breakdown of the ROP Emporium Challenges X64
comments: true
date: 2022-07-13
---
Note: Github repo and writeup avaliable here [https://github.com/hourglass492/hourglass492.github.io/tree/master/_posts/ROP_emporium](https://github.com/hourglass492/hourglass492.github.io/tree/master/_posts/ROP_emporium) in there broken down form and [https://github.com/hourglass492/ROP-Emporium-x64](https://github.com/hourglass492/ROP-Emporium-x64) in an overaching form with in the git repo I used for the project.


# General overview 

The rop emporium is a series of challenges designed to help teach about the concept of return oriented programing (ROP) which is an exploit technique that uses already existing chunks of machine code in an binary as part of the executable. This is an effective way of getting around write xor execut w^x stack protection.

They attempt to:
>ROP Emporium provides a series of challenges that are designed to teach ROP in isolation, with minimal requirement for reverse-engineering or bug hunting. Each challenge introduces a new concept with slowly increasing complexity.

While the challenges are supplied in x86, x86_64 amd, ARMv5 & MIPS binary format, I will be doing the challenge in the x86_64 amd format for easy use on my personal computer.

In order to attack these challenges, I will primarily be using GDB with pwndbg as my debugger and the pwntools Python library to craft my exploits. In particular the ropper tool built into pwndbg is being used to find the useful rop gadgets. Some of the rabin2 functions are also very useful in understanding what is in the binary.



## Why ROP


### Basic Buffer overflow on stack

To understand how ROP exploits came about, you have to understand a little bit of the history of binary exploits. Probably one of the first and simplest binary types of binary exploits that almost always gave remote code execution is a buffer overflow on the stack. In the most basic form, the program reads in more data then it was designed to handle. The extra data that get's read in overwrites important data in the program.

One of the things that communally gets over written is the return pointer in the stack frame. Without getting into too much detail, this is a value that tells the program what code to execute once the current function finishes executing. If the attacker controls this value, they can tell the program to execute whatever they want. In fact, they can even give the program code and tell it to execute that (that given code is normally called shell code).

```
         | stack          |
         | -------------- |
32 bytes | shell code     |<- 
8  bytes | shell code     |   \
8  bytes | shell code     |    |
8  bytes | base pointer   |   /
8  bytes | return pointer |-- 
8  bytes | -------------- |

```


In order to solve (or mitigate at least) that problem, operating systems began to implement the W^X (write xor execute) protection on the computer. The basic idea is if the program can write to a location in memory, it can't execute it and if a location in memory can be executed, it can't be written to. (Another prevention technique was Address space layout randomization (ALSR), but that's not directly related to this topic)

So the previous attack wouldn't work, because as soon as the program attempts to execute the shell code (in a writable location in memory) the program fails and exits, preventing the attack.


[ROP x64 Challenge 1](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-1) demonstrates this type of attack without the need for shell code.

### Getting to ROP

So, how do attackers keep exploiting these programs? After all, buffer overflows still exist on in the world even if you can't just drop your own shell code in and pwn them. The answer is to use the code that already exists in the binary. It's perfect because it's already marked executable so the W^X protection won't affect the attack. The next question is how to reuse the code already there.

Since we control the stack, we can make it look however we want. Normally, a stack is a basically a list of functions to return to and their local variables. So we create a list of functions (the pre existing code) to return to which effectively allows us to reuse the code. In fact, it doesn't even have to be an intended function in the binary. Any set of instructions that ends in the return instruction works.

[ROP x64 Challenge 2](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-2) shows how you can combine an unintended and intended set of instructions to win.

### More techniques
#### Using built in functions

However, to really expand what you can do it's nice to be able to call the intended functions in the binary. To do that, you have to understand how arguments are passed to functions. In x86_64, this is done in 4 registers (at least for the first 4 arguments). In order to properly call a function you need to control these registers.


[ROP x64 Challenge 3](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-3) shows how to do this.


#### Writing to memory

Although you can't execute any code that you write into memory in the process, it can have a bunch of other uses. For example, a function may need a string to execute properly. One challenge that shows this is [ROP x64 Challenge 4](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-4)


### Challenges with ROP

#### Write restrictions
However, things aren't all roses. Attackers often don't have complete control over what they can overflow the binary with. [ROP x64 Challenge 5](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-5) shows how needed values can be created in memory even if they can't be written in the initial stack smash

#### Size problems
ROP exploits require a certain amount of space and if you can't smash the stack with enough data it may be impossible to fully exploit a program. However, it's possible to store part of a ROP exploit in another space and then pivot to it which toften takes significantly less space as demonstrated in [ROP x64 Challenge 6](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-6).

#### Bad gadgets
Even with the ability to call any part of code that ends in a ret instruction, we may not have everything that we want to exploit a binary. Rather, some creative usage of instructions may be needed such as in [ROP x64 Challenge 7](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-7).


#### Small or hardened Binaries
Finally, a binary may just not have enough gadgets to exploit because it is too small or you're just unlucky. However, there is one last hope known as uROPs or universal ROPs. These are found by looking at the boiler palte code added by gcc (or other compiler) and looking for gadgets in those functions. [ROP x64 Challenge 8](https://nicholaskrabbenhoft.com/ROP_emporium/ROP-x64-Challenge-8) is an example of that.
