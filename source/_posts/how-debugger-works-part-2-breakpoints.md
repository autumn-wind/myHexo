---
title: 调试器工作原理：Part 2 － 断点
---

这篇文章基本上是国外一篇博客的翻译，部分地方做了一定删改。

原文链接：[http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints)

以下进入正文：

------------------------------------------------------------------------------

这是关于[“调试器原理系列文章”](http://eli.thegreenplace.net/tag/debuggers)的第二篇。确保你在阅读这一篇前阅读过[前一篇](http://autumn-wind.github.io/2016/05/02/how-debugger-works-part-1-basics/)。

<!-- more -->

### 在这一篇中

我将向大家示范在调试器中断点是如何实现的。设置断点是调试器的两大重要功能——另外一个是可以查看和修改被调试进程的内存内容。在这个系列的[第一篇](http://autumn-wind.github.io/2016/05/02/how-debugger-works-part-1-basics/)文章中我们已经见识了另一项功能的示范，但是断点对我们而言仍然是神秘的。在这篇文章的最后部分，它将不再神秘。

### int 3——理论

在x86体系结构里，断点实际上是使用一个特殊的软中断（也被称作“陷阱”）——*int 3*实现的。*int*其实是x86的对于“异常指令”的行话——它可以用来调用预先定义好的中断处理程序。x86用8位的操作数来支持*int*指令，用以表示发生的软中断具体的中断号，所以理论上可以支持256种中断。0～31号中断被Intel保留作己用，并且3号中断是我们所感兴趣的——它被称作“陷入调试的陷阱”。

为了准确明了，我将在此直接引用“Intel's Architecture software developer's manual, volume 2A”里面关于*int 3*的描述。

> The INT 3 instruction generates a special one byte opcode (CC) that is intended for calling the debug exception handler. (This one byte form is valuable because it can be used to replace the first byte of any instruction with a breakpoint, including other one byte instructions, without over-writing other code).

括号里的句子非常重要，但现在解释它还为时过早。在这篇文章的稍后部分我们再回过头来关注这句话。

### int 3——实践

是的，了解事情背后的原理是非常好的，但是，它到底是什么意思呢？我们到底怎样用*int 3*实现断点呢？或者用编程领域对于“Q&A”的行话——*“请给我看代码！”*

在实践中，这非常简单。一旦进程执行了*int 3*指令，操作系统会挂起这个进程。对于Linux（在这篇文章中我们将只关注Linux上的断点的实现），它还会向进程发送一个信号——*SIGTRAP*。

老实说，这就是所有的东西！现在回顾一下在前一篇中我们讲过，调试进程（调试器）会收到所有子进程（或者它所调试的进程）收到的信号，现在你能感觉到我们在向什么样的方向前进了吧。

就是这样，我不再喋喋不休的谈论体系结构相关的一些事情了。现在是时候看看例子和代码了。

### 手动设置断点

我现在将向大家展示一个在程序中设置断点的例子。被调试程序的代码如下：

```nasm
section    .text
    ; The _start symbol must be declared for the linker (ld)
    global _start

_start:

    ; Prepare arguments for the sys_write system call:
    ;   - eax: system call number (sys_write)
    ;   - ebx: file descriptor (stdout)
    ;   - ecx: pointer to string
    ;   - edx: string length
    mov     edx, len1
    mov     ecx, msg1
    mov     ebx, 1
    mov     eax, 4

    ; Execute the sys_write system call
    int     0x80

    ; Now print the other message
    mov     edx, len2
    mov     ecx, msg2
    mov     ebx, 1
    mov     eax, 4
    int     0x80

    ; Execute sys_exit
    mov     eax, 1
    int     0x80

section    .data

msg1    db      'Hello,', 0xa
len1    equ     $ - msg1
msg2    db      'world!', 0xa
len2    equ     $ - msg2
```

我现在先使用汇编代码，这会让我们的实验过程更加清晰。上述代码执行的效果是在终端第一行打印一个“Hello”，然后在第二行打印一个“World！”。它和上一篇文章中展示的一个例子非常相似。

我现在想在第一个输出后，但是在第二个输出前，设置一个断点。让我们就让它设置在紧接第一个 *int 0x80* 之后的那条 *mov edx, len2* 指令。首先，我们需要知道这条指令对应的地址是什么。运行 *objdump -d*：

```
traced_printer2:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000033  08048080  08048080  00000080  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         0000000e  080490b4  080490b4  000000b4  2**2
                  CONTENTS, ALLOC, LOAD, DATA

Disassembly of section .text:

08048080 <.text>:
 8048080:     ba 07 00 00 00          mov    $0x7,%edx
 8048085:     b9 b4 90 04 08          mov    $0x80490b4,%ecx
 804808a:     bb 01 00 00 00          mov    $0x1,%ebx
 804808f:     b8 04 00 00 00          mov    $0x4,%eax
 8048094:     cd 80                   int    $0x80
 8048096:     ba 07 00 00 00          mov    $0x7,%edx
 804809b:     b9 bb 90 04 08          mov    $0x80490bb,%ecx
 80480a0:     bb 01 00 00 00          mov    $0x1,%ebx
 80480a5:     b8 04 00 00 00          mov    $0x4,%eax
 80480aa:     cd 80                   int    $0x80
 80480ac:     b8 01 00 00 00          mov    $0x1,%eax
 80480b1:     cd 80                   int    $0x80
 ```

 所以，我们想设置断点的指令在0x08048096处。等等，这并不是真正的调试器做的，对吗？真正的调试器在代码的某一行或者某个函数入口设置断点，而不是通过赤裸裸的指令的内存地址。非常正确。但是我们距离这一点仍然很遥远——如果要像真正的调试器那样设置断点，我们还得首先处理编译产生的符号以及调试信息等，因此我们还需要在这个系列中再通过一两篇文章来讲述相关主题。现在，我们只需要处理通过指令的内存地址设置断点的情形。

### 在调试器中使用 *int 3* 设置断点

 为了在被调试的进程中的某个指令地址处设置断点，调试器做了以下两点：

 1. 记住被调试进程的指令地址处以前的值是多少
 2. 将那个指令地址的第一个字节换成*int 3*指令

然后，当调试器要求操作系统运行被调试的进程时（使用上一篇中提到的 *PRACE_CONT* 请求），被调试进程会开始执行并且最终会执行*int 3*指令，在那儿它会被挂起，并且操作系统会向它发送一个信号。这个时候就又是调试器登场的时候了，因为它会收到一个它的子进程（或者被调试的进程）被挂起的信号。然后它能做的是：

1. 用之前存储的指令值替换 *int 3* 指令。
2. 将被调试进程的指令指示器（*$eip*）往回指一条，因为它现在指向的是刚执行完的 *int 3* 指令的下一条指令。
3. 允许用户这个时候获取被调试进程的相关信息。因为这个时候被调试进程已经是执行到目标地址处被挂起，所以这个时候调试器可以让你去查看被调试进程的变量，寄存器，栈信息等。
4. 当用户想要继续运行的话，调试器会小心将断点放回去（因为在第一步中断点已经被移除），然后继续执行后面的指令，除非用户要求调试器删掉那个断点。

现在让我们来看看上述步骤是如何被翻译成具体代码的。我们将用到在上一篇文章中用到的调试器代码片段（父进程fork一个子进程然后对它进行跟踪）。在这篇文章的最后，你同样会找到一个包含这篇文章中所有代码的链接。

```c
/* Obtain and show child's instruction pointer */
ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
procmsg("Child started. EIP = 0x%08x\n", regs.eip);

/* Look at the word at the address we're interested in */
unsigned addr = 0x8048096;
unsigned data = ptrace(PTRACE_PEEKTEXT, child_pid, (void*)addr, 0);
procmsg("Original data at 0x%08x: 0x%08x\n", addr, data);
```

这是调试器从被调试进程中获取它现在的 *$eip* 的值，以及0x8048096处指令的值。当运行调试器跟踪本篇文章之前展示的汇编代码时，终端上会打印：

```
[13028] Child started. EIP = 0x08048080
[13028] Original data at 0x08048096: 0x000007ba
```

目前为止，进展一切顺利。接下来让我们继续：

```c
/* Write the trap instruction 'int 3' into the address */
unsigned data_with_trap = (data & 0xFFFFFF00) | 0xCC;
ptrace(PTRACE_POKETEXT, child_pid, (void*)addr, (void*)data_with_trap);

/* See what's there again... */
unsigned readback_data = ptrace(PTRACE_PEEKTEXT, child_pid, (void*)addr, 0);
procmsg("After trap, data at 0x%08x: 0x%08x\n", addr, readback_data);
```

请注意 *int 3* 是如何被插入到目标地址的。这个过程将在终端上打印：

```
[13028] After trap, data at 0x08048096: 0x000007cc
```

同样，正如预期的那样—— *0xba* 被替换成了 *0xcc*。调试器现在将运行被调试的进程，并且等待它在断点处被挂起：

```c
/* Let the child run to the breakpoint and wait for it to
** reach it
*/
ptrace(PTRACE_CONT, child_pid, 0, 0);

wait(&wait_status);
if (WIFSTOPPED(wait_status)) {
    procmsg("Child got a signal: %s\n", strsignal(WSTOPSIG(wait_status)));
}
else {
    perror("wait");
    return;
}

/* See where the child is now */
ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
procmsg("Child stopped at EIP = 0x%08x\n", regs.eip);
```

这个过程会打印：

```
Hello,
[13028] Child got a signal: Trace/breakpoint trap
[13028] Child stopped at EIP = 0x08048097
```

注意到“Hello，”是在断点前被打印出来——这正如我们所预期的那样。同样请注意子进程是在哪个地方被挂起的——刚好在一个字节的 *int 3* 指令之后。

最终，正如我们前面所解释的那样。为了让被调试进程重新继续执行，我们用原来的指令替换 *int 3* 指令并且让其 *$eip* 的值减1，然后重新运行被调试进程。

```c
/* Remove the breakpoint by restoring the previous data
** at the target address, and unwind the EIP back by 1 to
** let the CPU execute the original instruction that was
** there.
*/
ptrace(PTRACE_POKETEXT, child_pid, (void*)addr, (void*)data);
regs.eip -= 1;
ptrace(PTRACE_SETREGS, child_pid, 0, &regs);

/* The child can continue running now */
ptrace(PTRACE_CONT, child_pid, 0, 0);
```

这会让子进程打印“world！”并且退出，这正如我们所料。

注意到在这里我们并没有恢复断点。那可以通过使用单步执行模式运行被调试的进程，然后将 *int 3* 指令放回原处，然后再通过 **PTRACE_CONT** 请求使被调试进程继续执行。文章后面部分将介绍的调试库实现了这一点。

### 更多关于 *int 3* 的秘密

现在是时候回过头来重新审视一下 *int 3* 以及Intel手册上那句有趣的解释了。再次引用一下它：
> This one byte form is valuable because it can be used to replace the first byte of any instruction with a breakpoint, including other one byte instructions, without over-writing other code

在x86机器上 *int* 指令一般是占两个字节的—— *0xcd* 以及其后的8位的中断号。但是 *int 3* 是不能被加码为 *cd 03* 的，取而代之的是一个特殊的专门为它保留的指令—— *0xcc*。

为什么会这样呢？因为它允许我们在任何指令的第一个字节设置断点，这不会影响到它后面的指令。这是非常重要的，考虑下面这一个例子：

```nasm
    .. some code ..
    jz    foo
    dec   eax
foo:
    call  bar
    .. some code ..
```

假设我们想在 *dec eax* 处设置断点。这条指令正好对应着一个一字节指令（*0x48*）。假如我们使用一个超过一字节的断点指令去覆盖它，我们同样会覆盖掉它后面的一条指令（*call*），这可能会产生一些完全不合法的指令。如果执行了 *jz foo* 指令后，CPU并不会在断点处停下，并且会直接去执行后面的无效的指令。

使用一个特殊的一字节指令作为 *int 3* 指令解决了这个问题。因为在x86机器上一字节指令是最短的指令，我们可以保证只有我们想要停住的位置的那条指令会改变。

------------------------------------------------------------------------------

断点的设置原理上文已经介绍清楚。这篇文章剩下的几节不再准备翻译，它们是关于原文作者使用它写的调试库调试一个C程序的过程，有兴趣的读者可以移步[原文](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints)继续阅读，或者在网上找寻其它的对这篇文章的翻译，相信你能找到许多；）
