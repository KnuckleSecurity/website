---
layout:   post
title:   Memory Segmentation
description:   Memory Segmentation 
date:   2022-12-03 01:20:00 +0300
image: '/assets/img/posts/memory-segmentation/banner2.jpg'
tags:   [c,assembly,x86,32-bit,memory-segmentation,memory-segmentation]
featured:   false
---

# Segmentation on x86 Architecture

When a program is compiled and run, memory is divided into five segments: stack, heap, data, bss and text.
Each segment of the memory serves a certain purpose. The **text** segment is also called **code** segment.
This segment contains the compiled program's assembled machine code instructions to be read by the CPU.
The execution process is non-linear. When the executable first loaded into memory, EIP (instruction pointer) 
register is first set to the first instruction loaded in the text segment. Then, the processor gets into the 
execution loop state as following:

- Read the instruction written in the memory address that EIP points to.
- Adds the byte length of the instruction to EIP.
- Execute the instruction that was read in the first step.
- Loop back. Go back to the fist step.

As mentioned before, execution process is non-linear, sometimes in the text segment, instruction pointer will 
encounter a call instruction or a jump, which basically sets the EIP to a different address of the memory instead
of incrementing it by the previously read instruction's byte length. If the EIP is changed in the execution step,
EIP will point and loop back to the first step back, and then execute that instruction found at the address of
whatever EIP was changed to.<br>

## Text Segment
  Write permission in disabled in the text segment since it is not used to store variables, only code. This security 
mechanism is there for to prevent malicious activity which involves modifying the pre-compiled code into something else.
Any write attempt into this segment of the memory will cause the program to alert the user that something restricted 
happened, and program will be killed. Since that segment is in read-only mode, it can be shared among different copies 
of the program, allowing multiple proccesses or threads to run at the same time without problem. Plus, text segment
has a fixed size since nothing ever changes in it.

## Data and BSS Segment
  The data and bss segments are used to store static and global variables. The **data segment** is being used to store 
**initialized** global and static variables while the **bss segment** is being used to store their uninitialized
counterparts. Those segments also fixed size segments regardless of the fact that they are writable. Global
variables persist, despite the functional context. Both global and static variables are able to persist
because they are stored in their own memory segments.

## Heap Segment
  The **heap segment** is a memory segment which the programmers have a direct control over. It is programmable.
Blocks of the heap segment can be allocated and used for whatever the program needs. Unlike the text, bss and the 
data segment, heap does not have a fixed size, it can grow larger or smaller. The memory chunks within 
the heap is managed by de-allocator and allocator algorithms. Those algorithms enable programmers to reserve
a region of the heap memory for a required use or remove reservation to allow that portion of the memory 
to be reusable again for a later reservation. Heap will dynamically shrink and grow depending on how much memory 
is allocated for use. The growth of the heap moves downward toward the high memory addresses.

## Stack Segment
  The **stack** segment also has variable size and is used as a temporary pile 
to store local function variables and context during function calls. When a program calls for a function, EIP will 
point to the first instruction written for that function, and that function will have its own variables within its 
function scope, and a new stack frame will be created for that function. After a function call EIP does not immidiately
jumps to the address written on it, first the CPU pushes all the parameters passed to that function to the stack, then, since 
the EIP have to go back to the instruction where it was left before the function call, the return address and the
stack base pointer will be passed to the stack. When the last instruction of the called function is read by the EIP,
return address and base pointer address pushed to the stack before the function call will be read and by the 
instruction and stack pointer, and execution will continue from where it was left. All of this information is stored 
together on the stack in what is collectively called a stack frame. The stack contains many stack frames.<br>

In general computer science terms, a stack is an abstract data structure<br>
that is used frequently. It has first-in, last-out (FILO) ordering, which means the
first item that is put into a stack is the last item to come out of it. 
You can’t get the first bead off until you have removed all the other beads. When an
item is placed into a stack, it’s known as pushing, and when an item is removed
from a stack, it’s called popping.<br>

Stack segment, as the name implies,
is a stack data structure, which stores all the stack frames. The ESP (stack pointer) is used to store the address 
of the last data pushed onto the stack. That address is constantly changin as the PUSH and POP operations are executed 
by the CPU. As this behavior being dynamic, just like heap, stack segment does not have a fixed size. Stack segment 
grows upward in a visual listing of memory, toward lower memory addresses, as opposed to the heap.<br>

