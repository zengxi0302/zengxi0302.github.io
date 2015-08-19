---
layout: post
title:  "crash手工解析栈方法"
date:   2015-08-18 22:47:00
categories: linux
---

#引言
---

在处理主机重启的问题时，我们关心的入参或运行时变量值大多可以通过寄存器记录的数据获取。
但是，有时候有些我们关心的数据不一定就直接存在在寄存器，又或栈顶被破坏了，bt无法解析堆栈。
此时，我们就需要尝试通过手动解析栈的方式获得我们感兴趣的数据。

#一个例子
---

下面，以echo c > /proc/sysrq-trigger触发crash的例子说明如何手工解栈。

bt解析的调用栈如下。
{% highlight text %}
crash> bt
PID: 10109  TASK: ffff8801b80ee440  CPU: 0   COMMAND: "bash"
 #0 [ffff8801b283db30] crash_kexec at ffffffff8008d88a
 #1 [ffff8801b283dc00] oops_end at ffffffff8040edc8
 #2 [ffff8801b283dc20] __bad_area_nosemaphore at ffffffff8001e67d
 #3 [ffff8801b283dce0] do_page_fault at ffffffff80411b94
 #4 [ffff8801b283dde0] page_fault at ffffffff8040e0d8
    [exception RIP: sysrq_handle_crash+13]
    RIP: ffffffff802c43fd  RSP: ffff8801b283de90  RFLAGS: 00010096
    RAX: 0000000000000010  RBX: ffffffff807188c0  RCX: 0000000000000010
    RDX: 0000000000000000  RSI: 00000000cc95cc95  RDI: 0000000000000063
    RBP: 0000000000000063   R8: ffffffff808e779d   R9: 0000000000000056
    R10: 0000000000000056  R11: 0000000000000046  R12: 0000000000000000
    R13: 0000000000000003  R14: 0000000000000000  R15: 0000000000000002
    ORIG_RAX: ffffffffffffffff  CS: 10000e030  SS: e02b
 #5 [ffff8801b283de90] __handle_sysrq at ffffffff802c4b5d
 #6 [ffff8801b283dec0] write_sysrq_trigger at ffffffff802c4be8
 #7 [ffff8801b283ded0] proc_reg_write at ffffffff8018e882
 #8 [ffff8801b283df10] vfs_write at ffffffff8012d4ce
 #9 [ffff8801b283df40] sys_write at ffffffff8012d643
#10 [ffff8801b283df80] system_call_fastpath at ffffffff80415cc3
    RIP: 00007fe37bea9be0  RSP: 00007ffd65fc3958  RFLAGS: 00000246
    RAX: 0000000000000001  RBX: ffffffff80415cc3  RCX: 0000000000000001
    RDX: 0000000000000002  RSI: 00007fe37c646000  RDI: 0000000000000001
    RBP: 0000000000000002   R8: 00007fe37c7ad700   R9: 0000000000000020
    R10: 00007fe37c150e30  R11: 0000000000000246  R12: 00007fe37c14f7c0
    R13: 00007fe37c646000  R14: 0000000000000002  R15: 00000000006b4430
    ORIG_RAX: 0000000000000001  CS: e033  SS: e02b
{% endhighlight %}

#问题
---

假设，我们并不知道这是通过echo c > /proc/sysrq-trigger造成的crash。单纯的从堆栈来看，只能看出来是在调sys_write写一个文件时触发的主机重启。那么，

***问题是：***

写哪一个文件时触发的主机重启？
显然sysrq_handle_crash显示的寄存器内容并没有提供这样的信息。

***思路是：***

vfs_write肯定有一个打开要写的file对象，而file里的dentry字段会保存有文件名。虽然调用栈#8 vfs_write并提供寄存器信息，但是栈上可能会有我们需要的file对象地址，因此需要手工解析栈。

#Step1 打出栈内容
---

利用bt -f打印出每个桢的栈内容，截取其中部分。
{% highlight bash %}
#7 [ffff8801b283ded0] proc_reg_write at ffffffff8018e882
    ffff8801b283ded8: 00007fe37c646000 0000000000000002
    ffff8801b283dee8: ffff8801b7a5c5c0 ffff8801b283df50
    ffff8801b283def8: 00007fe37c646000 00007ffd65fc3a20
    ffff8801b283df08: 0000000000000000 "ffffffff8012d4ce <- vfs_write里调用proc_reg_write返回
