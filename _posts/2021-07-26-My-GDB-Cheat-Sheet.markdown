---
layout: post
title: My GDB Cheat Sheet (In Progress)
status: done
type: post
published: true
comments: true
date: '21/07/26 17:00 '
---
These are my notes on how to use GDB. They are not complete and I will probably continue updating them, however feel free to use them while I do.


### What is GDB

GDB or the **G**NU **D**e**B**ugger is a basic command line debugger that comes with most linux OS's. It is an incredibly useful tool when working with unknown binary's and in debugging your own code. I primarily use it for CTFs so that is what the below commands will focus on, but if you are planning to use it for your own code, make sure to google for the proper compile options to make it work better.



## startup
You can have gdb run a series of commands listed in file z by starting gdb with the flags

```bash
gdb -command=z x
```

otherwise starts up the program
```bash
gdb program_to_debug
```

## During Debugging
### set flags

set $eflags |= (1 << $flagNum) where zero flag is 6

### Stripped binaries

occosionally you have to run it once in order to populate the data and then do

[Working With Stripped Binaries in GDB](https://tr0id.medium.com/working-with-stripped-binaries-in-gdb-cacacd7d5a33)

```gdb
info file
```

and we can break at the entry point listed

to disas a region in front of you

```gdb
x/30i $eip
```

### Common gdb Commands

-   watch variable or (z > 28) - This breaks when the variable changes or changes when the conditional is met
-   TUI (or terminal user interface) displays the code as you step through it enter/exit CTRL-X-A
-   Display the stack frame: frame $number, where number is the depth backtrace is all of them

## Extensions
There are several exspansions to GDB, the one that I use primarally is 
[GEF](https://gef.readthedocs.io/en/master/) or GDB Enhanced Features.
I'll be honest, I haven't really had time to get to know these extra funtions yet, but the below image shows what the default screen looks like from the docs. You can see what all of the registers are, the memory, and code. All of this is very useful.

![GEF start page](https://i.imgur.com/E3EuQPs.png)