As I briefly mentioned before, when a function called, several things are pushed into to the stack.
The EBP (base pointer, sometimes called frame pointer FP), is used to reference local function variables
in the current stack frame, it stores the starting address of the active stack frame. Each stack frame
contains parameters to the function, its local variables, and two pointers that are neccesarry to put things
back the way they were. The saved frame pointer (SFP) and the return address. The SFP is used to restore EBP to 
its previous value, which is the starting address of the previous stack frame. And the return address is used to 
restore EIP to the next instruction found after the function call. This restores the functional context of the 
previous stack frame. 

## Stack Practices

The following C code has two functions, main() and test_function().
![](/assets/img/posts/memory-segmentation/9.png)<br>

Inside the main function the program declares a function called test_function(). This function 
takes four integer parameters. The local variables for this function are an integer called flag
and a 10 byte long character array called buffer. The machine instructions for this code will be 
placed in the text segment while the local variables and the parameters stored in the stack. After
compiling the program, we can examine the inner mechanisms with GDB (GNU debugger). The following 
picture illustrates the disassembled machine instructions for main() and test_function().
The main() function starts at 0x08048344. There is something called **function prologue** or 
**procedure prologue**. Each function has to start with a function prologue. The first three 
instructions for test_function() and the first six instructions for main() function are the procedure 
prologue instructions. They save base pointer in the stack, and open up space in the 
stack for the local variables. Depending on which compiler being used the disassembled instructions and the 
function prologue would differ slightly. However, in general, those instructions are required in order to build the 
stack frame for the called function.<br>

Machine instructions for main().
![](/assets/img/posts/memory-segmentation/10.png)<br>

Machine instructions for test_function().
![](/assets/img/posts/memory-segmentation/11.png)<br>

After the initial execution of the program, the main() function executes its own function prologue in those 
first six instructions. When the stack frame is set, main() function immediately calls for the test_function(). When the
the test_function() is called, after the stack frame is created, four values are pushed to the stack in 
reverse order, since stack is first in lost out (FILO). The arguments are 1 ,2 ,3 , 4. Therefore, they are 
pushed in the stack as 4, 3, 2 and finally 1. Those values correspond to the a, b, c, d variables.
{% highlight assembly %}
0x08048367 <main+16>: mov DWORD PTR [esp+12],0x4
0x0804836f <main+24>: mov DWORD PTR [esp+8],0x3
0x08048377 <main+32>: mov DWORD PTR [esp+4],0x2
0x0804837f <main+40>: mov DWORD PTR [esp],0x1
{% endhighlight %}

In C programming language each integer uses 4-bytes. Since a RAM block is 1-byte, the value of those 
integers will be spread into four consecutive RAM addresses. As you can see first integer 0x1 written
at the first address where the newly created stack frame's pointer points. esp, esp+1, esp+2, esp+3 will store 
0x1 as 0x00000001. After all those four parameters are pushed to the stack, the progam calls for the test_function(). 
Before setting the EIP to the first instructions of the test_function(), the **call** instruction will push the return 
address to the stack in order to restore the EIP when the execution of the called function is over. 
In this case, the return address would point to the leave instruction in main() at 0x0804838b

The call instruction both stores the return address on the stack and jumps
EIP to the beginning of test_function(), so test_function()’s procedure prologue
instructions finish building the stack frame. In this step, the current
value of EBP is pushed to the stack. This value is called the saved frame
pointer (SFP) and is later used to restore EBP back to its original state.
The current value of ESP is then copied into EBP to set the new frame pointer.
This frame pointer is used to reference the local variables of the function
(flag and buffer). Memory is saved for these variables by subtracting from
ESP. In the end, the stack frame looks like this:

![](/assets/img/posts/memory-segmentation/12.png)<br>

We can watch the stack frame construction on the stack using GDB. In the
following output, a breakpoint is set in main() before the call to test_function()
and also at the beginning of test_function(). GDB will put the first break-
point before the function arguments are pushed to the stack, and the second
breakpoint after test_function()’s procedure prologue. When the program is
run, execution stops at the breakpoint, where the register’s ESP (stack pointer),
EBP (frame pointer), and EIP (execution pointer) are examined.