后要执行的下一条指令。因为调用回来要知道从哪里继续执行，所以会压栈"
 8 [ffff8801b283df10] vfs_write at ffffffff8012d4ce
    ffff8801b283df18: ffff8801b518a080 ffff8801b7a5c5c0
    ffff8801b283df28: fffffffffffffff7 0000000000000002
    ffff8801b283df38: 00007fe37c646000 "ffffffff8012d643 <- 同上，这是sys_write调用vfs_write
返回后将要执行的下一条指令。"
 9 [ffff8801b283df40] sys_write at ffffffff8012d643
    ffff8801b283df48: 00000000006b4430 0000000000000000
    ffff8801b283df58: 00000000006b4430 0000000000000002
    ffff8801b283df68: 00007fe37c646000 00007fe37c14f7c0
    ffff8801b283df78: 0000000000000002 ffffffff80415cc3
#10 [ffff8801b283df80] system_call_fastpath at ffffffff80415cc3
{% endhighlight %}

vfs_write函数从进入，到执行到vfs_write再往下调用的最后一条指令*ffffffff8012d4ce*{:style="color:red"}，栈中内容分别是
{% highlight text%}
#8 [ffff8801b283df10] vfs_write at ffffffff8012d4ce
    ffff8801b283df18: ffff8801b518a080 ffff8801b7a5c5c0
    ffff8801b283df28: fffffffffffffff7 0000000000000002
    ffff8801b283df38: 00007fe37c646000
{% endhighlight %}

#Step2 反汇编解析
---

sys_write调用进入vfs_write时，栈顶指针*rsp = ffff8801b283df40*{:style="color:red"}。再结合vfs_write的反汇编，可以反推栈上的数据来源是什么。
{% highlight bash %}
0xffffffff8012d400 <vfs_write>: 	   sub    $0x28%rsp	       
"rsp=rsp-0x28=ffff8801b283df40-0x28=0xffff8801b283df18，开辟5个8字节的栈空间"
0xffffffff8012d404 <vfs_write+4>:       mov    %rbx0x8(%rsp)	
"rsp+0x8位置写入rbx的内容 ffff8801b7a5c5c0"
0xffffffff8012d409 <vfs_write+9>:       mov    %rbp0x10(%rsp)	
"rsp+0x10位置写入rbp的内容 fffffffffffffff7"
0xffffffff8012d40e <vfs_write+14>:      mov    $0xfffffffffffffff7%rbx
0xffffffff8012d415 <vfs_write+21>:      mov    %r120x18(%rsp)	
"rsp+0x18位置写入r12的内容 0000000000000002"
0xffffffff8012d41a <vfs_write+26>:      mov    %r130x20(%rsp)	
" rsp+0x20位置写入r13内容00007fe37c646000 buf -- 这是一个虚拟地址"
0xffffffff8012d41f <vfs_write+31>:      mov    %rdi%rbp
0xffffffff8012d422 <vfs_write+34>:      testb  $0x20x44(%rdi)
"既然知道了栈上数据来源分别来自rbx rbp r12 r13的值，但还不知道它们的含义。
 因此，需要往上追溯sys_write里对这几个寄存器的操作。去了解这几个寄存器内容的含义。
 继续反汇编sys_write分析。"
0xffffffff8012d5f0 <sys_write>: 	   sub    $0x38%rsp
0xffffffff8012d5f4 <sys_write+4>:       mov    %r130x30(%rsp)
0xffffffff8012d5f9 <sys_write+9>:       mov    %rsi%r13
0xffffffff8012d5fc <sys_write+12>:      lea    0x14(%rsp)%rsi
0xffffffff8012d601 <sys_write+17>:      mov    %rbx0x18(%rsp)
0xffffffff8012d606 <sys_write+22>:      mov    %rbp0x20(%rsp)

"rbp的内容由操作数0xfffffffffffffff7而来，跟对应栈上的内容一致"
0xffffffff8012d60b <sys_write+27>:      mov    $0xfffffffffffffff7%rbp	//-EBADF
0xffffffff8012d612 <sys_write+34>:      mov    %r120x28(%rsp)
0xffffffff8012d617 <sys_write+39>:      mov    %rdx%r12
0xffffffff8012d61a <sys_write+42>:      callq  0xffffffff8012e290 <fget_light>
0xffffffff8012d61f <sys_write+47>:      test   %rax%rax
0xffffffff8012d622 <sys_write+50>:      mov    %rax%rbx
0xffffffff8012d625 <sys_write+53>:      je     0xffffffff8012d657 <sys_write+103>
0xffffffff8012d627 <sys_write+55>:      mov    0x48(%rax)%rax
0xffffffff8012d62b <sys_write+59>:      lea    0x8(%rsp)%rcx

