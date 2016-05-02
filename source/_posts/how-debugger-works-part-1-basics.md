---
title: 调试器工作原理：Part 1 － 基础
---

这篇文章基本上是国外一篇博客的翻译，部分地方做了一定删改。

原文链接：[http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/)

以下进入正文：

------------------------------------------------------------------------------

这是关于[“调试器原理系列文章”](http://eli.thegreenplace.net/tag/debuggers)的第一篇，我目前还不确定这个系列最终会包含多少篇文章，但我准备从最基础的东西讲起。

<!-- more -->

### 在这一篇中

我准备向大家介绍在Linux上实现调试器所用到的最重要的工具——*ptrace*系统调用。这篇文章中包含的所有代码都是在32位的Ubuntu机器上编写的。需要注意的是，这些代码是非常平台相关的，不过如果要将它们移植到其它平台上应该也不会是一件太难的事。

### 动机

为了认识到我们正在做什么，请尝试着想一想调试器需要拥有哪些功能才能完成它的工作。一个调试器可以运行某个程序并且对它进行调试，或者连接到一个正在运行的进程并对它进行调试。它能单步执行，设置断点并且在断点处“停住”程序，查看变量的值以及栈的信息。很多高级的调试器甚至能动态的修改被调试程序的代码并观察运行效果。

即使现代的调试器是非常复杂的“野兽”，令人惊讶的是它赖以实现的根基竟然非常简单。调试器能完成许多复杂的工作，关键是靠少量的由操作系统和编译器／链接器提供的基本服务，剩下的就只是一些[Small matter of programming](https://en.wikipedia.org/wiki/Small_matter_of_programming)。（注意这里是反讽）

### Linux调试－*ptrace*

对于Linux调试器，它的瑞士军刀就是*ptrace*系统调用（使用 *man 2 ptrace* 看看它的具体信息吧）。它是一个多功能且复杂的工具，它可以用来让一个进程控制另一个进程的执行，并且能查看甚至修改被监控进程的内存。单单是*ptrace*就能用一本中等厚度的书来叙述了，这也是为什么我将仅仅关注几个它对于调试器来说特别实用的功能，并且配上具体的例子加以解释。

让我们这就开始吧！

### 单步调试一个进程

我准备了一个具体的例子，在这个例子中我们将使一个程序运行在“被跟踪”模式，然后我们将对这个程序的代码进行单步调试，注意这里的代码指的是程序在CPU上运行的一条一条指令对应的机器码（汇编代码）。我会分部分展示这个例子，分别进行解释，然后在文章的最后你会找到一个链接，从那里你可以下载所有相关的C语言代码来编译、运行和把玩。

高层的目标是编写这样一个程序，运行后它的子进程会执行一个新的程序并把自己设为“被跟踪”模式，而父进程则会去跟踪这个子进程。首先，main函数如下：

```c
int main(int argc, char** argv)
{
    pid_t child_pid;

    if (argc < 2) {
        fprintf(stderr, "Expected a program name as argument\n");
        return -1;
    }

    child_pid = fork();
    if (child_pid == 0)
        run_target(argv[1]);
    else if (child_pid > 0)
        run_debugger(child_pid);
    else {
        perror("fork");
        return -1;
    }

    return 0;
}
```

非常简单，不是吗？我们使用*fork*系统调用创建了一个新的子进程。满足*if*语句分支的情况会创建一个子进程（在这里称作“target”），*else if*语句分支则运行父进程（在这里称作“debugger”）。

下面是target进程：

```c
void run_target(const char* programname)
{
    procmsg("target started. will run '%s'\n", programname);

    /* Allow tracing of this process */
    if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0) {
        perror("ptrace");
        return;
    }

    /* Replace this process's image with the given program */
    execl(programname, programname, 0);
}
```

在这里最有趣的一行便是*ptrace*系统调用。*ptrace*的声明如下：（在*sys/ptrace.h*中）：
```c
long ptrace(enum __ptrace_request request, pid_t pid,
		                 void *addr, void *data);
```

第一个参数是*request*，它可能是许多预先定义好的*PTRACE_*\*类型的常量。第二个参数通常指出了指出了被调试进程的进程号。第三个参数和第四个参数是一个地址或数据的指针，它们被用来进行内存相关的操作。在上面这个代码片段中*ptrace*发出了**PTRACE_TRACEME**的*request*，这意味这个进程请求操作系统让它的父进程对它进行跟踪。这个*request*在man手册中描述的非常清楚：

> Indicates that this process is to be traced by its parent. Any signal (except SIGKILL) delivered to this process will cause it to stop and its parent to be notified via wait(). **Also, all subsequent calls to exec() by this process will cause a SIGTRAP to be sent to it, giving the parent a chance to gain control before the new program begins execution.** A process probably shouldn't make this request if its parent isn't expecting to trace it. (pid, addr, and data are ignored.)

我已经突出显示了在这个例子中，man手册中能让我们重点引起兴趣的一句话。注意到子进程在进行*ptrace*系统调用后，马上会使用命令行传来的一个参数，来执行*execl*系统调用，这将会使子进程运行这个参数对应的一个新的程序。于是，正如上文引用中所突出显示的，这会让操作系统在子进程通过*execl*执行新的程序前先挂起子进程，然后给父进程发送一个信号。

终于，是时候看看父进程干了些什么了：

```c
void run_debugger(pid_t child_pid)
{
    int wait_status;
    unsigned icounter = 0;
    procmsg("debugger started\n");

    /* Wait for child to stop on its first instruction */
    wait(&wait_status);

    while (WIFSTOPPED(wait_status)) {
        icounter++;
        /* Make the child execute another instruction */
        if (ptrace(PTRACE_SINGLESTEP, child_pid, 0, 0) < 0) {
            perror("ptrace");
            return;
        }

        /* Wait for child to stop on its next instruction */
        wait(&wait_status);
    }

    procmsg("the child executed %u instructions\n", icounter);
}
```

回忆上文提到的，一旦子进程开始执行*exec*类型的系统调用，它会被操作系统停住并且被发送一个**SIGTRAP**信号。父进程使用第一个*wait*系统调用等待这件事情的发生。一旦某些有趣的事情发生了，*wait*会马上返回，然后父进程会检查是否是子进程被挂起了（如果子进程因为被发送了一个信号而挂起了，**WIFSTOPPED**会返回true）。

父进程接下来做的将是这篇文章最精彩的部分。它会使用**PTRACE_SINGLESTEP**以及子进程号作为参数调用*ptrace*。这么做的效果是告诉操作系统——请重新运行子进程，但是当子进程执行一条指令后就再次将它挂起。再次地，父进程会通过*wait*等待子进程被挂起，循环也会继续。当父进程通过*wait*得知的信号并不是子进程被挂起，循环会终止。在子进程正常运行的情况下，这将会是一个信号，它告诉父进程子进程已经执行结束正常退出了（这种情况下**WIFEXISTED**会返回true）。

注意到*icounter*会记录子进程执行的指令条数。所以我们的简单示例程序其实做了一件很有意义的事情——通过命令行传入一个程序的名字，它会执行这个程序，并且统计这个程序从开始到结束一共执行的机器指令的条数。让我们通过实战来看看吧。

### 一个测试的运行

我编译了如下的简单程序，然后让它被调试器运行。

```c
#include <stdio.h>

int main()
{
    printf("Hello, world!\n");
    return 0;
}
```

令我惊讶的是，调试器花了很长的时间完成执行并且打印显示上面的这个简单程序执行了超过100,000条指令。仅仅是一个*printf*调用？为什么会这样？答案是很有趣的。默认情况下，Linux上gcc会将程序动态地链接C运行时库。这意味着当任何C程序开始运行时，首先动态库的加载器会去寻找需要的共享库。这将会是非常多的代码－－然后请记住我们的简易调试器会统计子进程执行的每一条指令，而不仅仅是*main*函数中所执行的指令。

于是，当我用*-static*选项链接测试程序（相应的可执行文件的大小增加了500KB，这对于一个C运行时库的静态链接来说是合理的），打印结果显示一共仅执行了大约7,000条指令。这还是有点多，但是也确实合乎情理，如果你意识到在*main*函数执行前后libc要做一些初始化和回收工作。并且，*printf*也是一个复杂的函数。

仍然不满意，我想要的是可以测试并验证的东西——比如，我事先知道在一个程序运行一次需要执行多少条指令。怎么做到呢？显然，使用汇编代码就能解决这个问题。于是我使用了下面这个汇编版本的“Hello world”程序，并且对它进行了汇编和链接：

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
    mov    edx, len
    mov    ecx, msg
    mov    ebx, 1
    mov    eax, 4

    ; Execute the sys_write system call
    int    0x80

    ; Execute sys_exit
    mov    eax, 1
    int    0x80

section   .data
msg db    'Hello, world!', 0xa
len equ    $ - msg
```

足够了。现在简易调试器打印显示正好7条指令被执行了，这恰好与上述的汇编代码是吻合的。

### 深入到执行流中

这个汇编程序允许我为你介绍*ptrace*另一个非常强大的功能——密切的监视被调试进程的状态，无论是寄存器还是内存。下面是run_debugger的另一个版本：

```c
void run_debugger(pid_t child_pid)
{
    int wait_status;
    unsigned icounter = 0;
    procmsg("debugger started\n");

    /* Wait for child to stop on its first instruction */
    wait(&wait_status);

    while (WIFSTOPPED(wait_status)) {
        icounter++;
        struct user_regs_struct regs;
        ptrace(PTRACE_GETREGS, child_pid, 0, &regs);
        unsigned instr = ptrace(PTRACE_PEEKTEXT, child_pid, regs.eip, 0);

        procmsg("icounter = %u.  EIP = 0x%08x.  instr = 0x%08x\n",
                    icounter, regs.eip, instr);

        /* Make the child execute another instruction */
        if (ptrace(PTRACE_SINGLESTEP, child_pid, 0, 0) < 0) {
            perror("ptrace");
            return;
        }

        /* Wait for child to stop on its next instruction */
        wait(&wait_status);
    }

    procmsg("the child executed %u instructions\n", icounter);
}
```

与之前的那个run_debugger版本相比，仅有的一些不同在于while循环中的前面几行代码。现在两个新的*ptrace*调用。第一个将被跟踪进程的寄存器内容读入到一个结构体中。*user_regs_struct*在*sys/user.h*被定义。下面是一个有趣的部分——如果你去查看了这个头文件，你会在靠近顶部的位置看到下面这段注释：

```c
/* The whole purpose of this file is for GDB and GDB only.
   Don't read too much into it. Don't use it for
   anything other than GDB unless know what you are
   doing.  */
```

现在，我并不知道你是怎么想的，但是我感到我们正行驶在正确的轨道上;-) 好吧，不论怎样，让我们先回到例子上来。一旦我们将被调试进程的所有寄存器的值放入到*regs*中，我们便能通过使用**PTRACE_PEEKTEXT**参数调用*ptrace*，同时传入*regs.eip*（x86上的扩展的指令指示器）作为地址值参数。我们通过*ptrace*得到的将是指令的机器码。现在让我们来看看这个新版本的调试器运行我们的汇编代码的效果：
```c
$ simple_tracer traced_helloworld
[5700] debugger started
[5701] target started. will run 'traced_helloworld'
[5700] icounter = 1.  EIP = 0x08048080.  instr = 0x00000eba
[5700] icounter = 2.  EIP = 0x08048085.  instr = 0x0490a0b9
[5700] icounter = 3.  EIP = 0x0804808a.  instr = 0x000001bb
[5700] icounter = 4.  EIP = 0x0804808f.  instr = 0x000004b8
[5700] icounter = 5.  EIP = 0x08048094.  instr = 0x01b880cd
Hello, world!
[5700] icounter = 6.  EIP = 0x08048096.  instr = 0x000001b8
[5700] icounter = 7.  EIP = 0x0804809b.  instr = 0x000080cd
[5700] the child executed 7 instructions
```

好的，现在除了*icounter*之外，我们还能看到指令指示器的值以及每步执行中它对应的机器码。怎么证实它是正确的呢？通过对可执行文件执行*objdump -d*即可：

```
$ objdump -d traced_helloworld

traced_helloworld:     file format elf32-i386


Disassembly of section .text:

08048080 <.text>:
 8048080:     ba 0e 00 00 00          mov    $0xe,%edx
 8048085:     b9 a0 90 04 08          mov    $0x80490a0,%ecx
 804808a:     bb 01 00 00 00          mov    $0x1,%ebx
 804808f:     b8 04 00 00 00          mov    $0x4,%eax
 8048094:     cd 80                   int    $0x80
 8048096:     b8 01 00 00 00          mov    $0x1,%eax
 804809b:     cd 80                   int    $0x80
 ```

 这里的反汇编和我们简易调试器的打印结果之间的相关性是很容易发现的。

### 连接到一个运行中的进程

 正如你所知道的，调试器可以连接到一个正在运行中的进程。现在你应该不会对这也是通过*ptrace*做到的感到惊奇，确实这也是通过给*ptrace*发送**PTRACE_ATTACH**请求做到的。在这里我不会再给出代码，因为通过适当地修改我们已经实现的代码它是很容易做到的。由于教学目的，这里采取的方式更为直接、便于理解（由于我们能在子进程开始运行新的程序前就将它停住）。

### 代码
 这篇文章中的简易调试器的完整代码（更高级的那个可以打印机器码的版本）在[这里](https://github.com/eliben/code-for-blog/blob/master/2011/simple_tracer.c)。在gcc 4.4上使用 *-Wall -pedantic --std=c99* 便能顺利完成编译。

### 结论和下一步
 必须承认的是，这篇文章并没有包含太多的东西——我们离一个真正的调试器还有很远的距离。然而，我希望它至少让调试一个程序的过程变得不再那么神秘。*ptrace*确实是一个有非常多的功能的系统调用，我们只是对其中的几个功能进行了举例阐述。

 单步调试是有用的，但是只是到一定程度上。用上面的C代码“Hello world”举个例子，如果要通过单步调试的话，那到达*main*函数恐怕得需要经过几千次指令的调试过程。这是非常不方便的（痛苦的）。我们理想中想要的是能在*main*函数开始的地方设置一个断点，然后直接运行到那里然后停住。好吧，在这个系列的下一篇文章中，我会来讲讲断点是如何实现的。

### 参考文章
我发现下面的几篇文章对阅读我的这篇文章可以起到一定的准备作用：

- [Playing with ptrace, Part I](http://www.linuxjournal.com/article/6100?page=0,1)
- [How debugger works](http://www.alexonlinux.com/how-debugger-works)