![](/assets/img/posts/memory-segmentation/13.png)<br>

As you can observe from the picture, the EIP points to the 0x8048367 <main+16>, 
where the EIP instructs the program to push the first parameter of the test_function() to 
the stack. However, that instruction is not executed yet since there is a breakpoint. 
It also means that since the call instruction is not invoked yet, the current stack frame is 
still belong to the main(). If we subtract the current value of the EBP from the ESP we can find out how many bytes are allocated 
in the stack for the main() function's stack frame.<br>
0xbffff818 - 0xbffff800 = 0x18<br>
0x18 (24) bytes are allocated in the stack for the main() function's stack frame. It is possible to verify this value
by observing the third instruction of the main() function's machine code where it subtracts 0x18 from the ESP.
Then, we can observe the upcoming five instructions after the first breakpoint. The CPU will push all the parameters
to the stack then invoke the call instruction.<br>

First, let's see how does the stack frame allocated for main() look like before and after the breakpoints.<br><br>
Before the call:
![](/assets/img/posts/memory-segmentation/14.png)<br>
Before the CPU pushes those integer parameters to the stack in order to 
prepare for the upcoming function call, this is how those allocated 24 bytes look like.<br>

After the call:
![](/assets/img/posts/memory-segmentation/16.png)<br>

The next breakpoint is just after the **function prologue** for the test_function(). Therefore, continuing
built the new stack frame. 
As you can see those function parameters are located at the top of the main() function's stack. The key takeaway
from that behavior is that, those function parameters won't be located in test_function()'s stack frame. Instead,
they will be referenced via its own base pointer (EBP).<br>

And this is how newly created stack fame looks like: 
Just like every 
local variable, flag and char local variables will be referenced relative to the frame pointer (EBP).
![](/assets/img/posts/memory-segmentation/17.png)<br>

You may wonder, in test_function's stack frame, none of the values listed are corresponds neither to the return 
address nor to the saved frame pointer. Those values are stored between the two stack frames. There is an 8-byte address gap 
between the new EBP and the old ESP. If that was not the case the new EBP would be sharing the old EBP's value, which is 
0xbffff800. Instead, the new value for the EBP is 0xbffff7f8.<br>
0xbffff800 - 0xbffff7f8 = 0x8<br>

Therefore, the new EBP's address stores the last byte (research little endian byte) of the saved frame pointer (SFP).
If we go further 8 more bytes from the new EBP, we will hit to the address of the old ESP. And that address
will be storing the last byte of the first integer parameter passed to the test_function(). Let's see.

![](/assets/img/posts/memory-segmentation/18.png)<br>

By referencing the new EBP, program can locate the parameters, SFP and return address.

![](/assets/img/posts/memory-segmentation/19.png)<br>

| Number               | Correlation                | 
|:---------------------|:---------------------------|
|1                     | Saved frame pointer (SFP)  |
|2                     | Return address             |
|3                     | int a                      |
|4                     | int b                      |
|5                     | int c                      |
|6                     | int d                      |

After the execution finishes, the entire stack frame is popped off of the
stack, and the EIP is set to the return address, so the program can continue
execution. If another function was called within the function, another stack
frame would be pushed onto the stack, and so on. As each function ends, its
stack frame is popped off of the stack, so execution can be returned to the
previous function. This behavior is the reason this segment of memory is
organized in a FILO data structure.
The various segments of memory are arranged in the order they
were presented, from the lower memory addresses to the higher memory
addresses. Since most people are familiar with seeing numbered lists that
count downward, the smaller memory addresses are shown at the top.
Some texts have this reversed, which can be very confusing; so for this
paper, smaller memory addresses
are always shown at the top. Most
debuggers also display memory in
this style, with the smaller memory
addresses at the top and the higher
ones at the bottom.
Since the heap and the stack
are both dynamic, they both grow
in different directions toward each
other. This minimizes wasted space,
allowing the stack to be larger if the
heap is small and vice versa.

![](/assets/img/posts/memory-segmentation/20.png)<br>

# Memory Segments in C language