"在调用vfs_write前，以下几条执令分别将rbx r12 r13赋值给rdi，rdx，rsi。
 而这几个寄存器的含义正是被调用函数的入参。
 根据vfs_write的定义vfs_write(struct file *file const char __user *buf size_t count loff_t *pos)"
0xffffffff8012d630 <sys_write+64>:      mov    %rbx%rdi	"rdi是入参1 - file"
0xffffffff8012d633 <sys_write+67>:      mov    %r12%rdx	"rdx是入参3 - count"
0xffffffff8012d636 <sys_write+70>:      mov    %r13%rsi	"rsi是入参2 - buf"
0xffffffff8012d639 <sys_write+73>:      mov    %rax0x8(%rsp)
0xffffffff8012d63e <sys_write+78>:      callq  0xffffffff8012d400 <vfs_write>
0xffffffff8012d643 <sys_write+83>:      mov    %rax%rbp	"sys_write调用完后返回的第一条指令0xffffffff8012d643"
0xffffffff8012d646 <sys_write+86>:      mov    0x8(%rsp)%rax
{% endhighlight %}

根据上面分析，可以清楚知道了栈上这几个数据的含义。
{% highlight bash %}
 #8 [ffff8801b283df10] vfs_write at ffffffff8012d4ce
    ffff8801b283df18: ffff8801b518a080 ffff8801b7a5c5c0 "<- file"
    ffff8801b283df28: fffffffffffffff7 0000000000000002 "<- count"
    ffff8801b283df38: 00007fe37c646000 "<- buf"
{% endhighlight %}

#解题
---

回到问题，写哪个文件触发的crash。根据file对象，查到dentry里的name字段，推出是写sysrq-trigger文件触发的crash。

{% highlight bash %}
crash> file ffff8801b7a5c5c0 | grep dentry
    dentry = 0xffff88008dbb2e90
crash> dentry 0xffff88008dbb2e90 | grep -E "inode|name"
  d_name = {
    name = 0xffff88008dbb2ec8 "sysrq-trigger"	//文件名称
  d_inode = 0xffff8801b87b7898					//文件inode对象
{% endhighlight %}

#题外话
---

实际上找哪个文件触发的crash并不需要这么麻烦，根据bash进程的task_struct可以查到进程打开的所有文件的。直接使用files命令就可以方便的查到有sysrq-trigger。

{% highlight bash %}
crash> files
PID: 10109  TASK: ffff8801b80ee440  CPU: 0   COMMAND: "bash"
ROOT: /    CWD: /tmp/disk/tmp/
 FD       FILE            DENTRY           INODE       TYPE PATH
  0 ffff8801b5321c80 ffff8800887e9e90 ffff8801b3a5c088 CHR  /dev/pts/2
  1 ffff8801b7a5c5c0 ffff88008dbb2e90 ffff8801b87b7898 REG  "/proc/sysrq-trigger"
  2 ffff8801b5321c80 ffff8800887e9e90 ffff8801b3a5c088 CHR  /dev/pts/2
 10 ffff8801b5321c80 ffff8800887e9e90 ffff8801b3a5c088 CHR  /dev/pts/2
255 ffff8801b5321c80 ffff8800887e9e90 ffff8801b3a5c088 CHR  /dev/pts/2
{% endhighlight %}

但，如果我们感兴趣的是执行过程中的某个变量，而crash工具没有提供简单命令查询，那还是要手动去解析，所以掌握这种分析方法仍然是非常有必要的。

#附：通用寄存器功能表
---

{% highlight text %}
 system_call x86_64
/*
 * Register setup:
 * rax  system call number
 * rdi  arg0
 * rcx  return address for syscall/sysret C arg3
 * rsi  arg1
 * rdx  arg2
 * r10  arg3    (--> moved to rcx for C)
 * r8   arg4
 * r9   arg5
 * r11  eflags for syscall/sysret temporary for C
 * r12-r15rbprbx saved by C code not touched.
 *
 * Interrupts are off on entry.
 * Only called from user space.
 *
 * XXX  if we had a free scratch register we could save the RSP into the stack fr<x>ame
 *      and report it properly in ps. Unfortunately we haven't.
 *
 * When user can change the fr<x>ames always force IRET. That is because
 * it deals with uncanonical addresses better. SYSRET has trouble
 * with them due to bugs in both AMD and Intel CPUs.
 */

{% endhighlight %}