In C programming language, just like the other languages, compiled code take its place in the code segment 
while, while variables sits in the remaining segments. Depending on the how the variable is defined, the variables
will reside in different segments. If a variable defined at the outside of a function, it is considered as a **global** 
variable. The **static** keyword is can also be prepended to any variable decleration to make the variable static. If a 
static or global variable initialized with a value, they will be stored in the **data segment**; otherwise, these 
variables will be located in the BSS segment. The memory on the heap memory must be allocated first by using a 
memory allocation function called malloc(). In general, pointers are used to reference memory on the heap segment.
Finally, the remaining variables will reside in the stack. Since the stack segment contains many stack frames
within, stack variables can maintain uniqueness within different functional contexts.<br>

The following code will explain these concepts in C.

![](/assets/img/posts/memory-segmentation/21.png)<br>
![](/assets/img/posts/memory-segmentation/22.png)<br>

Most of the concepts are self-explanatory since the variable names are descriptive and each variable grouped 
with their segment couples. Two identical named stack variables are declared in order to illustrade 
their functional contexts. The heap variable is declared as an integer pointer, which will point to the memory address 
allocated on the heap segment. malloc() function is called allocate four bytes in the heap segment. Since the newly 
allocated heap memory could be of any data type, the malloc() function returns a **void** pointer. It then type casted
into an integer pointer.

Summary:
- Initialized **global** and **static** variables will be stored in the Data Segment.
- Uninitialized **global** and **static** variables will be stored in the BSS Segment.
- Heap variables will be stored in the Heap Segment.
- Stack variables, in the both functions, will be stored in the Stack Segment in different Stack Frames.

The first two initialized variables have the lowers memory addresses. It is because the Text Segment starts from
the lowest memory address and the Data Segment resides where the Text Segment ends. The next two uninitialized variables 
are stored in the BSS segment. The memory addresses for those variables are slightly bigger then the addresses which points
to the variables in the Data Segment since the BSS Segment being initialized where the Data Segment ends. Since both 
segments are fixed size, there is verry litle wasted space, and the addresses are not very far apart.<br>

Heap Segment is not fixed size, and more space can be allocated on the run. And the first address of the Heap Segment
starts where the BSS Segment ends. Finally, the stack variables have the greatest addresses since the Stack Segment starts
where the memory ends. Stack Segment is also not fixed size, which means it will grow towards to the heap whereby heap 
grows toward to the stack. This allows both memory segments to be dynamic without wasting any unnecesarry space in memory.<br>

The first stack_var in the main() function's context resides in the Stack Segment within its own stack frame. The second 
stack_var in the function() has its own functional context, thereby that variable is stored within a different stack
frame in the Stack Segment. When the function() is called, CPU will initialize a function prologue in order to create 
a new stack frame in the stack to store stack_var (among other data) for function()'s context. Since the stack grow
back up toward the Heap Segment with each stack frame and pushed data, the memory address for the second stack_var 
(0xbffff814) is smaller than the address for the first stack_var (0xbffff834) found within main()'s context.

# Using the Heap

Using the other segments of the RAM is an automated process, programmer does not have to be attentive in to the 
segmentation process. However, when it comes to heap, the programmer have to manually set and calibrate the use 
of the variables. As previously illustrated, heap memory allocation is done by a memory allocation function called 
malloc() in C. This function accepts a size as its one and only argument and reserves that much space in the heap
segment as bytes, returning initial address of that space as a void pointer. If the malloc() function is not able 
to allocate any space for some reason, it will return a NULL pointer with a value of 0 instead. The corresponding 
de-allocation function is free(). This function accepts a pointer as its one and only argument and frees that 
previously reserved memory space for later use.

![](/assets/img/posts/memory-segmentation/23.png)<br>
![](/assets/img/posts/memory-segmentation/24.png)<br>

This program takes a command line argument and uses this argument as the size for the heap reservation which 
will be allocated by the malloc() function. Then it uses malloc() and free() functions to allocate and de-allocate
the memory. printf() statements are being used in order to debug what is actually happening. Since the malloc()
function does not know what kind of data will be stored in that reserved space, the void pointer should be type
casted into the appropriate type. After each malloc() call, there is an error-checking functionality that check 
whether or not the reservation process were successfull or not. If the allocation fails, fprintf() is called to 
direct a text message to the standart error file descriptor and program exits.

![](/assets/img/posts/memory-segmentation/25.png)<br>
